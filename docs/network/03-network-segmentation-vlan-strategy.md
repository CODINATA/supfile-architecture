# Network Segmentation and VLAN Strategy

**Document Version**: 1.1
**Project**: SUPFile — Multi-Site Cloud Storage Infrastructure
**Scope**: Internal network segmentation and VLAN design for a single data center (replicated across NY, Paris, Toronto)
**Last Updated**: April 2026

---

## 1. Overview

This document defines the internal network segmentation for a single data center. The design is replicated identically across all three DCs (New York, Paris, Toronto) with only the IP supernet changing per site. Consistent segmentation ensures uniform security posture, simplified operations, and predictable inter-DC routing.

### Design Principles

| Principle | Implementation |
|-----------|---------------|
| Defense in Depth | Four security zones with firewalls between each boundary |
| Least Privilege | Traffic flows restricted to minimum necessary paths; default-deny everywhere |
| Zero Trust | No implicit trust between segments; all inter-zone traffic inspected |
| Fault Isolation | Network failures contained within VLANs; broadcast domains limited |
| Operational Simplicity | Consistent VLAN IDs across all DCs; clear naming convention |

---

## 2. Security Zones

The data center is divided into four primary security zones, ordered from least trusted (Internet-facing) to most trusted (management).

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│  DMZ Zone (VLAN 10–20)                  │  ← Public-facing: LB, reverse proxy, WAF
│  Security Level: MEDIUM                 │
└─────────────┬───────────────────────────┘
              │ Firewall (stateful inspection)
              ▼
┌─────────────────────────────────────────┐
│  Internal Zone (VLAN 30–50)             │  ← App servers, DB, internal services
│  Security Level: HIGH                   │
└─────────────┬───────────────────────────┘
              │ Firewall (strict protocol filtering)
              ▼
┌─────────────────────────────────────────┐
│  Storage Zone (VLAN 60–80)              │  ← iSCSI SAN, storage controllers
│  Security Level: VERY HIGH              │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Management Zone (VLAN 90–140)          │  ← SIEM, monitoring, bastion, OOB
│  Security Level: VERY HIGH              │  ← Admin access to all zones
│  (Isolated, admin-only access)          │
└─────────────────────────────────────────┘
```

| Zone | Purpose | Security Level | Internet Access |
|------|---------|---------------|-----------------|
| DMZ | Public-facing services (load balancers, reverse proxy, WAF) | Medium | Inbound HTTPS only |
| Internal | Application servers, database servers, internal services | High | None |
| Storage | iSCSI SAN targets, storage controllers, SAN fabric | Very High | None |
| Management | SIEM, monitoring, bastion host, OOB management | Very High | None (admin VPN only) |

---

## 3. VLAN Design

### 3.1 VLAN Inventory

| VLAN ID | Name | Zone | Purpose | Subnet (NY: 10.1.x.x) | Hosts |
|---------|------|------|---------|----------------------|-------|
| 10 | DMZ-LB | DMZ | Load balancers, reverse proxy (HAProxy), SSL termination | 10.1.10.0/24 | HAProxy active + standby, VIP |
| 20 | DMZ-WAF | DMZ | WAF (ModSecurity) and IPS inline sensor (Suricata) | 10.1.20.0/24 | ModSecurity, Suricata |
| 30 | INT-App | Internal | Web/Application servers (Active/Passive cluster) | 10.1.30.0/24 | App active + passive, Pacemaker VIP |
| 40 | INT-DB | Internal | Database servers (PostgreSQL primary + hot standby) | 10.1.40.0/24 | PostgreSQL nodes, Patroni VIP |
| 50 | INT-Services | Internal | Internal services (Redis cache, RabbitMQ, auth service) | 10.1.50.0/24 | Cache, queue, auth servers |
| 60 | STO-iSCSI | Storage | iSCSI data network (target ↔ initiator traffic) | 10.1.60.0/24 | 3x SAN nodes (iSCSI targets) |
| 70 | STO-SAN | Storage | SAN replication and heartbeat between storage nodes | 10.1.70.0/24 | DRBD replication, heartbeat |
| 80 | STO-Controllers | Storage | Storage management and controller interfaces | 10.1.80.0/24 | Storage management servers |
| 90 | MGT-Servers | Management | Configuration management (Ansible), jump host, orchestration | 10.1.90.0/24 | Bastion host, Ansible server |
| 100 | MGT-Monitoring | Management | Monitoring (Zabbix, Grafana, Prometheus) | 10.1.100.0/24 | Zabbix, Grafana |
| 110 | SEC-SIEM | Management | SIEM (Wazuh manager + Elasticsearch) and log aggregation | 10.1.110.0/24 | Wazuh, ELK/Graylog |
| 120 | SEC-IPS | Management | IPS/IDS management interfaces and threat intelligence feeds | 10.1.120.0/24 | Suricata management, rule updates |
| 130 | MGT-Network | Management | Network device management (switch/router MGMT interfaces) | 10.1.130.0/24 | Switch MGMT, router MGMT |
| 140 | MGT-OOB | Management | Out-of-band management (IPMI/iLO/iDRAC) for STONITH fencing | 10.1.140.0/24 | Server BMC interfaces |

### 3.2 VLAN Details by Zone

#### DMZ Zone

**VLAN 10 — DMZ-LB (Load Balancers)**
- Hosts: HAProxy primary (10.1.10.11), HAProxy standby (10.1.10.12), VIP (10.1.10.10)
- Role: SSL/TLS termination, Layer 7 load balancing, health checks on backend app servers
- Inbound: Internet → VLAN 10 (HTTPS/443 only, via perimeter firewall)
- Outbound: VLAN 10 → VLAN 20 (to WAF) → VLAN 30 (to app servers)
- HA: Keepalived with floating VIP; automatic failover

**VLAN 20 — DMZ-WAF (Web Application Firewall + IPS)**
- Hosts: ModSecurity + OWASP CRS (10.1.20.11), Suricata IPS inline sensor (10.1.20.12)
- Role: HTTP request inspection, OWASP Top 10 protection, intrusion prevention
- Inbound: VLAN 10 → VLAN 20 (HTTP/HTTPS for inspection)
- Outbound: VLAN 20 → VLAN 30 (clean traffic to app servers)
- Note: Suricata operates in inline mode (not just detection) for active threat blocking

#### Internal Zone

**VLAN 30 — INT-App (Application Servers)**
- Hosts: App server ACTIVE (10.1.30.11), App server PASSIVE (10.1.30.12), Pacemaker VIP (10.1.30.10)
- Role: Nginx + application backend (REST API), file management logic
- Inbound: VLAN 20 → VLAN 30 (HTTP/HTTPS from WAF/LB)
- Outbound: VLAN 30 → VLAN 40 (PostgreSQL, port 5432), VLAN 30 → VLAN 50 (Redis, RabbitMQ), VLAN 30 → VLAN 60 (iSCSI, port 3260)
- HA: Pacemaker/Corosync Active/Passive cluster; STONITH via IPMI (VLAN 140)
- Failover: < 30 seconds; Pacemaker monitors Nginx + app process health

**VLAN 40 — INT-DB (Database Servers)**
- Hosts: PostgreSQL primary (10.1.40.11), PostgreSQL hot standby (10.1.40.12), Patroni VIP (10.1.40.10)
- Role: User data, file metadata, shared links, account information
- Inbound: VLAN 30 → VLAN 40 (PostgreSQL port 5432 only)
- Outbound: VLAN 40 → VLAN 100 (metrics to Zabbix), VLAN 40 → VLAN 110 (logs to SIEM)
- HA: Patroni + etcd for automatic PostgreSQL failover
- Replication: Streaming replication to standby; async replication to remote DC

**VLAN 50 — INT-Services (Internal Supporting Services)**
- Hosts: Redis cache (10.1.50.11), RabbitMQ (10.1.50.12), Authentication service (10.1.50.13)
- Role: Session caching, async job queue (ZIP generation, thumbnail creation), JWT token validation
- Inbound: VLAN 30 → VLAN 50 (Redis 6379, RabbitMQ 5672, Auth API)
- Outbound: VLAN 50 → VLAN 100 (metrics), VLAN 50 → VLAN 110 (logs)

#### Storage Zone

**VLAN 60 — STO-iSCSI (iSCSI Data Network)**
- Hosts: SAN Node 1 (10.1.60.11), SAN Node 2 (10.1.60.12), SAN Node 3 (10.1.60.13)
- Role: iSCSI targets (LIO/targetcli) serving block storage to application servers
- Inbound: VLAN 30 → VLAN 60 (iSCSI TCP 3260 only)
- Outbound: VLAN 60 → VLAN 100 (SMART alerts, health metrics), VLAN 60 → VLAN 110 (security logs)
- Configuration: Multipath I/O (dm-multipath) for redundant paths; RAID 6 on each node (12× 4TB disks); hot-swappable disks
- Note: This is a **dedicated network** — no non-iSCSI traffic permitted. Jumbo frames (MTU 9000) recommended for storage performance.

**VLAN 70 — STO-SAN (Storage Replication & Heartbeat)**
- Hosts: DRBD replication endpoints on each SAN node
- Role: Block-level replication between local SAN nodes (intra-DC) and to remote DC (inter-DC via VPN)
- Inbound: VLAN 60 nodes only (replication traffic)
- Outbound: VLAN 70 → VPN gateway (inter-DC DRBD async replication)
- Note: Heartbeat traffic between storage nodes also runs on this VLAN to detect node failures

**VLAN 80 — STO-Controllers (Storage Management)**
- Hosts: Storage management server (10.1.80.11)
- Role: LUN provisioning, quota management, user-to-server allocation ("least used server" logic)
- Inbound: VLAN 90 → VLAN 80 (admin management only)
- Outbound: VLAN 80 → VLAN 60 (storage commands), VLAN 80 → VLAN 70 (replication management)

#### Management Zone

**VLAN 90 — MGT-Servers (Infrastructure Management)**
- Hosts: Bastion/jump host (10.1.90.11), Ansible server (10.1.90.12)
- Role: Single entry point for all administrative access; configuration management
- Inbound: Admin VPN → VLAN 90 (SSH key-only, 2FA enforced)
- Outbound: VLAN 90 → all zones (SSH, management protocols — firewall-filtered per destination)
- Security: Session recording (asciinema); SSH keys rotated quarterly; no direct root access

**VLAN 100 — MGT-Monitoring (Monitoring Systems)**
- Hosts: Zabbix server (10.1.100.11), Grafana (10.1.100.12), Prometheus (10.1.100.13)
- Role: Infrastructure monitoring, performance dashboards, alerting
- Inbound: All zones → VLAN 100 (SNMP, Prometheus metrics push/scrape, HTTP)
- Outbound: VLAN 100 → VLAN 90 (alert notifications), VLAN 100 → VLAN 110 (correlated events to SIEM)

**VLAN 110 — SEC-SIEM (Security Information & Event Management)**
- Hosts: Wazuh manager (10.1.110.11), Elasticsearch (10.1.110.12), Graylog/Kibana (10.1.110.13)
- Role: Security event correlation, log aggregation, threat detection, compliance audit
- Inbound: All zones → VLAN 110 (syslog UDP/TCP 514, Wazuh agent 1514/1515)
- Outbound: VLAN 110 → VLAN 120 (threat intelligence to IPS), VLAN 110 → VLAN 90 (alerts)
- Retention: 1 year minimum for compliance

**VLAN 120 — SEC-IPS (IPS/IDS Management)**
- Hosts: Suricata management interface (10.1.120.11), rule update server (10.1.120.12)
- Role: IPS rule management, signature updates (ET Open/Pro), threat intelligence feeds
- Inbound: VLAN 110 → VLAN 120 (threat intelligence)
- Outbound: VLAN 120 → VLAN 20 (push rules to inline Suricata sensor)

**VLAN 130 — MGT-Network (Network Device Management)**
- Hosts: Switch management interfaces, router management interfaces
- Role: Network infrastructure configuration, firmware updates
- Inbound: VLAN 90 → VLAN 130 (SSH, SNMP v3 from bastion only)
- Outbound: VLAN 130 → VLAN 100 (network metrics via SNMP)

**VLAN 140 — MGT-OOB (Out-of-Band Management)**
- Hosts: IPMI/iLO/iDRAC interfaces for all physical servers
- Role: Hardware management, STONITH fencing for Pacemaker, KVM console access, firmware updates
- Inbound: VLAN 90 → VLAN 140 (IPMI from bastion only)
- Outbound: VLAN 140 → VLAN 100 (hardware health alerts)
- **Critical**: Used by Pacemaker for STONITH fencing — if a node becomes unresponsive, Pacemaker uses IPMI to power-cycle it, preventing split-brain

---

## 4. Traffic Flow Rules

### 4.1 Zone-to-Zone Traffic Matrix

| Source → Destination | Allowed | Protocols/Ports | Notes |
|---------------------|---------|----------------|-------|
| Internet → DMZ | ✅ | HTTPS (443) only | Via perimeter firewall |
| DMZ → Internal | ✅ | HTTP/HTTPS (80, 443) | Via WAF inspection first |
| Internal → Storage | ✅ | iSCSI (3260) only | App servers to SAN |
| Internal → Internal | ✅ | DB (5432), Redis (6379), AMQP (5672) | Between app, DB, services |
| All zones → MGT-Monitoring | ✅ | SNMP (161), HTTP (9090/3000), Syslog | Metrics push |
| All zones → SEC-SIEM | ✅ | Syslog (514), Wazuh (1514/1515) | Log shipping |
| MGT-Servers → All zones | ✅ | SSH (22), SNMP v3 | Admin access via bastion |
| SEC-IPS → DMZ | ✅ | Inline monitoring | Suricata inline sensor |
| MGT-OOB → Servers | ✅ | IPMI (623) | STONITH fencing |
| **Internet → Internal** | ❌ | — | **BLOCKED** |
| **Internet → Storage** | ❌ | — | **BLOCKED** |
| **Internet → Management** | ❌ | — | **BLOCKED** |
| **DMZ → Storage** | ❌ | — | **BLOCKED** |
| **DMZ → Management** | ❌ | — | **BLOCKED** |
| **Storage → Internal** | ❌ | — | **BLOCKED** (storage never initiates) |
| **Storage → DMZ** | ❌ | — | **BLOCKED** |
| **Internal → Internet** | ❌ | — | **BLOCKED** (no outbound) |

### 4.2 User Request Flow (Full Path)

```
User (HTTPS) → Perimeter Firewall → VLAN 10 (HAProxy LB)
    → VLAN 20 (ModSecurity WAF + Suricata IPS)
    → VLAN 30 (App Server — Active node via Pacemaker VIP)
    → VLAN 40 (PostgreSQL — via Patroni VIP)
    → VLAN 60 (iSCSI SAN — file retrieval via Multipath I/O)
    → Response back through same path
```

### 4.3 Storage Access Flow

```
App Server (VLAN 30) → iSCSI Initiator → Multipath I/O
    → VLAN 60 (SAN Node — iSCSI Target, port 3260)
    → Local RAID 6 disk array
    → VLAN 70 (DRBD replication to other SAN nodes / remote DC)
```

### 4.4 Monitoring and Security Flow

```
All servers (Wazuh agents) → VLAN 110 (Wazuh Manager → Elasticsearch)
All servers (Zabbix agents) → VLAN 100 (Zabbix Server → Grafana)
VLAN 110 (SIEM) → VLAN 120 (IPS rule updates)
VLAN 140 (IPMI) → Physical servers (STONITH fencing)
```

### 4.5 Backup Flow

```
VLAN 60 (SAN data) → Backup Agent (BorgBackup)
VLAN 40 (PostgreSQL) → pg_dump → Backup Agent
Backup Agent → VPN Gateway → IPsec Tunnel → Toronto DC
```

---

## 5. IP Addressing Scheme

### 5.1 Per-DC Supernet Allocation

| Data Center | Supernet | Example VLAN 30 |
|-------------|----------|-----------------|
| New York | 10.1.0.0/16 | 10.1.30.0/24 |
| Paris | 10.2.0.0/16 | 10.2.30.0/24 |
| Toronto | 10.3.0.0/16 | 10.3.30.0/24 |

This ensures no IP conflicts between DCs and makes source identification trivial from any IP address.

### 5.2 Standard Address Allocation Within Each /24

| Range | Purpose |
|-------|---------|
| .1 | Default gateway (VRRP/HSRP virtual IP) |
| .2–.10 | Network infrastructure (secondary gateways, switches) |
| .10 | Service VIP (Pacemaker, Patroni, HAProxy floating IP) |
| .11–.50 | Production servers |
| .51–.100 | Expansion / future servers |
| .101–.200 | Reserved |
| .201–.254 | Management interfaces, OOB |

### 5.3 Gateway Redundancy

Each VLAN gateway uses VRRP (Virtual Router Redundancy Protocol):
- Virtual IP: x.x.x.1 (always reachable)
- Primary router: x.x.x.2
- Secondary router: x.x.x.3
- Failover: < 3 seconds

---

## 6. Firewall Placement and Rules

### 6.1 Firewall Architecture

```
Internet
    │
    ▼
┌─────────────────┐
│ Perimeter FW    │  ← HA pair (pfSense/OPNsense + CARP)
│ (FW1 + FW2)    │  ← Allows only HTTPS inbound
└────────┬────────┘
         │
    ┌────▼────┐
    │  DMZ    │
    └────┬────┘
         │
┌────────▼────────┐
│ Internal FW     │  ← Filters DMZ → Internal traffic
│ (Zone firewall) │  ← Stateful inspection
└────────┬────────┘
         │
    ┌────▼─────┐
    │ Internal │
    └────┬─────┘
         │
┌────────▼────────┐
│ Storage FW      │  ← iSCSI (3260) only; strictest rules
│ (Zone firewall) │
└────────┬────────┘
         │
    ┌────▼────┐
    │ Storage │
    └─────────┘
```

### 6.2 Perimeter Firewall Rules (Simplified)

| # | Source | Destination | Protocol | Port | Action |
|---|--------|-------------|----------|------|--------|
| 1 | Any (Internet) | VLAN 10 VIP | TCP | 443 | ALLOW |
| 2 | Any (Internet) | VLAN 10 VIP | TCP | 80 | REDIRECT → 443 |
| 3 | Admin VPN | VLAN 90 (Bastion) | TCP | 22 | ALLOW |
| 4 | Any | Any | Any | Any | **DENY** (default) |

### 6.3 Internal Zone Firewall Rules (Simplified)

| # | Source | Destination | Protocol | Port | Action |
|---|--------|-------------|----------|------|--------|
| 1 | VLAN 20 (WAF) | VLAN 30 (App) | TCP | 80, 443 | ALLOW |
| 2 | VLAN 30 (App) | VLAN 40 (DB) | TCP | 5432 | ALLOW |
| 3 | VLAN 30 (App) | VLAN 50 (Services) | TCP | 6379, 5672 | ALLOW |
| 4 | VLAN 30 (App) | VLAN 60 (iSCSI) | TCP | 3260 | ALLOW |
| 5 | Any zone | VLAN 100 (Monitoring) | TCP/UDP | 161, 9090 | ALLOW |
| 6 | Any zone | VLAN 110 (SIEM) | TCP/UDP | 514, 1514 | ALLOW |
| 7 | VLAN 90 (Bastion) | Any zone | TCP | 22 | ALLOW |
| 8 | Any | Any | Any | Any | **DENY** (default) |

---

## 7. Layer 2 Implementation

### 7.1 Switch Configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| VLAN trunking | 802.1Q on all inter-switch links | Carry all VLANs between core/access switches |
| Access ports | Single VLAN per server port | No trunking on server connections (security) |
| Native VLAN | VLAN 999 (unused) | Prevent VLAN hopping attacks |
| STP | RSTP (Rapid Spanning Tree) | Loop prevention with fast convergence |
| Port security | MAC address limiting (2 per port) | Prevent MAC flooding |
| DHCP snooping | Enabled on all access VLANs | Prevent rogue DHCP servers |
| Dynamic ARP inspection | Enabled | Prevent ARP spoofing |

### 7.2 Link Redundancy

- Core-to-access: Dual uplinks with LACP (802.3ad) link aggregation
- Core-to-core: Dual links with LACP
- Core-to-firewall: Dual links with LACP
- STP root: Core switch 1 primary, core switch 2 secondary

### 7.3 Jumbo Frames

Jumbo frames (MTU 9000) enabled **only on storage VLANs** (60, 70, 80) to improve iSCSI throughput. All other VLANs use standard MTU 1500.

---

## 8. Replication Across Data Centers

### 8.1 Identical VLAN Design

All three DCs use the same VLAN IDs, same naming, same security zone boundaries. Only the second octet of the IP address changes:

| VLAN | NY | Paris | Toronto |
|------|-----|-------|---------|
| VLAN 30 (INT-App) | 10.1.30.0/24 | 10.2.30.0/24 | 10.3.30.0/24 |
| VLAN 40 (INT-DB) | 10.1.40.0/24 | 10.2.40.0/24 | 10.3.40.0/24 |
| VLAN 60 (STO-iSCSI) | 10.1.60.0/24 | 10.2.60.0/24 | 10.3.60.0/24 |

### 8.2 Inter-DC Traffic Handling

Inter-DC replication traffic (DRBD, PostgreSQL streaming, backup) is handled at the network edge via encrypted IPsec tunnels (see [02-inter-dc-encryption-vpn-strategy.md](./02-inter-dc-encryption-vpn-strategy.md)). Remote DCs do not have direct access to internal VLANs — all traffic enters/exits through the VPN gateway and is routed by the edge firewall to the appropriate internal VLAN.

### 8.3 Toronto Differences

Toronto uses the same VLAN structure but has reduced population:
- VLAN 30: No active app servers (cold standby only)
- VLAN 40: PostgreSQL delayed replica (24h lag) for DR
- VLAN 60: Backup storage (BorgBackup repository), no production iSCSI targets
- VLANs 90–140: Full management/monitoring stack (required for DR operations)

---

## 9. Security Benefits Summary

| Benefit | How Achieved |
|---------|-------------|
| Lateral movement prevention | Firewalls between every zone; compromised DMZ cannot reach Storage |
| Attack surface reduction | Only DMZ exposed to Internet; Internal/Storage have zero Internet access |
| Storage isolation | Dedicated VLANs with iSCSI-only access; jumbo frames isolated from other traffic |
| Admin access control | Single bastion entry point (VLAN 90); 2FA + SSH keys; session recording |
| Comprehensive logging | All zones ship logs to SIEM (VLAN 110); 1-year retention |
| STONITH fencing | Isolated OOB network (VLAN 140) prevents split-brain in HA clusters |
| VLAN hopping prevention | Native VLAN set to unused VLAN 999; port security enabled |
| ARP/DHCP attacks | Dynamic ARP inspection + DHCP snooping on all access VLANs |

---

## 10. Implementation Checklist

- [ ] Configure VLANs on all switches (core + access)
- [ ] Set up 802.1Q trunks between switches
- [ ] Configure access ports for each server (single VLAN assignment)
- [ ] Set native VLAN to 999 on all trunks
- [ ] Enable RSTP, port security, DHCP snooping, DAI
- [ ] Deploy and configure perimeter firewall HA pair
- [ ] Deploy zone firewalls between DMZ ↔ Internal ↔ Storage
- [ ] Configure all firewall rules (default deny + explicit allows)
- [ ] Enable jumbo frames on storage VLANs only
- [ ] Configure VRRP on all VLAN gateways
- [ ] Deploy LACP on all redundant links
- [ ] Deploy Suricata IPS inline at DMZ boundary
- [ ] Deploy Wazuh agents on all servers → ship to VLAN 110
- [ ] Deploy Zabbix agents on all servers → ship to VLAN 100
- [ ] Verify all denied traffic patterns with packet captures
- [ ] Document operational procedures per zone
- [ ] Replicate configuration to Paris and Toronto DCs

---

**Previous Document**: [02-inter-dc-encryption-vpn-strategy.md](./02-inter-dc-encryption-vpn-strategy.md)
**Related**: [Single DC PlantUML Diagram] — Visual representation of this segmentation
