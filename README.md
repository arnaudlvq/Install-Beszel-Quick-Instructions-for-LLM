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
    environment:
      - APP_URL=https://status.example.com
    volumes:
      - beszel-data:/beszel_data
    labels:
      - "traefik.enable=true"
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
   - **Host**: `localhost`
   - **Port**: `45876` (given)
4. Copy the **docker-compose.yml** shown in the dialog

### 5. Add the agent to `docker-compose.yml`

Add the agent service with the key and token from step 4: paste the peice of yml code you just copied.

### 6. Deploy the agent

```bash
scp docker-compose.yml your-vps:/home/ubuntu/beszel/
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

Create a `docker-compose.yml` with **only the agent**:

```yaml
services:
  beszel-agent:
    image: henrygd/beszel-agent:0.18.7
    container_name: beszel-agent
    restart: unless-stopped
    network_mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      LISTEN: 45876
      KEY: "ssh-ed25519 AAAA... (public key from hub)"
      TOKEN: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      HUB_URL: "https://status.example.com"
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
3. Go to the **Systems** table → click your system → **Alerts** tab
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
