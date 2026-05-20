# Slow-API Monitoring Stack

A learning-lab setup that runs a Node.js API across three replicas, load-balances them with Nginx, and observes everything with Prometheus, Alertmanager, and cAdvisor.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Services & Docker Compose](#services--docker-compose)
3. [Nginx — Load Balancer](#nginx--load-balancer)
4. [Prometheus — Metrics & Scraping](#prometheus--metrics--scraping)
5. [Alert Rules](#alert-rules)
6. [Alertmanager — Routing & Notifications](#alertmanager--routing--notifications)
7. [How Everything Connects](#how-everything-connects)
8. [Environment Variables & .env Files](#environment-variables--env-files)
9. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
                        ┌─────────────┐
         :8000          │   nginx-lb  │  load balances across replicas
  ──────────────────▶   │  (nginx)    │──────────────────────────────────────┐
                        └─────────────┘                                      │
                               │                          ┌──────────────────▼──────────────────┐
                               │ /nginx_status            │  slow-api   slow-api_2   slow-api_3 │
                               ▼                          │  :8081      :8082        :8083      │
                        ┌──────────────┐                  └─────────────────────────────────────┘
                        │nginx-exporter│
                        │  :9113       │ ◀─── prometheus scrapes
                        └──────────────┘
                                                          ┌───────────────┐
                                                          │  cadvisor     │ ◀─── prometheus scrapes
                                                          │  (host mounts)│      container metrics
                                                          └───────────────┘
                        ┌─────────────┐
                        │ prometheus  │  :9090   evaluates rules, fires alerts
                        └──────┬──────┘
                               │ alert
                               ▼
                        ┌─────────────┐
                        │alertmanager │  :9093   routes & sends email
                        └─────────────┘
                        ┌─────────────┐
                        │  loadgen    │  fires N req/s at nginx-lb/work
                        └─────────────┘
```

All services share a single Docker bridge network: **`slow-api-net`**.

---

## Services & Docker Compose

The `docker-compose.yml` defines every service, their environment, mounts, port bindings, and resource limits. All services are joined to one bridge network so they can resolve each other **by container name** (e.g. `http://nginx-lb`, `http://prometheus:9090`).

### Application replicas

Three identical instances of the Node app run in parallel. Each gets a unique `NUMBER` so you can tell them apart in logs and metrics.

| Service | Host port | `NUMBER` |
|---|---|---|
| `slow-api` | `8081` | `1` |
| `slow-api_2` | `8082` | `2` |
| `slow-api_3` | `8083` | `3` |

All share the same environment:

```
DESIRED_PATH=/work   # path the server listens on
PORT=8080            # internal container port
```

A **memory limit of 128 MB** is applied to every replica via:

```yaml
deploy:
  resources:
    limits:
      memory: 128m
```

### Load generator

`loadgen` is a lightweight Alpine container that fires `$RATE` requests per second at `http://nginx-lb/work`. Tune it without rebuilding:

```yaml
environment:
  - TARGET=http://nginx-lb/work
  - RATE=10
```

---

## Nginx — Load Balancer

`nginx-lb` sits in front of the three replicas and distributes traffic with a round-robin upstream. It also exposes the `stub_status` endpoint that `nginx-exporter` scrapes for metrics.

**`nginx/nginx.conf`**

```nginx
events {}
http {
  upstream slow-api-pool {
    server slow-api:8080;
    server slow-api_2:8080;
    server slow-api_3:8080;
  }
  server {
    listen 80;
    location /work {
      proxy_pass http://slow-api-pool;
    }
    location /nginx_status {
      stub_status;
      allow all;
    }
  }
}
```

`nginx-exporter` reads `/nginx_status` and translates it into Prometheus metrics (active connections, reading, writing, waiting).

---

## Prometheus — Metrics & Scraping

Prometheus is the central metrics database. It **pulls** (scrapes) metrics from exporters on a fixed interval and stores them as time series.

**`prometheus/prometheus.yml`**

```yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - "/etc/prometheus/rules/*.yml"     # ← absolute path, matches the volume mount

scrape_configs:
  - job_name: nginx
    static_configs:
      - targets: ["nginx-exporter:9113"]

  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

### Key points

- **`rule_files`** must use the **absolute container path** that matches the volume mount (`/etc/prometheus/rules/`). A relative path or wrong directory is the most common reason rules silently fail to load.
- **`--web.enable-lifecycle`** flag lets you reload config without restarting: `curl -X POST http://localhost:9090/-/reload`
- **cAdvisor** exposes per-container CPU, memory, and network metrics using Docker labels. The label `container_label_com_docker_compose_service` maps to your Compose service names.

### Validate config & rules

```bash
# Check prometheus.yml
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml

# Check a rule file
docker exec prometheus promtool check rules /etc/prometheus/rules/alerts.yml
```

---

## Alert Rules

Alert rules live in `prometheus/rules/alerts.yml` and are loaded by Prometheus at startup (or on reload). Each rule evaluates a PromQL expression on every scrape cycle.

#### Important: multi-line PromQL in YAML

YAML does not understand PromQL syntax. Curly braces, `=~`, and parentheses confuse the YAML parser if written as a plain scalar across multiple lines. Always use a **literal block scalar** (`|`) for multi-line expressions:

### Example rules

```yaml
groups:
- name: slow-app-rules
  rules:

  - alert: HighInFlightConnections
    expr: sum(nginx_connections_reading + nginx_connections_writing) > 20
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High in-flight connections at Nginx"
      description: "sum(reading+writing) > 20 for 2m."

  - alert: HighMemoryUsage
    expr: |
      (
        max by (container_label_com_docker_compose_service) (
          container_memory_working_set_bytes{container_label_com_docker_compose_service=~"slow-api.*"}
        )
        /
        max by (container_label_com_docker_compose_service) (
          container_spec_memory_limit_bytes{container_label_com_docker_compose_service=~"slow-api.*"}
        )
      ) > 0.5
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Container using >50% of its memory limit"
      description: "Working set / limit > 0.5 for 2m on one or more slow-api replicas."
```

---

## Alertmanager — Routing & Notifications

When Prometheus evaluates a rule as firing, it pushes the alert to Alertmanager. Alertmanager then decides **who to notify, how, and how often** — deduplicating, grouping, and silencing as configured.

#### Important: smtp config

the idea was to have the smtp-config in an `alertmager.env` file, however, the variables were not properly replaced on build/startup. So I hardcolded most of the values and use a `smtp_password.secret` file to store the smtp password (and add it to `.gitignore`). This requires an additional mount in the `docker-compose.yml`.

**`docker-compose.yml`**

```yaml
# extract...
alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
#    env_file:
#      - ./alertmanager/alertmanager.env  # smtp variable replacement failed
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - ./alertmanager/smtp_password.secret:/etc/alertmanager/smtp_password.secret:ro
```

**`alertmanager/alertmanager.yml`**

```yaml
global:
  smtp_smarthost: "mail.gmail.com:587"
  smtp_from: "myaddress@gmail.com"
  smtp_auth_username: "myaddress@gmail.com"
  smtp_auth_password_file: /etc/alertmanager/smtp_password.secret
  smtp_require_tls: true

route:
  receiver: email-me

receivers:
  - name: email-me
    email_configs:
      - to: "myaddress@gmail.com"

```

#### optionally gmail / modern SMTP

Use an **App Password** (not your login password) and enable 2FA on your Google account first. The App Password is generated at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords).

### Send a test alert manually

Bypasses Prometheus entirely — useful for testing Alertmanager config in isolation:

```bash
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"}}]'
```

---

## How Everything Connects

```
loadgen
  └─▶ nginx-lb:80/work
        └─▶ round-robin ▶ slow-api / slow-api_2 / slow-api_3

nginx-exporter
  └─▶ scrapes nginx-lb:80/nginx_status
        └─▶ exposes :9113/metrics

cadvisor
  └─▶ reads host Docker socket / cgroups
        └─▶ exposes :8080/metrics

prometheus
  └─▶ scrapes nginx-exporter:9113
  └─▶ scrapes cadvisor:8080
  └─▶ evaluates rules every 15 s
        └─▶ FIRING alert ▶ alertmanager:9093
              └─▶ sends email via SMTP
```

All inter-service communication uses **container names** as hostnames (e.g. `nginx-lb`, `cadvisor`) because every service is on the shared `slow-api-net` bridge network. No hardcoded IPs needed.

---

## Environment Variables & .env Files

Sensitive SMTP settings live in `alertmanager/alertmanager.env` and are injected via `env_file:` in Compose. **Add this file to `.gitignore`.**

```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_FROM=alerts@example.com
SMTP_USER=you@gmail.com
SMTP_PASS=your-app-password
ALERT_RECEIVER_EMAIL=oncall@example.com
```

> **Note:** `SMTP_PORT` as a variable does not work directly in Alertmanager's `smarthost` field — hardcode the port (`:587`) in `alertmanager.yml` or use the `envsubst` entrypoint approach described above.

---

## Troubleshooting

### General — check logs first

```bash
docker logs prometheus    2>&1 | grep -i "error\|warn\|rule"
docker logs alertmanager  2>&1 | grep -i "error\|smtp\|email\|warn"
docker logs nginx-lb      2>&1
docker logs cadvisor      2>&1 | grep -i "error"
```

### Prometheus rules not loading

```bash
# Are the files visible inside the container?
docker exec prometheus ls /etc/prometheus/rules/

# Validate syntax
docker exec prometheus promtool check rules /etc/prometheus/rules/alerts.yml

# Validate full config
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml
```

Common causes:

| Symptom | Cause | Fix |
|---|---|---|
| Rules dir empty | Wrong host path in volume mount | Check `./prometheus/rules/` exists on the host |
| Rules present but not evaluated | `rule_files:` path doesn't match mount target | Use absolute path `/etc/prometheus/rules/*.yml` |
| YAML parse error | Multi-line PromQL not using `\|` block scalar | Add `\|` after `expr:` |

### Alertmanager not sending email

```bash
# Watch logs live while sending a test alert
docker logs -f alertmanager

# Send test alert
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"}}]'

# Confirm env vars reached the container
docker exec alertmanager env | grep SMTP

# Confirm envsubst expanded the config
docker exec alertmanager cat /etc/alertmanager/alertmanager.yml
```

Common causes:

| Error in logs | Cause | Fix |
|---|---|---|
| `lookup tcp/${SMTP_PORT}: unknown port` | Variable not substituted | Hardcode port or use `envsubst` entrypoint |
| `535 Authentication failed` | Wrong password | Use an App Password for Gmail |
| `dial tcp: connection refused` | Wrong host or port | Check `SMTP_HOST` and port/TLS combo |
| `tls: no supported versions` | TLS mismatch | Match port to `require_tls` setting |

### Port / TLS reference

| Port | Protocol | `require_tls` |
|---|---|---|
| `25` | Plain SMTP | `false` |
| `587` | STARTTLS | `true` |
| `465` | SMTPS | `true` |

### Reload configs without restarting

```bash
# Prometheus (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload

# Alertmanager
curl -X POST http://localhost:9093/-/reload
```

### cAdvisor permission errors on Linux

If cAdvisor fails to read host metrics, add `privileged: true` to the service in Compose. This is acceptable in a learning lab but should be avoided in production.

```yaml
cadvisor:
  privileged: true
```
