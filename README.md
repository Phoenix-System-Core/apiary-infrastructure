# Apiary.AI Infrastructure

**The living blueprint for distributed AI infrastructure across three sites.** Not everything everywhere—intelligent tiered deployment by latency, resilience, and cost.

**Status:** 🔴 **Phase 0 (Foundation)** - Jan 9-17, 2026

---

## 🎉 What This Is

You said: *"I can't keep it all straight in my mind. We need to map networks, containers, etc."*

**This repo is the answer:** A complete, executable reference system for Apiary.AI that:

✅ **Answers "where does X run?"** (Service Deployment Matrix)
✅ **Shows network paths** (Physical topology, IP scheme, VPN routing)
✅ **Specifies container resources** (CPU, RAM, storage, ports)
✅ **Tracks deployment progress** (Phase 0-4 roadmap with checklists)
✅ **Documents procedures** (Runbooks, troubleshooting, weekly ops)
✅ **Scales from 3 sites to unlimited** (Tier 0 mainframe → Tier 1 sites → Tier 2 appliances)

---

## 🏃 Quick Start

### "How do I know what's deployed where?"

Read [APIARY_INFRASTRUCTURE_MASTER.md](APIARY_INFRASTRUCTURE_MASTER.md) (15 min overview)

**TL;DR:**
```
TIER 0: Bien Org mainframe (central services)
├─ Authentik (SSO)
├─ Netbird Coordinator (mesh VPN)
├─ PostgreSQL Primary (database hub)
├─ MinIO (backup storage)
├─ Restic Server (backup receiver)
├─ Prometheus + Grafana (monitoring)
└─ Support services (n8n, Gitea, Paperless, etc.)

TIER 1: Bien Home + Easy Pawn (site resilience)
├─ Ollama (local LLM inference)
├─ Open WebUI / Konstanze (chat)
├─ Odoo (accounting, works offline)
├─ PostgreSQL Replica (read-only sync from Tier 0)
└─ Restic Agent (backup client)

TIER 2: Customer appliances (minimal footprint, future)
├─ Ollama (local)
├─ OpenWebUI (chat)
├─ Odoo (per-customer accounting)
├─ Netbird Agent (connects to Tier 0)
└─ Restic Agent (backup home)
```

### "What's the network topology?"

Read [NETWORK_TOPOLOGY.md](NETWORK_TOPOLOGY.md) (visual + IP assignments)

**Key paths:**
- **LAN:** 192.168.9.x (BO) ↔ 192.168.50.x (BH) ↔ 192.168.30.x (EP)
- **VPN:** UniFi IPsec S2S tunnels (reliable infrastructure)
- **Mesh:** Netbird overlay (zero-trust, peer-to-peer)
- **Internet:** Fallback if VPN down

### "What resources does X need?"

Read [CONTAINER_DEPLOYMENT_MATRIX.md](CONTAINER_DEPLOYMENT_MATRIX.md) (detailed specs)

**Example:**
```
Authentik (SSO)
├─ Type: LXC Container
├─ Host: Dell R7920 (Proxmox primary)
├─ CPU: 2 vCPU | RAM: 4GB | Storage: 50GB
├─ Network: 192.168.9.30:9000 (HTTPS)
├─ Backend: PostgreSQL primary (central)
└─ Backup: Daily via Restic
```

### "What do I do today? This week?"

Read [PHASE_0_CHECKLIST.md](PHASE_0_CHECKLIST.md) (step-by-step execution)

**Phase 0 Goals (Jan 9-17):**
- [ ] Fix EP subnet (192.168.2.x → 192.168.30.x)
- [ ] Test S2S VPN tunnels (BH ↔ EP)
- [ ] Deploy Odoo on EP with real data
- [ ] Generate bank meeting reports
- [ ] Document current state

**Cost:** $0 (no new hardware, no API tokens)

---

## 📄 Document Map

| Document | Purpose | Read When |
|----------|---------|----------|
| **APIARY_INFRASTRUCTURE_MASTER.md** | Overall architecture, tier definitions, service matrix | Need big-picture overview |
| **NETWORK_TOPOLOGY.md** | Physical paths, IP addressing, VPN routing, DNS | Troubleshooting network issues |
| **CONTAINER_DEPLOYMENT_MATRIX.md** | Specs for every container (CPU/RAM/storage/ports) | Planning deployments |
| **PHASE_0_CHECKLIST.md** | Daily task breakdown for foundation phase | Executing Phase 0 |
| **README.md** (this file) | Navigation hub | First time here? |

---

## 🚰 The Three Tiers Explained

### Tier 0: Central Mainframe (Bien Org)

**Hardware:** Dell R7920 (primary) + R740xd (failover)  
**Purpose:** Heavy compute, central services, auth, backups, coordination  
**Uptime SLA:** 99.5% (redundant power, cooling, UPS)  
**Cost:** ~$30/month equivalent (CPU/RAM allocation) + hardware capex

**Why centralized?**
- Authentik (SSO) = single source of truth for identity
- PostgreSQL Primary = consistent writes (can't replicate a primary)
- Netbird Coordinator = mesh orchestration (must be central)
- MinIO + Restic = backup hub (all other sites feed into this)

**Why NOT here?**
- ❌ Ollama (latency penalty for chat, add 50-100ms)
- ❌ Odoo (must work if WAN fails, critical for POS)
- ❌ Home Assistant (IoT latency critical)

---

### Tier 1: Site Servers (Bien Home, Easy Pawn)

**Hardware:** BH-21100, EP-01-SN (existing Proxmox nodes)  
**Purpose:** Site-local services, resilience (survives WAN failure)  
**Uptime SLA:** 99% per site  
**Cost:** ~$5/month per site (ISP, power)

**Why local?**
- Ollama: User expects <200ms response, local = 5-10ms (vs 50+ ms over VPN)
- Odoo: Pawn shop can't close if internet goes down
- PostgreSQL Replica: Local reads fast, replication from BO for consistency

**Disaster scenario:**
- Internet down at EP? Bravo POS keeps working (offline mode)
- Odoo keeps working (local database)
- Ollama/Konstanze keep working (local inference)
- On reconnect: automatically sync to BO backups

---

### Tier 2: Customer Appliances (AIP Boxes) - Future

**Hardware:** Mac Mini M4 or Dell Micro (2-4 core, 16GB RAM)  
**Purpose:** Minimal footprint, "phones home" to Tier 0  
**Uptime SLA:** Best effort (non-critical customer device)  
**Cost:** ~$400-500 hardware + $5/month cloud backup per box

**Design:**
- Each customer gets their own mini-appliance
- Ollama runs locally (offline AI chat)
- Odoo instance (customer-specific accounting)
- Netbird agent (secure tunnel to BO)
- All customers isolated (Netbird peer rules prevent cross-talk)

**Why separate appliances, not shared Tier 1?**
- Scaling: 10 customers = 10 independent boxes, not shared overload
- Data privacy: Customer data never touches other customer hardware
- Resilience: One customer issue doesn't affect others

---

## 🗐️ Deployment Roadmap

### Phase 0: Foundation (THIS WEEK)
**Status:** 🔴 Not started  
**Duration:** Jan 9-17 (1 week)  
**API Cost:** $0

- Fix EP subnet
- Test S2S VPN
- Deploy Odoo on EP
- Generate bank reports
- Document current state

👉 See [PHASE_0_CHECKLIST.md](PHASE_0_CHECKLIST.md) for daily tasks

---

### Phase 1: Tier 0 Mainframe (Week 2-3)
**Status:** 🔵 Planning  
**Duration:** Jan 20-Feb 3 (2 weeks)  
**API Cost:** $0

**Services:**
- Authentik (SSO)
- Netbird Coordinator (mesh VPN)
- PostgreSQL Primary (central database)
- MinIO (backup storage)
- Restic Server (backup receiver)

**Deliverables:**
- Tier 0 mainframe online
- All sites can backup to central Restic
- Netbird mesh active (zero-trust VPN)

👉 See APIARY_INFRASTRUCTURE_MASTER.md → Phase 1 Roadmap

---

### Phase 2: Central Services (Week 3-4)
**Status:** 🔵 Planning  
**Duration:** Feb 3-17 (2 weeks)  
**API Cost:** $0

**Services:**
- Prometheus + Grafana (monitoring)
- Uptime Kuma (health checks)
- Gitea (code repos)
- n8n (automation)
- Paperless-NGX (document archive)

**Deliverables:**
- Full observability (metrics from all sites)
- Automated workflows
- Centralized audit trail

---

### Phase 3: API Integration (Week 5+)
**Status:** 🔵 Planning  
**Duration:** Feb 17+ (ongoing)  
**API Cost:** $100/month (strategic, not excessive)

**Integrations:**
- Claude API (Aider/Windsurf code development)
- Claude API (Konstanze fallback when local Ollama insufficient)
- Perplexity (batch research queries)

**Strategy:** Use local Ollama for 80%+ of queries, cloud API only for:
- Complex reasoning (code, math)
- Real-time information (Perplexity)
- Batch processing (save tokens)

---

### Phase 4: Customer Appliances (Month 2+)
**Status:** 🔵 Planning  
**Duration:** Feb+ (pilot, then scale)
**API Cost:** Incremental (per-appliance)

**First AIP Box:**
- Pilot with real customer
- Measure cost/performance/support burden
- Document onboarding
- Decide: scale to 5-10, or iterate design?

---

## 🗑️ How to Use This Repo

### First Time?
1. Read this README (5 min)
2. Skim APIARY_INFRASTRUCTURE_MASTER.md (10 min)
3. Jump to PHASE_0_CHECKLIST.md (if executing)

### Daily Operations?
- Update PHASE_0_CHECKLIST.md (progress log)
- Reference CONTAINER_DEPLOYMENT_MATRIX.md (specs)
- Check NETWORK_TOPOLOGY.md (if networking issues)

### Planning Next Phase?
- APIARY_INFRASTRUCTURE_MASTER.md → Phase X Roadmap section
- Cross-ref with CONTAINER_DEPLOYMENT_MATRIX.md (resource planning)
- Update checklist, commit to main branch

### Troubleshooting?
- Network issue? Check NETWORK_TOPOLOGY.md (IP scheme, VPN routing)
- Container down? Check CONTAINER_DEPLOYMENT_MATRIX.md (specs, ports)
- Service missing? Check APIARY_INFRASTRUCTURE_MASTER.md (what goes where)
- Recovery procedure? Check runbooks section

---

## 📦 What's NOT Here (Yet)

**Intentionally deferred to Phase 2+:**
- ❌ Wazuh (SIEM) - too heavy for Phase 0, need Tier 0 stable
- ❌ CrowdSec (DDoS protection) - premature
- ❌ Rocket.Chat (team comms) - you're solo right now
- ❌ ERPNext (alternative ERP) - already using Odoo, no split focus
- ❌ Jellyfin (media server) - nice-to-have, not critical path

**Philosophy:** Do foundational well before adding complexity.

---

## 🔐 Access & Security

**This repo:** Public (infrastructure docs, no secrets)

**Secrets stored separately:**
- PostgreSQL passwords → Password manager (Bitwarden, 1Password)
- API keys → Environment variables (`.env` files, not in Git)
- TLS certs → Ansible vault or Secrets manager
- PSK for VPN tunnels → UniFi controller (encrypted at rest)

**GitOps philosophy:** Infrastructure code in Git, secrets out of Git

---

## 🦁 Contributors

- **Eric Bien** - Architecture, implementation, documentation
- **Apiary.AI team** - TBD (when hiring)

---

## 🎆 Key Metrics

**Capex (One-time):**
- R7920: ~$15K
- R740xd: ~$5K
- Networking (switches, cabling): ~$3K
- Total Phase 0-1: ~$25K

**Opex (Monthly):**
- ISP (3 sites): ~$300
- Power/cooling (BO): ~$150
- Cloud API (Phase 3+): $100 (capped)
- Total: ~$550/month (~$6.6K/year)

**ROI:** 
- BH/EP Odoo automates accounting ($5K/year CPA time saved?)
- BO Authentik handles 100s of identities (future customer scale)
- Netbird mesh + backups prevent data loss (invaluable)

---

## 📁 Version History

| Version | Date | Status | Changes |
|---------|------|--------|----------|
| 1.0 | 2026-01-09 | Phase 0 | Initial foundation (master doc, network, containers, checklist) |
| 1.1 | TBD | Phase 0 | Post-execution (subnet fixed, VPN tested, Odoo deployed) |
| 2.0 | TBD | Phase 1 | Tier 0 online (Authentik, Netbird, PostgreSQL, backups) |
| 2.1 | TBD | Phase 2 | Central services (monitoring, automation, archive) |
| 3.0 | TBD | Phase 3 | API integration (Claude, Perplexity) |
| 4.0 | TBD | Phase 4+ | AIP appliances (customer scale) |

---

## 특 Next Steps

**Right now (Jan 9):**
1. Read this README
2. Read APIARY_INFRASTRUCTURE_MASTER.md (big picture)
3. Open PHASE_0_CHECKLIST.md in editor
4. Start Task 1.1 (fix EP subnet)

**By Jan 17:**
- ✅ Phase 0 complete
- ✅ All docs updated
- ✅ Bank ready for Q1 review

**By Feb 17:**
- ✅ Phase 1-2 complete
- ✅ Central infrastructure online
- ✅ Monitoring + automation working

---

**Status:** 🔴 Phase 0 Foundation - Ready to Execute  
**Last Updated:** 2026-01-09 09:12 CST  
**Next Update:** After Phase 0 completion (Jan 17)

[Go to APIARY_INFRASTRUCTURE_MASTER.md](APIARY_INFRASTRUCTURE_MASTER.md) →
