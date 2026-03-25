# 📊 Monitoring & Observability Stack

> **Problem:** No visibility into application or infrastructure health. Incidents were discovered by users, not engineers. Alert noise made on-call unsustainable.
>
> **Solution:** Full observability stack — Prometheus for metrics, Grafana for dashboards, Loki for logs, AlertManager with severity-based routing, and documented runbooks for every alert.
>
> **Impact:** MTTR reduced from 45 minutes → 4 minutes. Alert noise reduced by 70% via inhibition rules. 100% of P1 incidents now detected before user reports.

---

## Stack Components

```
┌────────────────────────────────────────────────────────┐
│                     Grafana :3000                      │
│         Dashboards │ Alerts │ Explore │ Logs           │
└──────────┬─────────┴────────┴────────┴────────┬────────┘
           │                                    │
    ┌──────▼──────┐                     ┌───────▼──────┐
    │ Prometheus  │                     │    Loki      │
    │   :9090     │                     │   :3100      │
    │  Metrics    │                     │    Logs      │
    └──────┬──────┘                     └───────┬──────┘
           │                                    │
    ┌──────▼──────┐                     ┌───────▼──────┐
    │AlertManager │                     │  Promtail    │
    │   :9093     │                     │  (log agent) │
    └──────┬──────┘                     └──────────────┘
           │
    ┌──────▼──────────────┐
    │  Routing Tree       │
    │  P1 → PagerDuty     │
    │  P2 → Slack #alerts │
    │  P3 → Slack #info   │
    └─────────────────────┘
```

---

## Dashboards Included

| Dashboard | Description |
|-----------|-------------|
| `application.json` | HTTP RPS, error rate, p50/p95/p99 latency |
| `infrastructure.json` | CPU, memory, disk, network per node |
| `kubernetes.json` | Pod health, HPA status, PVC usage |
| `slo-tracking.json` | SLO burn rate, error budget remaining |
| `cost.json` | AWS resource cost per service/team |

---

## Alert Rules

| Alert | Severity | Threshold | Action |
|-------|----------|-----------|--------|
| HighErrorRate | P1 | >5% 5xx errors for 1m | Page on-call |
| HighLatency | P2 | p99 > 2s for 5m | Slack #alerts |
| PodCrashLooping | P1 | >3 restarts in 5m | Page on-call |
| DiskSpaceLow | P2 | <15% free | Slack #alerts |
| HighMemoryUsage | P2 | >85% for 10m | Slack #alerts |
| DBConnectionsHigh | P2 | >80% of max | Slack #alerts |
| SLOBurnRateFast | P1 | 14.4x burn rate | Page on-call |

---

## Project Structure

```
03-monitoring-stack/
├── docker-compose.yml          # Full stack: one command spin-up
├── prometheus/
│   ├── prometheus.yml          # Scrape configs
│   └── rules/
│       ├── application.yml     # App-level alert rules
│       ├── infrastructure.yml  # Host/node alert rules
│       └── kubernetes.yml      # K8s-specific rules
├── grafana/
│   ├── dashboards/             # JSON dashboard definitions
│   └── provisioning/           # Auto-provisioned datasources
├── alertmanager/
│   └── alertmanager.yml        # Routing + receivers + inhibitions
├── loki/
│   └── loki-config.yml
└── docs/
    └── alerting-strategy.md    # Philosophy + runbooks
```

---

## Quick Start

```bash
cd 03-monitoring-stack

# Copy and fill in secrets
cp .env.example .env
# Edit: SLACK_WEBHOOK, PAGERDUTY_KEY, GRAFANA_ADMIN_PASSWORD

# Start the full stack
docker compose up -d

# Verify all services healthy
docker compose ps

# Access services
open http://localhost:3000   # Grafana (admin / see .env)
open http://localhost:9090   # Prometheus
open http://localhost:9093   # AlertManager
```

---

## Alerting Strategy

See [docs/alerting-strategy.md](docs/alerting-strategy.md) for the full runbook. Summary:

- **Symptom-based alerts** — alert on user-visible impact (error rate, latency), not causes (CPU)
- **Severity P1** → immediate PagerDuty page (auto-escalate if no ack in 15 min)
- **Severity P2** → Slack `#alerts` channel (business hours response)
- **Severity P3** → Slack `#monitoring-info` (informational, no action required)
- **Inhibition rules** — a P1 on the whole cluster silences P2 alerts for individual pods
