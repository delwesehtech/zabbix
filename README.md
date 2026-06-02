# Zabbix server (Docker Compose)

Zabbix **7.4** server stack for Linux: PostgreSQL, Zabbix server, and web UI (Nginx).

Monitored machines use **Zabbix Agent** (installed on the host) or **Agent 2 in Docker** — see [Connecting agents](#connecting-agents). This setup assumes agents are on the **same trusted LAN** (plaintext on port 10051, no PSK/TLS).

## Requirements

- Linux host with Docker Engine and Docker Compose v2.24+
- Ports reachable from monitored hosts:
  - **10051/TCP** — Zabbix server (agents send data here)
  - **8051/TCP** (or your `ZABBIX_WEB_PORT`) — web UI

## Quick start

```bash
cd ~/code/monitoring/zabbix
cp .env.example .env
# Edit .env: set POSTGRES_PASSWORD (required), TZ, ZABBIX_WEB_PORT if needed
docker compose up -d
```

First startup can take 1–3 minutes while the database is initialized.

- **Web UI:** `http://<server-host>:8051` (or your `ZABBIX_WEB_PORT`)
- **Default login:** `Admin` / `zabbix` — change the password immediately.

Check status:

```bash
docker compose ps
docker compose logs -f zabbix-server
```

## Connecting agents

Agents must reach the Zabbix server on **port 10051** at the server host’s IP or DNS name (not `localhost` from remote hosts).

### Agent on the host (recommended)

Install Zabbix Agent 2 from your distro or [Zabbix packages](https://www.zabbix.com/download), then set in `/etc/zabbix/zabbix_agent2.conf` (paths vary):

```ini
Server=<zabbix-server-ip>
ServerActive=<zabbix-server-ip>
Hostname=<unique-hostname-matching-zabbix-ui>
```

Restart the agent, then in the Zabbix UI add the host (Configuration → Hosts) with the same **Hostname** and link templates (e.g. “Linux by Zabbix agent”).

### Agent in Docker on a monitored host

Use `docker-compose.agent.example.yml` on each host:

```bash
export ZBX_HOSTNAME=myserver01
export ZBX_SERVER_HOST=10.0.0.5   # IP of the machine running this compose stack
docker compose -f docker-compose.agent.example.yml up -d
```

Register the host in the UI with hostname `myserver01`.

### Agent on the same host as this stack

Point the agent at the host’s LAN IP (or `host.docker.internal` is not reliable on Linux). Example:

```bash
ZBX_SERVER_HOST=192.168.1.10 ZBX_HOSTNAME=zabbix-monitor-host docker compose -f docker-compose.agent.example.yml up -d
```

Do not set `ZBX_SERVER_HOST=zabbix-server` unless the agent container shares a network with this stack (not the default setup).

## Firewall (example)

```bash
# ufw on the Zabbix server host
sudo ufw allow 8051/tcp comment 'Zabbix web'
sudo ufw allow 10051/tcp comment 'Zabbix agents'
```

## Upgrades

1. Update `ZBX_IMAGE_TAG` in `.env` (e.g. `alpine-7.4-latest` pulls the latest 7.4 patch).
2. `docker compose pull && docker compose up -d`

Database migrations run automatically on server start.

## Optional components

This stack omits the Java gateway, SNMP traps, and the web service (scheduled PDF reports). Add them in `docker-compose.yml` if you later need JMX, SNMP traps, or reports — see [official Zabbix Docker docs](https://www.zabbix.com/documentation/current/en/manual/installation/containers).

## Data layout

The database lives in a Docker **named volume** (`zabbix_postgres_data`), so there are no host-permission issues.

Custom scripts you provide are bind-mounted read-only from `./data/` (gitignored):

- `data/alertscripts/` — custom alert/notification scripts
- `data/externalscripts/` — external checks

Backups:

```bash
# Database dump
docker compose exec -T postgres pg_dump -U zabbix zabbix | gzip > zabbix-$(date +%F).sql.gz

# Or snapshot the raw volume
docker run --rm -v zabbix_postgres_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/postgres_data.tar.gz -C /data .
```

Back up the database and your `.env` before major upgrades.

## License

Zabbix 7.0+ is AGPLv3. See [Zabbix licensing](https://www.zabbix.com/license).
