# Apiary.AI Infrastructure – Claude Context

This repository is the **living blueprint** for the Apiary.AI distributed infrastructure across three tiers and three primary sites (Bien Org, Bien Home, Easy Pawn). It is the source of truth for where services run, how networks connect, what resources containers need, and how deployment progresses over time.

---

## Core Purpose

- Answer "where does X run?" via the service deployment matrix in `APIARY_INFRASTRUCTURE_MASTER.md`.
- Show network paths and IP schemes in `NETWORK_TOPOLOGY.md`.
- Specify container resources and placements in `CONTAINER_DEPLOYMENT_MATRIX.md`.
- Track execution with phase roadmaps and `PHASE_0_CHECKLIST.md`.

Claude should treat this repo as the canonical infrastructure and operations reference for Apiary.AI.

---

## 1. High-Level Purpose

- Apiaries.ai / **Bienenstaat** is a distributed AI orchestration and automation platform for The Bien Organization (Bien Real Estate, Bien Construction, Easy Pawn, Bien Company).
- It implements a "Queen and Swarm" model: centralized governance plus site-level and future customer appliances for AI-driven operations.

---

## 2. Tiers and Sites (High-Level Mental Model)

### Tier 0 – Bien Org Mainframe (Central)

- Hardware: Dell R7920 (primary) + R740xd (failover).
- Role: Heavy compute, central services, identity, backups, coordination.
- Key services (examples):
  - Authentik (SSO)
  - Netbird Coordinator (mesh VPN)
  - PostgreSQL Primary
  - MinIO + Restic Server
  - Prometheus + Grafana (metrics and observability)
  - Wazuh SIEM (security monitoring and event correlation)
  - Supporting services like n8n, Gitea, Paperless, etc.

**Design intent:** Centralized, highly available backbone. Do **not** place latency‑sensitive, WAN‑fragile services (e.g., Ollama, Odoo, Home Assistant) here.

### Tier 1 – Site Servers (Bien Home, Easy Pawn)

- Hardware: Existing Proxmox nodes (BH-21100, EP-01-SN).
- Role: Site-local services that must survive WAN/ISP failure (e.g., pawn shop operations).
- Typical services:
  - Ollama + Open WebUI/Konstanze (local LLM/chat)
  - Odoo (local accounting / POS)
  - PostgreSQL replica
  - Restic agent

**Design intent:** Low-latency user experience and business continuity if the internet is down.

### Tier 2 – Customer Appliances (Future AIP Boxes)

- Hardware: Mac Mini M4 or Dell Micro, 2–4 cores, 16 GB RAM.
- Role: Per-customer mini-appliance that "phones home" to Tier 0 over Netbird.
- Services:
  - Local Ollama + OpenWebUI
  - Customer-scoped Odoo instance
  - Netbird agent
  - Restic agent

**Design intent:** Isolation, scalability, and privacy: each customer has a dedicated box; no cross‑customer data sharing.

---

## 3. Proxmox Host Overview (BH-21100)

### Node Summary

- **Node name:** BH-21100
- **Platform:** Proxmox VE 9.1.4
- **Uptime:** 7 days 2 hours (as of Jan 10, 2026)
- **Host resources:**
  - Disk used: 53%
  - Memory used: 29.6%
  - CPU used: 7.8% of 12 cores

### Storage Pools

| Storage | Used |
|---------|------|
| backup-plus | 7.8% |
| local | 53% |
| local-lvm | 34.7% |
| local-vz | 53% |

### Running Containers (LXC) with Live Metrics

| CT ID | Name | Role | vCPUs | CPU% | RAM% | Disk% | Uptime |
|-------|------|------|-------|------|------|-------|--------|
| 101 | hive-db | PostgreSQL 16 + pgvector | 2 | 0.0% | 1.6% | 41.3% | 5d 7h |
| 102 | sentinel | Monitoring / security | 1 | 0.2% | 5.6% | 9.0% | 3d 2h |
| 103 | gitea | Git repository server | 1 | 0.1% | 7.5% | 9.2% | 5d 6h |
| 104 | ollama | AI / LLM backend | 4 | 0.1% | 1.8% | 61.3% | 5d 7h |
| 105 | central-agent | Agent orchestration | 4 | 0.0% | 4.5% | 5.5% | 4d 8h |
| 106 | research-agent | Research/automation agent | 4 | 0.0% | 5.2% | 4.7% | 3d 2h |
| 107 | konstanze-core | Core orchestration/service layer | 6 | 0.0% | 3.2% | 2.4% | 4d 8h |
| 108 | open-webui | Web UI for AI/LLM | 4 | 0.2% | 59.0% | 73.0% | 1h* |
| 109 | datacenter-manager | Datacenter/infra management | 4 | 0.0% | 4.0% | 6.2% | 2d 12h |
| 110 | dns-server | BIND9 DNS service | 2 | 0.0% | 1.8% | 9.0% | 2d 1h |
| 112 | unifi-monitoring | UniFi network monitoring | 2 | 0.3% | 15.6% | 16.4% | 3d 2h |
| 200 | easy-pawn-bh | Easy Pawn application | 4 | 0.0% | 9.0% | 1.5% | 1h* |

*Recent restarts visible in task log.

### Recent Activity Notes

- Recent restarts of CT 108 (open-webui) and CT 200 (easy-pawn-bh) visible in task log.
- Periodic package database updates and shell sessions recorded.
- System health: all 12 containers running, no task errors in last 48 hours.
- Only notable warning: missing Proxmox subscription (cosmetic, does not block functionality).
- Storage note: Bienenstaat infra is built on a 2TB drive attached to BH-21100; may need to be rediscovered and re-added as storage.

---

## 4. Networking Model (Essential Facts)

For reasoning about connectivity, Claude should assume:

### LAN Ranges

- Bien Org (BO): 192.168.9.x
- Bien Home (BH): 192.168.50.x
- Easy Pawn (EP): 192.168.30.x (Phase 0 task: migrate from 192.168.2.x)

### VPN & Mesh

- **VPN:** UniFi IPsec site‑to‑site tunnels for stable inter‑site connectivity.
- **Mesh:** Netbird overlay for zero‑trust, peer‑to‑peer connections across devices and sites.
- **Internet:** Fallback path if VPN is unavailable.

When generating configs, diagrams, or troubleshooting steps, Claude should align with these ranges and fabrics.

### Critical Current Blocker

- **Subnet conflict:**
  - Easy Pawn VLAN 2: 192.168.2.0/24
  - Bien Home VLAN 2: 192.168.2.0/24
- **Impact:**
  - BH ↔ EP VPN cannot be reliably established.
  - EP-01-SN cannot be properly onboarded into Proxmox Datacenter Manager.
  - Odoo deployment at Easy Pawn is blocked.
  - AIP pilot validation and multi-site demonstration are blocked.
- **Proposed fix:**
  - Renumber **Easy Pawn** to 192.168.30.0/24 or 192.168.32.0/24.
  - Estimated work: ~30 minutes of focused networking change.

### Subnet Migration Procedure (Easy Pawn)

Planned concrete steps:

1. Log into UniFi controller (UDM Pro Max) at `https://192.168.9.1/`.
2. Change EP VLAN network from `192.168.2.0/24` → `192.168.30.0/24`.
3. Assign static IP `192.168.30.10` to EP-01-SN Proxmox node.
4. Set DHCP pool to `192.168.30.100–200`.
5. If issues occur, rollback by reverting to `192.168.2.0/24` and rebooting (recommended during low-traffic hours).

---

## 5. Container & Resource Conventions

Claude should use `CONTAINER_DEPLOYMENT_MATRIX.md` as the authoritative source for:

- Container type (LXC vs VM), host node, CPU/RAM/storage allocation, ports, and backing services.
- Example pattern (Authentik SSO): LXC on primary Proxmox node with 2 vCPU, 4 GB RAM, 50 GB storage, HTTPS on 192.168.9.30:9000, backed by PostgreSQL primary, backed up via Restic.

When proposing new containers or changes, mirror this style and keep resource allocations realistic given Tier and hardware.

---

## 6. Execution Phases & Time Horizon

Claude should anchor planning and task suggestions to the deployment roadmap:

### Phase 0 – Foundation (Jan 9–17, 2026)

**Current Status:** Day 2 (Saturday, Jan 10) with 11 days until the Jan 21 bank meeting.

**Focus:** Fix EP subnet, test S2S VPN, deploy Odoo at EP, generate bank reports, document current state.

**Cost:** $0 additional capex/API spend.

**Timeline Snapshot:**

| Day | Date | Tasks |
|-----|------|-------|
| 1 | Jan 9 (Thu) | Task 1.1: subnet fix (EP renumber), Task 1.2: BH ↔ EP VPN configuration |
| 2 | Jan 10 (Sat) | Task 2.1: Odoo preparation at Easy Pawn |
| 3 | Jan 11 (Sun) | Complete 2.1, start 2.2 (data import into Odoo / reporting pipeline) |
| 4 | Jan 12 (Mon) | Generate reports for Jan 21 bank meeting |

**Phase 0 Success Criteria:**

- [ ] EP subnet renumbered to 192.168.30.x
- [ ] BH ↔ BO and EP ↔ BO VPN tunnels are online (Netbird / UniFi)
- [ ] Odoo deployed on EP-01-SN, expected endpoint 192.168.30.30:8069
- [ ] Real Easy Pawn operational data imported into Odoo
- [ ] Financial reports generated in time for Jan 21 bank meeting
- [ ] All relevant docs and configs committed to GitHub

### Phase 1 – Tier 0 Mainframe (Week 2–3)

- Bring Authentik, Netbird Coordinator, PostgreSQL Primary, MinIO, Restic Server online at Tier 0.

### Phase 2 – Central Services (Week 3–4)

- Deploy Prometheus + Grafana, Uptime Kuma, Gitea, n8n, Paperless‑NGX.

### Phase 3 – API Integration (Week 5+)

- Enable Claude API, Perplexity, and related cloud integrations with a budget of about $100/month, used sparingly and strategically.

### Phase 4 – Customer Appliances (Month 2+)

- Pilot and then scale AIP Boxes to customers.

Any auto‑generated task lists or schedules should respect this phase ordering and cost philosophy.

---

## 7. Cost, ROI, and Philosophy

- Capex so far: ~$25K for servers and networking (R7920, R740xd, switches, cabling).
- Opex target: ~$550/month including ISP, power/cooling, and cloud APIs.
- Claude should **optimize for local compute first**, then cloud APIs when they materially add value (complex reasoning, real‑time info, batch jobs).

Guiding principle: **do foundational work well before adding complexity** (e.g., defer heavy SIEM, DDoS, and non‑critical services until Tier 0/1 are stable).

---

## 8. Cloudflare DNS Context for apiaries.ai

### Account & Domain

- Cloudflare account email: Administrator@bien.company
- Managed domain: apiaries.ai
- Console section: DNS → Records

### Current DNS Records

All records are managed through Cloudflare DNS with automatic TTL and standard proxy settings unless otherwise noted.

#### A Records

- `api.apiaries.ai` → 192.0.2.1
- `gitea.apiaries.ai` → 192.0.2.1
- `easypawn.apiaries.ai` → 192.0.2.1
- `konstanze.apiaries.ai` → 192.0.2.1
- `hq.apiaries.ai` → 192.0.2.1
- `apiaries.ai` (root) → 192.0.2.1

#### CNAME Records

- `www.apiaries.ai` → apiaries.ai

### Nameservers

- cheryl.ns.cloudflare.com
- lamar.ns.cloudflare.com

### Email & Security Notes

- No MX records currently configured for apiaries.ai (email to @apiaries.ai will not be delivered).
- Recommendations:
  - Add MX records if email service is needed.
  - Add SPF, DKIM, and DMARC records if email is not used, to prevent spoofing.
- Defensive example (domain sends no mail):
  - TXT `@`: `v=spf1 -all`
  - TXT `*._domainkey`: `v=DKIM1; p=` (locked down)
  - TXT `_dmarc`: `v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s;`

### Planned Future DNS Entries

- `odoo.apiaries.ai` → Odoo ERP (after EP deployment)
- `auth.apiaries.ai` → Authentik SSO at Tier 0
- `monitor.apiaries.ai` → Prometheus / Grafana dashboards

---

## 9. GitHub Repositories & Docs

- Primary infra repo: `Phoenix-System-Core/apiary-infrastructure`
- Key files (approx sizes):
  - `APIARY_INFRASTRUCTURE_MASTER.md` (~20 KB) – master architecture
  - `CONTAINER_DEPLOYMENT_MATRIX.md` (~21 KB) – every container, image, ports, resources
  - `NETWORK_TOPOLOGY.md` (~17 KB) – VLANs, IP plans, inter-site routing
  - `PHASE_0_CHECKLIST.md` (~14 KB) – execution plan, status flags, runbooks
  - `README.md` (~11 KB) – quick reference to the repo

---

## 10. Operational To-Dos For Claude

When Claude acts as an infra/ops copilot, it should:

1. **Treat the EP subnet renumber + VPN as the top-priority unblocker** for all other work.
2. Use the container list and roles above as the ground truth for what is currently running on BH-21100.
3. Refer to `PHASE_0_CHECKLIST.md` for day-by-day tasks and to quickly assess whether actions are BLOCKED, IN-PROGRESS, or DONE.
4. Assume Cloudflare is fronting services (192.0.2.1 as placeholder) and that internal reachability is primarily governed by Netbird + UniFi VLANs.
5. When giving command examples, target:
   - Proxmox host shell (for storage, containers, cluster)
   - UniFi network controller for VLAN/subnet changes
   - Git workflows for `Phoenix-System-Core/apiary-infrastructure` and related repos.

---

## 11. How Claude Should Use This Repo

When assisting with this project, Claude should:

1. Use `APIARY_INFRASTRUCTURE_MASTER.md` for big‑picture architecture, tiers, and service placement.
2. Use `NETWORK_TOPOLOGY.md` for IP plans, VPN paths, and debugging network flows.
3. Use `CONTAINER_DEPLOYMENT_MATRIX.md` for exact container specs and to validate new proposals.
4. Use `PHASE_0_CHECKLIST.md` and later phase checklists as the authoritative task list and status tracker.

Claude's outputs (configs, runbooks, checklists, diagrams described in text) should remain consistent with these documents and phase boundaries.

---

## 12. Storytelling Angles For Bank Meeting

When asked to generate documents, plans, or explanations, Claude should:

### Emphasize:

- Production-ready **Proxmox-based** platform with central and site tiers.
- Strong **security and observability** (Wazuh, Prometheus/Grafana, Authentik).
- **AI-native** design (Ollama, Konstanze, agents) tightly integrated with business workflows.
- Clear path from **internal deployment** (Bien Org + Easy Pawn) to **customer-facing AIP appliances**.

### Be Aware Of:

- The subnet conflict and VPN onboarding as the **short-term technical task**.
- The bank meeting's need for:
  - Plain-language explanations of architecture.
  - Risk, reliability, and growth story.
  - Concrete, near-term milestones (e.g., fixing EP subnet, onboarding EP-01-SN, deploying Odoo).

---

## 13. What Claude Can Safely Assume

- Donald Trump is the current US president (as of January 2026).
- The Bien Organization operates out of Dexter, Missouri.
- Key entities:
  - Bien Real Estate
  - Bien Construction
  - Easy Pawn
  - Bien Company (core / holding)
- The infrastructure described here is **real** and should be treated as current state, not hypothetical.
