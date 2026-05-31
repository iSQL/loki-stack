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

| File                                              | Purpose                                |
|---------------------------------------------------|----------------------------------------|
| `docker-compose.yml`                              | Stack definition + Traefik label       |
| `loki/Dockerfile`, `loki/config.yml`              | Loki image with single-binary config   |
| `alloy/Dockerfile`, `alloy/config.alloy`          | Alloy image with tail/parse pipeline   |
| `grafana/Dockerfile`, `grafana/provisioning/...`  | Grafana image: Loki datasource + dashboards |
| `.env.example`                                    | Required env vars                      |

Each service builds from its own subdirectory rather than bind-mounting a config file from the repo. This is a deliberate Coolify workaround: Coolify's build container clones the repo into its own filesystem and then invokes `docker compose up` against the host daemon — relative bind mount sources and `configs: file:` references both resolve to paths the host daemon can't see. Baking configs into per-service images via `COPY` keeps deploys repo-driven without needing the Coolify Storages UI per file.

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

Supports JSON log lines from either logger out of the box:

- **Pino** (Node/Fastify): `{"level":30,"time":1716998400000,"msg":"..."}`
- **Monolog** (Laravel): `{"level":400,"level_name":"ERROR","datetime":"2026-05-31T14:23:01.123456+00:00","message":"..."}`

Extraction:

- **Stream labels** (low cardinality, indexed):
  - `app` — from the path segment
  - `level` — normalized to `trace|debug|info|warn|error|fatal` (Monolog's `WARNING` → `warn`, `EMERGENCY` → `fatal`; unknown → `unknown`)
- **Body** — the raw JSON, so all fields stay queryable via `| json` parser.
- **Timestamp** — from the in-payload `time` (Pino, unix ms) or `datetime` (Monolog, ISO 8601). Falls back to ingest time only if neither is present.

Onboarding a new app that uses a different JSON shape: either map it onto one of these two field sets in the app's logger config, or add a third pair of extractions in [`alloy/config.alloy`](alloy/config.alloy).

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

## Dashboards

Anything checked into [`grafana/provisioning/dashboards/`](grafana/provisioning/dashboards/) is auto-loaded at Grafana boot. Add a new dashboard by exporting its JSON from the UI (Dashboard settings → JSON Model) and dropping it next to the existing files — next deploy bakes it into the Grafana image.

Out of the box:

- **Zabari — Overview** ([`zabari-overview.json`](grafana/provisioning/dashboards/zabari-overview.json)) — 5 panels: lines/sec, error rate %, registrations (24h), stacked volume by level, recent errors. Has an `$app` template variable, so it works for any app that ships logs to Loki, not just zabari.

## Alerting (Grafana managed)

Single tenant + low ingestion volume = Grafana built-in alerting is enough; an Alertmanager sidecar would be over-architected. To add a rule:

1. **Alerting → Alert rules → New alert rule** → type **Grafana managed**.
2. **Query A**: pick Loki datasource, paste the LogQL that returns the metric value (e.g. error rate). Pick query type **Instant** for ratios, **Range** if you want per-step values.
3. **Expression B**: type **Threshold**, set `IS ABOVE 0.05` (5%), input `A`.
4. **Alert condition**: `B`.
5. **Folder**: `Zabari` (create on first use). **Evaluation group**: `zabari` with `Interval: 1m`. **For**: `5m` (don't fire on transients).
6. **Annotations + labels**: `summary = Error rate above 5%`, `severity = warning`, `app = zabari`.
7. **Contact point**: configure once under **Contact points** (email/Slack/webhook). Default policy delivers all rules unless you carve specific ones out.

Suggested first rules (LogQL given, threshold suggested):

| Rule | Query | Threshold | For |
|---|---|---|---|
| Error rate spike | `sum(count_over_time({app="zabari", level=~"error\|fatal\|critical"}[5m])) / sum(count_over_time({app="zabari"}[5m]))` | `> 0.05` | 5m |
| No logs (zabari down?) | `sum(count_over_time({app="zabari"}[5m]))` | `< 1` | 5m |
| Fatal in last minute | `sum(count_over_time({app="zabari", level=~"fatal"}[1m]))` | `>= 1` | 0m |

Once a rule is stable in the UI, **Export → YAML** and commit the result under `grafana/provisioning/alerting/` for future replay.

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
