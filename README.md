# Beszel — Server Monitoring

Lightweight server monitoring with Docker stats, historical data, and alerts.
Uses a **hub** (web dashboard) + **agent** (per-server metrics collector).

Docs: https://beszel.dev

## Architecture

- **Hub**: PocketBase web app — serves the dashboard, stores metrics, sends alerts.
- **Agent**: Runs on each monitored server — reports CPU, RAM, disk, network, container stats to the hub.

One hub can monitor **multiple systems** (servers). Each system runs its own agent.

## Setup on a new VPS (single system, hub + agent on same server)

### Prerequisites

- Docker + Docker Compose
- Traefik running with `traefik-net` network (or adapt labels for your reverse proxy)
- A DNS record pointing your domain to the VPS (e.g. `status.example.com`)

### 1. Create the directory on the VPS

```bash
ssh your-vps 'mkdir -p /home/ubuntu/beszel'
```

### 2. Adapt `docker-compose.yml`

Edit the compose file:
- Change `APP_URL` and Traefik `Host()` rule to your domain
- The agent section will be completed after step 4

```yaml
services:
  beszel:
    image: henrygd/beszel:0.18.7
    container_name: beszel
    restart: unless-stopped
    user: "65534:65534"
    security_opt:
      - no-new-privileges:true
    environment:
      - APP_URL=https://status.example.com
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - beszel-data:/beszel_data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.beszel-http.rule=Host(`status.example.com`)"
      - "traefik.http.routers.beszel-http.entrypoints=web"
      - "traefik.http.routers.beszel-http.middlewares=beszel-https"
      - "traefik.http.middlewares.beszel-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.beszel-https.redirectscheme.permanent=true"
      - "traefik.http.routers.beszel.rule=Host(`status.example.com`)"
      - "traefik.http.routers.beszel.entrypoints=websecure"
      - "traefik.http.routers.beszel.tls.certresolver=letsencrypt"
      - "traefik.http.services.beszel.loadbalancer.server.port=8090"
    networks:
      - traefik-net

volumes:
  beszel-data:
    name: beszel-data

networks:
  traefik-net:
    external: true
    name: traefik-net
```

### 3. Deploy the hub

```bash
scp docker-compose.yml your-vps:/home/ubuntu/beszel/
ssh your-vps 'cd /home/ubuntu/beszel && docker compose up -d beszel'
```

### 4. Create admin account + add system

1. Open `https://status.example.com` in your browser
2. Create your admin account
3. Click **Add System**:
   - **Name**: your server name (e.g. `my-vps`)
   - **Host**: `host.docker.internal`
   - **Port**: `45876` (given)
4. Copy the **docker-compose.yml** shown in the dialog

### 5. Add the agent to `docker-compose.yml`

Create a `.env` file next to your `docker-compose.yml` with the key and token from step 4:

```bash
# .env  (never commit this)
BESZEL_AGENT_KEY=ssh-ed25519 AAAA...
BESZEL_AGENT_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Get the docker group GID: `getent group docker | cut -d: -f3` (e.g. `988`), then add the agent service:

```yaml
  beszel-agent:
    image: henrygd/beszel-agent:0.18.7 # version important
        container_name: beszel-agent
        restart: unless-stopped
        network_mode: host
        user: "65534:988"  # nobody:docker — replace 988 with your docker GID
        security_opt:
            - no-new-privileges:true # important


    environment:
    ...
      KEY: ${BESZEL_AGENT_KEY}
      TOKEN: ${BESZEL_AGENT_TOKEN}
      ...
```

### 6. Deploy the agent

```bash
scp docker-compose.yml your-vps:/home/ubuntu/beszel/
scp .env your-vps:/home/ubuntu/beszel/
# Fix volume ownership for non-root hub (run once)
ssh your-vps 'docker run --rm -v beszel-data:/data alpine chown -R 65534:65534 /data'
ssh your-vps 'cd /home/ubuntu/beszel && docker compose up -d'
```

### 7. Verify

```bash
ssh your-vps 'docker logs beszel-agent 2>&1 | tail -3'
# Should show: "WebSocket connected host=status.example.com"
```

Click **Add System** in the Beszel UI — it should turn green.

## Adding a second server (multi-system)

If you have another VPS you want to monitor from the same hub:

### 1. On the hub (status.example.com)

Click **Add System** → set **Host** to the remote server's IP, **Port** `45876`.
Copy the **public key** and **token**.

### 2. On the remote server

```bash
mkdir -p /home/ubuntu/beszel-agent
```

Create a `docker-compose.yml` with **only the agent**: (Same structure as what you got for the first setup)

Create a `.env` with the key and token, then a `docker-compose.yml`:

```bash
# .env
BESZEL_AGENT_KEY=ssh-ed25519 AAAA...
BESZEL_AGENT_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

```yaml
services:
  beszel-agent:
    image: henrygd/beszel-agent:0.18.7
    container_name: beszel-agent
    restart: unless-stopped
    network_mode: host
    user: "65534:988"  # nobody:docker — replace 988 with your docker GID
    security_opt:
      - no-new-privileges:true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      LISTEN: 45876  # exposed to hub — open in firewall if needed
      KEY: ${BESZEL_AGENT_KEY}
      TOKEN: ${BESZEL_AGENT_TOKEN}
      HUB_URL: https://status.example.com
```

```bash
docker compose up -d
```

The new system should appear green in the hub dashboard.

> **Note**: Port 45876 must be reachable from the hub server. Open it in the firewall if needed,
> or use Beszel's SSH connection method instead (see https://beszel.dev/guide/agent-installation).

## Telegram Alerts

1. Go to **Settings** → **Notifications** in the Beszel UI
2. Add URL: `telegram://BOT_TOKEN@telegram?chats=CHAT_ID`
3. Go to the **Systems** table → locate your system → **Alerts** icon on the row
4. Enable alerts for CPU, memory, disk, status, etc.

## Useful commands

```bash
# Check status
ssh your-vps 'docker ps | grep beszel'

# View agent logs
ssh your-vps 'docker logs beszel-agent 2>&1 | tail -20'

# View hub logs
ssh your-vps 'docker logs beszel 2>&1 | tail -20'

# Update to a new version (edit image tag in docker-compose.yml, then:)
ssh your-vps 'cd /home/ubuntu/beszel && docker compose pull && docker compose up -d'

# Full reset (loses all data)
ssh your-vps 'cd /home/ubuntu/beszel && docker compose down -v'
```
