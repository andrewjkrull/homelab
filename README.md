# Homelab Runbook

A reproducible personal homelab stack on any Debian/Ubuntu Linux host running
Docker Engine. Services are managed via Docker Compose, proxied through Caddy
with TLS issued by Caddy's built-in internal CA. No external CA or ACME
infrastructure required.

## Stack

| Service | URL | Purpose |
|---|---|---|
| Caddy | :8080 / :8443 | Reverse proxy / TLS termination |
| Dockhand | https://dockhand.test:8443 | Docker management UI |
| Flatnotes | https://flatnotes.test:8443 | Markdown notes |
| Vaultwarden | https://vaultwarden.test:8443 | Password manager |
| Tasks | https://tasks.test:8443 | Task management |

## Host Requirements

- Debian 12 or Ubuntu 22.04+
- Docker Engine (not Docker Desktop)
- Git
- Ports 8080 and 8443 available
- Primary user with uid 1000

## Design Decisions

**Caddy internal CA** — Caddy issues TLS certificates itself using its built-in local CA. No ACME challenges, no port 80/443 requirements, no external CA infrastructure. Import the root cert once to each workstation and all services get a trusted green padlock automatically.

**Shared Docker network** — All containers join a single `homelab` bridge network. Caddy reaches services by container name. The network is created by the Caddy compose and treated as external by all other composes.

**Ports 8080/8443** — Non-standard ports avoid conflicts with other services on the host (e.g. a Kubernetes cluster running on 80/443).

**`/data` layout** — All persistent state lives under `/data/<service>/`. Easy to back up, easy to find, consistent across rebuilds.

**Secrets never in git** — `.env` files are gitignored. Each service directory has a `.env.example` showing what variables are needed. Real values are created manually on the host and stored in Vaultwarden once it is running.

---

## Host Preparation

### 1. Get the Docker group GID

```bash
getent group docker
```

Note the GID (typically 989 or 998 depending on the host). You will need this when setting up Dockhand.

### 2. Create data directories

```bash
sudo mkdir -p /data/caddy
sudo mkdir -p /data/dockhand
sudo mkdir -p /data/flatnotes/data
sudo mkdir -p /data/flatnotes/index
sudo mkdir -p /data/vaultwarden
sudo mkdir -p /data/tasks/tasks
sudo mkdir -p /data/tasks/config
sudo chown -R 1000:1000 /data
```

### 3. Add /etc/hosts entries

Add to `/etc/hosts` on the host and on any workstation that needs access:

```
<host-ip>  dockhand.test flatnotes.test vaultwarden.test tasks.test
```

### 4. Clone the homelab repo

```bash
git clone https://github.com/yourusername/homelab.git ~/Projects/homelab
cd ~/Projects/homelab
```

---

## Deployment Order

Services must be started in this order. Caddy creates the `homelab` network that everything else depends on:

1. Caddy (creates the homelab network, issues all TLS certs)
2. Dockhand
3. Flatnotes
4. Vaultwarden
5. Tasks

---

## Step 1 — Caddy

Caddy has no `.env` file — all config is in the Caddyfile. Start it directly:

```bash
cd ~/Projects/homelab/caddy
docker compose up -d
```

Verify certificates were issued for all hostnames:

```bash
docker logs caddy 2>&1 | grep "certificate obtained"
```

You should see `certificate obtained successfully` with `issuer: local` for each hostname within a few seconds of startup.

### Export and import the root CA cert

Caddy's internal CA root cert must be imported to each workstation once. Extract it from the running container:

```bash
docker exec caddy find / -name "root.crt" 2>/dev/null
# Expected: /data/caddy/pki/authorities/local/root.crt

docker exec caddy cat /data/caddy/pki/authorities/local/root.crt > ~/caddy-root-ca.crt
```

Copy to your workstation and import:

**Windows (covers Chrome and Edge) — run PowerShell as Administrator:**
```powershell
Import-Certificate -FilePath "$env:USERPROFILE\Downloads\caddy-root-ca.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

**WSL Debian:**
```bash
sudo cp caddy-root-ca.crt /usr/local/share/ca-certificates/caddy-root-ca.crt
sudo update-ca-certificates
```

**macOS:**
```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain caddy-root-ca.crt
```

**Linux:**
```bash
sudo cp caddy-root-ca.crt /usr/local/share/ca-certificates/caddy-root-ca.crt
sudo update-ca-certificates
```

Restart your browser after importing. Firefox manages its own trust store — import via Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import.

---

## Step 2 — Dockhand

### Set correct data directory ownership

Dockhand runs as `1000:<docker-gid>` to access the Docker socket. The data directory must be owned accordingly. Replace `989` with your actual Docker GID:

```bash
sudo chown -R 1000:989 /data/dockhand
```

### Verify the user directive in docker-compose.yml

Ensure `dockhand/docker-compose.yml` has the correct GID:

```yaml
user: "1000:989"
```

Update this value if your Docker GID differs before starting.

### Start Dockhand

```bash
cd ~/Projects/homelab/dockhand
docker compose up -d
docker logs dockhand 2>&1 | tail -10
```

Look for `Listening on http://0.0.0.0:3000/` with no errors. If you see `attempt to write a readonly database` see Troubleshooting below.

Browse to https://dockhand.test:8443 and complete initial account setup.

### Add the local Docker environment

In the Dockhand UI go to Environments → Add Environment → Local socket → `/var/run/docker.sock`. All running containers will then appear.

### Adopt existing stacks

If containers were started outside Dockhand, go to Stacks → Adopt to import them so Dockhand can manage them going forward.

---

## Step 3 — Flatnotes

### Create .env

```bash
cd ~/Projects/homelab/flatnotes
cp .env.example .env
nano .env
```

Set these values:
- `FLATNOTES_USERNAME` — your login username
- `FLATNOTES_PASSWORD` — your login password
- `FLATNOTES_SECRET_KEY` — generate with `openssl rand -base64 32`

### Start Flatnotes

```bash
docker compose up -d
```

Browse to https://flatnotes.test:8443

### Migrate existing notes

```bash
cp /path/to/old/notes/*.md /data/flatnotes/data/
```

Flatnotes detects new files immediately — no restart needed.

---

## Step 4 — Vaultwarden

### Create .env

```bash
cd ~/Projects/homelab/vaultwarden
cp .env.example .env
nano .env
```

Generate a strong admin token:

```bash
openssl rand -base64 48
```

Set this as `ADMIN_TOKEN` in `.env`.

### Start Vaultwarden

```bash
docker compose up -d
```

Browse to https://vaultwarden.test:8443

### Create your account

`SIGNUPS_ALLOWED` is `false` by default. Create your account via the admin panel:

1. Go to https://vaultwarden.test:8443/admin
2. Enter your `ADMIN_TOKEN`
3. Go to Users → Invite User
4. Complete registration

Once Vaultwarden is running, store all `.env` files and secrets in it. That becomes the source of truth for credentials going forward.

---

## Step 5 — Tasks

```bash
cd ~/Projects/homelab/tasks
docker compose up -d
```

Browse to https://tasks.test:8443

---

## Adding a New Service

Follow this process for every new service added to the homelab.

### 1. Create the service directory and files

```bash
mkdir ~/Projects/homelab/myservice
cd ~/Projects/homelab/myservice
```

Create `docker-compose.yml`. All services must join the `homelab` network. Do not expose ports directly — Caddy handles all ingress:

```yaml
services:
  myservice:
    image: someimage:latest
    container_name: myservice
    restart: unless-stopped
    environment:
      SOME_VAR: ${SOME_VAR}
    volumes:
      - /data/myservice:/data
    networks:
      - homelab
    logging:
      driver: json-file
      options:
        max-size: 1M
        max-file: "2"

networks:
  homelab:
    name: homelab
    external: true
```

Create `.env.example` documenting required variables with no real values:

```
# Copy to .env and fill in values — never commit .env to git
SOME_VAR=REPLACE_ME
```

### 2. Create the data directory on the host

```bash
sudo mkdir -p /data/myservice
sudo chown -R 1000:1000 /data/myservice
```

### 3. Add the route to the Caddyfile

Edit `caddy/Caddyfile` and add a new block:

```
myservice.test:8443 {
  tls internal
  reverse_proxy myservice:<container-port>
}
```

### 4. Add the hostname to /etc/hosts

On the host and each workstation add to `/etc/hosts`:

```
<host-ip>  myservice.test
```

On Windows the hosts file is at `C:\Windows\System32\drivers\etc\hosts` — edit as Administrator. Save the file before testing.

### 5. Commit to git safely

Always add files explicitly — never use `git add .`:

```bash
cd ~/Projects/homelab
git add myservice/docker-compose.yml myservice/.env.example caddy/Caddyfile
git status  # verify .env is NOT listed before committing
git commit -m "add myservice"
git push
```

### 6. Create the real .env on the host

```bash
cd ~/Projects/homelab/myservice
cp .env.example .env
nano .env  # fill in real values
```

### 7. Reload Caddy and start the service

```bash
cd ~/Projects/homelab/caddy && docker compose down && docker compose up -d
cd ~/Projects/homelab/myservice && docker compose up -d
```

Verify the cert was issued:

```bash
docker logs caddy 2>&1 | grep "myservice.test"
```

---

## Day-to-Day Operations

### Start all services

```bash
for svc in caddy dockhand flatnotes vaultwarden tasks; do
  cd ~/Projects/homelab/$svc && docker compose up -d && cd -
done
```

### Stop all services

```bash
for svc in tasks vaultwarden flatnotes dockhand caddy; do
  cd ~/Projects/homelab/$svc && docker compose down && cd -
done
```

### Update all images

```bash
for svc in caddy dockhand flatnotes vaultwarden tasks; do
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
3. Get the Docker GID: `getent group docker`
4. Update `dockhand/docker-compose.yml` `user` directive with correct GID
5. Create `/data` directories and set ownership (see Host Preparation)
6. Copy `/data` from old host or restore from backup
7. Create `.env` files in each service directory — retrieve values from Vaultwarden
8. Add `/etc/hosts` entries on host and workstations
9. Start services in order
10. Extract Caddy root CA cert and import to workstation

The `/data` directory contains all persistent state — back this up regularly. The git repo contains all configuration. The `.env` files contain all secrets — store them in Vaultwarden.

---

## Troubleshooting

### Dockhand: "attempt to write a readonly database"

The data directory was created by root before the `user` directive was set. Fix ownership and restart:

```bash
sudo chown -R 1000:<docker-gid> /data/dockhand
cd ~/Projects/homelab/dockhand && docker compose down && docker compose up -d
```

If the database is corrupted from repeated failed starts, wipe and recreate it:

```bash
cd ~/Projects/homelab/dockhand && docker compose down
sudo rm -rf /data/dockhand/db
docker compose up -d
```

You will need to recreate your Dockhand account and re-add the Docker environment.

### Dockhand: containers not visible / Docker socket permission denied

The container must run with the Docker group GID. Verify:

```bash
getent group docker
docker inspect dockhand --format '{{.HostConfig.User}}'
```

The second command should return `1000:<docker-gid>`. If not, update `docker-compose.yml` and restart.

### Certificate not trusted in browser

Extract the cert and re-import to the system trust store:

```bash
docker exec caddy cat /data/caddy/pki/authorities/local/root.crt > caddy-root-ca.crt
```

On Windows, Firefox requires a separate import via its own certificate manager — the Windows system trust store does not cover Firefox.

### New service not reachable after adding to Caddyfile

Caddy must be restarted to pick up Caddyfile changes:

```bash
cd ~/Projects/homelab/caddy && docker compose down && docker compose up -d
docker logs caddy 2>&1 | grep "myservice.test"
```

Also verify the `/etc/hosts` entry was saved on the workstation — unsaved changes are a common cause of "can't reach that page" errors in the browser.

### Service returns 502 Bad Gateway

Caddy is running but cannot reach the backend container. Check the backend is running and on the homelab network:

```bash
docker ps | grep myservice
docker inspect myservice --format '{{json .NetworkSettings.Networks}}'
```

The output must show `homelab`. If it shows `myservice_default` the compose file is missing the network block — add it and redeploy.

### Flatnotes search index stale

Trigger a search in the UI to force a sync. If still stale:

```bash
docker restart flatnotes
```

### Vaultwarden won't start

Requires `DOMAIN` set to the full HTTPS URL including port:

```
DOMAIN=https://vaultwarden.test:8443
```
