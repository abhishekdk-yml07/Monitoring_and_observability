# Alerting Strategy & Runbooks

## Philosophy

> "Alert on symptoms, not causes. Every alert must be actionable."

We follow Google SRE principles:
- **Precision** — alerts fire only when action is needed
- **Recall** — all real incidents trigger an alert
- **Detection time** — catch issues before users do
- **Reset time** — alerts resolve automatically when fixed

---

## Severity Levels

| Severity | Definition | Response Time | Channel |
|----------|-----------|---------------|---------|
| **P1 / critical** | User-facing impact, data loss risk, SLO burn | Immediate (< 5 min) | PagerDuty → on-call |
| **P2 / warning** | Degraded performance, risk of P1 if unaddressed | Business hours (< 2 hr) | Slack `#alerts` |
| **P3 / info** | Anomalies, capacity warnings | Next business day | Slack `#monitoring-info` |

---

## Runbooks

### HighErrorRate
**Alert:** HTTP 5xx error rate > 5% for 1 minute

1. Check Grafana Application dashboard for affected endpoints
2. Check pod logs: `kubectl logs -n production -l app=myapp --tail=200`
3. Check recent deployments: `kubectl rollout history deployment/myapp -n production`
4. If recent deploy: `helm rollback myapp -n production`
5. If no recent deploy: check downstream dependencies (DB, Redis, external APIs)
6. Scale up if traffic-related: `kubectl scale deployment myapp --replicas=10 -n production`

---

### PodCrashLooping
**Alert:** Container restart count > 3 in 15 minutes

1. Identify crashing pods: `kubectl get pods -n production | grep CrashLoop`
2. Get logs: `kubectl logs -n production <pod-name> --previous`
3. Describe pod for events: `kubectl describe pod -n production <pod-name>`
4. Common causes:
   - OOMKilled → increase memory limits in Helm values
   - Failed DB connection → check secrets and DB status
   - Bad config → check ConfigMaps match expected schema
5. Force rollout if needed: `kubectl rollout restart deployment/myapp -n production`

---

### DiskSpaceLow
**Alert:** Disk space < 15%

1. SSH to affected node or use SSM Session Manager
2. Find large files: `du -sh /* 2>/dev/null | sort -rh | head -20`
3. Common culprits:
   - Docker logs: `docker system prune -f`
   - Application logs: check log rotation config
   - Container images: `docker image prune -a`
4. If persistent: expand EBS volume via AWS Console, then extend filesystem
   ```bash
   sudo growpart /dev/nvme0n1 1
   sudo xfs_growfs /
   ```

---

### SLOBurnRateFast
**Alert:** Error budget burning 14.4x faster than target

1. This means the 30-day error budget will be exhausted in ~1 hour
2. Immediately escalate to incident commander
3. Review error budget dashboard
4. Consider emergency maintenance window
5. Identify top error-contributing endpoints from Grafana
6. Decision tree:
   - Known bad deploy → rollback immediately
   - Infrastructure issue → scale or failover
   - Third-party dependency → enable circuit breaker / fallback

---

## Inhibition Logic

```
NodeNotReady (critical)
  └── suppresses → PodCrashLooping, PodNotReady on same node
      (pod alerts are noise when the whole node is gone)

ApplicationDown (critical)
  └── suppresses → HighErrorRate, HighLatency for same instance
      (can't have errors if the app is completely down)

Critical severity
  └── suppresses → Warning severity for same alertname + namespace
      (avoid duplicate pages for same underlying issue)
```

---

## On-Call Rotation

- Weekly rotation, handoff every Monday 9 AM UTC
- PagerDuty escalation: primary → 15 min → secondary → 30 min → manager
- Post-incident review (PIR) required for all P1 incidents within 48 hours
- PIR template: [link to template]
