# Apiary.AI Container Deployment Matrix

**Purpose:** Canonical source for what runs where, how much it costs (CPU/RAM/storage), and when

**Status:** Phase 0-2 deployment roadmap

---

## Tier 0: Bien Org Mainframe (Dell R7920 + R740xd)

**Host Specs:**
- R7920: 2× Xeon Platinum, 256GB RAM, SSD+HDD
- R740xd: 2× Xeon Gold, 128GB RAM, SSD
- Network: 1Gbps copper to UDM Pro Max, 10Gbps SFP uplink (future)
- Power: Dual PSU 1400W ea (redundant)
- Cluster: Proxmox HA-enabled (R7920 primary, R740xd secondary)

---

### BO-1: Authentik (SSO)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Lightweight, fast boot |
| **Host** | Dell R7920 (primary) | Failover to R740xd if needed |
| **Image** | Ubuntu 22.04 LTS | Standard Proxmox template |
| **CPU** | 2 vCPU (pinned) | SSO not CPU-intensive |
| **RAM** | 4 GB | Headroom for caching |
| **Storage** | 50 GB (SSD) | Config + user database |
| **Network** | vmbr0 (bridge) | 192.168.9.30 (static) |
| **Ports** | 8000 (HTTP), 9000 (HTTPS) | Redirect HTTP → HTTPS |
| **Backend** | PostgreSQL Primary (192.168.9.20) | Central auth database |
| **Backup** | Daily via Restic | Nightly 0100 CST |
| **Startup** | Auto (always-on) | Essential service |
| **Persistent Dirs** | /var/lib/authentik/ | Mount from host or PVC |
| **Status** | Phase 1 deployment (W2) | |
| **Cost** | 2vCPU + 4GB RAM = ~$8/month equivalent |

**Deployment Script:**
```bash
#!/bin/bash
# pct create 101 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
#   -hostname authentik -cores 2 -memory 4096 -storage local -swap 0 -net0 name=eth0,bridge=vmbr0,ip=192.168.9.30/24,gw=192.168.9.1
#
# Inside container:
# apt-get update && apt-get install -y authentik
# systemctl enable authentik
```

**Access:** https://192.168.9.30:9000/admin (after deployment)

---

### BO-2: Netbird Coordinator

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Self-contained, no external deps |
| **Host** | Dell R7920 | Failover to R740xd |
| **Image** | Ubuntu 22.04 LTS | |
| **CPU** | 1 vCPU | Low utilization (coordination only) |
| **RAM** | 2 GB | Peer metadata in memory |
| **Storage** | 20 GB (SSD) | SQLite database |
| **Network** | vmbr0 (bridge) | 192.168.9.31 (static) |
| **Ports** | 80 (HTTP API), 443 (HTTPS) | Management API |
| **Backend** | SQLite (local) | Self-hosted, no external DB |
| **Backup** | Daily via Restic | Config + peer list |
| **Startup** | Auto (critical) | Mesh won't function without it |
| **Persistent Dirs** | /var/lib/netbird/ | Peer registry |
| **Status** | Phase 1 deployment (W2) | |

**Config:**
```yaml
# /etc/netbird/setup.yaml
ManagementAPI:
  Address: 192.168.9.31:443
  TLSCertFile: /etc/netbird/certs/management.crt
  TLSKeyFile: /etc/netbird/certs/management.key

SignalAPI:
  Address: 192.168.9.31:443

OIDC:
  IssuerURL: https://192.168.9.30:9000  # Authentik
  ClientID: netbird
  ClientSecret: <secret>
  Audience: netbird

RelayServers:
  - HostedNAT:
    - IP: 192.168.9.31
      Port: 51820  # WireGuard listen port
```

---

### BO-3: PostgreSQL Primary

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Direct SSD access |
| **Host** | Dell R7920 | Primary (R740xd = failover) |
| **Image** | Ubuntu 22.04 LTS with PostgreSQL 15 | Streaming replication |
| **CPU** | 4 vCPU | Index operations, archival |
| **RAM** | 16 GB | Shared buffers = 4GB, effective_cache_size = 12GB |
| **Storage** | 500 GB SSD | WAL + data files |
| **Network** | vmbr0 (bridge) | 192.168.9.20 (internal only, no world access) |
| **Ports** | 5432 (PostgreSQL) | Restricted to replicas + Authentik |
| **Databases** | authentik, iron_honeycomb, odoo_bo | Multiple databases for isolation |
| **Backup** | Continuous archiving to MinIO | WAL segments + daily snapshots |
| **Replication** | Streaming to BH-50 (192.168.50.22) and EP-30 (192.168.30.22) | Async replication (RPO ~1min) |
| **Startup** | Auto (critical) | Central auth + business data |
| **Persistent Dirs** | /var/lib/postgresql/15/main/ | Mount from host SSD pool |
| **Status** | Phase 1 deployment (W3) | |
| **Cost** | 4vCPU + 16GB RAM = ~$30/month equivalent |

**Init SQL:**
```sql
-- Create replication role
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica_pass';

-- Create databases
CREATE DATABASE authentik;
CREATE DATABASE iron_honeycomb;
CREATE DATABASE odoo_bo;

-- Grant access
GRANT ALL PRIVILEGES ON DATABASE authentik TO authentik_user;
GRANT ALL PRIVILEGES ON DATABASE iron_honeycomb TO honeycomb_user;
GRANT ALL PRIVILEGES ON DATABASE odoo_bo TO odoo_user;

-- Enable WAL archiving
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 3;
ALTER SYSTEM SET max_replication_slots = 3;
ALTER SYSTEM SET wal_keep_size = '1GB';
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'aws s3 cp %p s3://apiary-backups/wal/%f';
SELECT pg_ctl_reload_conf();
```

**Replica Config (BH/EP):**
```bash
# On replica container
pg_basebackup -h 192.168.9.20 -U replicator -D /var/lib/postgresql/15/main -Fp -Xs -v
echo "standby_mode = 'on'" >> /var/lib/postgresql/15/main/recovery.conf
chown postgres:postgres /var/lib/postgresql/15/main/recovery.conf
systemctl start postgresql
```

---

### BO-4: MinIO (S3-Compatible Backup Storage)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Direct block storage |
| **Host** | Dell R7920 | Failover to external NAS |
| **Image** | Ubuntu 22.04 + MinIO binary | S3-compatible API |
| **CPU** | 2 vCPU | I/O bound, not CPU-bound |
| **RAM** | 8 GB | Caching + metadata |
| **Storage** | 2 TB SSD | Initial capacity (expandable) |
| **Network** | vmbr0 (bridge) | 192.168.9.21 (static) |
| **Ports** | 9000 (API), 9001 (Console) | Reverse proxy recommended |
| **Buckets** | backups/, archives/, media/ | Lifecycle policies, versioning |
| **Backup** | Not backed up (primary storage) | Replicate to offsite (Phase 2+) |
| **Startup** | Auto | Needed for Restic pipeline |
| **Persistent Dirs** | /minio/data/ | Mount from SSD pool |
| **Status** | Phase 1 deployment (W3) | |

**Init:**
```bash
# Inside container
minio server /minio/data --console-address :9001 &
minio admin user add <username> <password>
minio mb minio/backups
minio mb minio/archives
minio mb minio/media
```

**Access:** https://192.168.9.21:9001 (console)

---

### BO-5: Restic Server

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | REST API wrapper around Restic |
| **Host** | Dell R7920 | Failover to R740xd |
| **Image** | Ubuntu 22.04 + Restic + rest-server | |
| **CPU** | 1 vCPU | Mostly idle (nightly backups) |
| **RAM** | 4 GB | Deduplication metadata |
| **Storage** | 500 GB SSD | Backup repository (incremental) |
| **Network** | vmbr0 (bridge) | 192.168.9.22 (static) |
| **Ports** | 8000 (REST API) | Restic client → server |
| **Backend** | MinIO (s3://backups/) | Offsite repository storage |
| **Backup Schedule** | Nightly 0100-0300 CST | Incremental, ~5GB/night |
| **Retention** | 30 daily, 12 weekly, 12 monthly | Automated cleanup |
| **Startup** | Auto | Backup reception |
| **Status** | Phase 1 deployment (W3) | |

**Config:**
```bash
# /etc/restic/server.conf
[rest]
address = 0.0.0.0:8000
path = /minio/backups/restic

# /var/lib/restic/.env (agent config on clients)
RESTC_REPOSITORY=rest:http://192.168.9.22:8000/
RESTC_PASSWORD=<repo-password>
```

---

### BO-6: Prometheus + Grafana

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container (dual) | Can split if needed |
| **Host** | Dell R7920 | Split to R740xd if memory pressure |
| **Image** | Ubuntu 22.04 + Prometheus + Grafana | |
| **CPU** | 2 vCPU (Prometheus) + 2 vCPU (Grafana) | ~4 vCPU total |
| **RAM** | 8 GB (Prometheus) + 4 GB (Grafana) | ~12 GB total |
| **Storage** | 100 GB SSD | 30-day retention |
| **Network** | vmbr0 (bridge) | Prometheus 192.168.9.23:9090, Grafana 192.168.9.24:3000 |
| **Scrape Targets** | All Proxmox nodes, MinIO, Restic, PostgreSQL | 30s interval |
| **Dashboards** | Infrastructure, Backups, Database, Network | Pre-built + custom |
| **Backup** | Daily via Restic | Dashboards + alert rules |
| **Startup** | Auto | Operational visibility |
| **Status** | Phase 2 deployment (W4) | |

**Scrape Config:**
```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['192.168.9.10:9100', '192.168.50.10:9100', '192.168.30.10:9100']

  - job_name: 'minio'
    static_configs:
      - targets: ['192.168.9.21:9000']

  - job_name: 'postgres'
    static_configs:
      - targets: ['192.168.9.20:9187']  # postgres_exporter

  - job_name: 'restic'
    static_configs:
      - targets: ['192.168.9.22:9753']  # restic_exporter
```

---

### BO-7: Uptime Kuma

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Simple HTTP heartbeat |
| **Host** | Dell R7920 | |
| **Image** | Ubuntu 22.04 + Uptime Kuma | |
| **CPU** | 1 vCPU | Minimal |
| **RAM** | 2 GB | |
| **Storage** | 20 GB | Health check history |
| **Network** | vmbr0 (bridge) | 192.168.9.25:3001 |
| **Monitors** | Critical services only (Authentik, PostgreSQL, Restic) | Ping every 60s |
| **Alerts** | Slack/Email on failure | 3 consecutive failures |
| **Backup** | Daily via Restic | |
| **Status** | Phase 2 deployment (W4) | |

---

### BO-8: Wazuh (SIEM)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | VM (resource-heavy) | Not LXC due to Elasticsearch |
| **Host** | Dell R740xd (secondary) | Offload from R7920 primary |
| **Image** | CentOS 8 + Wazuh Manager/Indexer | |
| **CPU** | 4 vCPU | Index operations |
| **RAM** | 16 GB | Elasticsearch heap = 8GB |
| **Storage** | 200 GB SSD | Log retention (30 days) |
| **Network** | vmbr0 (bridge) | 192.168.9.26 (manager), 192.168.9.27 (indexer) |
| **Log Sources** | Authentik, Netbird, PostgreSQL, system syslogs | Syslog + REST API |
| **Retention** | 30 days rolling | Purge old indices via Kibana |
| **Backup** | Weekly snapshots | Index snapshots to MinIO |
| **Status** | Phase 2 deployment (W4) | Optional, good-to-have |

---

### BO-9: Gitea (Code Repositories)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Self-hosted Git |
| **Host** | Dell R7920 | |
| **Image** | Ubuntu 22.04 + Gitea | |
| **CPU** | 2 vCPU | Clone/push operations |
| **RAM** | 4 GB | |
| **Storage** | 100 GB SSD | Git repos |
| **Network** | vmbr0 (bridge) | 192.168.9.40:3000 |
| **Repos** | apiary-infrastructure, apiary-ai-agents, apiary-web, apiary-odoo-mods | |
| **Backup** | Daily via Restic | |
| **Auth** | OAuth via Authentik | OIDC integration |
| **Status** | Phase 2 deployment (W4) | |

---

### BO-10: n8n (Workflow Automation)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Node.js-based |
| **Host** | Dell R7920 | |
| **Image** | Ubuntu 22.04 + n8n | |
| **CPU** | 2 vCPU | Workflow execution |
| **RAM** | 4 GB | |
| **Storage** | 50 GB SSD | Execution history |
| **Network** | vmbr0 (bridge) | 192.168.9.41:5678 |
| **Backend** | PostgreSQL Primary (shared) | |
| **Workflows** | Odoo → Slack, Backup reports, Invoicing | Scheduled + triggered |
| **Backup** | Daily via Restic | |
| **Auth** | OAuth via Authentik | |
| **Status** | Phase 2 deployment (W4) | |

---

### BO-11: Paperless-NGX (Document Archive)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Document scanning + archival |
| **Host** | Dell R7920 | |
| **Image** | Ubuntu 22.04 + Paperless-NGX | |
| **CPU** | 2 vCPU | OCR (optional, CPU-heavy) |
| **RAM** | 4 GB | |
| **Storage** | 500 GB SSD | Scanned documents + search index |
| **Network** | vmbr0 (bridge) | 192.168.9.42:8000 |
| **Backend** | PostgreSQL Primary (shared) | |
| **Purpose** | Centralized audit trail (company documents, invoices) | Full-text search |
| **Backup** | Daily via Restic | |
| **Auth** | OAuth via Authentik | |
| **Status** | Phase 2 deployment (W4) | |

---

## Tier 1: Site Servers

### BH-21100 (Bien Home)

**Host Specs:** Custom/refurb, 4-8 core CPU, 16-32GB RAM

#### BH-1: Ollama (Local LLM)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | GPU passthrough optional |
| **CPU** | 4 vCPU (all available) | Model inference |
| **RAM** | 16 GB | Model cache (llama2-7b = ~4GB) |
| **Storage** | 100 GB SSD | Model files |
| **Network** | 192.168.50.20:11434 | Localhost binding only |
| **Models** | llama2-7b, mistral-7b, orca-mini | Minimal for latency |
| **Startup** | Auto | User-facing service |
| **Backup** | Models from Gitea (not user data) | |
| **Status** | Phase 1 deployment (W1-ready) | |

#### BH-2: Open WebUI (Konstanze for BH)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Web frontend |
| **CPU** | 2 vCPU | |
| **RAM** | 4 GB | |
| **Storage** | 20 GB | Chat history, user data |
| **Network** | 192.168.50.21:8080 | |
| **Upstream** | localhost:11434 (Ollama) | Local by default |
| **Fallback** | Netbird tunnel to BO Ollama | If local saturated |
| **Backup** | Daily via Restic | Chat history archive |
| **Status** | Phase 1 (W1-ready) | |

#### BH-3: PostgreSQL Replica

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Read-only |
| **CPU** | 2 vCPU | |
| **RAM** | 8 GB | |
| **Storage** | 200 GB SSD | Replica of BO (192.168.9.20) |
| **Network** | 192.168.50.22 (internal) | No external access |
| **Sync** | Streaming replication from BO | Via S2S VPN |
| **Use** | Read operations, offline cache | Not a write source |
| **Status** | Phase 1 (W3) | |

#### BH-4: Odoo (Development)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Dev/test instance |
| **CPU** | 4 vCPU | |
| **RAM** | 8 GB | |
| **Storage** | 100 GB SSD | Dev data only |
| **Network** | 192.168.50.30:8069 | |
| **Database** | PostgreSQL replica (read) + local SQLite (dev) | Not production |
| **Purpose** | Test customizations, modules | Isolated from EP |
| **Status** | Phase 1 (W1-ready) | |

#### BH-5: Home Assistant (Optional)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container / VM | IoT automation |
| **CPU** | 2 vCPU | |
| **RAM** | 4 GB | |
| **Storage** | 50 GB | Automation history |
| **Network** | 192.168.50.40:8123 | |
| **Devices** | Local lights, thermostats, sensors | Zigbee/Z-Wave mesh |
| **Status** | Phase 2+ (optional) | Nice-to-have |

#### BH-6: Restic Agent

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | Systemd timer (host-level) | Backup client |
| **Schedule** | Nightly 0200 CST | |
| **Target** | Restic Server (BO 192.168.9.22:8000) | Via S2S VPN |
| **Dataset** | /var/lib/odoo/, /var/lib/postgres/, /root/.ollama/ | Site data |
| **Retention** | 30 daily, 12 weekly, 12 monthly | Automated pruning |
| **Status** | Phase 1 (W1-ready) | |

---

### EP-01-SN (Easy Pawn)

**Host Specs:** Custom/refurb, similar to BH-21100

#### EP-1: Odoo (Production - Easy Pawn)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Production ERP |
| **CPU** | 4 vCPU | User-facing, higher load |
| **RAM** | 8 GB | |
| **Storage** | 100 GB SSD | Pawn shop inventory + accounting |
| **Network** | 192.168.30.30:8069 | Internal only |
| **Database** | PostgreSQL replica (read) + offline fallback | Works if WAN down |
| **Modules** | Easy Pawn custom: inventory, POS sync, accounting | Production modules |
| **Backup** | Continuous via Restic | Critical data |
| **Status** | Phase 0 (W1-3) | PRIORITY |

#### EP-2: Bravo POS (Native)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | Native app (Windows/Mac) | Not containerized |
| **Network** | 192.168.30.40 (LAN static) | Sync with Odoo |
| **API** | REST to Odoo | On-demand sync |
| **Offline** | Full operation, manual sync | Critical for sales |
| **Status** | Phase 0 baseline | |

#### EP-3: Ollama (Easy Pawn local)

| Property | Value | Notes |
|----------|-------|-------|
| **Type** | LXC Container | Customer-facing AI queries |
| **CPU** | 2 vCPU | |
| **RAM** | 8 GB | |
| **Storage** | 50 GB | |
| **Network** | 192.168.30.20:11434 | |
| **Models** | mistral-7b, orca-mini | Minimal |
| **Status** | Phase 0+ (optional) | |

#### EP-4-6: Konstanze, PostgreSQL Replica, Restic Agent

**Same as BH** with network addresses shifted to 192.168.30.x

---

## Tier 2: Customer Appliances (AIP Boxes) - Future

**Hardware:** Mac Mini M4 or Dell Micro (2-4 core, 16GB RAM)

**Stack per appliance:**

| Service | CPU | RAM | Storage | Network | Status |
|---------|-----|-----|---------|---------|--------|
| Ollama | 2 | 8GB | 40GB | localhost:11434 | Phase 4 |
| Open WebUI | 1 | 2GB | 10GB | localhost:8080 | Phase 4 |
| Odoo | 1 | 4GB | 40GB | localhost:8069 | Phase 4 |
| Netbird Agent | shared | shared | <1GB | 10.x.x.x (Netbird) | Phase 4 |
| Restic Agent | shared | shared | <1GB | Netbird to BO | Phase 4 |
| **TOTAL** | **4-6** | **16-20GB** | **100GB** | **Netbird mesh** | |

**Cost per AIP Box:** ~$400-500 hardware + $5/month cloud backup

---

## Resource Summary

### CPU Allocation (by Tier)

```
TIER 0 (BO):
├─ R7920: 48 cores physical, ~24 cores for VMs/containers
├─ Primary services: Authentik (2), Netbird (1), PostgreSQL (4), MinIO (2) = 9 cores
├─ Monitoring: Prometheus (2), Grafana (2) = 4 cores
├─ Utility: Restic (1), Uptime Kuma (1) = 2 cores
└─ Reserved: 9 cores for failover, bursts

TIER 1 (BH + EP):
├─ BH-21100: 8 cores, allocated 6 cores (Ollama 4, Odoo 2, reserve 2)
├─ EP-01-SN: 8 cores, allocated 6 cores (Odoo 4, Ollama 2, reserve 2)
└─ Both sites: Restic agents + Replica PostgreSQL (shared overhead)

TIER 2 (per AIP):
├─ Mac Mini M4: 8-10 cores
├─ Allocation: Ollama (4), Odoo (2), services (1), reserve (1-2)
└─ Scale: 10 boxes = 40-60 cores dedicated
```

### RAM Allocation (by Tier)

```
TIER 0 (BO):
├─ R7920: 256 GB physical
├─ Databases: PostgreSQL (16GB) + Wazuh/Elasticsearch (16GB) = 32GB
├─ Services: Authentik (4), Netbird (2), MinIO (8), Prometheus (8), n8n (4), Gitea (4), Paperless (4) = 28GB
├─ OS/Overhead: ~20GB
└─ Available for growth: ~170GB (headroom for replication, cache)

TIER 1 (Both sites):
├─ Each (BH, EP): 16-32GB physical
├─ Allocation: Ollama (8), Odoo (8), Replica (8) = ~24GB
└─ Headroom: Varies

TIER 2 (per AIP):
├─ Mac Mini: 16GB physical
├─ Allocation: Ollama (8), Odoo (4), services (2), OS (2)
└─ Headroom: ~0-2GB (tight, but acceptable for appliance)
```

### Storage Allocation (by Tier)

```
TIER 0 (BO):
├─ SSD: 1-2TB
├─ PostgreSQL + WAL: 100GB
├─ Containers (base): 50GB + 20GB + 50GB + 100GB + 500GB = ~720GB
├─ MinIO: 2TB (backups, growing)
├─ Restic: 500GB (incremental, ~5GB/night growth)
└─ Total: 3-4TB needed, ~3TB currently allocated

TIER 1 (BH + EP each):
├─ SSD: 500GB-1TB
├─ Containers: Ollama (100GB) + Odoo (100GB) + Replica (200GB) = ~400GB
├─ Headroom: 100-600GB
└─ Annual growth: ~50GB (backups via Restic)

TIER 2 (per AIP):
├─ SSD: 250GB
├─ Services: Ollama (50GB) + Odoo (40GB) + local cache (20GB) = ~110GB
└─ Headroom: ~140GB
```

---

## Deployment Checklist

### Phase 0 (This Week)

- [ ] Proxmox: Verify R7920 + R740xd exist and are connectable
- [ ] Network: Fix EP subnet (192.168.2.x → 192.168.30.x)
- [ ] Network: Bring BH ↔ EP S2S VPN online, test ping
- [ ] BH/EP: Deploy Restic agents (client-side)
- [ ] BH/EP: Deploy Odoo (Odoo already running on EP?)
- [ ] BH/EP: Connect to BO backups (even though BO not ready yet)

### Phase 1 (Week 2-3)

- [ ] BO: Rack R7920, install Proxmox 8.x, setup cluster
- [ ] BO: Deploy Authentik (LXC)
- [ ] BO: Deploy Netbird Coordinator (LXC)
- [ ] BO: Deploy PostgreSQL Primary (LXC) + setup replication to BH/EP
- [ ] BO: Deploy MinIO (LXC)
- [ ] BO: Deploy Restic Server (LXC)
- [ ] BH/EP: Enroll in Netbird mesh
- [ ] BH/EP: Connect PostgreSQL replicas to BO primary
- [ ] All: Test backup pipeline (BH/EP → BO Restic → MinIO)

### Phase 2 (Week 3-4)

- [ ] BO: Deploy Prometheus + Grafana (LXC)
- [ ] BO: Deploy Uptime Kuma (LXC)
- [ ] BO: Deploy Gitea (LXC)
- [ ] BO: Deploy n8n (LXC)
- [ ] BO: Deploy Paperless-NGX (LXC)
- [ ] All: Setup monitoring (Prometheus scrape configs)
- [ ] All: Create Grafana dashboards

### Phase 3 (Week 5+)

- [ ] Setup Claude API integration in Konstanze
- [ ] Setup Perplexity integration for research
- [ ] Track API costs (spreadsheet or dashboard)
- [ ] Test fallback (local Ollama insufficient, cloud query)

### Phase 4+ (Month 2+)

- [ ] Procure AIP Box #1 (Mac Mini or Dell Micro)
- [ ] Install minimal stack (Ollama, OpenWebUI, Odoo, agents)
- [ ] Test with real customer data
- [ ] Document onboarding process
- [ ] Evaluate before scaling to 5-10 boxes

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-09  
**Next Update:** After Phase 1 deployment
