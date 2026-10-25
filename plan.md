# PLAN PROJEKTU: Centralne Reverse Proxy (Nginx Proxy Manager) + SSL na VPS (Tryb Hybrydowy)

Niniejszy dokument stanowi kompletny opis architektury, stanu obecnego oraz zaktualizowanej procedury wdrożenia i dalszej rozbudowy środowiska serwerowego na VPS. Został przygotowany tak, aby dowolny agent AI lub inżynier mógł podjąć pracę od dowolnego punktu bez konieczności odtwarzania kontekstu.

---

## 1. Informacje podstawowe i środowisko

* **Serwer VPS:** Linux Mint / Debian-based
* **Główny adres IP:** `92.55.194.2`
* **Domyślny rDNS (PTR):** `92-55-194-2.net.hawetelekom.pl` (pozostaje bez zmian)
* **Środowisko uruchomieniowe:** Docker + Docker Compose (`docker compose`)
* **Standard lokalizacji projektów:** Standard FHS – niezależne katalogi w `/opt/`
* **Architektura sieciowa:** 
  * Dedykowana, wspólna sieć mostkowa Dockera (`proxy-net`).
  * **Tryb hybrydowy:** Strony WWW (`/opt/elektrotechnika` i `/opt/worldclock`) są dostępne dwutorowo – po bezpiecznej domenie z SSL przez NPM oraz bezpośrednio po adresie IP i porcie na hoście.
  * **Pełna izolacja n8n:** Aplikacja automatyzacji `/opt/n8n` nie publikuje portu na hoście – działa wyłącznie w sieci wewnętrznej i jest dostępna jedynie przez NPM z certyfikatem SSL.
* **Strategia DNS / Domeny:** Brak płatnych domen. Użycie publicznego Wildcard DNS `sslip.io`, który dynamicznie rozwiązuje adresy w formacie:
  * `[nazwa]-[IP-z-myslnikami].sslip.io` -> `92.55.194.2`
* **Zarządzanie certyfikatami SSL:** Let's Encrypt z automatycznym odnawianiem obsługiwany centralnie przez Nginx Proxy Manager (NPM).

---

## 2. Architektura docelowa

Architektura łączy wygodę bezpośredniego dostępu po IP z pełnym bezpieczeństwem HTTPS:
1. **Warstwa wejściowa (Centralne Proxy):** Kontener NPM w `/opt/proxy` przejmuje kontrolę nad publicznymi portami `80` (HTTP) i `443` (HTTPS).
2. **Bezpieczny panel administracyjny:** Port administracyjny NPM (`81`) jest zbindowany wyłącznie do pętli zwrotnej (`127.0.0.1:81`), uniemożliwiając skanowanie i ataki brute-force z internetu (dostęp realizowany jest przez tunel SSH).
3. **Tryb hybrydowy dla serwisów WWW:** Kontenery `elektrotechnika-portal` i `worldclock` są podłączone do sieci `proxy-net` (gdzie NPM przekazuje do nich ruch z domen SSL), a jednocześnie mają zmapowane porty na hoście (`8080` i `8081`), dzięki czemu działają również pod czystym IP.
4. **Izolacja dla n8n:** n8n działa wyłącznie w sieci `proxy-net` (bez portu na hoście), co eliminuje problemy z odrzucaniem ciasteczek sesyjnych (`Secure` cookie loop), błędami CORS, zerwanymi WebSocketami i wymogami webhooków (np. Telegram/GitHub wymagają HTTPS).

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

### Przypisanie domen, portów i dostępów

| Usługa / Projekt | Katalog | Nazwa kontenera | Port na hoście | Dostęp przez IP | Dostęp przez domenę (SSL) | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Bramka / Proxy** | `/opt/proxy` | `nginx_proxy_manager` | `80`, `443`, `127.0.0.1:81` | Tunel SSH (`localhost:8181`) | Let's Encrypt na 80/443 | Wdrażany |
| **Elektrotechnika** | `/opt/elektrotechnika` | `elektrotechnika-portal` | `8080:80` | `http://92.55.194.2:8080` | `https://elektro-92-55-194-2.sslip.io` | Aktywny (Hybrydowy) |
| **Worldclock** | `/opt/worldclock` | `worldclock` | `8081:80` | `http://92.55.194.2:8081` | `https://worldclock-92-55-194-2.sslip.io` | Aktywny (Hybrydowy) |
| **n8n Automation** | `/opt/n8n` | `n8n_app` | *Brak* | *Zablokowany (Brak portu)* | `https://n8n-92-55-194-2.sslip.io` | Zaplanowany (Tylko NPM) |

*Uwaga dotycząca `/opt/containerd`: jest to wewnętrzny katalog silnika Docker/containerd – nie należy go modyfikować ani przenosić.*

---

## 3. Plan wdrożenia krok po kroku

### Krok 1: Weryfikacja portów, firewall (UFW) i utworzenie sieci Dockera

1. Upewnij się, że porty `80` i `443` na hoście są wolne:
   ```bash
   sudo ss -tulpn | grep -E ':(80|443)\s'
   ```
2. Otwórz wymagane porty publiczne w firewallu UFW:
   ```bash
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw allow 8080/tcp
   sudo ufw allow 8081/tcp
   sudo ufw status
   ```
   *(Port 81 dla panelu NPM oraz port 5678 dla n8n pozostają zamknięte w UFW)*.
3. Utwórz wspólną, zewnętrzną sieć mostkową Dockera:
   ```bash
   docker network create proxy-net
   ```

---

### Krok 2: Uruchomienie Nginx Proxy Manager w `/opt/proxy`

1. Utwórz strukturę katalogów:
   ```bash
   sudo mkdir -p /opt/proxy
   cd /opt/proxy
   ```
2. Utwórz plik `/opt/proxy/docker-compose.yml`:
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
3. Uruchom kontener:
   ```bash
   docker compose up -d
   ```
4. Sprawdź poprawność startu w logach:
   ```bash
   docker compose logs -f --tail 30
   ```

---

### Krok 3: Konfiguracja usług WWW w trybie hybrydowym

Do plików `docker-compose.yml` dodajemy podpięcie do sieci `proxy-net`, zachowując mapowanie portów na hoście.

1. **W `/opt/elektrotechnika/docker-compose.yml` (Elektrotechnika):**
   ```yaml
   services:
     elektrotechnika:
       build:
         context: .
         dockerfile: Dockerfile
       container_name: elektrotechnika-portal
       restart: unless-stopped
       ports:
         - "8080:80"
       environment:
         - TZ=Europe/Warsaw
       networks:
         - proxy-net
       logging:
         driver: "json-file"
         options:
           max-size: "10m"
           max-file: "3"

   networks:
     proxy-net:
       external: true
   ```
   *Zastosuj zmiany:* `cd /opt/elektrotechnika && docker compose up -d`

2. **W `/opt/worldclock/docker-compose.yml` (Worldclock):**
   ```yaml
   services:
     worldclock:
       build:
         context: .
         dockerfile: Dockerfile
       image: worldclock:latest
       container_name: worldclock
       restart: unless-stopped
       ports:
         - "8081:80"
       volumes:
         - ./index.html:/usr/share/nginx/html/index.html:ro
         - ./style.css:/usr/share/nginx/html/style.css:ro
         - ./app.js:/usr/share/nginx/html/app.js:ro
         - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
       networks:
         - proxy-net

   networks:
     proxy-net:
       external: true
   ```
   *Zastosuj zmiany:* `cd /opt/worldclock && docker compose up -d`

---

### Krok 4: Konfiguracja Proxy Hostów i certyfikatów SSL

1. **Bezpieczny dostęp do panelu NPM:**
   Połącz się z poziomu swojej lokalnej maszyny przez tunel SSH:
   ```bash
   ssh -L 8181:localhost:81 user@92.55.194.2
   ```
   Otwórz w przeglądarce: `http://localhost:8181`

2. **Logowanie domyślne:**
   * **Email:** `admin@example.com`
   * **Hasło:** `changeme`
   * *Natychmiast zmień adres e-mail i hasło administratora po zalogowaniu.*

3. **Dodaj wpis dla Elektrotechnika:**
   * **Hosts** -> **Proxy Hosts** -> **Add Proxy Host**
   * **Domain Names:** `elektro-92-55-194-2.sslip.io`
   * **Scheme:** `http`
   * **Forward Hostname / IP:** `elektrotechnika-portal`
   * **Forward Port:** `80`
   * Zaznacz: `Block Common Exploits`, `Websockets Support`
   * Zakładka **SSL**:
     * SSL Certificate: `Request a new SSL Certificate`
     * Zaznacz: `Force SSL`, `HTTP/2 Support`
     * Wpisz e-mail kontaktowy i zaakceptuj warunki Let's Encrypt.
   * Kliknij **Save**.

4. **Dodaj wpis dla Worldclock:**
   * **Domain Names:** `worldclock-92-55-194-2.sslip.io`
   * **Scheme:** `http`
   * **Forward Hostname / IP:** `worldclock`
   * **Forward Port:** `80`
   * Zaznacz: `Block Common Exploits`, `Websockets Support`
   * Zakładka **SSL**: `Request a new SSL Certificate` + `Force SSL` + `HTTP/2 Support`.
   * Kliknij **Save**.

---

## 4. Moduł rozbudowy: Wdrożenie n8n (Plan na przyszłość)

n8n wdrażamy w trybie pełnej izolacji sieciowej – **bez wystawiania portu na hoście**.

### Wymagania systemowe dla n8n
* Minimum 2 GB RAM na VPS (jeśli serwer ma 1-2 GB RAM, upewnij się, że skonfigurowano minimum 2 GB pliku Swap).

### Wdrożenie usługi w `/opt/n8n`
1. Utwórz katalog oraz nadaj uprawnienia użytkownikowi kontenera n8n (`node` - UID 1000):
   ```bash
   sudo mkdir -p /opt/n8n/n8n_data
   sudo chown -R 1000:1000 /opt/n8n/n8n_data
   cd /opt/n8n
   ```
2. Utwórz plik `/opt/n8n/docker-compose.yml`:
   ```yaml
   services:
     n8n:
       image: docker.n8n.io/n8nio/n8n
       container_name: n8n_app
       restart: unless-stopped
       environment:
         - N8N_HOST=n8n-92-55-194-2.sslip.io
         - N8N_PORT=5678
         - N8N_PROTOCOL=https
         - WEBHOOK_URL=https://n8n-92-55-194-2.sslip.io
         - GENERIC_TIMEZONE=Europe/Warsaw
       volumes:
         - ./n8n_data:/home/node/.n8n
       networks:
         - proxy-net

   networks:
     proxy-net:
       external: true
   ```
3. Uruchom kontener:
   ```bash
   docker compose up -d
   ```
4. Podepnij pod NPM:
   * W panelu NPM (`http://localhost:8181` przez tunel SSH) dodaj nowy Proxy Host:
     * **Domain Names:** `n8n-92-55-194-2.sslip.io`
     * **Scheme:** `http`
     * **Forward Hostname / IP:** `n8n_app`
     * **Forward Port:** `5678`
     * Zaznacz: `Websockets Support` (bezwzględnie wymagane dla edytora n8n), `Block Common Exploits`
     * Zakładka **SSL**: `Request a new SSL Certificate` + `Force SSL` + `HTTP/2 Support`.

---

## 5. Przydatne polecenia diagnostyczne

* **Podgląd statusu kontenerów w sieci proxy:**
  ```bash
  docker network inspect proxy-net
  ```
* **Podgląd logów proxy w czasie rzeczywistym:**
  ```bash
  cd /opt/proxy && docker compose logs -f npm
  ```
* **Weryfikacja odpowiedzi bezpośredniej po IP (HTTP):**
  ```bash
  curl -I http://92.55.194.2:8080
  curl -I http://92.55.194.2:8081
  ```
* **Weryfikacja odpowiedzi przez domenę i certyfikatu SSL (HTTPS):**
  ```bash
  curl -Iv https://elektro-92-55-194-2.sslip.io
  curl -Iv https://worldclock-92-55-194-2.sslip.io
  ```
* **Test komunikacji wewnętrznej (z kontenera NPM do aplikacji):**
  ```bash
  docker exec -it nginx_proxy_manager curl -I http://elektrotechnika-portal:80
  docker exec -it nginx_proxy_manager curl -I http://worldclock:80
  docker exec -it nginx_proxy_manager curl -I http://n8n_app:5678
  ```
* **Kopia zapasowa konfiguracji proxy i certyfikatów:**
  ```bash
  sudo tar -czvf /opt/backup_proxy_$(date +%F).tar.gz /opt/proxy/data /opt/proxy/letsencrypt
  ```
