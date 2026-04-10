# Inter-Data Center Network Architecture

**Document Version**: 1.1
**Project**: SUPFile — Multi-Site Cloud Storage Infrastructure
**Scope**: Inter-DC connectivity between New York, Paris, and Toronto data centers
**Last Updated**: April 2026

---

## 1. Logical Inter-DC Topology

### 1.1 Topology Overview

```
         ┌──────────────────────────────────────────────┐
         │          GeoDNS / GSLB (PowerDNS)            │
         │        files.supfile.com                      │
         │  Americas → NY  |  EU/Asia/Africa → Paris    │
         └──────────┬───────────────────┬───────────────┘
                    │                   │
        ┌───────────▼──────┐  ┌────────▼───────────┐
        │  NY DC (Primary) │◄─►  Paris DC (Primary) │
        │  Active/Active   │  │  Active/Active      │
        └────────┬─────────┘  └────────┬────────────┘
                 │                     │
                 │   ┌─────────────┐   │
                 └──►│ Toronto DC  │◄──┘
                     │ Backup / DR │
                     └─────────────┘
```

### 1.2 Connection Matrix

| Source | Destination | Direction | Traffic Type |
|--------|-------------|-----------|--------------|
| NY | Paris | Bidirectional | Production replication (DRBD + PostgreSQL), user failover |
| NY | Toronto | Unidirectional (data), Bidirectional (control) | Daily backup, DB dumps, management |
| Paris | Toronto | Unidirectional (data), Bidirectional (control) | Daily backup, DB dumps, management |
| Toronto | NY/Paris | Read-only | Disaster recovery restore only |

### 1.3 Design Rationale

- **Partial mesh with hub-and-spoke**: NY–Paris mesh for Active/Active production; Toronto as spoke receiving backups from both primaries.
- **No full triangle mesh**: Toronto's backup-only role does not justify a dedicated leased line or bidirectional replication.
- **Cost optimization**: Leased line only where Active/Active demands it (NY↔Paris). Internet VPN is sufficient for Toronto's scheduled, asynchronous backup traffic.

---

## 2. Link Types and Connectivity

### 2.1 NY ↔ Paris (Active/Active Primary)

**Primary Path — Dedicated Leased Line (MPLS / Private WAN)**

| Parameter | Value |
|-----------|-------|
| Type | Dedicated leased line (MPLS or private WAN) |
| Bandwidth | 1–10 Gbps (scaled to replication load) |
| Latency | ~70–80 ms (transatlantic) |
| SLA | 99.9% uptime |
| Encryption | IPsec ESP Transport Mode (AES-256-GCM, IKEv2) |
| Use case | DRBD async block replication, PostgreSQL WAL streaming, metadata sync |

**Backup Path — IPsec VPN over Internet**

| Parameter | Value |
|-----------|-------|
| Type | IPsec VPN tunnel over public Internet (managed ISP) |
| Bandwidth | 500 Mbps – 1 Gbps |
| Latency | ~80–90 ms |
| Encryption | IPsec ESP Tunnel Mode (AES-256-GCM, IKEv2) |
| Use case | Automatic failover if leased line fails; non-critical traffic |
| Failover time | < 30 seconds (BFD + DPD detection) |

**Layer 2 Enhancement (Optional)**:
MACsec (IEEE 802.1AE) may be deployed on the leased line for defense-in-depth (L2 encryption under L3 IPsec). This is optional for the POC and not relied upon for baseline security guarantees.

### 2.2 NY → Toronto (Backup Path)

| Parameter | Value |
|-----------|-------|
| Type | IPsec VPN over Internet (managed service) |
| Bandwidth | 1–2 Gbps |
| Latency | ~15–20 ms (intra–North America) |
| Encryption | IPsec ESP Tunnel Mode (AES-256-GCM, IKEv2) |
| Use case | Daily incremental backup (BorgBackup), weekly full backup, pg_basebackup DB dumps |

### 2.3 Paris → Toronto (Backup Path)

| Parameter | Value |
|-----------|-------|
| Type | IPsec VPN over Internet (managed service) |
| Bandwidth | 1–2 Gbps |
| Latency | ~90–100 ms (transatlantic) |
| Encryption | IPsec ESP Tunnel Mode (AES-256-GCM, IKEv2) |
| Use case | Redundant backup path, DR access |

### 2.4 Cost Justification

| Link | Type | Estimated Monthly Cost | Justification |
|------|------|----------------------|---------------|
| NY ↔ Paris Primary | Dedicated leased line 10 Gbps | $8,000–$15,000 | Required for Active/Active near-real-time replication |
| NY ↔ Paris Backup | Internet VPN 1 Gbps | $500–$1,000 | Failover only; commodity Internet sufficient |
| NY → Toronto | Internet VPN 2 Gbps | $400–$800 | Backup traffic only; scheduled off-peak |
| Paris → Toronto | Internet VPN 2 Gbps | $600–$1,200 | Backup traffic only; scheduled off-peak |

> **Total estimated inter-DC connectivity**: $9,500–$18,000/month. No leased line to Toronto saves $5,000–$10,000/month vs. full mesh.

---

## 3. Global Traffic Management (Active/Active)

### 3.1 GeoDNS / GSLB Architecture

| Component | Detail |
|-----------|--------|
| Technology | PowerDNS with GeoIP module (or Cloudflare DNS as managed alternative) |
| Domain | `files.supfile.com` |
| Routing logic | Americas → NY DC public IP; EU/Asia/Africa → Paris DC public IP |
| Health checks | HTTP health probe every 10 seconds against each DC's HAProxy VIP |
| Failover behavior | If one DC fails health check → all traffic shifts to surviving DC |
| DNS TTL | 30 seconds (enables fast failover) |

### 3.2 Why Active/Active

- **Reduced latency**: Users hit the geographically nearest DC.
- **Load distribution**: Read/write balanced across both DCs; no single bottleneck.
- **High availability**: Automatic failover without manual intervention. If NY fails, Paris absorbs 100% of traffic (and vice versa).
- **Compliance**: EU user data can be served from Paris, aiding GDPR considerations.

### 3.3 Replication Strategy Between Primaries

| Data Type | Tool | Mode | RPO |
|-----------|------|------|-----|
| Block storage (user files) | DRBD | Asynchronous | < 30 seconds |
| Database (PostgreSQL) | Streaming replication (Patroni) | Asynchronous | < 5 seconds |
| Metadata/config | rsync or application-level | Periodic | < 1 minute |

> **Note**: Synchronous replication is impractical across transatlantic latency (~75 ms round-trip). Async replication with short intervals provides an acceptable RPO while maintaining write performance.

### 3.4 Conflict Resolution

With Active/Active and async replication, write conflicts are possible. Conflict resolution is handled at the application layer (outside network scope) using:
- User affinity: Each user is assigned a "home DC" for writes; reads can come from either.
- Last-write-wins with vector clocks for file metadata.
- Database conflict resolution via Patroni leader election.

---

## 4. Toronto's Limited Role

### 4.1 Design Rationale

| Aspect | Detail |
|--------|--------|
| Purpose | Backup and Disaster Recovery only |
| User traffic | **None** — Toronto never serves production users |
| Replication direction | Receive-only (NY → Toronto, Paris → Toronto) |
| Activation trigger | Only activated in catastrophic DR scenario (both main DCs down) |
| RTO | 4 hours (cold standby with warm restore) |
| RPO | 24 hours (daily backup cycle) |

### 4.2 Toronto Operations

| Operation | Schedule | Tool | Source |
|-----------|----------|------|--------|
| Incremental file backup | Daily at 02:00 UTC | BorgBackup (deduplicated, encrypted) | NY + Paris |
| Full file backup | Weekly (Sunday 00:00 UTC) | BorgBackup | NY + Paris |
| Database backup | Daily at 03:00 UTC | pg_basebackup + WAL archive | NY (primary PG) |
| Tape archival | Weekly full → LTO-9 tape | GFS rotation (30 daily, 12 weekly, 12 monthly, 5 yearly) | Toronto backup server |
| DR validation test | Quarterly | Manual failover drill | — |

### 4.3 Bandwidth Allocation

| Period | Bandwidth Usage | Traffic Type |
|--------|----------------|--------------|
| Off-peak (backup window) | 1–2 Gbps burst | Backup replication |
| Baseline (outside window) | < 50 Mbps | Monitoring, heartbeat, management |
| DR activation | Full link capacity | Restore traffic |

### 4.4 Why Limited Connectivity is Sufficient

- **Cost efficiency**: Backup traffic is bursty and scheduled; VPN over Internet costs ~90% less than a dedicated line.
- **Latency tolerance**: Async backup operations tolerate 100+ ms latency without issue.
- **Geographic isolation**: Physical separation from NYC and Paris ensures DR site survives regional disasters.
- **Reduced attack surface**: No user-facing services means smaller security footprint.

---

## 5. Failure Scenarios and Recovery

### 5.1 Scenario Matrix

| # | Scenario | Impact | Recovery | RTO | RPO |
|---|----------|--------|----------|-----|-----|
| 1 | NY–Paris primary leased line failure | Replication degrades to VPN backup; +10–20 ms latency | Automatic failover to backup VPN (< 30s); async replication continues | < 30s | < 30s |
| 2 | NY DC complete failure | Americas users rerouted to Paris via GeoDNS | Automatic; Paris becomes sole primary | < 2 min | < 30s |
| 3 | Paris DC complete failure | EU users rerouted to NY via GeoDNS | Automatic; NY becomes sole primary | < 2 min | < 30s |
| 4 | NY → Toronto link failure | NY backup to Toronto interrupted | Reroute via Paris → Toronto; or queue until restored | N/A (backup only) | Up to 48h |
| 5 | Paris → Toronto link failure | Paris backup to Toronto interrupted | Reroute via NY → Toronto; or queue until restored | N/A (backup only) | Up to 48h |
| 6 | Complete NY–Paris network partition | Both DCs operate independently; split-brain risk | Application-layer conflict resolution; manual arbitration | Degraded mode | Varies |
| 7 | Toronto DC failure | Backup/DR capability suspended | Production unaffected; rebuild Toronto from either primary | N/A | N/A |
| 8 | Both NY + Paris failure (catastrophic) | Full outage | Activate Toronto DR; restore from latest backup | 4 hours | 24 hours |

### 5.2 Detailed Recovery Procedures

#### Scenario 1: NY–Paris Leased Line Failure

1. BFD detects link down within 3 seconds.
2. Routing tables update: traffic shifts to IPsec VPN backup tunnel.
3. DRBD and PostgreSQL replication continue over backup path (reduced bandwidth).
4. Alert sent to NOC via SIEM/Zabbix.
5. Leased line provider notified for restoration.
6. Once restored, traffic automatically prefers primary path (lower route metric).

#### Scenario 2: NY DC Complete Failure

1. GeoDNS health check fails for NY (3 consecutive failures at 10s intervals = 30s detection).
2. DNS TTL expires (30s) → all new requests resolve to Paris IP.
3. Paris absorbs all user traffic (capacity must be sized for 100% load).
4. Toronto backup continues from Paris only.
5. NY restoration: rebuild from Paris replication + Toronto backup if needed.

#### Scenario 6: Complete Network Partition

1. Both DCs continue serving their regional users independently.
2. Replication paused — data diverges.
3. Both DCs continue independent backup streams to Toronto.
4. On reconnection: Patroni leader election resolves DB split; DRBD resync resolves storage divergence.
5. Application-level conflict resolution merges diverged file metadata.
6. Post-incident: manual review of conflicting writes.

---

## 6. Network Capacity Planning

### 6.1 Bandwidth Requirements

| Traffic Type | Estimated Volume | Required Bandwidth | Link |
|-------------|-----------------|-------------------|------|
| DRBD storage replication | ~500 GB/day (delta) | ~50 Mbps sustained, bursts to 500 Mbps | NY ↔ Paris |
| PostgreSQL WAL streaming | ~10–50 GB/day | ~5–25 Mbps sustained | NY ↔ Paris |
| User failover (worst case) | Full user traffic of one DC | 1–5 Gbps | NY ↔ Paris |
| Daily backup to Toronto | ~500 GB–1 TB/day | 1–2 Gbps during window | NY/Paris → Toronto |
| Monitoring/management | Negligible | < 10 Mbps | All links |

### 6.2 Scaling Considerations

- NY ↔ Paris leased line can be upgraded from 1 Gbps to 10 Gbps without topology changes.
- Toronto VPN bandwidth can be increased by upgrading ISP tier.
- If user base grows significantly, consider adding a CDN layer for static file delivery to reduce inter-DC replication needs.

---

## 7. Monitoring and Management

### 7.1 Key Metrics

| Category | Metric | Alert Threshold |
|----------|--------|-----------------|
| Link health | Tunnel status (up/down) | Immediate alert on down |
| Latency | Round-trip time per link | NY–Paris > 150 ms; NY–Toronto > 50 ms |
| Packet loss | Per-link loss rate | > 0.5% |
| Bandwidth | Utilization percentage | > 80% sustained for 5 min |
| Replication lag | DRBD sync status | > 60s behind |
| Replication lag | PostgreSQL WAL replay delay | > 30s behind |
| Backup jobs | Success/failure per DC | Any failure |
| Certificate expiry | VPN gateway certs | 30 days before expiry |

### 7.2 Monitoring Tools

| Tool | Purpose |
|------|---------|
| Zabbix | Infrastructure monitoring (link health, bandwidth, latency, server metrics) |
| Grafana | Dashboards and visualization |
| Wazuh (SIEM) | Security event correlation, IKEv2 auth logs, anomaly detection |
| BFD | Bidirectional Forwarding Detection for rapid link failure detection (< 1s) |
| SNMP | Gateway and switch polling |
| NetFlow/sFlow | Traffic analysis and capacity planning |

---

## 8. Summary

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Dedicated leased line for NY–Paris only | Active/Active replication demands low-latency, guaranteed bandwidth |
| Internet VPN for all Toronto links | Backup traffic is scheduled and latency-tolerant; saves $5K–$10K/month |
| Hybrid primary + backup for NY–Paris | Eliminates single point of failure on the most critical link |
| GeoDNS for global load balancing | Routes users to nearest DC; enables automatic failover |
| Async replication everywhere | Transatlantic latency makes synchronous replication impractical |
| Unidirectional Toronto links | Cost efficiency; Toronto only receives, never initiates production traffic |
| 30s DNS TTL | Balances failover speed against DNS query volume |

---

**Next Document**: [02-inter-dc-encryption-vpn-strategy.md](./02-inter-dc-encryption-vpn-strategy.md) — Detailed encryption, key exchange, and VPN configuration.
