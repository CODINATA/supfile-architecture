# Inter-Data Center Encryption and VPN Strategy

## 1. Overview

This document defines the encryption and VPN strategy for secure inter-data center connectivity. All inter-DC communications are encrypted at the network layer to ensure confidentiality, integrity, and authenticity of data in transit.

### Design Principles
- **Defense in Depth**: Multiple layers of encryption where applicable
- **Zero Trust**: No implicit trust between data centers
- **Standards-Based**: Use industry-standard protocols (IPsec, IKEv2)
- **Operational Resilience**: Automatic key rotation, failover-capable tunnels
- **Performance-Aware**: Encryption choices balance security with latency and throughput

---

## 2. Encryption Technology by Connection Type

### 2.1 NY ↔ Paris (Active/Active Primary Path)

**Technology**: **IPsec (ESP) in Transport Mode** over dedicated leased line

**Rationale**:
- Dedicated leased line provides physical isolation; IPsec adds cryptographic protection
- Transport mode preserves original IP headers, reducing overhead for point-to-point links
- Supports near-real-time replication with minimal latency impact
- Enables application-aware routing and QoS preservation

**Layer 2 Encryption (Optional Enhancement)**:
- **MACsec (IEEE 802.1AE)** may be deployed as an additional layer if end-to-end support is available
- Provides hop-by-hop encryption at Layer 2
- Protects against intermediate device compromise on leased line path
- **Note**: MACsec is optional and not relied upon for baseline security guarantees in this POC. IPsec remains the primary protection mechanism. Combined with IPsec: MACsec (L2) + IPsec (L3) provides defense in depth, but IPsec alone satisfies security requirements.

**Backup Path**: IPsec VPN in Tunnel Mode (see section 2.3)

---

### 2.2 NY ↔ Toronto and Paris ↔ Toronto (Backup/DR Links)

**Technology**: **IPsec VPN in Tunnel Mode** over Internet

**Rationale**:
- Internet-based connectivity requires tunnel mode for full encapsulation
- Protects entire IP packet including source/destination addresses
- Standard approach for site-to-site VPN over public networks
- Cost-effective for backup traffic patterns

**Implementation**:
- Each link: One primary IPsec tunnel (always-on)
- Optional: Secondary tunnel for redundancy (backup tunnel on standby)
- Both tunnels terminate at dedicated VPN gateways/firewalls at each DC

---

## 3. Key Exchange and Authentication Mechanisms

### 3.1 IKEv2 Protocol

**Primary Key Exchange Protocol**: **IKEv2 (Internet Key Exchange version 2)**

**Rationale**:
- Industry standard for IPsec key exchange
- Supports EAP for certificate-based authentication
- More efficient than IKEv1 (fewer message exchanges)
- Better NAT traversal support
- Built-in DoS protection (cookie mechanism)

### 3.2 Authentication Methods

#### NY ↔ Paris (Leased Line)
- **X.509 Certificate-based (Primary)**: Mutual certificate authentication for IKEv2 SA establishment
  - Certificates issued by private PKI/CA
  - Subject Alternative Names (SANs) include gateway IP addresses
  - Certificate revocation checked via OCSP/CRL
- **Pre-shared Key (PSK) (Controlled Fallback)**: Retained only as an emergency fallback option if certificate services are unavailable (deployed via secure key management)

#### NY ↔ Toronto and Paris ↔ Toronto (VPN)
- **X.509 Certificate-based (Primary)**: Mutual certificate authentication (same PKI as above)
- **Pre-shared Key (PSK) (Controlled Fallback)**: Retained only as an emergency fallback option if certificate services are unavailable
- Certificates deployed to VPN gateways via secure provisioning process

### 3.3 Key Exchange Parameters

**IKEv2 Proposal Sets**:

**Diffie-Hellman Groups**:
- Primary: **Group 19 (ECDH with 256-bit modulus)** for forward secrecy
- Fallback: Group 14 (MODP 2048-bit) for interoperability

**IKEv2 Encryption Algorithms**:
- Primary: **AES-256-GCM** for authenticated encryption
- Fallback: AES-128-GCM

**IKEv2 Integrity/Hash Algorithms**:
- Primary: **SHA-384**
- Fallback: SHA-256

**IKEv2 PRF (Pseudo-Random Function)**:
- Primary: **HMAC-SHA384**
- Fallback: HMAC-SHA256

### 3.4 Perfect Forward Secrecy (PFS)

- **PFS Enabled**: All IKEv2 exchanges require PFS
- **PFS Group**: Group 19 (ECDH) or Group 14 (MODP 2048-bit)
- **Benefit**: Compromise of long-term keys does not compromise past session keys

---

## 4. Encryption Algorithms and Modes

### 4.1 IPsec ESP (Encapsulating Security Payload)

#### NY ↔ Paris (Transport Mode)
- **Encryption Algorithm**: **AES-256-GCM**
  - Key length: 256 bits
  - Mode: Galois/Counter Mode (GCM)
  - Benefits: Authenticated encryption (confidentiality + integrity in single operation), low latency, hardware acceleration support
- **Integrity**: Embedded in GCM (no separate AH needed)
- **Key Derivation**: From IKEv2 SA via PRF

#### NY ↔ Toronto and Paris ↔ Toronto (Tunnel Mode)
- **Encryption Algorithm**: **AES-256-GCM** (same as above)
- **Additional Protection**: Full IP packet encapsulation protects source/destination IPs

### 4.2 Key Lifetimes

**IKEv2 SA Lifetime**:
- Time-based: **8 hours**
- Volume-based: **Not used** (time-based only for consistency)
- Rekey margin: **15 minutes** before expiration (automatic rekey)

**IPsec SA (ESP) Lifetime**:
- Time-based: **1 hour**
- Volume-based: Not used
- Rekey margin: **10 minutes** before expiration

**Rationale**:
- Frequent rekeying limits exposure window if keys are compromised
- Short ESP SA lifetime allows for rapid key rotation
- IKEv2 SA re-established periodically to refresh long-term key material

### 4.3 Anti-Replay Protection

- **Enabled**: All IPsec tunnels use sequence numbers for anti-replay protection
- **Window Size**: 64 packets (default)
- **Behavior**: Packets outside replay window are dropped

---

## 5. Tunnel Topology and Routing Behavior

### 5.1 NY ↔ Paris Topology

**Primary Path (Leased Line)**:
- **Mode**: IPsec Transport Mode
- **Endpoints**: 
  - NY Gateway: Dedicated VPN gateway/firewall (primary DC router interface)
  - Paris Gateway: Dedicated VPN gateway/firewall (primary DC router interface)
- **Tunnel Configuration**: Single active tunnel (always-on)
- **Failover**: Automatic failover to VPN backup path if leased line fails

**Backup Path (Internet VPN)**:
- **Mode**: IPsec Tunnel Mode
- **Endpoints**: Same gateways, different logical interfaces (public IPs)
- **Tunnel Configuration**: Standby tunnel activated on primary path failure
- **Routing**: BGP or static routes redirect traffic to backup tunnel

**Topology Diagram**:
```
NY DC                    Paris DC
[Gateway A] <---------> [Gateway B]
   |                        |
   | (Leased Line)          |
   | IPsec Transport        |
   |                        |
   | (Internet VPN Backup)  |
   | IPsec Tunnel           |
   v                        v
[Gateway A Public] <---> [Gateway B Public]
```

### 5.2 NY ↔ Toronto Topology

**Mode**: IPsec Tunnel Mode
- **Endpoints**:
  - NY Gateway: VPN gateway (public IP)
  - Toronto Gateway: VPN gateway (public IP)
- **Tunnel Configuration**: Single primary tunnel (always-on)
- **Routing**: Static routes or BGP advertise Toronto backup network ranges to NY

**Traffic Flow**:
- **Unidirectional (Primary)**: NY → Toronto (backup replication)
- **Bidirectional (Control)**: Management, monitoring, DR orchestration

### 5.3 Paris ↔ Toronto Topology

**Mode**: IPsec Tunnel Mode
- **Endpoints**:
  - Paris Gateway: VPN gateway (public IP)
  - Toronto Gateway: VPN gateway (public IP)
- **Tunnel Configuration**: Single primary tunnel (always-on)
- **Routing**: Static routes or BGP advertise Toronto backup network ranges to Paris

**Traffic Flow**:
- **Unidirectional (Primary)**: Paris → Toronto (backup replication)
- **Bidirectional (Control)**: Management, monitoring, DR orchestration

### 5.4 Routing Behavior

#### NY ↔ Paris
- **Primary Path**: Routes prefer leased line tunnel (lower metric/administrative distance)
- **Backup Path**: Routes to VPN tunnel used only on primary failure
- **Load Balancing**: Not used (single primary path, single backup path)
- **Path Selection**: OSPF/IS-IS internal routing or BGP for path preference

#### NY/Paris ↔ Toronto
- **Static Routes**: Prefer IPsec tunnel for Toronto destination networks
- **NAT Considerations**: No NAT required (tunnel mode preserves addressing)
- **Backup Routing**: If direct tunnel fails, backup traffic may route via alternative path (e.g., NY → Paris → Toronto)

### 5.5 Tunnel Interface Configuration

**IPsec Tunnel Interfaces**:
- Each tunnel creates a virtual interface (e.g., `tunnel0`, `ipsec0`)
- Interface IPs: Private addressing (e.g., point-to-point /30 or /31)
- MTU: Adjusted for IPsec overhead (see section 6.2)

**Example Addressing**:
- NY-Paris Primary: `10.0.1.1/30` ↔ `10.0.1.2/30`
- NY-Paris Backup: `10.0.1.5/30` ↔ `10.0.1.6/30`
- NY-Toronto: `10.0.2.1/30` ↔ `10.0.2.2/30`
- Paris-Toronto: `10.0.3.1/30` ↔ `10.0.3.2/30`

---

## 6. Operational Considerations

### 6.1 Latency Impact

#### NY ↔ Paris (Leased Line + IPsec Transport Mode)
- **Additional Latency**: ~0.5-1ms (hardware-accelerated AES-256-GCM)
- **Total Latency**: <100ms (meets architecture requirement)
- **Mitigation**: Hardware crypto acceleration on gateways, transport mode minimizes overhead

#### NY ↔ Toronto (VPN Tunnel Mode)
- **Additional Latency**: ~1-2ms (IPsec encapsulation + internet routing)
- **Total Latency**: ~15-20ms (acceptable for backup operations)
- **Mitigation**: Hardware acceleration, optimized tunnel configuration

#### Paris ↔ Toronto (VPN Tunnel Mode)
- **Additional Latency**: ~1-2ms
- **Total Latency**: ~80-100ms (transatlantic routing dominates; acceptable for backup)

**Performance Optimization**:
- Hardware crypto acceleration (ASIC/FPGA-based encryption)
- TCP/IP offload on gateways
- Avoid software-based encryption for production traffic

### 6.2 MTU (Maximum Transmission Unit) Handling

**IPsec Overhead**:
- ESP header: 8 bytes
- ESP trailer: 16-20 bytes (AES-GCM authentication tag)
- IP header: 20 bytes (if tunnel mode adds outer IP header)
- **Total Overhead**: ~28-48 bytes (transport mode) or ~48-68 bytes (tunnel mode)

**MTU Configuration**:
- **Physical Interface MTU**: 1500 bytes (standard Ethernet)
- **Tunnel Interface MTU**: 
  - Transport Mode (NY-Paris): 1452 bytes (1500 - 48)
  - Tunnel Mode (Toronto links): 1432 bytes (1500 - 68)
- **Path MTU Discovery (PMTUD)**: Enabled to detect smaller MTU on path
- **ICMP Handling**: Allow ICMP fragmentation needed messages for PMTUD

**Application Impact**:
- TCP MSS (Maximum Segment Size) adjusted automatically by gateway
- UDP applications: Handle fragmentation or use smaller payloads
- Backup applications: Configure appropriate block sizes to avoid fragmentation

### 6.3 Monitoring and Visibility

#### Key Metrics

**Tunnel Health**:
- Tunnel status (up/down)
- IKEv2 SA status and lifetime remaining
- IPsec SA status and lifetime remaining
- Rekey success/failure rates

**Performance Metrics**:
- Latency (ICMP or BFD-based)
- Packet loss rate
- Jitter
- Throughput (bytes/sec, packets/sec)
- Bandwidth utilization

**Security Metrics**:
- Failed authentication attempts
- Replay window violations
- Invalid SPI (Security Parameter Index) errors
- Certificate expiration warnings

#### Monitoring Tools

**Network Monitoring**:
- SNMP polling of gateway devices (tunnel status, counters)
- BFD (Bidirectional Forwarding Detection) for rapid failure detection
- NetFlow/sFlow for traffic analysis (encrypted payloads not visible, but metadata available)

**Logging**:
- IKEv2 logs: Authentication events, rekey events, failures
- IPsec logs: Tunnel establishment, teardown, errors
- Log aggregation: Centralized SIEM for security event correlation

**Alerting**:
- Tunnel down alerts (< 1 minute detection)
- High latency alerts (> 150ms for NY-Paris)
- High packet loss alerts (> 1%)
- Rekey failures
- Certificate expiration warnings (30 days advance)

### 6.4 Key Rotation and Management

#### Key Rotation Schedule

**IKEv2 SA Keys**:
- Automatic rekey every 8 hours (via IKEv2 protocol)
- Manual rotation: Quarterly review and update of long-term credentials (certificates/PSKs)

**IPsec SA Keys**:
- Automatic rekey every 1 hour (via IKEv2 rekey exchange)
- No manual intervention required

**Pre-shared Keys (PSKs)** (for emergency fallback only):
- Rotation: Every 90 days (if PSK fallback is enabled)
- Deployment: Secure key management system (HSM or key management service)
- Zero-downtime rotation: Deploy new PSK before retiring old PSK

**X.509 Certificates**:
- Validity: 1 year
- Renewal: 30 days before expiration
- OCSP/CRL checking: Real-time validation during IKEv2 handshake

#### Key Management Infrastructure

**Certificate Authority (CA)**:
- Private PKI: Internal CA for issuing gateway certificates
- Certificate profiles: Specific OIDs for IPsec/IKEv2 usage
- CRL/OCSP: Certificate revocation services

**Key Storage**:
- Gateway devices: Secure storage (HSM or hardware security module if available)
- PSK storage: Encrypted key vault (e.g., HashiCorp Vault, AWS Secrets Manager equivalent)
- Access control: Least privilege, audit logging of key access

**Operational Procedures**:
- Key rotation runbook: Step-by-step procedures for manual rotation
- Emergency key rotation: Procedures for compromised key scenarios
- Backup and recovery: Secure backup of CA keys and certificates

### 6.5 Failover and High Availability

#### NY ↔ Paris Failover

**Primary to Backup Path Failover**:
- Detection: BFD or routing protocol (OSPF/BGP) detects primary link failure
- Activation: Backup VPN tunnel becomes active (< 30 seconds)
- Routing: Routes automatically prefer backup tunnel
- Rekey: IKEv2 SA established on backup path if not already active

**Tunnel Resilience**:
- Dead Peer Detection (DPD): Enabled to detect tunnel failures
- DPD interval: 30 seconds (keepalive)
- DPD timeout: 120 seconds (declare peer dead)
- Automatic tunnel restart on DPD timeout

#### Toronto Link Failover

**NY → Toronto or Paris → Toronto Failure**:
- Detection: DPD or routing protocol detects tunnel down
- Behavior: Backup operations queued or redirected to alternative path
- Alternative path: NY → Paris → Toronto (if NY-Toronto fails) or Paris → NY → Toronto (if Paris-Toronto fails)
- Manual intervention: May be required for backup job rerouting (application layer)

### 6.6 Security Considerations

#### Threat Mitigation

**Man-in-the-Middle (MitM) Attacks**:
- Mitigated by: Certificate-based authentication, PFS, strong encryption
- Protection: X.509 certificates prevent spoofing of gateway identity

**Replay Attacks**:
- Mitigated by: IPsec sequence numbers, anti-replay window
- Protection: Replayed packets are dropped

**DoS Attacks**:
- Mitigated by: IKEv2 cookie mechanism, rate limiting on IKEv2 port (UDP 500, 4500)
- Protection: Gateways ignore IKEv2 requests without valid cookies

**Key Compromise**:
- Mitigated by: PFS (past sessions secure), frequent key rotation, certificate revocation
- Protection: Compromised keys have limited window of exposure

#### Compliance and Audit

**Encryption Standards Compliance**:
- FIPS 140-2 validated crypto modules (if required)
- AES-256-GCM: Approved for classified information (when properly implemented)
- IKEv2: Industry-standard key exchange

**Audit Trail**:
- All IKEv2 authentication events logged
- Certificate validation events logged
- Key rotation events logged
- Tunnel establishment/teardown events logged
- Log retention: 1 year minimum

---

## 7. Implementation Notes

### 7.1 Gateway Hardware/Software Requirements

**Minimum Specifications**:
- Hardware crypto acceleration for AES-256-GCM
- Sufficient CPU for IKEv2 processing
- Multiple network interfaces (LAN, WAN, dedicated tunnel interfaces)
- High-availability pair (active-standby) for critical gateways (NY, Paris)

**Gateway Platforms** (examples, not exhaustive):
- Enterprise firewalls/VPN gateways (e.g., Palo Alto, Fortinet, Cisco ASA)
- Linux-based solutions (e.g., StrongSwan, Libreswan) with hardware acceleration
- Router-based IPsec (e.g., Cisco IOS, Juniper Junos) if integrated into DC routers

### 7.2 Configuration Template (IKEv2/IPsec)

**IKEv2 Proposal (Example)**:
```
ikev2-proposal AES256-GCM-SHA384
 encryption aes-256-gcm
 integrity sha384
 prf sha384
 group 19
```

**IPsec Proposal (ESP)**:
```
ipsec-proposal ESP-AES256-GCM
 protocol esp
 encryption aes-256-gcm
```

**IKEv2 Policy**:
```
ikev2-policy NY-Paris
 match remote-identity fqdn paris-gateway.example.com
 proposal AES256-GCM-SHA384
 pfs group19
 lifetime seconds 28800
```

**IPsec Policy**:
```
ipsec-policy NY-Paris-ESP
 ikev2-policy NY-Paris
 proposal ESP-AES256-GCM
 lifetime seconds 3600
```

### 7.3 Testing and Validation

**Pre-Production Testing**:
- Tunnel establishment and teardown
- Failover scenarios (primary to backup)
- Rekey operations (manual and automatic)
- Performance testing (latency, throughput)
- MTU path discovery testing
- Certificate validation testing

**Ongoing Validation**:
- Quarterly failover drills
- Annual penetration testing of VPN endpoints
- Regular review of security logs
- Certificate expiration monitoring

---

## 8. Summary

### Encryption Strategy Summary

| Connection | Technology | Mode | Key Exchange | Encryption |
|-----------|-----------|------|--------------|------------|
| NY ↔ Paris (Primary) | IPsec ESP | Transport | IKEv2 (Cert primary, PSK fallback) | AES-256-GCM |
| NY ↔ Paris (Backup) | IPsec VPN | Tunnel | IKEv2 (Cert primary, PSK fallback) | AES-256-GCM |
| NY ↔ Toronto | IPsec VPN | Tunnel | IKEv2 (Cert primary, PSK fallback) | AES-256-GCM |
| Paris ↔ Toronto | IPsec VPN | Tunnel | IKEv2 (Cert primary, PSK fallback) | AES-256-GCM |

### Key Design Decisions

1. **IPsec as Primary Technology**: Industry-standard, well-supported, hardware-accelerated
2. **IKEv2 for Key Exchange**: Modern, efficient, supports PFS and certificate authentication
3. **AES-256-GCM**: Strong encryption with authenticated encryption mode, low latency overhead
4. **Transport Mode for NY-Paris**: Lower overhead for dedicated links, preserves original IP headers
5. **Tunnel Mode for Toronto Links**: Full encapsulation required for internet-based VPNs
6. **Frequent Key Rotation**: 1-hour ESP SA lifetime limits exposure window
7. **Certificate-Based Authentication (Primary)**: X.509 certificates are the primary authentication method; PSKs retained only as controlled emergency fallback
8. **Operational Resilience**: DPD, BFD, automatic failover ensure high availability

---

**Document Version**: 1.0  
**Last Updated**: [Date]  
**Next Review**: Quarterly

