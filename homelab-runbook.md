# Homelab Runbook

A reproducible personal homelab stack on any Debian/Ubuntu Linux host running Docker Engine.
Services are managed via Docker Compose, proxied through Caddy with TLS from Step CA.

## Stack

| Service | URL | Purpose |
|---|---|---|
| Step CA | https://step-ca.test:9000 | Internal CA / ACME provider |
| Caddy | :8080 / :8443 | Reverse proxy / TLS termination |
| Dockhand | https://dockhand.test:8443 | Docker management UI |
| Flatnotes | https://flatnotes.test:8443 | Markdown notes |
| Vaultwarden | https://vaultwarden.test:8443 | Password manager |

## Host Requirements

- Debian 12 or Ubuntu 22.04+
- Docker Engine (not Docker Desktop)
- Git
- Ports 8080 and 8443 available
- User with uid 1000

## Host Preparation

### 1. Create data directories

```bash
sudo mkdir -p /data/step-ca
sudo mkdir -p /data/caddy
sudo mkdir -p /data/dockhand
sudo mkdir -p /data/flatnotes/data
sudo mkdir -p /data/flatnotes/index
sudo mkdir -p /data/vaultwarden
sudo chown -R 1000:1000 /data
```

### 2. Add /etc/hosts entries

Add to `/etc/hosts` on the **host** and on any **workstation** that needs access:

```
192.168.10.77  step-ca.test dockhand.test flatnotes.test vaultwarden.test
```

### 3. Clone the homelab repo

```bash
git clone https://github.com/yourusername/homelab.git ~/Projects/homelab
cd ~/Projects/homelab
```

---

## Deployment Order

Services must be started in this order:

1. Step CA (foundation — everything else depends on it for TLS)
2. Caddy (reverse proxy — depends on Step CA for ACME)
3. Dockhand (management UI)
4. Flatnotes
5. Vaultwarden

---

## Step 1 — Step CA

### Create .env

```bash
cd ~/Projects/homelab/step-ca
cp .env.example .env
```

Edit `.env` and set a strong password for `STEP_CA_PASSWORD`. This password
encrypts the CA private key. Store it somewhere safe — you'll need it if you
ever need to restart Step CA from scratch.

### Start Step CA

```bash
docker compose up -d
```

On first run Step CA initialises itself, generates the root and intermediate CA,
and writes everything to `/data/step-ca`.

### Get the root CA fingerprint

```bash
docker logs step-ca 2>&1 | grep "Root fingerprint"
```

Copy the fingerprint — you'll need it for the Caddy `.env` in the next step.

### Export the root CA certificate

```bash
docker exec step-ca step ca root > /data/caddy/step-ca-root.crt
```

Caddy reads this file to trust the Step CA ACME endpoint.

### Import root CA to your workstation

Copy the root cert to your workstation and trust it:

**Linux (Chrome/Firefox system trust):**
```bash
# Copy cert from server first
scp andrew@192.168.10.77:/data/caddy/step-ca-root.crt ~/step-ca-root.crt

sudo cp ~/step-ca-root.crt /usr/local/share/ca-certificates/homelab-root-ca.crt
sudo update-ca-certificates
```

**macOS:**
```bash
scp andrew@192.168.10.77:/data/caddy/step-ca-root.crt ~/step-ca-root.crt
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/step-ca-root.crt
```

**Windows:**
Download the cert then double-click → Install Certificate → Local Machine →
Place in "Trusted Root Certification Authorities".

---

## Step 2 — Caddy

### Create .env

```bash
cd ~/Projects/homelab/caddy
cp .env.example .env
```

Edit `.env` and paste the `STEP_CA_FINGERPRINT` value from the Step CA logs.

### Start Caddy

```bash
docker compose up -d
```

Caddy will contact Step CA via ACME and obtain certificates for all configured
hostnames automatically. Check logs to confirm:

```bash
docker logs caddy -f
```

You should see certificate obtained messages for each `.test` hostname. If Caddy
can't reach Step CA, check that the `extra_hosts` entries in the Caddy
`docker-compose.yml` match your host IP.

---

## Step 3 — Dockhand

```bash
cd ~/Projects/homelab/dockhand
docker compose up -d
```

Browse to https://dockhand.test:8443 — you should get a green padlock.
Complete initial setup to create your admin account.

From this point you can manage all remaining services through the Dockhand UI
by pointing it at each `docker-compose.yml` in the repo, or continue via CLI.

---

## Step 4 — Flatnotes

### Create .env

```bash
cd ~/Projects/homelab/flatnotes
cp .env.example .env
```

Edit `.env`:
- `FLATNOTES_USERNAME` — your login username
- `FLATNOTES_PASSWORD` — your login password
- `FLATNOTES_SECRET_KEY` — a random string, generate with:

```bash
openssl rand -base64 32
```

### Start Flatnotes

```bash
docker compose up -d
```

Browse to https://flatnotes.test:8443

### Migrate existing notes

If migrating notes from another host:

```bash
cp /path/to/old/notes/*.md /data/flatnotes/data/
```

Flatnotes picks up new files immediately — no restart needed.

---

## Step 5 — Vaultwarden

### Create .env

```bash
cd ~/Projects/homelab/vaultwarden
cp .env.example .env
```

Edit `.env` and set `ADMIN_TOKEN`. Generate a secure token:

```bash
openssl rand -base64 48
```

### Start Vaultwarden

```bash
docker compose up -d
```

Browse to https://vaultwarden.test:8443

Admin panel is at https://vaultwarden.test:8443/admin

> **Note:** `SIGNUPS_ALLOWED` is set to `false` by default. Create your account
> via the admin panel first, then you can leave signups disabled.

---

## Day-to-Day Operations

### Start all services

```bash
for svc in step-ca caddy dockhand flatnotes vaultwarden; do
  cd ~/Projects/homelab/$svc && docker compose up -d && cd -
done
```

### Stop all services

```bash
for svc in vaultwarden flatnotes dockhand caddy step-ca; do
  cd ~/Projects/homelab/$svc && docker compose down && cd -
done
```

### Update all images

```bash
for svc in step-ca caddy dockhand flatnotes vaultwarden; do
  cd ~/Projects/homelab/$svc && docker compose pull && docker compose up -d && cd -
done
```

Or use Dockhand's built-in auto-update and vulnerability scanning.

### View logs

```bash
docker logs <container-name> -f
```

---

## Rebuilding on a New Host

1. Install Docker Engine
2. Clone the homelab repo
3. Create `/data` directories (Step 1 above)
4. Copy `/data` from old host or restore from backup
5. Copy `.env` files from old host (these are not in git)
6. Add `/etc/hosts` entries
7. Start services in order
8. Import root CA cert to workstation

> The `/data` directory contains all persistent state. Back this up regularly.
> The git repo contains all config. The `.env` files contain all secrets —
> store these separately (e.g. in Vaultwarden itself once it's running).

---

## Troubleshooting

### Caddy can't reach Step CA
Check `extra_hosts` in `caddy/docker-compose.yml` matches the host IP.
Confirm Step CA is running: `docker ps | grep step-ca`

### Certificate not trusted in browser
Ensure you've imported the root CA cert and restarted the browser.
On Linux you may need to restart Chrome/Firefox after `update-ca-certificates`.

### Flatnotes search index stale
The index syncs on every search and on startup. If notes added via filesystem
aren't appearing, trigger a search in the UI to force a sync. If still stale,
restart the container: `docker restart flatnotes`

### Vaultwarden won't start
Vaultwarden requires the `DOMAIN` env var to be set to the full HTTPS URL
including port. Confirm `.env` has `DOMAIN=https://vaultwarden.test:8443`.
