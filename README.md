# Centralne Reverse Proxy (Nginx Proxy Manager) + SSL na VPS

Repozytorium zawiera konfigurację oraz dokumentację wdrożeniową centralnej bramy wejściowej (**Reverse Proxy**) opartej na **Nginx Proxy Manager (NPM)** z automatyczną obsługą certyfikatów SSL (**Let's Encrypt**) na serwerze VPS.

Projekt realizuje architekturę **hybrydową**:
* **Usługi WWW** (`Elektrotechnika`, `Worldclock`) działają dwutorowo – dostępne bezpiecznie po domenie z certyfikatem SSL (`https://*.sslip.io`) oraz bezpośrednio po adresie IP i porcie (`http://IP:PORT`).
* **Usługi wrażliwe / zaawansowane** (`n8n Automation`) są w pełni odizolowane w wewnętrznej sieci Dockera i dostępne wyłącznie za pośrednictwem proxy po HTTPS (z obsługą WebSockets).
* **Panel administracyjny NPM** (port 81) jest zabezpieczony na pętli lokalnej (`127.0.0.1`) i dostępny wyłącznie przez tunel SSH.

Szczegółowy przewodnik wdrożeniowy krok po kroku znajduje się w pliku [plan.md](plan.md).

---

## 🏛️ Architektura systemu

```text
                                 INTERNET
                                    │
       ┌────────────────────────────┼───────────────────────────┐
       │ Port 80 (HTTP)             │ Port 443 (HTTPS)          │ Porty bezpośrednie HTTP
       ▼                            ▼                           ▼
┌────────────────────────────────────────────────────────┐ ┌─────────────────────────┐
│             /opt/proxy (Nginx Proxy Manager)           │ │ Dostęp po IP:           │
│  - Odbiera zapytania SNI dla domen *.sslip.io          │ │ http://92.55.194.2:8080 │
│  - Terminacja SSL / certyfikaty Let's Encrypt          │ │ http://92.55.194.2:8081 │
│  - Panel UI na 127.0.0.1:81 (dostęp przez tunel SSH)   │ └────────────┬────────────┘
└───────────────────────────┬────────────────────────────┘              │
                            │                                           │
                Wspólna sieć Dockera (proxy-net)                        │
     ┌──────────────────────┼───────────────────────────┐               │
     │                      │                           │               │
     ▼                      ▼                           ▼               │
┌─────────────────────────┐ ┌───────────────────┐ ┌───────────────────┐ │
│  /opt/elektrotechnika   │ │  /opt/worldclock  │ │     /opt/n8n      │ │
│    (Elektrotechnika)    │ │   (Worldclock)    │ │  (Automatyzacja)  │ │
│  Kontener:              │ │ Kontener:         │ │ Kontener:         │ │
│  elektrotechnika-portal │ │ worldclock        │ │ n8n_app           │ │
│  Port w sieci: 80       │ │ Port w sieci: 80  │ │ Port w sieci: 5678│ │
│  Port hosta: 8080 ◄─────┼─┼───────────────────┼─┼───────────────────┼─┘
│  (Tryb hybrydowy)       │ │ Port hosta: 8081 ◄┼─┼───────────────────┘
└─────────────────────────┘ │ (Tryb hybrydowy)  │ │ (BRAK portu hosta)│
                            └───────────────────┘ └───────────────────┘
```

---

## 🌐 Przypisanie usług, domen i portów

| Usługa / Projekt | Katalog | Kontener | Port hosta | Dostęp przez IP | Domena sslip.io (SSL) | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Bramka / Proxy** | `/opt/proxy` | `nginx_proxy_manager` | `80`, `443`, `127.0.0.1:81` | Tunel SSH (`localhost:8181`) | Let's Encrypt na 80/443 | Aktywny |
| **Elektrotechnika** | `/opt/elektrotechnika` | `elektrotechnika-portal` | `8080:80` | `http://92.55.194.2:8080` | `https://elektro-92-55-194-2.sslip.io` | Aktywny (Hybrydowy) |
| **Worldclock** | `/opt/worldclock` | `worldclock` | `8081:80` | `http://92.55.194.2:8081` | `https://worldclock-92-55-194-2.sslip.io` | Aktywny (Hybrydowy) |
| **n8n Automation** | `/opt/n8n` | `n8n_app` | *Brak* | *Zablokowany* | `https://n8n-92-55-194-2.sslip.io` | Zaplanowany (Tylko NPM) |

---

## 🚀 Szybki start (Wdrożenie)

### 1. Przygotowanie sieci Dockera i firewalla
```bash
# Utworzenie współdzielonej sieci dla kontenerów
docker network create proxy-net

# Otwarcie portów w UFW
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 8081/tcp
```

### 2. Uruchomienie Nginx Proxy Manager
W katalogu `/opt/proxy` utwórz `docker-compose.yml`:
```yaml
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx_proxy_manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '127.0.0.1:81:81'
    environment:
      - DISABLE_IPV6=true
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```
Uruchom stos:
```bash
docker compose up -d
```

### 3. Bezpieczne połączenie z panelem administracyjnym
Połącz się z serwerem przez tunel SSH ze swojej stacji roboczej:
```bash
ssh -L 8181:localhost:81 user@92.55.194.2
```
Otwórz w przeglądarce: [http://localhost:8181](http://localhost:8181)
* Domyślne dane logowania: `admin@example.com` / `changeme` (wymagana zmiana przy 1. logowaniu).

### 4. Konfiguracja Proxy Hostów
W panelu NPM dodaj Proxy Hosty kierujące ruch do kontenerów:
* **Elektrotechnika:** domena `elektro-92-55-194-2.sslip.io` $\rightarrow$ `elektrotechnika-portal:80` (Force SSL, HTTP/2).
* **Worldclock:** domena `worldclock-92-55-194-2.sslip.io` $\rightarrow$ `worldclock:80` (Force SSL, HTTP/2).
* **n8n:** domena `n8n-92-55-194-2.sslip.io` $\rightarrow$ `n8n_app:5678` (Force SSL, HTTP/2, WebSockets Support).

---

## 🔧 Przydatne polecenia diagnostyczne

```bash
# Podgląd kontenerów w sieci proxy
docker network inspect proxy-net

# Logi Nginx Proxy Manager
cd /opt/proxy && docker compose logs -f npm

# Sprawdzenie certyfikatu SSL z wiersza poleceń
curl -Iv https://elektro-92-55-194-2.sslip.io

# Test komunikacji wewnętrznej w sieci Dockera
docker exec -it nginx_proxy_manager curl -I http://elektrotechnika-portal:80

# Kopia zapasowa konfiguracji proxy i certyfikatów
sudo tar -czvf /opt/backup_proxy_$(date +%F).tar.gz /opt/proxy/data /opt/proxy/letsencrypt
```

---

## 📖 Dokumentacja
Kompletna specyfikacja architektury, rozważania dotyczące routingu i procedury awaryjne znajdują się w pliku [plan.md](plan.md).
