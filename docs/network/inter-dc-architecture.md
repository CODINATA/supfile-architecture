# Inter-Data Center Network Architecture

## 1. Logical Inter-DC Topology

### Topology Overview
```
NY (Primary) <----> Paris (Primary)
     |                    |
     |                    |
     v                    v
     Toronto (Backup/DR)
```

### Connection Matrix
- **NY ↔ Paris**: Full bidirectional connectivity (Active/Active)
- **NY → Toronto**: Primary backup path (unidirectional data flow, bidirectional control)
- **Paris → Toronto**: Secondary backup path (unidirectional data flow, bidirectional control)
- **Toronto → NY/Paris**: Disaster recovery read-only access

### Rationale
- **Hub-and-spoke pattern with partial mesh**: NY-Paris mesh for Active/Active; Toronto as spoke to both primaries
- **Avoids full-mesh**: No direct NY-Toronto-Paris triangle required since Toronto is backup-only
- **Cost optimization**: Toronto's limited role justifies asymmetrical connectivity

---

## 2. Link Types and Connectivity

### NY ↔ Paris Connection
**Type**: Hybrid connectivity (primary + backup paths)

**Primary Path**:
- **Dedicated leased line** (MPLS or private WAN)
  - Guaranteed bandwidth: 1-10 Gbps (depending on replication requirements)
  - SLA: 99.9% uptime, <100ms latency
  - Use case: Near-real-time replication for lightweight metadata, bulk data replication

**Backup Path**:
- **IPsec VPN over Internet** (managed by enterprise ISP)
  - Bandwidth: 500 Mbps - 1 Gbps
  - Use case: Failover for primary link, non-critical traffic

**Encryption**:
- Primary: IPsec (Layer 3) on leased line (AES-256-GCM, IKEv2); MACsec (Layer 2) optional if supported end-to-end
- Backup: IPsec VPN tunnels (AES-256-GCM, IKEv2)

### NY → Toronto Connection
**Type**: IPsec VPN over Internet (managed service)

- **Bandwidth**: 1-2 Gbps
- **Latency**: ~15-20ms (within North America)
- **Encryption**: IPsec VPN (AES-256-GCM)
- **Use case**: Daily backup replication (asynchronous, scheduled batches)

### Paris → Toronto Connection
**Type**: IPsec VPN over Internet (managed service)

- **Bandwidth**: 1-2 Gbps
- **Latency**: ~80-100ms (transatlantic)
- **Encryption**: IPsec VPN (AES-256-GCM)
- **Use case**: Redundant backup path, DR access

### Rationale
- **Leased line for NY-Paris**: Justified by Active/Active requirements and near-real-time metadata replication needs
- **VPN for Toronto links**: Sufficient for backup traffic; avoids expensive dedicated circuits
- **Hybrid approach**: Balances performance (NY-Paris) with cost (Toronto links)

---

## 3. Active/Active Architecture: NY and Paris

### Justification

**Geographic Distribution**:
- Reduces latency for end users in North America (NY) and Europe (Paris)
- Local traffic stays local; cross-region only for replication

**High Availability**:
- Automatic failover without service interruption
- No single point of failure for primary operations
- Near-real-time replication for metadata enables low RPO

**Load Distribution**:
- Users routed to nearest DC via DNS/anycast
- Balanced read/write operations across both DCs
- Reduces network transit costs (regional traffic)

**Replication Strategy**:
- **Semi-synchronous or near-real-time replication** for lightweight metadata (via leased line, accounting for transatlantic latency)
- **Asynchronous replication** for bulk storage data
- Note: Conflict resolution is handled at the application layer, outside network scope

**Network Requirements**:
- Low latency between NY-Paris (<100ms target) for near-real-time metadata replication
- High bandwidth for replication traffic
- Bidirectional, symmetrical connectivity

**Failure Tolerance**:
- If NY fails: Paris continues serving; Toronto provides backup restore
- If Paris fails: NY continues serving; Toronto provides backup restore
- If link fails: Primary leased line fails over to VPN backup; operations continue in degraded mode

---

## 4. Toronto's Limited Role

### Design Rationale

**Purpose**: Backup and Disaster Recovery only
- Not part of Active/Active production traffic
- No user-facing services
- No bidirectional replication with primaries

**Connectivity Pattern**:
- **Unidirectional data flow**: NY → Toronto and Paris → Toronto (backup writes)
- **Bidirectional control**: Management, monitoring, DR orchestration
- **No Toronto → NY/Paris production traffic**: Avoids unnecessary bandwidth costs

**Why Limited Connectivity**:
- **Cost efficiency**: Backup traffic is bursty and scheduled; VPN is sufficient
- **Latency tolerance**: Backups can tolerate higher latency (async operations)
- **Geographic isolation**: Ensures DR site is independent of primary failures
- **Reduced complexity**: No need for real-time replication or routing policies

**Toronto Operations**:
- Daily incremental backups from both primaries (scheduled windows)
- Full backup snapshots (weekly/monthly)
- DR testing and validation
- Cold standby with warm restore capabilities

**Network Bandwidth Allocation**:
- Peak usage: 1-2 Gbps during backup windows (off-peak hours)
- Baseline: Minimal (monitoring only)
- No need for dedicated leased lines

---

## 5. Failure Scenarios

### Scenario 1: NY-Paris Primary Leased Line Failure

**Impact**:
- Near-real-time metadata replication degrades to VPN backup path
- Latency increases (~20-50ms additional)
- Possible brief service interruption during failover (<30 seconds)

**Recovery**:
- Automatic failover to IPsec VPN backup path
- Operations continue with asynchronous replication mode
- Leased line auto-restoration or manual intervention

**Network Behavior**:
- Routing tables update via BGP/static routes
- IPsec tunnels remain active (always-on)
- Bandwidth throttling may occur if VPN capacity is lower

---

### Scenario 2: NY Data Center Failure

**Impact**:
- NY users automatically routed to Paris (DNS failover, anycast)
- Paris becomes sole primary DC
- Toronto backup remains available

**Recovery**:
- Users reconnect to Paris (automatic via DNS/load balancer)
- Read operations continue from Paris
- Write operations continue in Paris
- NY restoration from Paris or Toronto backup

**Network Behavior**:
- Paris-Toronto link continues to function (backup path intact)
- No impact on Toronto backup operations
- No routing changes needed (NY routes marked down)

---

### Scenario 3: Paris Data Center Failure

**Impact**:
- Paris users automatically routed to NY (DNS failover)
- NY becomes sole primary DC
- Toronto backup remains available

**Recovery**:
- Users reconnect to NY (automatic)
- Operations continue from NY
- Paris restoration from NY or Toronto backup

**Network Behavior**:
- NY-Toronto link continues to function
- No impact on backup operations
- Transatlantic latency increases for European users (acceptable during DR)

---

### Scenario 4: Toronto Link Failure (NY → Toronto)

**Impact**:
- NY backup operations to Toronto interrupted
- Paris → Toronto link remains operational

**Recovery**:
- Backup traffic rerouted: NY → Paris → Toronto (indirect path)
- Or: Backup operations delayed until link restoration
- No impact on production operations

**Network Behavior**:
- Backup jobs queued or redirected
- Production traffic unaffected
- Manual intervention may be required

---

### Scenario 5: Toronto Link Failure (Paris → Toronto)

**Impact**:
- Paris backup operations to Toronto interrupted
- NY → Toronto link remains operational

**Recovery**:
- Backup traffic rerouted: Paris → NY → Toronto (indirect path)
- Or: Backup operations delayed
- No impact on production operations

**Network Behavior**:
- Similar to Scenario 4
- Backup jobs handled via alternative path or queued

---

### Scenario 6: Complete Network Partition (NY isolated from Paris)

**Impact**:
- Both NY and Paris continue operating independently
- Near-real-time metadata replication impossible
- Split-brain risk for conflicting writes

**Recovery**:
- Conflict resolution handled at application layer (outside network scope)
- Manual intervention to determine authoritative state
- One DC may be designated as temporary primary

**Network Behavior**:
- Both DCs continue serving users
- Replication paused until connectivity restored
- Both DCs maintain separate backup streams to Toronto

---

### Scenario 7: Toronto Data Center Failure

**Impact**:
- Backup operations suspended
- DR capability temporarily unavailable
- No impact on production (NY and Paris)

**Recovery**:
- Production operations continue normally
- Toronto restoration from either NY or Paris backup
- Alternative DR site may be provisioned if extended outage

**Network Behavior**:
- NY-Paris link unaffected
- Backup jobs fail or queue
- No routing changes needed

---

## Summary

### Key Design Principles
- **Cost-optimized**: Leased line only for critical NY-Paris path; VPN for backup links
- **Availability-focused**: Active/Active between primaries; isolated DR site
- **Security**: All inter-DC traffic encrypted (IPsec primary, MACsec optional if supported)
- **Simplicity**: Hub-and-spoke avoids unnecessary full-mesh complexity
- **Scalability**: Bandwidth can be upgraded without topology changes

### Network Capacity Planning
- NY-Paris: 1-10 Gbps (dedicated) + 500 Mbps-1 Gbps (VPN backup)
- NY-Toronto: 1-2 Gbps (VPN)
- Paris-Toronto: 1-2 Gbps (VPN)

### Monitoring and Management
- Network monitoring: Link health, latency, packet loss, jitter
- Backup job monitoring: Success/failure rates, transfer speeds
- DR readiness: Regular failover tests, backup verification

