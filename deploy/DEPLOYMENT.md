# Mira — przewodnik wdrożenia na VPS (Ubuntu + Docker)

Kompletny przewodnik produkcyjnego wdrożenia Mira na własnym VPS:
instalacja Dockera, pobranie kodu, sekrety, TLS, firewall, start
kontenerów i konfiguracja GitHub App.

Wszystkie komendy wykonujesz **na VPS przez SSH jako `root`**
(weryfikacja: `whoami` → `root`, `id` → `uid=0(root) ... groups=0(root),987(docker)`).
Kod projektu znajduje się w katalogu **`/opt/mira`**.

> Wszystkie wartości w przykładach (`twoja-domena.example.com`, klucze,
> sekrety) to **placeholdery** — zastąp je własnymi. Nigdy nie commituj
> `.env`, certyfikatów ani kluczy prywatnych.

---

## 0. Instalacja git, Dockera i Docker Compose

```bash
ssh root@IP_VPS
apt update
apt install -y git ca-certificates curl

# oficjalne repo Dockera
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# weryfikacja (jako root nie potrzeba sudo; root jest już w grupie docker)
docker --version
docker compose version
```

> Wariant dla użytkownika nie-root: po instalacji wykonaj
> `usermod -aG docker $USER`, wyloguj się i zaloguj ponownie (lub użyj
> `sudo docker ...`). Uwaga: grupa `docker` daje efektywnie uprawnienia
> roota na hoście — nie dodawaj do niej użytkowników, którym nie ufasz.

## 1. Pobranie kodu projektu

```bash
mkdir -p /opt/mira
git clone -b develop-0.11.1 https://github.com/petrus92/mira.git /opt/mira
cd /opt/mira
git log --oneline -1
```

Aktualizacje: `cd /opt/mira && git pull`

## 2. Sekrety — plik `/opt/mira/.env`

```bash
cd /opt/mira
cp .env.example .env
nano .env
```

```bash
MIRA_DOMAIN=twoja-domena.example.com          # domena = rekord DNS + certyfikat TLS
MISTRAL_API_KEY=<klucz z console.mistral.ai> # La Plateforme (mira.yaml używa api_key_env: MISTRAL_API_KEY)
MIRA_GITHUB_APP_ID=123456                     # z ustawień GitHub App (sekcja 7)
MIRA_GITHUB_PRIVATE_KEY=@/app/secrets/mira-app.private-key.pem
MIRA_WEBHOOK_SECRET=<openssl rand -hex 32>    # taki sam jak "Webhook secret" w GitHub App
```

Klucz prywatny GitHub App (pobrany jako plik `.pem`) umieść w `deploy/secrets`:

```bash
# na komputerze lokalnym:
scp mira-app.2026-XX-XX.private-key.pem root@IP_VPS:/tmp/
# na VPS:
mkdir -p /opt/mira/deploy/secrets
mv /tmp/mira-app.2026-XX-XX.private-key.pem /opt/mira/deploy/secrets/
chmod 600 /opt/mira/deploy/secrets/*.pem
```

Forma `@/app/secrets/...` jest obsługiwana przez aplikację
(`src/mira/platforms/github/auth.py`); `docker-compose.yml` montuje
`deploy/secrets` do kontenera jako `/app/secrets:ro`.

## 3. Certyfikaty TLS (Let's Encrypt)

Przed pierwszym startem nginx (port 80 musi być wolny dla trybu `standalone`):

```bash
apt install -y certbot
certbot certonly --standalone -d twoja-domena.example.com
mkdir -p /opt/mira/deploy/nginx/certs
cp /etc/letsencrypt/live/twoja-domena.example.com/fullchain.pem /opt/mira/deploy/nginx/certs/
cp /etc/letsencrypt/live/twoja-domena.example.com/privkey.pem  /opt/mira/deploy/nginx/certs/
chmod 600 /opt/mira/deploy/nginx/certs/privkey.pem
```

Odnowienia przez webroot (gdy nginx już działa) — dodaj do crona na VPS:

```bash
certbot renew --webroot -w /opt/mira/deploy/nginx/acme
cd /opt/mira && docker compose restart nginx
```

## 4. Firewall (ufw)

Kolejność krytyczna — najpierw SSH, żeby nie odciąć sobie dostępu:

```bash
ufw allow OpenSSH
ufw allow 80,443/tcp
ufw enable
ufw status verbose
```

## 5. DNS

U dostawcy domeny ustaw rekord **A**: `twoja-domena.example.com → IP_VPS`.

## 6. Start aplikacji

```bash
cd /opt/mira
docker compose up -d --build
docker compose ps                                   # serwis "mira" powinien mieć status healthy
curl -s https://twoja-domena.example.com/health
```

> `docker compose` uruchamiaj zawsze z katalogu `/opt/mira` — ścieżki
> w compose (`./deploy/...`, `./mira.yaml`, `./.env`) są względne.

## 7. GitHub App

github.com → **Settings → Developer settings → GitHub Apps → New GitHub App**
(lub analogicznie w ustawieniach organizacji):

- **Webhook URL**: `https://twoja-domena.example.com/webhook`
- **Webhook secret**: wartość `MIRA_WEBHOOK_SECRET` z `/opt/mira/.env`
- **Uprawnienia i eventy**: zgodnie z
  [docs.miracode.ai/quickstart](https://docs.miracode.ai/quickstart)
  (typowo: Pull requests read/write, Issues read/write, Contents read,
  Metadata read; eventy `pull_request`, `issue_comment`, `pull_request_review*`)
- **Generate a private key** → pobrany plik `.pem` umieść jak w sekcji 2
- **App ID** (strona główna aplikacji) → `MIRA_GITHUB_APP_ID` w `/opt/mira/.env`
- Na końcu **Install** aplikację na docelowe repozytoria

## 8. Kolejność czynności — skrót

1. SSH → instalacja git + Docker + Compose → `ufw` (OpenSSH, potem 80/443) → DNS
2. `git clone -b develop-0.11.1 https://github.com/petrus92/mira.git /opt/mira`
3. `/opt/mira/.env`, `/opt/mira/deploy/secrets/*.pem`, `/opt/mira/deploy/nginx/certs/`
4. `cd /opt/mira && docker compose up -d --build`
5. GitHub App: webhook + sekret + instalacja na repozytoria

## Uwagi techniczne

- `MIRA_DOMAIN` jest renderowane do vhostu nginx przez envsubst
  (`deploy/nginx/mira.conf.template` → `/etc/nginx/conf.d/default.conf`).
  Zmień tylko wartość w `.env`, nie edytuj szablonu.
- Baza SQLite i indeksy per-repo żyją na wolumenie `mira-data`
  (`MIRA_INDEX_DIR=/app/data/indexes`). Do Postgresa przejdziesz przez
  `DATABASE_URL` w `.env` — bez zmiany plików.
- Logi kontenerów są rotowane (json-file, 10 MB × 3) — zdefiniowane
  w `docker-compose.yml`.
