# weekend-project

A self-contained monitoring stack running on Docker Compose: MySQL instrumented with
Prometheus, visualised in Grafana, with alerts routed through Alertmanager.

Everything — datasource, dashboards, alert rules — is provisioned from files in this
repo. `docker compose up -d` gives you a fully configured stack with nothing to click.

## Architecture

```
                          BROWSER
                             │
              :3000          │          :9090        :9093
                │            │            │            │
  ══════════════╪════════════╪════════════╪════════════╪══════════
                │  published ports (host → container)  │
  ┌─────────────┼────────────┼────────────┼────────────┼─────────┐
  │ sql-network ▼            ▼            ▼            ▼         │
  │                                                              │
  │   ┌─────────┐   PromQL    ┌────────────┐  alerts  ┌────────┐ │
  │   │ grafana ├────────────►│ prometheus ├─────────►│ alert- │ │
  │   └─────────┘             └──────┬─────┘          │manager │ │
  │                                  │                └────────┘ │
  │                    scrape /metrics every 15s                 │
  │            ┌─────────────────────┼─────────────────┐         │
  │            ▼                     ▼                 ▼         │
  │   ┌────────────────┐     ┌───────────────┐   ┌──────────┐    │
  │   │ mysql-exporter │     │ node-exporter │   │ cadvisor │    │
  │   └───────┬────────┘     └───────────────┘   └──────────┘    │
  │           │ SQL: SHOW GLOBAL STATUS                          │
  │           ▼                                                  │
  │   ┌────────────┐                                             │
  │   │  mysql-db  │──► volume: mysql-data                       │
  │   └────────────┘                                             │
  └──────────────────────────────────────────────────────────────┘
```

Every arrow is a **pull**. Grafana queries Prometheus, Prometheus scrapes the exporters,
and the exporter logs into MySQL. Nothing pushes upward.

MySQL speaks SQL, not Prometheus — `mysql-exporter` is the translation layer that runs
`SHOW GLOBAL STATUS` and re-exposes the results as metrics.

## Requirements

- Docker Desktop (or any Docker engine with Compose v2+)

## Quick start

```bash
git clone https://github.com/xShakedx/weekend-project.git
cd weekend-project
cp .env.example .env     # then edit the passwords
docker compose up -d
```

Give MySQL ~30s to initialise on first run, then open Grafana at
**http://localhost:3000** → Dashboards → *weekend-project*.

## Services

| Service | Port | What it does |
|---|---|---|
| `grafana` | 3000 | Dashboards. Login from `.env` |
| `prometheus` | 9090 | Scrapes metrics, evaluates alert rules, 30d retention |
| `alertmanager` | 9093 | Receives firing alerts and routes notifications |
| `mysql-db` | 3306 | MySQL 8.0, the monitored database |
| `mysql-exporter` | 9104 | Translates MySQL status into Prometheus metrics |
| `node-exporter` | 9100 | Host CPU, memory, disk, load |
| `cadvisor` | 8080 | Per-container CPU and memory |

Only port 3000 is strictly needed; the rest are exposed for debugging.

## Useful URLs

| URL | Purpose |
|---|---|
| http://localhost:3000 | Grafana dashboards |
| http://localhost:9090/targets | Scrape health — all four jobs should be **UP** |
| http://localhost:9090/alerts | Alert rule states |
| http://localhost:9093 | Firing alerts in Alertmanager |

## Configuration

| File | Purpose |
|---|---|
| `docker-compose.yaml` | Service definitions |
| `prometheus.yml` | Scrape jobs, rule files, Alertmanager address |
| `alerts.yml` | Alert rules |
| `alertmanager.yml` | Notification routing |
| `grafana/provisioning/datasources/` | Prometheus datasource, created automatically |
| `grafana/provisioning/dashboards/` | Dashboard provider config |
| `grafana/dashboards/` | Dashboard JSON (MySQL Overview, Node Exporter Full, cAdvisor) |
| `.env` | Passwords — **git-ignored, never commit this** |
| `.env.example` | Template listing the required variables |

Secrets live only in `.env`. The compose file references them as
`${MYSQL_ROOT_PASSWORD:?...}`, so a missing variable fails loudly at startup rather than
silently becoming an empty password.

## Alert rules

| Alert | Condition | Waits |
|---|---|---|
| `MySQLDown` | `mysql_up == 0` | 1m |
| `MySQLTooManyConnections` | connections > 80% of `max_connections` | 5m |
| `HighMemoryUsage` | memory > 90% | 5m |

The wait (`for:`) is what separates a useful alert from a noisy one — a single failed
scrape resolves quietly while still Pending, and only a sustained problem fires.

### Testing an alert

```bash
docker compose stop mysql-db
```

Watch http://localhost:9090/alerts: `MySQLDown` goes **Pending** within ~15s, then
**Firing** a minute later, and appears in Alertmanager. Restore with:

```bash
docker compose start mysql-db
```

## Common tasks

```bash
docker compose ps                      # service status
docker compose logs -f prometheus      # follow logs
docker compose restart prometheus      # reload after editing prometheus.yml
docker compose down                    # stop, keep data
docker compose down -v                 # stop and wipe all volumes
```

Prometheus reads its config **only at startup**, so editing `prometheus.yml` or
`alerts.yml` requires a restart. Validate before restarting:

```bash
docker run --rm \
  -v "$PWD/prometheus.yml:/etc/prometheus/prometheus.yml:ro" \
  -v "$PWD/alerts.yml:/etc/prometheus/alerts.yml:ro" \
  --entrypoint promtool prom/prometheus:v2.53.0 \
  check config /etc/prometheus/prometheus.yml
```

Connect to the database with the client inside the container:

```bash
docker compose exec mysql-db mysql -uroot -p mydatabase
```

## Gotchas

**Browsers use `localhost:PORT`, containers use `service-name:PORT`.** Inside the Grafana
container `localhost:9090` means Grafana itself — the datasource must point at
`http://prometheus:9090`. Published ports exist only for your browser.

**Grafana serves plain HTTP.** `https://localhost:3000` will not connect.

**Prometheus images are tagged with a `v` prefix** (`v2.53.0`), Grafana and MySQL are not
(`11.1.0`, `8.0`).

**node-exporter reports the Docker VM on macOS**, not your Mac — Docker Desktop runs a
Linux VM and the exporter only sees inside it. cAdvisor's per-container numbers are
accurate regardless.

**cAdvisor needs the Docker socket** (`/var/run/docker.sock`) to label containers by name;
without it you get raw cgroup hashes.

**YAML forbids tabs for indentation.** A tab produces the cryptic
`found character that cannot start any token`.

## Possible extensions

- A `README` badge and CI that runs `promtool check config` on every push
- An instrumented app exposing its own `/metrics` (rate, errors, duration)
- Loki + Promtail for logs alongside the metrics
- Port the stack to Kubernetes — service names become Services, named volumes become PVCs
