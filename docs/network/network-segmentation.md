# Network Segmentation and VLAN Strategy

## 1. Overview

This document defines the internal network segmentation and VLAN strategy for a single data center in the multi-DC cloud file storage platform. This design is replicated identically across all data centers (NY, Paris, Toronto) to ensure consistent security posture and operational procedures.

### Design Principles
- **Defense in Depth**: Multiple layers of network isolation
- **Least Privilege**: Traffic flows restricted to minimum necessary paths
- **Zero Trust**: No implicit trust between network segments
- **Fault Isolation**: Network failures contained within segments
- **Operational Simplicity**: Clear, maintainable segmentation boundaries

---

## 2. Security Zones

The data center network is divided into four primary security zones, each with distinct security requirements and traffic patterns:

### 2.1 DMZ (Demilitarized Zone)
**Purpose**: Public-facing services exposed to the internet
**Security Level**: Medium (external-facing, but isolated from internal networks)
**Components**: Web frontend servers, load balancers, reverse proxies

### 2.2 Internal Zone
**Purpose**: Application and compute infrastructure
**Security Level**: High (internal only, no direct internet access)
**Components**: Application servers, database servers, internal services

### 2.3 Management Zone
**Purpose**: Infrastructure management and monitoring
**Security Level**: Very High (restricted access, administrative controls)
**Components**: Management servers, monitoring systems, configuration management

### 2.4 Storage Zone
**Purpose**: Storage infrastructure with strict isolation
**Security Level**: Very High (dedicated network, minimal external access)
**Components**: SAN storage systems, iSCSI targets, storage controllers

---

## 3. VLAN Design

### 3.1 VLAN Inventory

| VLAN ID | VLAN Name | Security Zone | Purpose | IP Range | Subnet Mask |
|---------|-----------|---------------|---------|----------|-------------|
| 10 | DMZ-Web | DMZ | Web frontend servers | 10.1.10.0/24 | 255.255.255.0 |
| 20 | DMZ-LB | DMZ | Load balancers | 10.1.20.0/24 | 255.255.255.0 |
| 30 | Internal-App | Internal | Application servers | 10.1.30.0/24 | 255.255.255.0 |
| 40 | Internal-DB | Internal | Database servers | 10.1.40.0/24 | 255.255.255.0 |
| 50 | Internal-Services | Internal | Internal services | 10.1.50.0/24 | 255.255.255.0 |
| 60 | Storage-iSCSI | Storage | iSCSI storage targets | 10.1.60.0/24 | 255.255.255.0 |
| 70 | Storage-SAN | Storage | SAN storage systems | 10.1.70.0/24 | 255.255.255.0 |
| 80 | Storage-Controllers | Storage | Storage controllers | 10.1.80.0/24 | 255.255.255.0 |
| 90 | Management-Servers | Management | Management servers | 10.1.90.0/24 | 255.255.255.0 |
| 100 | Management-Monitoring | Management | Monitoring systems | 10.1.100.0/24 | 255.255.255.0 |
| 110 | Security-SIEM | Management | SIEM systems | 10.1.110.0/24 | 255.255.255.0 |
| 120 | Security-IPS | Management | IPS/IDS systems | 10.1.120.0/24 | 255.255.255.0 |
| 130 | Infrastructure-Network | Management | Network infrastructure | 10.1.130.0/24 | 255.255.255.0 |
| 140 | Infrastructure-Storage | Management | Storage management | 10.1.140.0/24 | 255.255.255.0 |

### 3.2 VLAN Descriptions

#### DMZ Zone VLANs

**VLAN 10 - DMZ-Web**
- **Purpose**: Hosts web frontend servers that receive user requests
- **Components**: HTTP/HTTPS web servers, application frontends
- **Access**: Internet → DMZ-Web (HTTP/HTTPS only)
- **Outbound**: DMZ-Web → DMZ-LB (application requests)

**VLAN 20 - DMZ-LB**
- **Purpose**: Load balancers and reverse proxies
- **Components**: Layer 4/7 load balancers, SSL termination
- **Access**: DMZ-Web → DMZ-LB (forward requests)
- **Outbound**: DMZ-LB → Internal-App (backend requests)

#### Internal Zone VLANs

**VLAN 30 - Internal-App**
- **Purpose**: Application servers processing business logic
- **Components**: Application servers, microservices, API servers
- **Access**: DMZ-LB → Internal-App (HTTP/HTTPS)
- **Outbound**: Internal-App → Internal-DB (database queries), Internal-App → Internal-Services (service calls)

**VLAN 40 - Internal-DB**
- **Purpose**: Database servers storing application data
- **Components**: Database servers (SQL, NoSQL), database clusters
- **Access**: Internal-App → Internal-DB (database protocols only)
- **Outbound**: Internal-DB → Management-Monitoring (metrics), Internal-DB → Security-SIEM (logs)

**VLAN 50 - Internal-Services**
- **Purpose**: Internal supporting services
- **Components**: Authentication services, caching servers, message queues
- **Access**: Internal-App → Internal-Services (service protocols)
- **Outbound**: Internal-Services → Management-Monitoring (metrics)

#### Storage Zone VLANs

**VLAN 60 - Storage-iSCSI**
- **Purpose**: iSCSI storage targets for block storage
- **Components**: iSCSI target servers, storage arrays with iSCSI interfaces
- **Access**: Internal-App → Storage-iSCSI (iSCSI protocol only, port 3260)
- **Outbound**: Storage-iSCSI → Management-Monitoring (health checks), Storage-iSCSI → Infrastructure-Storage (management)

**VLAN 70 - Storage-SAN**
- **Purpose**: SAN storage systems (Fibre Channel over Ethernet or native SAN)
- **Components**: SAN arrays, storage controllers, FC switches
- **Access**: Storage-Controllers → Storage-SAN (SAN protocols only)
- **Outbound**: Storage-SAN → Management-Monitoring (health checks)

**VLAN 80 - Storage-Controllers**
- **Purpose**: Storage controllers and management interfaces
- **Components**: Storage controller nodes, storage management servers
- **Access**: Infrastructure-Storage → Storage-Controllers (management protocols)
- **Outbound**: Storage-Controllers → Storage-SAN (SAN commands), Storage-Controllers → Storage-iSCSI (iSCSI management)

#### Management Zone VLANs

**VLAN 90 - Management-Servers**
- **Purpose**: Infrastructure management servers
- **Components**: Configuration management (Ansible, Puppet), orchestration platforms, jump hosts
- **Access**: Admin networks → Management-Servers (SSH, management protocols)
- **Outbound**: Management-Servers → All zones (management access, restricted by firewall rules)

**VLAN 100 - Management-Monitoring**
- **Purpose**: Monitoring and observability systems
- **Components**: Monitoring servers (Prometheus, Grafana), log aggregators, metrics collectors
- **Access**: All zones → Management-Monitoring (metrics/logs push, SNMP)
- **Outbound**: Management-Monitoring → Management-Servers (alerting), Management-Monitoring → Security-SIEM (correlated events)

**VLAN 110 - Security-SIEM**
- **Purpose**: Security Information and Event Management
- **Components**: SIEM servers, log analysis systems, security analytics
- **Access**: All zones → Security-SIEM (security logs, syslog)
- **Outbound**: Security-SIEM → Management-Servers (alerting), Security-SIEM → Security-IPS (threat intelligence)

**VLAN 120 - Security-IPS**
- **Purpose**: Intrusion Prevention/Detection Systems
- **Components**: IPS/IDS sensors, network security appliances
- **Access**: Security-IPS → All zones (monitoring, inline protection)
- **Outbound**: Security-IPS → Security-SIEM (threat events), Security-IPS → Management-Monitoring (metrics)

**VLAN 130 - Infrastructure-Network**
- **Purpose**: Network infrastructure management
- **Components**: Network management systems, router/switch management interfaces
- **Access**: Management-Servers → Infrastructure-Network (SNMP, SSH, management protocols)
- **Outbound**: Infrastructure-Network → Management-Monitoring (network metrics)

**VLAN 140 - Infrastructure-Storage**
- **Purpose**: Storage infrastructure management
- **Components**: Storage management systems, SAN management interfaces
- **Access**: Management-Servers → Infrastructure-Storage (storage management protocols)
- **Outbound**: Infrastructure-Storage → Storage-Controllers (management commands), Infrastructure-Storage → Management-Monitoring (storage metrics)

---

## 4. Traffic Flow Rules

### 4.1 Zone-to-Zone Traffic Matrix

| Source Zone | Destination Zone | Allowed Traffic | Protocols | Direction |
|-------------|------------------|-----------------|-----------|-----------|
| Internet | DMZ | HTTP (80), HTTPS (443) | TCP | Inbound |
| DMZ | Internal | HTTP (80), HTTPS (443) | TCP | Outbound |
| Internal | Storage | iSCSI (3260) | TCP | Outbound |
| Internal | Management | Metrics, logs | TCP/UDP | Outbound |
| Management | All Zones | Management protocols | SSH, SNMP, etc. | Outbound |
| Storage | Management | Health checks, metrics | TCP/UDP | Outbound |
| Security-IPS | All Zones | Monitoring, inline protection | Various | Bidirectional |
| All Zones | Security-SIEM | Security logs | Syslog (514), TCP | Outbound |
| All Zones | Management-Monitoring | Metrics, logs | SNMP, HTTP, TCP | Outbound |

### 4.2 Detailed Traffic Flows

#### User Request Flow
```
Internet → DMZ-Web (VLAN 10) → DMZ-LB (VLAN 20) → Internal-App (VLAN 30) → Internal-DB (VLAN 40)
```

**Rules**:
- Internet to DMZ-Web: Allow HTTP/HTTPS only
- DMZ-Web to DMZ-LB: Allow HTTP/HTTPS only
- DMZ-LB to Internal-App: Allow HTTP/HTTPS only
- Internal-App to Internal-DB: Allow database protocols only (MySQL: 3306, PostgreSQL: 5432, etc.)
- No direct internet access from Internal or Storage zones

#### Storage Access Flow
```
Internal-App (VLAN 30) → Storage-iSCSI (VLAN 60) → Storage-SAN (VLAN 70)
```

**Rules**:
- Internal-App to Storage-iSCSI: Allow iSCSI protocol only (TCP 3260)
- Storage-Controllers to Storage-SAN: Allow SAN protocols only
- No direct access from DMZ to Storage zones
- Storage zones cannot initiate connections to Internal zones

#### Management Access Flow
```
Management-Servers (VLAN 90) → All Zones (SSH, SNMP, management protocols)
```

**Rules**:
- Management-Servers can access all zones for administrative purposes
- Access restricted by firewall rules (specific protocols and ports)
- All management access logged and monitored
- No reverse access from production zones to Management-Servers (except responses)

#### Monitoring and Security Flow
```
All Zones → Management-Monitoring (VLAN 100) [Metrics]
All Zones → Security-SIEM (VLAN 110) [Logs]
Security-IPS (VLAN 120) → All Zones [Monitoring/Protection]
```

**Rules**:
- All zones can push metrics to Management-Monitoring
- All zones can send logs to Security-SIEM
- Security-IPS can monitor and protect all zones
- Management-Monitoring and Security-SIEM cannot initiate connections to production zones (pull-only where applicable)

### 4.3 Denied Traffic Patterns

**Explicitly Denied**:
- DMZ → Storage zones (no direct storage access from DMZ)
- Storage → Internal zones (storage cannot initiate connections to applications)
- Internal → Internet (no direct internet access from internal zones)
- Production zones → Management-Servers (no reverse access, except responses)
- Storage zones → DMZ (no storage access from DMZ)

---

## 5. IP Addressing Scheme

### 5.1 IP Range Allocation

Each VLAN uses a /24 subnet (256 addresses) with the following allocation:

**Base Network**: `10.1.0.0/16` (reserved for data center internal networks)

**VLAN IP Ranges**:
- **DMZ Zone**: `10.1.10.0/24` - `10.1.29.255` (20 subnets reserved)
- **Internal Zone**: `10.1.30.0/24` - `10.1.59.255` (30 subnets reserved)
- **Storage Zone**: `10.1.60.0/24` - `10.1.89.255` (30 subnets reserved)
- **Management Zone**: `10.1.90.0/24` - `10.1.159.255` (70 subnets reserved)

### 5.2 IP Address Allocation Within VLANs

**Standard Allocation Pattern** (per /24 subnet):
- **.1 - .10**: Network infrastructure (gateways, switches)
- **.11 - .50**: Server addresses (production)
- **.51 - .100**: Server addresses (expansion)
- **.101 - .200**: Reserved for future use
- **.201 - .254**: Management interfaces, out-of-band management

**Example - VLAN 30 (Internal-App)**:
- `10.1.30.1`: Default gateway
- `10.1.30.11-50`: Application servers
- `10.1.30.51-100`: Reserved for expansion
- `10.1.30.201-254`: Management interfaces

### 5.3 Gateway Addressing

Each VLAN has a single default gateway:
- Gateway IP: First usable address in subnet (e.g., `10.1.30.1`)
- Gateway redundancy: VRRP/HSRP for high availability
- Virtual gateway IP: Same as physical gateway (VRRP virtual IP)

---

## 6. Security Benefits

### 6.1 Security Improvements

**Lateral Movement Prevention**:
- Compromised DMZ systems cannot directly access Internal or Storage zones
- Storage systems isolated from application compromise
- Management systems protected from production network threats

**Attack Surface Reduction**:
- Only DMZ exposed to internet (minimal attack surface)
- Internal and Storage zones have no internet connectivity
- Management zone accessible only from trusted administrative networks

**Traffic Inspection**:
- All inter-zone traffic passes through firewalls for inspection
- Security-IPS can monitor and block malicious traffic
- SIEM collects security events from all zones for correlation

**Compliance and Audit**:
- Clear network boundaries enable compliance with security frameworks
- Traffic flows documented and auditable
- Logging and monitoring comprehensive across all zones

### 6.2 Availability Improvements

**Fault Isolation**:
- Network issues in one VLAN do not propagate to others
- Broadcast storms contained within individual VLANs
- Storage network failures isolated from application networks

**Load Distribution**:
- Traffic segmented reduces contention on shared links
- Storage traffic isolated from application traffic (no interference)
- Management traffic separated from production traffic

**Redundancy**:
- Each zone can have independent redundancy mechanisms
- Storage network redundancy independent of application network
- Management network redundancy ensures operational continuity

### 6.3 Operational Benefits

**Simplified Troubleshooting**:
- Clear network boundaries make issue isolation straightforward
- VLAN-based segmentation provides logical separation
- Traffic flow rules documented and enforceable

**Scalability**:
- Each VLAN can be scaled independently
- New services can be added to appropriate zones without affecting others
- IP addressing scheme allows for growth within each zone

**Change Management**:
- Changes to one zone do not require changes to others
- Network policies can be updated per zone
- Testing and validation isolated to affected zones

---

## 7. Implementation Considerations

### 7.1 Layer 2/Layer 3 Boundaries

**VLAN Trunking**:
- Trunk ports between switches carry all VLANs
- 802.1Q tagging used for VLAN identification
- Trunk links between core and access switches

**Inter-VLAN Routing**:
- Layer 3 switches or routers perform inter-VLAN routing
- Default gateways for each VLAN on Layer 3 devices
- Routing tables configured to enforce traffic flow rules

**Access Ports**:
- Server connections use access ports (single VLAN)
- No trunking on server ports (security best practice)
- VLAN assignment static (not dynamic)

### 7.2 Firewall Placement

**Zone Firewalls**:
- Firewalls between security zones (DMZ ↔ Internal, Internal ↔ Storage, etc.)
- Stateful inspection of all inter-zone traffic
- Default deny policy with explicit allow rules

**Inline Security**:
- Security-IPS deployed inline at zone boundaries
- Traffic inspection and threat prevention
- Fail-open or fail-close configuration based on zone criticality

### 7.3 High Availability

**Network Redundancy**:
- Redundant switches in each zone (active-standby or active-active)
- Redundant firewalls (active-standby)
- Redundant gateways (VRRP/HSRP)

**Link Redundancy**:
- Multiple physical links between switches (LACP/EtherChannel)
- Redundant paths between zones
- Spanning tree protocol (STP) or similar for loop prevention

---

## 8. Replication Across Data Centers

### 8.1 Identical Design

This network segmentation design is replicated identically across all data centers:
- **NY Data Center**: Same VLAN IDs, same IP ranges (different base network)
- **Paris Data Center**: Same VLAN IDs, same IP ranges (different base network)
- **Toronto Data Center**: Same VLAN IDs, same IP ranges (different base network)

### 8.2 IP Address Space Separation

**NY Data Center**: `10.1.0.0/16`
**Paris Data Center**: `10.2.0.0/16`
**Toronto Data Center**: `10.3.0.0/16`

This ensures:
- No IP address conflicts between data centers
- Clear identification of traffic source by IP range
- Simplified routing and firewall rules for inter-DC traffic

### 8.3 Inter-DC Traffic Considerations

Inter-DC traffic (as defined in inter-dc-architecture.md) uses dedicated inter-DC links and does not traverse the internal VLAN segmentation. Inter-DC replication traffic is handled at the network edge and does not require access to internal VLANs from remote data centers.

---

## 9. Summary

### Key Design Elements

1. **Four Security Zones**: DMZ, Internal, Management, Storage
2. **14 VLANs**: Logical segmentation within and across zones
3. **Strict Traffic Controls**: Explicit allow rules, default deny
4. **Isolated Storage Network**: SAN/iSCSI completely isolated from other zones
5. **Protected Management**: Management zone with restricted access
6. **Comprehensive Monitoring**: Security and monitoring systems in dedicated VLANs

### Benefits Achieved

- **Security**: Lateral movement prevention, attack surface reduction
- **Availability**: Fault isolation, independent redundancy
- **Operational**: Simplified troubleshooting, scalable design
- **Compliance**: Clear boundaries, auditable traffic flows

### Next Steps

- Implement VLANs on network switches
- Configure firewall rules between zones
- Deploy security-IPS at zone boundaries
- Configure SIEM and monitoring systems
- Document operational procedures for each zone

---

**Document Version**: 1.0  
**Last Updated**: [Date]  
**Next Review**: Quarterly

