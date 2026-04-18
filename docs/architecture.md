# Architecture

## Cluster Design

The homelab runs on a 3-node Proxmox VE cluster providing high availability and resource distribution.

### Why 3 nodes?

3 is the minimum number of nodes required for **quorum** — the cluster can survive the loss of 1 node and still make decisions (fence the failed node, relocate services). With 2 nodes, losing 1 means the remaining node doesn't have a majority and the cluster locks up.

### Node Specifications

| Node | CPU | RAM | OS Disk | Extra Storage | Role |
|------|-----|-----|---------|---------------|------|
| pve | Intel i5 | 16GB | 238GB NVMe | 931GB HDD (ZFS) | Primary storage, backup |
| node2 | Intel i5 | 8GB | 119GB NVMe | — | Services |
| node3 | Intel i5 | 8GB | 119GB SSD | — | Workload distribution |


### Networking

All nodes and containers use bridged networking via `vmbr0`, connected to the same physical LAN (192.168.1.0/24). No VLANs are currently configured.

Remote access is handled via Tailscale mesh VPN — no ports are exposed to the public internet.

### High Availability

All critical services are configured in Proxmox HA with:
- Automatic restart on the same node (up to 3 attempts)
- Automatic migration to another node if restart fails

**Limitation:** All storage is local (no Ceph or shared NFS). This means HA failover requires copying the container disk to the target node, which takes minutes rather than seconds. This is acceptable for homelab use.

### Monitoring Architecture

```
Data flow:
node_exporter (port 9100)      → Prometheus (scrapes every 15s)
blackbox_exporter (port 9115)  → Prometheus (scrapes every 15s)
                                    ↓
                                 Prometheus (stores, evaluates rules)
                                    ↓                    ↓
                                 Grafana              Alertmanager
                                 (dashboards)         (notifications)
                                                         ↓
                                                      PagerDuty
```

