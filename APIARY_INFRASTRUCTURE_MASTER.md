# APIARY.AI: Master Infrastructure Reference

**Last Updated:** January 9, 2026 | **Version:** 1.0 (Foundation)
**Status:** Phase 0 (Foundation) - Roadmap locked, execution phase

---

## 🎯 Quick Navigation

- **For "where does X run?"** → [Service Deployment Matrix](#service-deployment-matrix)
- **For network topology** → [Network Topology](#network-topology)
- **For hardware specs** → [Physical Inventory](#physical-inventory)
- **For what to build next** → [Deployment Roadmap](#deployment-roadmap)
- **For emergency procedures** → [Runbooks](#runbooks--procedures)

---

## Core Architecture: The Three-Tier Model

### Design Philosophy

**Problem:** You can't keep everything in your head. Current state:
- 3 physical locations (Bien Org, Bien Home, Easy Pawn)
- 15+ services across multiple Proxmox nodes
- 2+ network trunks (UniFi S2S, Netbird mesh)
- Growing customer appliance fleet (AIP boxes)
- Distributed PostgreSQL replicas
- API budgets to manage

**Solution:** Tiered deployment by **criticality**, **latency requirements**, and **resilience needs** — not "install everything everywhere."

### The Three Tiers

```
TIER 0: CENTRAL MAINFRAME (Bien Org)
├─ Dell R7920 / R740xd
├─ Purpose: Heavy compute, central services, SSO, backups, coordination
├─ Uptime SLA: 99.5% (redundant power, cooling, UPS)
└─ Acts as: Auth hub, backup receiver, monitoring center, API gateway

TIER 1: SITE SERVERS (BH-21100, EP-01-SN)
├─ Existing Proxmox nodes at Bien Home & Easy Pawn
├─ Purpose: Site-local resilience (survives WAN failure)
├─ Uptime SLA: 99% per site
├─ Services: Ollama, Konstanze, Odoo, PostgreSQL replica, Home Assistant
└─ Design: Works standalone if internet down

TIER 2: CUSTOMER APPLIANCES (AIP boxes)
├─ Mac Mini / Dell Micro form factor (future)
├─ Purpose: Minimal footprint, "phones home" to Tier 0
├─ Uptime SLA: Best effort (non-critical customer devices)
├─ Services: Ollama, OpenWebUI, Odoo, Netbird agent, backups
└─ Design: Can operate offline, sync when online
```

---

## Physical Inventory

### Bien Org (BO) - 401 BUS 60 E, Dexter, MO

**Tier 0 Mainframe:**

| Item | Model | Specs | Status | Role |
|------|-------|-------|--------|------|
| Primary Compute | Dell PowerEdge R7920 | 2× Xeon Platinum, 256GB RAM, SSD | ⚠️ To be racked | Proxmox primary, compute-heavy |
| Secondary/Failover | Dell PowerEdge R740xd | 2× Xeon Gold, 128GB RAM, SSD | ✅ Baseline | Proxmox secondary, HA candidate |
| Storage | MinIO backend | 1-2TB initial | 📋 Planned | S3-compatible backup target |
| Router | UniFi UDM Pro Max | 10Gbps capable | ✅ Deployed | Primary, controller, aggregation |
| Switching | UniFi 24-port PoE | SFP uplinks | 📋 Planned | Proxmox cluster switching |
| Power | Dual PSU (R7920) | 1400W ea | ✅ Spec'd | Redundant distribution |
| UPS | TBD | 30-60 min reserve | 📋 NEEDED | Graceful shutdown |

**Network Segment:**
```
192.168.9.0/24 (Bien Org)
├─ .1        → UDM Pro Max (gateway)
├─ .10       → R7920 Proxmox (primary)
├─ .11       → R740xd Proxmox (secondary)
├─ .20-29    → Storage (MinIO, NAS, etc)
├─ .50-99    → Management (IPMI, iLO, out-of-band)
└─ .100-200  → Client/service pool
```

---

### Bien Home (BH) - 730 Susan St, Dexter, MO

**Tier 1 Site Server:**

| Item | Model | Specs | Status | Role |
|------|-------|-------|--------|------|
| Proxmox Node | Custom/Refurb | BH-21100 | ✅ Deployed | Dev/test + local services |
| Router | UniFi Dream Router | Wi-Fi 7, built-in | ✅ Deployed | Secondary router, backup |
| Storage | TBD | Local SSD/NAS | 📋 Planned | Odoo database backup, media |

**Network Segment:**
```
192.168.50.0/24 (Bien Home)
├─ .1        → UniFi Dream Router (gateway)
├─ .10       → BH-21100 Proxmox
├─ .20-99    → Services (Ollama, Konstanze, Odoo, HA, etc)
├─ .100-200  → Workstations
└─ .200-254  → IoT / Home Assistant
```

---

### Easy Pawn (EP) - 2 S Kitchen St, Dexter, MO

**Tier 1 Site Server:**

| Item | Model | Specs | Status | Role |
|------|-------|-------|--------|------|
| Proxmox Node | Custom/Refurb | EP-01-SN | ✅ Deployed | Odoo + POS + local services |
| Router | UniFi UDR7 | Compact PoE | ✅ Deployed | Site router, backup |
| Storage | TBD | Local SSD | 📋 Planned | Odoo + POS database |
| POS System | Bravo POS / Square | TBD | ✅ Baseline | Integrates with Odoo |

**Network Segment:**
```
192.168.30.0/24 (Easy Pawn) [CURRENT: 192.168.2.x — PHASE 0 FIX]
├─ .1        → UDR7 (gateway)
├─ .10       → EP-01-SN Proxmox
├─ .20-99    → Services (Ollama, Konstanze, Odoo, backups)
├─ .100-200  → Workstations / Staff
└─ .250-254  → POS / Bravo terminals
```

**⚠️ CRITICAL PHASE 0 TASK:** Fix EP subnet from 192.168.2.x → 192.168.30.x in UniFi controller

---

## Network Topology

### High-Level Architecture

```
                    ┌──────────────────────────────────────┐
                    │   INTERNET (Shared ISP)              │
                    └──────────────┬───────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
    ┌────────────┐            ┌────────────┐            ┌────────────┐
    │ BIEN HOME  │            │ BIEN ORG   │            │EASY PAWN   │
    │ BH-21100   │            │ MAINFRAME  │            │ EP-01-SN   │
    │192.168.50.x│            │192.168.9.x │            │192.168.30.x│
    │  UDR       │            │ UDM Pro Max│            │   UDR7     │
    └────┬───────┘            └────┬───────┘            └────┬───────┘
         │                         │                         │
         │   UniFi S2S VPN         │   UniFi S2S VPN         │
         └─────────────────────────┼─────────────────────────┘
                                   │
                       ┌───────────┴──────────┐
                       │  Netbird Mesh VPN    │
                       │  Zero-Trust Layer    │
                       │  Tunnel: 10.0.0.0/8  │
                       └───────────┬──────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │ AIP Customer  │      │ AIP Customer  │      │ AIP Customer  │
    │ Box #1        │      │ Box #2        │      │ Box #N        │
    │ Netbird Agent │      │ Netbird Agent │      │ Netbird Agent │
    └───────────────┘      └───────────────┘      └───────────────┘
```

### Connectivity Model

| Mode | Link | Protocol | Purpose | Latency | Cost |
|------|------|----------|---------|---------|------|
| **S2S VPN** | BO ↔ BH ↔ EP | UniFi IPsec | Site-local LAN access | <50ms | Zero (embedded) |
| **Mesh VPN** | All nodes | Netbird WireGuard | Zero-trust overlay, customer isolation | +10-20ms | Free (self-hosted) |
| **Internet** | Any → Public IP | HTTPS/REST | Fallback if VPN down | ISP-dependent | ISP only |

---

## Service Deployment Matrix

### Master Service List

```
SERVICE                 | TIER 0 (BO) | TIER 1 (BH) | TIER 1 (EP) | TIER 2 (AIP) | TYPE | CRITICAL
────────────────────────┼─────────────┼─────────────┼─────────────┼──────────────┼──────┼──────────
Authentik (SSO)         | ✅          | ––          | ––          | ––           | LXC  | YES
Netbird Coordinator     | ✅          | ––          | ––          | ––           | LXC  | YES
PostgreSQL Primary      | ✅          | ––          | ––          | ––           | LXC  | YES
PostgreSQL Replica      | ––          | ✅          | ✅          | ––           | LXC  | HIGH
MinIO (S3 Backup)       | ✅          | ––          | ––          | ––           | LXC  | YES
Restic Server           | ✅          | ––          | ––          | ––           | LXC  | YES
Restic Agent            | ––          | ✅          | ✅          | ✅           | LXC  | HIGH
Prometheus + Grafana    | ✅          | ––          | ––          | ––           | LXC  | HIGH
Uptime Kuma             | ✅          | ––          | ––          | ––           | LXC  | MEDIUM
Wazuh (SIEM)            | ✅ (W4)     | ––          | ––          | ––           | VM   | MEDIUM
Gitea (Code Repos)      | ✅          | ––          | ––          | ––           | LXC  | MEDIUM
n8n (Automation)        | ✅          | ––          | ––          | ––           | LXC  | MEDIUM
Paperless-NGX (Archive) | ✅          | ––          | ––          | ––           | LXC  | LOW
───────────────────────────────────────────────────────────────────────────────────────────────
Ollama (LLM)            | ––          | ✅          | ✅          | ✅           | LXC  | HIGH
Open WebUI (Konstanze)  | ––          | ✅          | ✅          | ✅           | LXC  | HIGH
Odoo (Accounting)       | ––          | ✅ (dev)    | ✅ (prod)   | ✅ (cust)    | LXC  | YES
Bravo POS               | ––          | ––          | ✅ (native) | ––           | App  | YES
Home Assistant          | ––          | ✅ (opt)    | ––          | ✅ (opt)     | LXC  | LOW
PDM (Dev Env)           | ––          | ✅          | ––          | ––           | LXC  | LOW
```

### Key Design Decisions

**Why Ollama/OpenWebUI at EDGE (Tier 1/2), NOT Tier 0:**
- Latency: User expects <200ms response, central adds 50-100ms network overhead
- Bandwidth: Each user query is ~1KB input, ~2KB output (minimal)
- Compute: Can run on modest hardware (no need for R7920)
- Redundancy: Local model works if WAN fails (customer value)

**Why Odoo at TIER 1 (BH/EP) + TIER 2 (AIP), NOT Tier 0 only:**
- Must work offline: Shop can't close if WAN fails
- Data locality: Each site/customer owns their accounting data
- Scaling: 10 AIP boxes = 10 Odoo instances, not one shared database
- Replication: Central PostgreSQL primary, local replicas for fast reads

**Why Authentik/Netbird ONLY on Tier 0:**
- Single source of truth for identity
- Coordinator must be centralized (mesh topology)
- Low latency not critical (cached tokens on clients)
- High availability via HA setup (R7920 + R740xd failover)

---

## Phase 0: Foundation (THIS WEEK — $0 API cost)

### Goal
Get existing infrastructure working, fix critical gaps, prepare for centralized services.

### Tasks

| Day | Task | Location | Duration | Status | Notes |
|-----|------|----------|----------|--------|-------|
| **1 (TODAY)** | Fix EP subnet (192.168.2.x → 192.168.30.x) | EP UniFi | 1h | 🔴 BLOCKED | Critical for S2S VPN |
| **1** | Bring BH ↔ EP VPN online | Both sites | 2h | 🔴 BLOCKED | Depends on subnet fix |
| **2-3** | Deploy Odoo on EP-01-SN | EP Proxmox | 8h | 📋 READY | Import real Easy Pawn data |
| **4** | Test Odoo with real EP data | Easy Pawn | 4h | 📋 READY | Verify accounting integrity |
| **5-7** | Generate bank meeting reports | Odoo BO | 6h | 📋 READY | Use for Q1 review |
| **EOW** | Document current state | GitHub | 2h | 📋 READY | Update this file weekly |

### Deliverables
- ✅ EP subnet fixed (no more 192.168.2.x)
- ✅ BH ↔ EP S2S VPN active, tested
- ✅ Odoo running on EP with live data
- ✅ Bank reports generated from Odoo
- ✅ Infrastructure docs on GitHub

**API Cost:** $0 (no tokens spent)

---

## Phase 1: Bien Org Mainframe Setup (Week 2-3)

### Goal
Stand up Tier 0 central infrastructure, enable all sites to backup/sync.

### Key Services
- Authentik (SSO)
- Netbird Coordinator (zero-trust mesh)
- PostgreSQL Primary (central database)
- MinIO (S3 backup storage)
- Restic Server (backup receiver)

### Deliverables
- ✅ Tier 0 mainframe online (R7920 + R740xd cluster)
- ✅ Authentik SSO working
- ✅ Netbird mesh active (all sites + appliances can join)
- ✅ MinIO + Restic backup pipeline working
- ✅ Site-to-site backup over Netbird tested

**API Cost:** $0 (still self-hosted)

---

## Phase 2: Central Services (Week 3-4)

### Goal
Deploy monitoring, automation, and document archive.

### Key Services
- PostgreSQL Primary (with replication to EP/BH)
- Prometheus + Grafana (monitoring all sites)
- Uptime Kuma (health checks)
- Gitea (code repository)
- n8n (workflow automation)
- Paperless-NGX (document archive)

### Deliverables
- ✅ Central PostgreSQL with replication to all sites
- ✅ Monitoring stack (Prometheus + Grafana) scraping all sites
- ✅ Health checks (Uptime Kuma) on critical services
- ✅ Gitea repo with infrastructure code
- ✅ n8n automation workflows
- ✅ Document archive (Paperless-NGX)

**API Cost:** $0 (still self-hosted)

---

## Phase 3: API Integration (Week 5+)

### Goal
Integrate cloud APIs strategically, keep costs low.

### Budget: $100/month

| API | Allocation | Purpose |
|-----|-----------|---------|
| Claude API | $50 | Code development (Aider, Windsurf) |
| Claude API | $30 | Konstanze fallback (when local Ollama insufficient) |
| Perplexity | $20 | Batch research (not real-time) |
| **Buffer** | **$10** | Unexpected needs, overages |

### Deployment Strategy
1. Get local Ollama working first
2. Use local for 80%+ of queries
3. Fall back to cloud API only when:
   - Local model insufficient (reasoning, coding)
   - Batch research needed
   - Real-time info required (Perplexity)

**Deliverables**
- ✅ Claude API integration in Konstanze (fallback)
- ✅ Perplexity integration for research agents
- ✅ Cost tracking dashboard
- ✅ Documented API budgets per agent

---

## Phase 4: AIP Customer Appliances (Month 2+)

### Goal
Onboard first customer appliance, validate Tier 2 model.

### Hardware
- Mac Mini M4 (2-4 core, 16GB RAM minimum)
- OR Dell Micro (similar specs)

### Services per Appliance
- Ollama (local inference)
- Open WebUI (Konstanze)
- Odoo (customer accounting)
- Netbird Agent (zero-trust VPN)
- Restic Agent (backup to BO)

### Decision Point (Week 8)
Should we scale to 5-10 AIP boxes?
- Cost analysis
- Performance benchmarks
- Customer feedback
- Support burden

---

## What NOT to Install (Yet)

| Tool | Why Wait | When to Install |
|------|---------|-----------------|
| **Wazuh** | Heavy, need Tier 0 first | After Phase 1 stable (W4) |
| **CrowdSec** | Requires stable base | After Authentik + Netbird tested |
| **Rocket.Chat** | Team comms, you're solo | When hiring first team member |
| **ERPNext** | Chose Odoo, don't split | Phase 2 evaluation only |
| **Jellyfin** | Nice-to-have | Phase 4+ (after AIP validated) |
| **Jitsi** | No video meetings needed | When remote team needed |
| **Discourse** | Community forum, not urgent | Phase 5+ |

---

## Runbooks & Procedures

### Emergency: Site WAN Failure

**Symptoms:** S2S VPN tunnel down, services unreachable over WAN

**MTTR Target:** <5 min for local services, <30 min for central sync

**Steps:**
1. Check UniFi controller dashboard (web UI)
2. Verify link status (BH UDR / EP UDR7 → Status page)
3. Restart WAN interface if stuck (UniFi settings)
4. Trigger Netbird re-connect (should establish in <1 min)
5. Test local services (Ollama, Odoo at 192.168.50.x or 192.168.30.x)
6. Alert BO when WAN restored for backup catch-up

**Service Continuity:**
- Ollama: ✅ Works offline
- Open WebUI: ✅ Works offline
- Odoo: ✅ Works offline (syncs on restore)
- Bravo POS: ✅ Works offline
- Authentik: ✅ Cached tokens work ~1 hour

### Emergency: PostgreSQL Primary Failover

**Symptoms:** R7920 PostgreSQL down, replicas can't sync

**MTTR Target:** <15 minutes

**Steps:**
1. Detect failure in Prometheus alert
2. Verify R7920 status (power, network)
3. Manually promote R740xd replica
4. Update replication config on BH/EP replicas
5. Restart Odoo services on all sites
6. Verify backup pipeline to Restic

### Weekly Infrastructure Check

**Every Monday 0900 CST, run:**

```bash
# 1. Verify all Proxmox nodes reachable
for node in bo bh ep; do ping -c 1 $node && echo "$node OK"; done

# 2. Check PostgreSQL replication lag
psql -h 192.168.9.20 -U postgres -c "SELECT slot_name, restart_lsn FROM pg_replication_slots;"

# 3. Verify backups completed (last 24h)
restic -r s3:... ls --last 1d

# 4. Check disk usage (all sites)
df -h / | grep -v Avail

# 5. Review Prometheus alerts
curl -s http://192.168.9.23:9090/api/v1/alerts | jq '.data[] | select(.state=="firing")'
```

---

## Document Control

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-09 | Initial foundation (Phase 0-1 roadmap) |
| 1.1 | TBD | Post-Phase 0 (subnet fixed, VPN tested) |
| 2.0 | TBD | Post-Phase 1 (Tier 0 online) |
| 2.1 | TBD | Post-Phase 2 (central services) |
| 3.0 | TBD | Post-Phase 3 (API integration) |

---

## Quick Reference Commands

```bash
# Proxmox cluster status
pvecm status                    # Cluster quorum
pve-ha-manager status           # HA failover status

# Network
ping 192.168.9.1               # BO gateway
ping 192.168.50.1              # BH gateway
ping 192.168.30.1              # EP gateway

# Netbird
netbird status                 # Mesh connectivity
netbird list peers             # Connected appliances

# PostgreSQL
psql -h 192.168.9.20 -U postgres -l    # List databases
psql -h 192.168.9.20 -U postgres -c "SELECT VERSION();"

# Restic backups
restic -r rest:http://10.x.x.x:8000 snapshots
restic -r rest:http://10.x.x.x:8000 restore <snapshot-id> --target /recovery

# Monitoring
curl http://192.168.9.23:9090/api/v1/targets    # Prometheus scrape targets
curl http://192.168.9.24:3000/api/health        # Grafana health
```

---

**NEXT ACTION: Execute Phase 0 → Fix EP subnet, bring VPN online, deploy Odoo. Update this document weekly.**
