# Zabbix server (Docker Compose)

Zabbix **7.4** server stack for Linux: PostgreSQL, Zabbix server, and web UI (Nginx).

Monitored machines use **Zabbix Agent** (installed on the host) or **Agent 2 in Docker** — see [Connecting agents](#connecting-agents). This setup assumes agents are on the **same trusted LAN** (plaintext on port 10051, no PSK/TLS).

## Requirements

- Linux host with Docker Engine and Docker Compose v2.24+
- Ports reachable from monitored hosts:
  - **10051/TCP** — Zabbix server (agents send data here)
  - **8051/TCP** (or your `ZABBIX_WEB_PORT`) — web UI

## Configuration files

Server and agent config are kept separate (the agent host never needs the DB password):

| Template | Copy to | Used by | Runs on | Command |
|----------|---------|---------|---------|---------|
| `.env.example` | `.env` | `docker-compose.agent.yml` | every monitored host | auto-loaded (no flag) |
| `.env.server.example` | `.env.server` | `docker-compose.yml` | the server host | `--env-file .env.server` |

Both real files are gitignored. Agents use the default `.env` (Compose loads it automatically); only the server is explicit, so the two never collide — even on a host that runs both.

## Quick start (server host)

```bash
cd ~/code/monitoring/zabbix
cp .env.server.example .env.server
# Edit .env.server: set POSTGRES_PASSWORD (required), TZ, ZABBIX_WEB_PORT if needed
docker compose --env-file .env.server -f docker-compose.yml up -d
```

First startup can take 1–3 minutes while the database is initialized.

- **Web UI:** `http://<server-host>:8051` (or your `ZABBIX_WEB_PORT`)
- **Default login:** `Admin` / `zabbix` — change the password immediately.

Check status:

```bash
docker compose --env-file .env.server -f docker-compose.yml ps
docker compose --env-file .env.server -f docker-compose.yml logs -f zabbix-server
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

### Agents: active checks + autoregistration (fleet model)

All Linux hosts are monitored the same way: **active checks only** (agent connects out to the server on 10051) and **autoregistration** (the server auto-creates each host on first connect). `docker-compose.agent.yml` is byte-identical on every host — only `.env` differs.

#### One-time server setup (Zabbix UI)

Create an autoregistration action so new agents become hosts automatically:

1. **Data collection → Actions → Autoregistration actions → Create action**
   - Name: `Linux autoregister`
   - Condition: *Host metadata* **contains** `linux` (matches `ZBX_METADATA`)
2. **Operations** tab — add:
   - *Add host*
   - *Add to host groups*: `Linux servers`
   - *Link templates*: **Linux by Zabbix agent (active)**
3. Enable the action.

#### Per monitored host

Ship `docker-compose.agent.yml` + `.env.example` to the host, then:

```bash
cp .env.example .env
# Edit .env:
#   ZBX_SERVER_HOST=zabbix.example.com   # the server's IP/DNS (same on every host)
#   ZBX_HOSTNAME=                        # empty = use this machine's system hostname
#   ZBX_METADATA=linux
docker compose -f docker-compose.agent.yml up -d
```

Within a minute the host appears under **Data collection → Hosts** named after its system hostname, with the active Linux template linked. No manual host creation, no inbound 10050, no per-host interface IPs.

> The server's own host is just another agent: put the agent values in `.env` and run `docker compose -f docker-compose.agent.yml up -d`. The server stack uses `.env.server` via `--env-file`, so the agent's default `.env` and the server's `.env.server` coexist cleanly on that one host.

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
