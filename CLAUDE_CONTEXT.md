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

## Tiers and Sites (High-Level Mental Model)

### Tier 0 – Bien Org Mainframe (Central)

- Hardware: Dell R7920 (primary) + R740xd (failover).
- Role: Heavy compute, central services, identity, backups, coordination.
- Key services (examples):
  - Authentik (SSO)
  - Netbird Coordinator (mesh VPN)
  - PostgreSQL Primary
  - MinIO + Restic Server
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

## Networking Model (Essential Facts)

For reasoning about connectivity, Claude should assume:

- LAN ranges:
  - Bien Org (BO): 192.168.9.x
  - Bien Home (BH): 192.168.50.x
  - Easy Pawn (EP): 192.168.30.x (Phase 0 task: migrate from 192.168.2.x).
- VPN:
  - UniFi IPsec site‑to‑site tunnels for stable inter‑site connectivity.
- Mesh:
  - Netbird overlay for zero‑trust, peer‑to‑peer connections across devices and sites.
- Internet:
  - Fallback path if VPN is unavailable.

When generating configs, diagrams, or troubleshooting steps, Claude should align with these ranges and fabrics.

---

## Container & Resource Conventions

Claude should use `CONTAINER_DEPLOYMENT_MATRIX.md` as the authoritative source for:

- Container type (LXC vs VM), host node, CPU/RAM/storage allocation, ports, and backing services.
- Example pattern (Authentik SSO): LXC on primary Proxmox node with 2 vCPU, 4 GB RAM, 50 GB storage, HTTPS on 192.168.9.30:9000, backed by PostgreSQL primary, backed up via Restic.

When proposing new containers or changes, mirror this style and keep resource allocations realistic given Tier and hardware.

---

## Execution Phases & Time Horizon

Claude should anchor planning and task suggestions to the deployment roadmap:

- **Phase 0 – Foundation (Jan 9–17, 2026)**
  - Focus: Fix EP subnet, test S2S VPN, deploy Odoo at EP, generate bank reports, document current state.
  - Cost: $0 additional capex/API spend.

- **Phase 1 – Tier 0 Mainframe (Week 2–3)**
  - Bring Authentik, Netbird Coordinator, PostgreSQL Primary, MinIO, Restic Server online at Tier 0.

- **Phase 2 – Central Services (Week 3–4)**
  - Deploy Prometheus + Grafana, Uptime Kuma, Gitea, n8n, Paperless‑NGX.

- **Phase 3 – API Integration (Week 5+)**
  - Enable Claude API, Perplexity, and related cloud integrations with a budget of about $100/month, used sparingly and strategically.

- **Phase 4 – Customer Appliances (Month 2+)**
  - Pilot and then scale AIP Boxes to customers.

Any auto‑generated task lists or schedules should respect this phase ordering and cost philosophy.

---

## Cost, ROI, and Philosophy

- Capex so far: ~$25K for servers and networking (R7920, R740xd, switches, cabling).
- Opex target: ~$550/month including ISP, power/cooling, and cloud APIs.
- Claude should **optimize for local compute first**, then cloud APIs when they materially add value (complex reasoning, real‑time info, batch jobs).

Guiding principle: **do foundational work well before adding complexity** (e.g., defer heavy SIEM, DDoS, and non‑critical services until Tier 0/1 are stable).

---

## Cloudflare DNS Context for apiaries.ai

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

---

## Proxmox Datacenter Context

### Core Objective (Next 11 Days)

Prepare a clear, professional technical story for a **bank meeting** explaining:

- The current production-ready state of the Apiaries.ai / Bien Organization infrastructure.
- The near-term roadmap for Easy Pawn and other Bien businesses.
- How the platform scales to AIP (Apiaries Infrastructure Platform) customers.

### Critical Current Blocker

- **Subnet conflict**:
  - Easy Pawn VLAN 2: 192.168.2.0/24
  - Bien Home VLAN 2: 192.168.2.0/24
- Impact:
  - BH ↔ EP VPN cannot be reliably established.
  - EP-01-SN cannot be properly onboarded into Proxmox Datacenter Manager.
  - Odoo deployment at Easy Pawn is blocked.
  - AIP pilot validation and multi-site demonstration are blocked.
- Proposed fix:
  - Renumber **Easy Pawn** to 192.168.30.0/24 or 192.168.32.0/24.
  - Estimated work: ~30 minutes of focused networking change.

### Datacenter Manager / Proxmox Overview

- Platform: **Proxmox VE 9.1.4**
- Main node: **BH-21100** (Bien Home)
- Proxmox Datacenter Manager: Version 1.0.2
- Nodes: 1 Virtual Environment node online
- Containers: 12 LXC containers running
- VMs: 0 currently deployed

### Active Container Fleet (BH-21100)

| Container ID | Name | Role |
|--------------|------|------|
| 101 | hive-db | PostgreSQL 16 + pgvector |
| 102 | sentinel | Monitoring / security |
| 103 | gitea | Git repository server |
| 104 | ollama | AI / LLM service |
| 105 | central-agent | Agent orchestration |
| 106 | research-agent | AI research / experimentation |
| 107 | konstanze-core | Core system services |
| 108 | open-webui | Konstanze AI interface |
| 109 | datacenter-manager | Infrastructure management |
| 110 | dns-server | BIND9 DNS |
| 112 | unifi-monitoring | UniFi monitoring |
| 200 | easy-pawn-bh | Easy Pawn business app |

---

## How Claude Should Use This Repo

When assisting with this project, Claude should:

1. Use `APIARY_INFRASTRUCTURE_MASTER.md` for big‑picture architecture, tiers, and service placement.
2. Use `NETWORK_TOPOLOGY.md` for IP plans, VPN paths, and debugging network flows.
3. Use `CONTAINER_DEPLOYMENT_MATRIX.md` for exact container specs and to validate new proposals.
4. Use `PHASE_0_CHECKLIST.md` and later phase checklists as the authoritative task list and status tracker.

Claude's outputs (configs, runbooks, checklists, diagrams described in text) should remain consistent with these documents and phase boundaries.

---

## Storytelling Angles For Bank Meeting

When asked to generate documents, plans, or explanations, Claude should:

- Emphasize:
  - Production-ready **Proxmox-based** platform with central and site tiers.
  - Strong **security and observability** (Wazuh, Prometheus/Grafana, Authentik).
  - **AI-native** design (Ollama, Konstanze, agents) tightly integrated with business workflows.
  - Clear path from **internal deployment** (Bien Org + Easy Pawn) to **customer-facing AIP appliances**.

- Be aware of:
  - The subnet conflict and VPN onboarding as the **short-term technical task**.
  - The bank meeting's need for:
    - Plain-language explanations of architecture.
    - Risk, reliability, and growth story.
    - Concrete, near-term milestones (e.g., fixing EP subnet, onboarding EP-01-SN, deploying Odoo).

---

## What Claude Can Safely Assume

- Donald Trump is the current US president (as of January 2026).
- The Bien Organization operates out of Dexter, Missouri.
- Key entities:
  - Bien Real Estate
  - Bien Construction
  - Easy Pawn
  - Bien Company (core / holding)
- The infrastructure described here is **real** and should be treated as current state, not hypothetical.
