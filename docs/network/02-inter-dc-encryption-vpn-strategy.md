# Inter-Data Center Encryption and VPN Strategy

**Document Version**: 1.1
**Project**: SUPFile — Multi-Site Cloud Storage Infrastructure
**Scope**: Encryption, key exchange, and VPN configuration for all inter-DC communications
**Last Updated**: April 2026

---

## 1. Overview

All inter-data center communications are encrypted at the network layer. No cleartext traffic traverses any inter-DC link under any circumstance — this is a non-negotiable requirement per the project specification: *"All the communications made between the 3 data centers must be fully encrypted."*

### Design Principles

| Principle | Implementation |
|-----------|---------------|
| Defense in Depth | IPsec at L3; optional MACsec at L2 on leased line |
| Zero Trust | No implicit trust between DCs; mutual certificate authentication required |
| Standards-Based | IPsec, IKEv2, AES-256-GCM — industry-standard, FIPS-validated |
| Operational Resilience | Automatic key rotation, DPD, BFD, failover-capable tunnels |
| Performance-Aware | Hardware-accelerated encryption; transport mode where possible to reduce overhead |

---

## 2. Encryption Technology by Connection

### 2.1 Summary Matrix

| Connection | Technology | IPsec Mode | Key Exchange | Cipher | Auth Method |
|-----------|-----------|------------|--------------|--------|-------------|
| NY ↔ Paris (Primary) | IPsec ESP | Transport | IKEv2 | AES-256-GCM | X.509 cert (primary), PSK (emergency fallback) |
| NY ↔ Paris (Backup) | IPsec VPN | Tunnel | IKEv2 | AES-256-GCM | X.509 cert (primary), PSK (emergency fallback) |
| NY → Toronto | IPsec VPN | Tunnel | IKEv2 | AES-256-GCM | X.509 cert (primary), PSK (emergency fallback) |
| Paris → Toronto | IPsec VPN | Tunnel | IKEv2 | AES-256-GCM | X.509 cert (primary), PSK (emergency fallback) |

### 2.2 NY ↔ Paris Primary Path (Leased Line)

**IPsec ESP in Transport Mode**

- The dedicated leased line provides physical isolation. IPsec adds cryptographic protection on top.
- Transport mode preserves original IP headers, reducing overhead for this point-to-point link.
- Lower overhead (~28–48 bytes vs. ~48–68 bytes in tunnel mode) benefits near-real-time replication performance.

**Optional L2 Enhancement — MACsec (IEEE 802.1AE)**:
MACsec provides hop-by-hop Layer 2 encryption as additional defense-in-depth. Combined with IPsec: MACsec (L2) + IPsec (L3) = two independent encryption layers. This is optional for the POC; IPsec alone satisfies all security requirements.

### 2.3 NY ↔ Paris Backup Path (Internet VPN)

**IPsec VPN in Tunnel Mode**

- Internet-based connectivity requires tunnel mode for full packet encapsulation (hides internal IP addressing from public Internet).
- Always-on standby tunnel; activates automatically on primary path failure.
- Same cipher suite as primary path for consistency.

### 2.4 NY/Paris → Toronto (Backup Links)

**IPsec VPN in Tunnel Mode**

- Full encapsulation required over public Internet.
- Each link: one primary tunnel (always-on).
- Sufficient for scheduled, asynchronous backup traffic.

---

## 3. Key Exchange and Authentication

### 3.1 IKEv2 Protocol

IKEv2 is the sole key exchange protocol for all inter-DC tunnels.

| Feature | Benefit |
|---------|---------|
| Fewer message exchanges than IKEv1 | Lower setup latency |
| Built-in NAT traversal | Works through Internet NAT devices (Toronto links) |
| Cookie mechanism | DoS protection on IKEv2 responders |
| EAP support | Flexible certificate-based authentication |
| MOBIKE | Session continuity during IP changes (failover scenarios) |

### 3.2 Authentication

**Primary: X.509 Certificate-Based (Mutual Authentication)**

- Certificates issued by a private internal PKI/CA (self-hosted, not public CA).
- Each VPN gateway has a unique certificate with SAN matching its gateway IP/FQDN.
- Certificate revocation checked via OCSP (primary) and CRL (fallback).
- Certificate validity: 1 year; renewal 30 days before expiry.

**Emergency Fallback: Pre-Shared Key (PSK)**

- Retained only for scenarios where certificate services are completely unavailable.
- PSK stored in encrypted key vault (HashiCorp Vault or equivalent).
- PSK rotation: every 90 days.
- Deployment via secure key management; never transmitted in cleartext.

### 3.3 Cryptographic Parameters

**IKEv2 SA Parameters**:

| Parameter | Primary | Fallback |
|-----------|---------|----------|
| Encryption | AES-256-GCM | AES-128-GCM |
| Integrity/Hash | SHA-384 | SHA-256 |
| PRF | HMAC-SHA384 | HMAC-SHA256 |
| DH Group | Group 19 (ECDH P-256) | Group 14 (MODP 2048-bit) |
| Authentication | X.509 RSA-4096 | PSK (emergency only) |

**IPsec ESP Parameters**:

| Parameter | Value |
|-----------|-------|
| Encryption | AES-256-GCM (authenticated encryption — no separate integrity algorithm needed) |
| PFS | Enabled — Group 19 (ECDH) per child SA |
| Anti-replay | Enabled, window size 64 packets |

### 3.4 Perfect Forward Secrecy (PFS)

PFS is mandatory on all tunnels. Each child SA performs a fresh DH exchange, ensuring that compromise of long-term keys does not expose past session traffic.

---

## 4. Key Lifetimes and Rotation

### 4.1 Automatic Rotation

| SA Type | Lifetime | Rekey Margin | Notes |
|---------|----------|-------------|-------|
| IKEv2 SA | 8 hours | 15 min before expiry | Automatic rekey via IKEv2 protocol |
| IPsec SA (ESP) | 1 hour | 10 min before expiry | Automatic rekey; limits exposure window |

### 4.2 Manual Rotation Schedule

| Credential | Rotation | Procedure |
|-----------|----------|-----------|
| X.509 certificates | Annually (renewal 30 days prior) | Reissue from internal CA; deploy to gateways |
| PSK (if enabled) | Every 90 days | Deploy new PSK via Vault before retiring old |
| CA root certificate | Every 5 years | Planned renewal with overlap period |

### 4.3 Key Management Infrastructure

| Component | Implementation |
|-----------|---------------|
| Certificate Authority | Private internal CA (e.g., EJBCA, StepCA, or OpenSSL-based) |
| Certificate profiles | Specific OIDs for IPsec/IKEv2 gateway usage |
| Revocation | OCSP responder (primary) + CRL distribution (fallback) |
| Key storage | HSM on gateway hardware (if available); otherwise secure TPM or encrypted filesystem |
| PSK vault | HashiCorp Vault (encrypted, audit-logged, RBAC) |
| Audit | All key access, rotation, and revocation events logged to SIEM |

---

## 5. Tunnel Topology and Addressing

### 5.1 Tunnel Interface Addressing

| Tunnel | Endpoint A | Endpoint B | Subnet |
|--------|-----------|-----------|--------|
| NY–Paris Primary (leased line) | 10.0.1.1 | 10.0.1.2 | /30 |
| NY–Paris Backup (Internet VPN) | 10.0.1.5 | 10.0.1.6 | /30 |
| NY–Toronto | 10.0.2.1 | 10.0.2.2 | /30 |
| Paris–Toronto | 10.0.3.1 | 10.0.3.2 | /30 |

### 5.2 Routing Behavior

**NY ↔ Paris**:
- Primary path (leased line tunnel) preferred via lower route metric / administrative distance.
- Backup path (VPN tunnel) activated on primary failure (BFD/DPD detection).
- No load balancing between paths — single active, single standby.
- Path selection via OSPF or static routes with preference metrics.

**NY/Paris → Toronto**:
- Static routes for Toronto destination networks via IPsec tunnel.
- If direct tunnel fails, backup routing via the other primary DC (e.g., NY → Paris → Toronto).
- NAT not required (tunnel mode preserves internal addressing).

### 5.3 Topology Diagram

```
NY DC                              Paris DC
┌──────────────┐                   ┌──────────────┐
│ VPN Gateway  │═══════════════════│ VPN Gateway  │
│ 10.1.200.1   │  Leased Line     │ 10.2.200.1   │
│              │  IPsec Transport  │              │
│              │───────────────────│              │
│              │  Internet VPN     │              │
│              │  IPsec Tunnel     │              │
└──────┬───────┘  (Standby)       └──────┬───────┘
       │                                  │
       │ IPsec Tunnel                     │ IPsec Tunnel
       │ (Internet VPN)                   │ (Internet VPN)
       │                                  │
       └──────────┐          ┌────────────┘
                  ▼          ▼
            ┌──────────────────┐
            │   Toronto DC     │
            │   VPN Gateway    │
            │   10.3.200.1     │
            └──────────────────┘
```

---

## 6. MTU and Performance

### 6.1 MTU Configuration

| Interface | MTU | Rationale |
|-----------|-----|-----------|
| Physical Ethernet | 1500 bytes | Standard |
| Transport mode tunnel (NY–Paris primary) | 1452 bytes | 1500 − 48 bytes ESP overhead |
| Tunnel mode tunnels (all others) | 1432 bytes | 1500 − 68 bytes (outer IP + ESP) |

**PMTUD (Path MTU Discovery)**: Enabled on all tunnels. ICMP "fragmentation needed" messages must be allowed through firewalls.

**TCP MSS clamping**: Applied at VPN gateways to automatically adjust TCP segment sizes, avoiding fragmentation.

### 6.2 Latency Impact of Encryption

| Link | Base Latency | Encryption Overhead | Total Latency |
|------|-------------|-------------------|---------------|
| NY ↔ Paris (leased line) | ~70–75 ms | +0.5–1 ms (hardware AES-NI) | ~71–76 ms |
| NY ↔ Paris (Internet VPN) | ~75–85 ms | +1–2 ms | ~77–87 ms |
| NY → Toronto | ~15–18 ms | +1–2 ms | ~17–20 ms |
| Paris → Toronto | ~85–95 ms | +1–2 ms | ~87–97 ms |

**Performance optimization**: All gateways must have hardware crypto acceleration (AES-NI / ASIC). Software-based encryption is not acceptable for production traffic.

---

## 7. Failover and High Availability

### 7.1 NY ↔ Paris Primary-to-Backup Failover

| Step | Action | Timing |
|------|--------|--------|
| 1 | BFD detects leased line failure | < 3 seconds |
| 2 | DPD confirms peer unreachable on primary | < 30 seconds |
| 3 | Routing tables updated (metric change) | Immediate |
| 4 | Traffic shifts to backup VPN tunnel | < 30 seconds total |
| 5 | IKEv2 SA established on backup path (if not pre-established) | < 5 seconds |
| 6 | Replication resumes over backup path | Automatic |

### 7.2 Dead Peer Detection (DPD)

| Parameter | Value |
|-----------|-------|
| DPD interval | 30 seconds (keepalive) |
| DPD timeout | 120 seconds (declare peer dead) |
| Action on timeout | Tear down SA; trigger route failover |
| Automatic restart | Yes — tunnel re-established when peer recovers |

### 7.3 Gateway High Availability

Each primary DC (NY, Paris) runs a pair of VPN gateways in active-standby configuration:
- VRRP/CARP for gateway IP failover.
- Shared tunnel state via IKEv2 session resumption or IPsec SA sync.
- Failover time: < 10 seconds for gateway hardware failure.

Toronto runs a single VPN gateway (cost optimization; backup-only role).

---

## 8. Security Considerations

### 8.1 Threat Mitigation

| Threat | Mitigation |
|--------|-----------|
| Man-in-the-Middle | X.509 mutual certificate authentication; PFS; strong encryption |
| Replay attacks | IPsec anti-replay window (sequence numbers) |
| DoS on VPN gateways | IKEv2 cookie mechanism; rate limiting on UDP 500/4500 |
| Key compromise | PFS limits exposure; 1-hour ESP SA lifetime; certificate revocation via OCSP |
| Leased line interception | IPsec encryption even on "private" lines (zero trust) |
| Certificate spoofing | Private CA; certificate pinning on gateway configs; OCSP stapling |

### 8.2 Compliance and Audit

| Requirement | Implementation |
|-------------|---------------|
| Encryption standard | AES-256-GCM (FIPS 140-2 approved) |
| Key exchange | IKEv2 with ECDH Group 19 |
| Audit trail | All IKEv2 auth events, rekey events, tunnel up/down → SIEM |
| Log retention | 1 year minimum |
| Testing | Quarterly failover drills; annual penetration testing of VPN endpoints |

---

## 9. Implementation Configuration Templates

### 9.1 StrongSwan Configuration (Linux-based Gateway)

**IKEv2 Connection — NY to Paris Primary (Transport Mode)**:

```
# /etc/swanctl/conf.d/ny-paris-primary.conf
connections {
    ny-paris-primary {
        version = 2
        local_addrs = 10.1.200.1
        remote_addrs = 10.2.200.1

        local {
            auth = pubkey
            certs = ny-gateway.pem
            id = ny-gateway.supfile.internal
        }
        remote {
            auth = pubkey
            id = paris-gateway.supfile.internal
        }

        children {
            ny-paris-esp {
                mode = transport
                esp_proposals = aes256gcm16-sha384-ecp256
                life_time = 3600       # 1 hour ESP SA
                rekey_time = 3300      # rekey 5 min before expiry
                dpd_action = restart
                start_action = start
            }
        }

        proposals = aes256gcm16-sha384-prfsha384-ecp256
        rekey_time = 28800             # 8 hour IKE SA
        dpd_delay = 30s
        dpd_timeout = 120s
    }
}
```

**IKEv2 Connection — NY to Toronto (Tunnel Mode)**:

```
# /etc/swanctl/conf.d/ny-toronto.conf
connections {
    ny-toronto {
        version = 2
        local_addrs = %any
        remote_addrs = toronto-vpn.supfile.internal

        local {
            auth = pubkey
            certs = ny-gateway.pem
            id = ny-gateway.supfile.internal
        }
        remote {
            auth = pubkey
            id = toronto-gateway.supfile.internal
        }

        children {
            ny-toronto-esp {
                mode = tunnel
                local_ts = 10.1.0.0/16
                remote_ts = 10.3.0.0/16
                esp_proposals = aes256gcm16-sha384-ecp256
                life_time = 3600
                rekey_time = 3300
                dpd_action = restart
                start_action = start
            }
        }

        proposals = aes256gcm16-sha384-prfsha384-ecp256
        rekey_time = 28800
        dpd_delay = 30s
        dpd_timeout = 120s
    }
}
```

### 9.2 IPsec Proposal Reference

```
# IKEv2 Proposal
Encryption:     AES-256-GCM (16-byte ICV)
Integrity:      SHA-384 (used for IKE SA only; GCM provides its own for ESP)
PRF:            HMAC-SHA384
DH Group:       19 (ECP-256)

# ESP Proposal
Encryption:     AES-256-GCM (16-byte ICV)
PFS Group:      19 (ECP-256)
```

---

## 10. Summary

### Key Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | IPsec as primary encryption | Industry-standard, hardware-accelerated, well-supported on all gateway platforms |
| 2 | IKEv2 for key exchange | Modern, efficient, built-in DoS protection, NAT traversal |
| 3 | AES-256-GCM cipher | Authenticated encryption in single pass; low latency; FIPS approved |
| 4 | Transport mode for NY–Paris leased line | Lower overhead (~20 bytes saved per packet); suitable for point-to-point |
| 5 | Tunnel mode for Internet VPN links | Full encapsulation required; hides internal addressing |
| 6 | 1-hour ESP SA lifetime | Limits key exposure window; automatic rotation |
| 7 | X.509 certificates as primary auth | Stronger than PSK; supports revocation; scalable |
| 8 | PFS mandatory | Protects historical traffic even if long-term keys are later compromised |
| 9 | Hardware crypto acceleration required | Software encryption introduces unacceptable latency for production replication |

---

**Previous Document**: [01-inter-dc-network-architecture.md](./01-inter-dc-network-architecture.md)
**Next Document**: [03-network-segmentation-vlan-strategy.md](./03-network-segmentation-vlan-strategy.md)
