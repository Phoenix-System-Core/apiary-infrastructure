# Apiary.AI Network Topology

**Status:** Phase 0 (Foundation) - IP assignments, VPN routing, mesh configuration

---

## Physical Network Diagram

```
┌──────────────────────────────────────────────────────────┐
│                                                              │
│                         INTERNET (ISP)                      │
│                  Shared WAN egress point                    │
│                                                              │
└────────────────────┬────────────────────┬────────────────────┘
                             │                             │                             │
                             │                             │                             │
                      Fiber / Copper                  Fiber / Copper                  Fiber / Copper
                             │                             │                             │
                             ▼                             ▼                             ▼
                    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
                    │ UniFi Dream    │    │ UniFi UDM   │    │ UniFi UDR7  │
                    │ Router (BH)   │    │ Pro Max    │    │ (EP)        │
                    │ 192.168.50.1  │    │ (BO)       │    │ 192.168.30.1│
                    │ Wi-Fi 7       │    │ 192.168.9.1│    │ LTE/Backup  │
                    └──┬───────────┘    └──┬───────────┘    └──┬───────────┘
                         │                    │                    │
                         │ Ethernet/Wi-Fi    │ 10Gbps trunk      │ 1Gbps copper
                         │ to clients        │ to switching     │ to Proxmox
                         │                    │                    │
                         ▼                    ▼                    ▼
                    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
                    │ BH-21100       │    │ Dell R7920   │    │ EP-01-SN    │
                    │ Proxmox Node  │    │ + R740xd    │    │ Proxmox Node│
                    │ 192.168.50.10 │    │ 192.168.9.10│    │ 192.168.30.10
                    │               │    │             │    │             │
                    │ Services:     │    │ Services:   │    │ Services:   │
                    │ • Ollama       │    │ • Authentik  │    │ • Ollama      │
                    │ • Konstanze    │    │ • Netbird   │    │ • Konstanze  │
                    │ • Odoo (dev)   │    │ • PostgreSQL│    │ • Odoo (prod)│
                    │ • Home Asst.   │    │ • MinIO     │    │ • Bravo POS  │
                    │ • Restic Agent │    │ • Restic Srv│    │ • Restic Agnt│
                    └──────────────┘    └──────────────┘    └──────────────┘
```

---

## IP Addressing Scheme

### Overview

```
CIDR Block      | Purpose                  | Sites         | Allocation
──────────────┼─────────────────┼─────────────┼───────────────────
192.168.9.0/24  | Bien Org (Mainframe)     | BO only       | 254 hosts
192.168.30.0/24 | Easy Pawn (Production)   | EP only       | 254 hosts
192.168.50.0/24 | Bien Home (Dev/Test)     | BH only       | 254 hosts
10.0.0.0/8      | Netbird Mesh (Overlay)   | All sites+AIP | Unlimited peers
```

### Bien Org (192.168.9.0/24) — Tier 0 Mainframe

**Scope:** Central infrastructure only, not for general clients

```
192.168.9.1        | UniFi UDM Pro Max (gateway, DHCP disabled)
192.168.9.10       | Dell R7920 (Proxmox primary, LXC host)
192.168.9.11       | Dell R740xd (Proxmox secondary, HA failover)

192.168.9.20       | PostgreSQL Primary (read/write)
192.168.9.21       | MinIO (S3-compatible backup)
192.168.9.22       | Restic Server (backup receiver)
192.168.9.23       | Prometheus (metrics scraper)
192.168.9.24       | Grafana (dashboards)
192.168.9.25       | Uptime Kuma (health checks)
192.168.9.26       | Wazuh Manager (SIEM, Phase 2+)
192.168.9.27       | Wazuh Indexer (Elasticsearch)

192.168.9.30       | Authentik OIDC/SAML (SSO)
192.168.9.31       | Netbird Coordinator (mesh control)
192.168.9.40       | Gitea (code repos)
192.168.9.41       | n8n (automation workflows)
192.168.9.42       | Paperless-NGX (document archive)

192.168.9.50-99    | Management/IPMI/iLO (out-of-band)
192.168.9.100-200  | Service pool (reserved for future LXC containers)
192.168.9.201-254  | Reserved (clients shouldn't be here)
```

### Bien Home (192.168.50.0/24) — Tier 1 Site

**Scope:** Local site services + clients + development

```
192.168.50.1       | UniFi Dream Router (gateway)
192.168.50.10      | BH-21100 Proxmox Node

192.168.50.20      | Ollama (local LLM inference)
192.168.50.21      | Open WebUI / Konstanze (chat interface)
192.168.50.22      | PostgreSQL Replica (read-only sync from BO)
192.168.50.30      | Odoo (development instance)
192.168.50.40      | Home Assistant (IoT automation)
192.168.50.50      | Restic Agent (backup to BO)

192.168.50.100-200 | DHCP client pool (workstations, devices)
192.168.50.200-254 | IoT / WiFi devices
```

### Easy Pawn (192.168.30.0/24) — Tier 1 Site

**Scope:** POS operations + production accounting + backups

**⚠️ CURRENT STATE: 192.168.2.x (WRONG, must fix in Phase 0)**

```
192.168.30.1       | UniFi UDR7 (gateway)
192.168.30.10      | EP-01-SN Proxmox Node

192.168.30.20      | Ollama (local LLM)
192.168.30.21      | Open WebUI / Konstanze (staff access)
192.168.30.22      | PostgreSQL Replica (read-only, accounting data)
192.168.30.30      | Odoo (PRODUCTION Easy Pawn accounting)
192.168.30.40      | Bravo POS (native app, LAN access)
192.168.30.50      | Restic Agent (backup to BO)

192.168.30.100-200 | DHCP client pool (workstations)
192.168.30.250-254 | POS terminals (static or DHCP reserved)
```

---

## VPN Configuration

### UniFi Site-to-Site (S2S) IPsec Tunnels

**Purpose:** Encrypted LAN-to-LAN connectivity for infrastructure traffic

**Tunnel 1: BO ↔ BH**
```
Initiator       : UniFi UDM Pro Max (BO)
Responder       : UniFi Dream Router (BH)
Mode            : IPsec (IKEv2)
Encryption      : AES-256-GCM
Authentication  : Pre-shared key (stored in UniFi)

BO Local Subnet : 192.168.9.0/24
BH Local Subnet : 192.168.50.0/24

Status          : Phase 0 baseline (currently down)
MTU Overhead    : -70 bytes (auto-adjusted)
```

**Tunnel 2: BO ↔ EP**
```
Initiator       : UniFi UDM Pro Max (BO)
Responder       : UniFi UDR7 (EP)
Mode            : IPsec (IKEv2)
Encryption      : AES-256-GCM
Authentication  : Pre-shared key

BO Local Subnet : 192.168.9.0/24
EP Local Subnet : 192.168.30.0/24 (currently 192.168.2.x)

Status          : Phase 0 BLOCKED (subnet issue)
MTU Overhead    : -70 bytes
```

**Tunnel 3: BH ↔ EP (Optional)**
```
Initiator       : UniFi Dream Router (BH)
Responder       : UniFi UDR7 (EP)
Mode            : IPsec (IKEv2)

BH Local Subnet : 192.168.50.0/24
EP Local Subnet : 192.168.30.0/24

Status          : Phase 0 NOT REQUIRED (relay via BO is acceptable)
Reason          : BO is central hub, reduces complexity
```

### Netbird Mesh (Zero-Trust Overlay)

**Purpose:** Encrypted peer-to-peer VPN tunnel with identity-based access control

**Configuration:**
```
Coordinator     : Netbird Coordinator on BO (192.168.9.31:80)
Tunnel Network  : 10.0.0.0/8 (WireGuard interface)
Protocol        : WireGuard (UDP/51820)
Encryption      : Curve25519 (256-bit ECDH)

Peers (Phase 1+):
├─ BO Proxmox nodes (R7920, R740xd)
├─ BH-21100 Proxmox node
├─ EP-01-SN Proxmox node
├─ AIP Box #1, #2, #N (future)
└─ Workstations (optional, for remote access)

Auth            : Authentik OIDC integration (Phase 1)
Isolation       : Peer-level ACL rules (customer isolation)
Latency         : +10-20ms added to native traffic
```

**Netbird Peer Topology:**
```
    Coordinator
    (BO 10.x.x.x)
    ┌───┬───┐
    │   │   │
    ▼   ▼   ▼
  R7920 R740 BH-21100  <- Direct encrypted tunnels to coordinator
    │   │   │
    └──────┘ <- Peer-to-peer discovery via coordinator

Each peer can reach any other peer directly (NAT traversal)
No single point of failure for peer communication (coordinator down → cached routes)
```

---

## Container Networking (Per-Site)

### Bien Org (BO) Proxmox Bridge

**Host Interface:** eth0 (1Gbps copper to UDM Pro Max)
**Proxmox Bridge:** vmbr0 (192.168.9.0/24)

```
          eth0 (physical)
            │
            ▼
        vmbr0 (bridge)
        192.168.9.0/24
            │
┌──────┴──────────────────────┐
│ LXC Containers / VMs
├─ net0: 192.168.9.20 (PostgreSQL)
├─ net0: 192.168.9.21 (MinIO)
├─ net0: 192.168.9.22 (Restic Server)
├─ net0: 192.168.9.23 (Prometheus)
├─ net0: 192.168.9.30 (Authentik)
├─ net0: 192.168.9.31 (Netbird Coordinator)
├─ net0: 192.168.9.40 (Gitea)
├─ net0: 192.168.9.41 (n8n)
└─ net0: 192.168.9.42 (Paperless-NGX)
```

**VLAN Plan (Future):**
Reserve VLAN 100-110 for management traffic (IPMI, iLO, console)
Reserve VLAN 200 for replication (PostgreSQL streaming)

### Bien Home (BH) Proxmox Bridge

**Host Interface:** eth0 (1Gbps to Dream Router)
**Proxmox Bridge:** vmbr0 (192.168.50.0/24)

```
          eth0 (physical)
            │
            ▼
        vmbr0 (bridge)
        192.168.50.0/24
            │
┌──────┴──────────────────────┐
│ LXC Containers
├─ net0: 192.168.50.20 (Ollama)
├─ net0: 192.168.50.21 (Open WebUI)
├─ net0: 192.168.50.22 (PostgreSQL Replica)
├─ net0: 192.168.50.30 (Odoo)
├─ net0: 192.168.50.40 (Home Assistant)
└─ net0: 192.168.50.50 (Restic Agent)
```

### Easy Pawn (EP) Proxmox Bridge

**Host Interface:** eth0 (1Gbps to UDR7)
**Proxmox Bridge:** vmbr0 (192.168.30.0/24)

```
          eth0 (physical)
            │
            ▼
        vmbr0 (bridge)
        192.168.30.0/24
            │
┌──────┴──────────────────────┐
│ LXC Containers
├─ net0: 192.168.30.20 (Ollama)
├─ net0: 192.168.30.21 (Open WebUI)
├─ net0: 192.168.30.22 (PostgreSQL Replica)
├─ net0: 192.168.30.30 (Odoo)
├─ net0: 192.168.30.50 (Restic Agent)
└─ eth0: 192.168.30.40 (Bravo POS, native/isolated)
```

---

## Netbird Mesh Design

### Per-Container Netbird Agent

**Why per-container, not per-host?**
- Isolation: Customers never cross paths
- Scaling: Add 100 containers, not 100 host routes
- Security: Granular ACL rules at service level

**Netbird Agent Installation (each container):**
```bash
# Inside container (LXC)
apt-get install -y netbird

# Configure agent
netbird up --setup-key <setup-key-from-coordinator>

# Verify
ip addr show | grep 10.  # Should see 10.x.x.x address
ping 10.x.x.x          # Coordinator
```

### Peer Access Rules (Example)

```
Rule 1: Internal Site Access (Same Site)
  From: 192.168.9.* (BO subnet)
  To:   192.168.9.* (BO subnet)
  VIA:  Native LAN (UniFi bridge)
  Fallback: Netbird if LAN down

Rule 2: Backup Replication
  From: Restic Agent (BH/EP/AIP)
  To:   Restic Server (BO 192.168.9.22)
  VIA:  Netbird tunnel (10.x.x.x)
  Port: 8000 (REST API)
  MTU:  1400 (WireGuard overhead)

Rule 3: PostgreSQL Replication
  From: PostgreSQL Replica (BH/EP 192.168.50.22, 192.168.30.22)
  To:   PostgreSQL Primary (BO 192.168.9.20)
  VIA:  UniFi S2S VPN (IPsec) or Netbird (fallback)
  Port: 5432
  Auth: pguser/pgpass (encrypted)

Rule 4: Authentik OAuth
  From: OpenWebUI (all sites)
  To:   Authentik (BO 192.168.9.30)
  VIA:  UniFi S2S + Netbird
  Port: 8000, 9000

Rule 5: Customer Isolation
  From: AIP Box #1 (10.x.x.1)
  To:   AIP Box #2 (10.x.x.2)
  Allow: NO (customers isolated)
  To:   BO Tier 0 (10.x.x.y)
  Allow: YES (auth, backup only)
```

---

## DNS Resolution

### Internal DNS Records (Phase 1+)

When Authentik + Netbird online, add DNS records:

```
Record Type      | Hostname              | IP Address      | Purpose
──────────┼──────────────────┼─────────────┼──────────────────

A (Local)        | auth.local            | 192.168.9.30    | Authentik SSO
A (Local)        | netbird.local         | 192.168.9.31    | Netbird coordinator
A (Local)        | db.local              | 192.168.9.20    | PostgreSQL primary
A (Local)        | backup.local          | 192.168.9.22    | Restic server
A (Local)        | s3.local              | 192.168.9.21    | MinIO
A (Local)        | metrics.local         | 192.168.9.23    | Prometheus
A (Local)        | dashboards.local      | 192.168.9.24    | Grafana
A (Local)        | status.local          | 192.168.9.25    | Uptime Kuma

CNAME (Public)   | auth.apiary.ai        | *.biencompanies | Public endpoint (reverse proxy)
CNAME (Public)   | api.apiary.ai         | *.biencompanies | Netbird API gateway
CNAME (Public)   | konstanze.apiary.ai   | *.biencompanies | OpenWebUI public
```

**DNS Server:** Run Dnsmasq in LXC container (BO) or use UniFi built-in

---

## Firewall Rules (Phase 1)

### UniFi Firewall (Inbound from Internet)

```
Rule 1: Block all inbound (default deny)
  From: 0.0.0.0/0
  To:   Any
  Action: DROP
  Log: YES

Rule 2: Allow VPN (Netbird coordinator)
  From: 0.0.0.0/0
  To:   UDM Pro Max port 80 (Netbird API)
  Action: ACCEPT
  Log: NO (high volume)

Rule 3: Allow public HTTPS (OAuth, reverse proxy)
  From: 0.0.0.0/0
  To:   UDM Pro Max port 443 (HTTPS)
  Action: ACCEPT
  Log: YES

Rule 4: Allow DNS (if public resolver)
  From: 0.0.0.0/0
  To:   UDM Pro Max port 53 (DNS)
  Action: ACCEPT
  Log: NO
```

### Proxmox Host Firewall (Inbound from LAN)

```
Rule 1: Allow local subnet (same site)
  From: 192.168.9.0/24 (or 192.168.50.x, 192.168.30.x)
  To:   All Proxmox interfaces
  Protocol: Any
  Action: ACCEPT

Rule 2: Allow Netbird tunnel
  From: 10.0.0.0/8
  To:   All Proxmox interfaces
  Protocol: WireGuard (UDP)
  Action: ACCEPT

Rule 3: Allow UniFi S2S VPN
  From: VPN peers (dynamic IPs)
  To:   All containers
  Protocol: IPsec
  Action: ACCEPT

Rule 4: Allow SSH (management, restricted)
  From: 192.168.9.50-99 (management VLAN)
  To:   Proxmox host:22
  Protocol: TCP
  Action: ACCEPT

Rule 5: Block everything else (default deny)
  Action: DROP
  Log: YES
```

---

## Phase 0 Network Checklist

- [ ] Fix EP subnet (192.168.2.x → 192.168.30.x)
- [ ] Verify BO ↔ BH S2S tunnel status (UniFi controller)
- [ ] Verify BO ↔ EP S2S tunnel status (wait for subnet fix)
- [ ] Test ping between all three sites
  - `ping 192.168.50.10` from BO
  - `ping 192.168.9.10` from EP
  - `ping 192.168.30.10` from BH
- [ ] Verify MTU (should be 1400-1480)
  - `ping -M do -s 1450 192.168.9.10`
- [ ] Document actual VPN IKE/ESP proposals (UniFi logs)
- [ ] Test failover (kill S2S, verify fallback)

---

## Phase 1 Network Additions

- [ ] Deploy Netbird Coordinator on BO
- [ ] Generate setup keys for all Proxmox nodes
- [ ] Enroll BO/BH/EP Proxmox nodes in Netbird mesh
- [ ] Configure Authentik OIDC for Netbird
- [ ] Test peer-to-peer communication (BO ↔ BH, BO ↔ EP)
- [ ] Create network diagram with Netbird tunnel assignments
- [ ] Document DNS records (local + public)
- [ ] Deploy Prometheus service discovery (scrape all sites)

---

## Phase 2+ Network Scale

When AIP boxes are deployed:
- Generate unique setup keys per AIP device
- Configure customer isolation rules (no cross-peer access)
- Monitor Netbird peer count (status.local dashboard)
- Plan IP space (10.0.0.0/8 can handle 16M unique peers)
- Evaluate site-to-site throughput (backup traffic analysis)

---

**Document Version:** 1.0 (Phase 0)  
**Last Updated:** 2026-01-09  
**Next Update:** After Phase 0 completion (subnet fixed, VPNs tested)
