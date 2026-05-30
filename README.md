# Loki + Grafana + Alloy stack

Host-wide log aggregation for Coolify-deployed apps.

```
   app rolling-log files                  Alloy             Loki        Grafana
┌──────────────────────────┐         ┌──────────┐      ┌─────────┐    ┌────────┐
│ /var/lib/coolify/storage │ ──tail─►│  parse   │─push─►│  store  │◄──►│   UI   │
│   /logs/zabari/*.log     │         │  Pino    │       │  TSDB+  │    │  Loki  │
│   /logs/<other>/*.log    │         │  JSON    │       │  chunks │    │  data- │
└──────────────────────────┘         └──────────┘      └─────────┘    │  source│
                                                                       └────────┘
```

## Layout

| File                                              | Purpose                          |
|---------------------------------------------------|----------------------------------|
| `docker-compose.yml`                              | Stack definition + Traefik label |
| `loki/config.yml`                                 | Single-binary, 30-day retention  |
| `alloy/config.alloy`                              | Tail + parse + ship to Loki      |
| `grafana/provisioning/datasources/loki.yml`       | Auto-wire Loki on Grafana boot   |
| `.env.example`                                    | Required env vars                |

## Deploy on Coolify

1. **New Application** → Build Pack: **Docker Compose**.
2. Source: this repo (`Base Directory: /`).
3. **Environment** panel — copy `.env.example` keys, set:
   - `GRAFANA_DOMAIN=logs.cloudfrog.cc`
   - `GRAFANA_ADMIN_USER=admin`
   - `GRAFANA_ADMIN_PASSWORD=<strong>`
4. **Domains** — point `${GRAFANA_DOMAIN}` at this app (Coolify provisions TLS via Traefik labels in compose).
5. **Persistent Storage** — *not required*; named docker volumes (`loki-data`, `alloy-data`, `grafana-data`) survive redeploys. Add UI Persistent Storage mounts only if you want host-readable paths for chunks/dashboards.
6. **First-time host setup** — Alloy bind-mounts `/var/lib/coolify/storage/logs` read-only. The dir must exist:
   ```bash
   sudo mkdir -p /var/lib/coolify/storage/logs
   ```
   Per-app subdirs (`/zabari`, etc.) get created by the producing apps. Producer apps must run with uid 1000 (which is the default `node` user in Node.js images) and own their subdir:
   ```bash
   sudo mkdir -p /var/lib/coolify/storage/logs/zabari
   sudo chown 1000:1000 /var/lib/coolify/storage/logs/zabari
   ```
7. **Deploy**. Open `https://${GRAFANA_DOMAIN}` → log in → Explore → pick Loki → query:
   ```
   {app="zabari"} | json
   ```

## How a new app onboards

Two steps, no Loki-side config changes:

1. App writes JSON-line logs (one event per line) into `/var/lib/coolify/storage/logs/<your-app>/<anything>.log` on the host. Easiest path: mount that host dir into the app container at some `/storage/logs` path and have its logger write rolling files there.
2. On the host, pre-create the subdir with the right uid:
   ```bash
   sudo mkdir -p /var/lib/coolify/storage/logs/<your-app>
   sudo chown <container-uid>:<container-uid> /var/lib/coolify/storage/logs/<your-app>
   ```

Alloy auto-discovers the new `<your-app>/*.log` files on next scan (`local.file_match` polls every ~15s) and labels them with `app="<your-app>"`. Done.

## What Alloy extracts

For each line of Pino JSON (`{"level":30,"time":1716998400000,"msg":"...",...}`):

- **Stream labels** (low cardinality, indexed):
  - `app` — from the path segment
  - `level` — `info|warn|error|...` (translated from Pino's numeric `level`)
- **Body** — the raw JSON, so all fields stay queryable via `| json` parser.
- **Timestamp** — from the in-payload `time` (unix ms), not ingest time.

Useful queries to bookmark in Grafana:

```logql
# everything from one app, last hour
{app="zabari"}

# errors only, last 24h
{app="zabari", level=~"error|fatal"}

# requests with status >= 500, parsing JSON fields
{app="zabari"} | json | res_statusCode >= 500

# user registrations
{app="zabari"} |= "user registered" | json | line_format "{{.email}}"
```

## Retention + sizing

- **30-day retention** is set in `loki/config.yml` (`limits_config.retention_period`). Change + restart Loki container.
- **Compactor** runs every ~10min and unlinks chunks/indexes past retention. Disk usage stabilizes once daily-write ≈ daily-prune.
- **Disk footprint** scales with log volume + label cardinality. Rule of thumb at this scale: ~10–50 MB/day per low-traffic app, ~few GB/day per busy one.
- The `loki-data` named volume lives on the Coolify host's Docker volume root (`/var/lib/docker/volumes/`). Watch host disk; alert at 75% via Grafana → Alerting if needed.

## Security notes

- Loki has **`auth_enabled: false`** — fine because no host port is exposed and no Traefik label points at it. If you ever expose it, put a reverse proxy with basic auth in front, or switch to a multi-tenant setup.
- Grafana admin password is required at compose boot (`?GRAFANA_ADMIN_PASSWORD is required`). No default.
- Anonymous Grafana sign-up disabled. Add users manually under Server Admin.
- Analytics phone-home disabled for Loki, Alloy, and Grafana.
