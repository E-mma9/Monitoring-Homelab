# Monitoring

The monitoring setup for my 3-node Proxmox cluster. Everything here was installed and configured manually — no helper scripts, no Docker, no pre-built stacks. Each config file in this repo is the actual config running on my infrastructure.

## What's in this stack

| Component | Version | What it does |
|-----------|---------|-------------|
| Prometheus | 3.4.0 | Scrapes metrics from all nodes every 15 seconds, stores them for 90 days |
| Grafana | 11.x | Dashboards — both custom-built and imported from the community |
| Alertmanager | 0.28.1 | Groups alerts and routes them to PagerDuty |
| blackbox_exporter | 0.25.0 | Probes my services with HTTP requests to check if they're alive |
| node_exporter | 1.9.1 | Runs on each Proxmox host, exposes CPU/RAM/disk/network metrics |

Everything except node_exporter runs inside a single LXC container (CT 110, Debian 13, 2GB RAM, 2 cores). node_exporter runs directly on each Proxmox host since it needs access to the host's kernel stats.

## How it all connects

```
pve (:9100)  ─┐
node2 (:9100) ─┼── Prometheus (:9090) ──► Alertmanager (:9093) ──► PagerDuty
node3 (:9100) ─┘        │
                         │
blackbox (:9115) ────────┘        Grafana (:3000)
  checks services                   reads from Prometheus
```

Prometheus pulls data — it reaches out to each target every 15 seconds and asks "give me your current numbers." This is the industry standard approach (pull model). The targets don't need to know where Prometheus is.

## Alert rules

Seven rules, split into two categories.

**Infrastructure** — is the host itself okay?

- `NodeDown` — can't reach node_exporter → node probably crashed or lost network (fires after 1 min)
- `DiskSpaceWarning` — root partition above 85% (fires after 5 min)
- `DiskSpaceCritical` — root partition above 95%, things start breaking at this point (fires after 2 min)
- `HighMemoryUsage` — RAM above 90% (fires after 5 min)
- `HighCPUUsage` — CPU above 95% sustained (fires after 10 min — short spikes are normal so the long window filters those out)

**Services** — are my apps responding?

- `EndpointDown` — blackbox HTTP check fails, service is unreachable (fires after 2 min)
- `SlowResponse` — service responds but takes more than 5 seconds (fires after 5 min, usually a warning sign before a full outage)

Every `for` duration is intentional. Without it, a 2-second network blip would wake me up at 3 AM for nothing.

## The alerting pipeline

When something breaks, this is what happens:

1. Prometheus detects the issue (within 15 seconds)
2. Alert enters "Pending" — the `for` timer starts
3. If the problem persists past the `for` duration, alert starts "Firing"
4. Alertmanager receives it, waits 30 seconds to group related alerts
5. Sends to PagerDuty via the Events API
6. PagerDuty creates an Incident and pushes to my phone + email

From crash to phone buzz: about 3 minutes for most alerts. That's the `for` duration + Alertmanager grouping window.

I chose PagerDuty over Telegram or Discord because it's what companies actually use. The free tier is enough for a homelab (up to 5 users).

## PromQL queries I wrote

These are in the custom Grafana dashboard. Writing these by hand instead of importing a template was the whole point — I wanted to understand what each function does.

**CPU usage as a percentage:**
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
`rate()` calculates the per-second change over 5 minutes. We take the idle time, average it across all cores per host, and subtract from 100 to get usage.

**RAM usage:**
```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```
Straightforward — available divided by total, flipped to show "used" instead of "free."

**Disk usage:**
```promql
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)
```
Same pattern but for the root partition.

**Network traffic in bits/s:**
```promql
rate(node_network_receive_bytes_total{device!="lo"}[5m]) * 8
```
Bytes to bits (multiply by 8) because network speeds are measured in bits. `{device!="lo"}` excludes loopback.

## Services being monitored by blackbox

| URL | Service |
|-----|---------|
| `https://192.168.1.118:8000` | Vaultwarden |
| `http://192.168.1.30:81` | Nginx Proxy Manager |
| `http://192.168.1.223/admin` | Pi-hole |
| `http://localhost:3000` | Grafana |
| `http://192.168.1.250:8000` | Paperless-ngx |
| `http://192.168.1.96:8080` | Glance |

SSL verification is disabled in the blackbox config because Vaultwarden uses a Tailscale-issued cert that doesn't match the LAN IP. For internal monitoring this is fine.

## Config files

| File | What it controls |
|------|-----------------|
| [`prometheus.yml`](./prometheus/prometheus.yml) | What to scrape, how often, where to send alerts |
| [`alert_rules.yml`](./prometheus/alert_rules.yml) | When to fire alerts, thresholds, descriptions |
| [`alertmanager.yml`](./alertmanager/alertmanager.yml) | PagerDuty routing, grouping logic, repeat intervals |
| [`blackbox.yml`](./blackbox_exporter/blackbox.yml) | HTTP/TCP/ICMP probe definitions |
| [`*.service`](./systemd/) | Systemd units that keep everything running after reboots |

## Dashboards

**Custom — Cluster Overview**
Built from scratch. CPU, RAM, disk as gauges per node. Network traffic over time. Node up/down status. Nothing fancy, but every query is hand-written.

**Imported — Node Exporter Full (ID 1860)**
The standard community dashboard for node_exporter. Way more detailed than my custom one — shows everything from CPU steal time to disk I/O latency. Good for debugging specific issues.

**Imported — Blackbox Exporter (ID 13659)**
Service availability and response times in a clean layout.

I keep both the custom and imported dashboards. The custom one proves I can write PromQL. The imported ones are what I actually look at day-to-day because they're more complete.
