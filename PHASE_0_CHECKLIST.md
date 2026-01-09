# Phase 0 Execution Checklist: Foundation (This Week)

**Goal:** Get existing infrastructure working, fix critical gaps, prepare for centralized services.

**Status:** 🔴 **NOT STARTED** (as of 2026-01-09 09:12 CST)

**Duration:** 1 week (Jan 9-17, 2026)

**Success Criteria:**
- ✅ EP subnet fixed (192.168.2.x → 192.168.30.x)
- ✅ BH ↔ EP S2S VPN tested and working
- ✅ Odoo running on EP with live data
- ✅ Bank reports generated from Odoo
- ✅ Infrastructure docs on GitHub

**API Cost:** $0 (no tokens spent)

---

## Day 1 (Thursday, Jan 9) - Today

### Task 1.1: Fix EP Subnet (192.168.2.x → 192.168.30.x)

**Owner:** Eric Bien  
**Duration:** 1 hour  
**Status:** 🔴 BLOCKED (critical path blocker)  
**Impact:** UniFi VPN tunnel BO ↔ EP cannot establish without this

**Steps:**

1. **Access UniFi Controller**
   ```bash
   # Option A: Web UI
   https://192.168.9.1/  # UDM Pro Max (BO)
   # Login with UniFi credentials
   
   # Option B: SSH (if needed)
   ssh ubnt@192.168.9.1
   ```

2. **Navigate to Settings → Network → Advanced**
   - Select network interface connected to EP-01-SN
   - Current: LAN with 192.168.2.0/24
   - Change to: 192.168.30.0/24

3. **Update DHCP Pool**
   - Start: 192.168.30.100
   - End: 192.168.30.200
   - Gateway: 192.168.30.1
   - Subnet: 255.255.255.0

4. **Static IP Assignment for EP-01-SN**
   ```
   Device: EP-01-SN (by MAC address)
   IP: 192.168.30.10
   Subnet: 255.255.255.0
   Gateway: 192.168.30.1
   ```

5. **Update Bravo POS Devices (if static)**
   ```
   Device: Bravo Terminal 1
   IP: 192.168.30.250 (or DHCP with reservation)
   ```

6. **Update EP Router (UDR7)**
   ```
   Device: UniFi UDR7
   IP: 192.168.30.1 (gateway)
   Announce: No (BO UDM is primary controller)
   ```

7. **Reboot EP network**
   ```bash
   # From BO UDM (optional, if UniFi allows remote reboot)
   # OR manually at EP site
   ```

8. **Verify**
   ```bash
   # From BO or BH, try:
   ping 192.168.30.10  # Should reach EP-01-SN
   ping 192.168.30.1   # Should reach EP router
   
   # From EP, verify BO is visible:
   ping 192.168.9.1    # BO gateway
   ```

**Completion Criteria:**
- [ ] EP subnet changed in UniFi controller
- [ ] EP-01-SN has static IP 192.168.30.10
- [ ] EP gateway is 192.168.30.1
- [ ] Ping test successful from all three sites
- [ ] Document current UniFi config (screenshot or export)

**Notes:**
- Do this during low-traffic window (Pawn shop slow time?)
- Have rollback plan (can revert to 192.168.2.x if issues)
- May require EP site visit or remote access

**✅ COMPLETED:** _____ (date/time)  
**❌ BLOCKED:** _____ (reason)

---

### Task 1.2: Verify UniFi S2S VPN Tunnels

**Owner:** Eric Bien  
**Duration:** 30 min  
**Status:** 🔴 BLOCKED (depends on 1.1)  
**Prerequisite:** Task 1.1 complete

**Steps:**

1. **Check BO ↔ BH Tunnel**
   - UniFi Controller → Devices → UDR (BH)
   - Status: Should show "Connected" (green)
   - If "Disconnected" (red), click "Restart"

2. **Check BO ↔ EP Tunnel** (after subnet fix)
   - UniFi Controller → Devices → UDR7 (EP)
   - Status: Should show "Connected" (green)
   - If "Disconnected", check:
     - Is EP subnet fixed? (1.1)
     - Is WAN accessible at EP?
     - Are PSK keys matching?

3. **Test Tunnel Connectivity**
   ```bash
   # From BO (192.168.9.x), try:
   ping -c 3 192.168.50.10   # BH Proxmox
   ping -c 3 192.168.30.10   # EP Proxmox
   
   # From BH (192.168.50.x), try:
   ping -c 3 192.168.9.10    # BO Proxmox
   ping -c 3 192.168.30.10   # EP Proxmox (via BO relay)
   
   # From EP (192.168.30.x), try:
   ping -c 3 192.168.9.10    # BO Proxmox
   ping -c 3 192.168.50.10   # BH Proxmox (via BO relay)
   ```

4. **Check Tunnel Stats**
   - UniFi Controller → Network → Tunnels
   - Note tunnel latency, packet loss
   - Document for future baseline

5. **Test Failover** (optional, Phase 1)
   - Disconnect BH WAN, verify BO ↔ EP still works
   - Reconnect, verify sync

**Completion Criteria:**
- [ ] BO ↔ BH tunnel connected
- [ ] BO ↔ EP tunnel connected (after subnet fix)
- [ ] Ping tests successful between all three sites
- [ ] Tunnel latency <50ms (typical)
- [ ] Packet loss <1%

**✅ COMPLETED:** _____ (date/time)  
**❌ BLOCKED:** _____ (reason)

---

## Days 2-3 (Jan 10-11): Odoo Deployment on EP

### Task 2.1: Prepare EP-01-SN for Odoo

**Owner:** Eric Bien  
**Duration:** 4 hours  
**Status:** 🔴 BLOCKED (depends on subnet fix)  
**Prerequisites:** Task 1.1, 1.2 complete

**Steps:**

1. **SSH into EP-01-SN**
   ```bash
   ssh root@192.168.30.10
   # Or via Proxmox console
   ```

2. **Verify Proxmox is operational**
   ```bash
   pve-version  # Should show Proxmox version
   pvecm status # Should show cluster status
   pvesh get /nodes
   ```

3. **Create Odoo LXC Container**
   ```bash
   pct create 300 \
     local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
     -hostname odoo-ep \
     -cores 4 \
     -memory 8192 \
     -storage local \
     -swap 0 \
     -net0 name=eth0,bridge=vmbr0,ip=192.168.30.30/24,gw=192.168.30.1
   ```

4. **Start container**
   ```bash
   pct start 300
   ```

5. **Inside container, install Odoo**
   ```bash
   pct enter 300
   
   # Update packages
   apt-get update && apt-get upgrade -y
   
   # Install dependencies
   apt-get install -y python3 python3-dev postgresql-client python3-pip wget
   
   # Download and install Odoo
   wget https://github.com/odoo/odoo/archive/15.0.tar.gz
   tar -xzf 15.0.tar.gz && cd odoo-15.0
   pip3 install -e .
   
   # Create Odoo user
   useradd -m -d /var/lib/odoo -s /bin/bash odoo
   
   # Configure Odoo
   mkdir -p /etc/odoo
   cat > /etc/odoo/odoo.conf << 'EOF'
   [options]
   admin_passwd = <admin-password>
   db_host = 192.168.9.20  # BO PostgreSQL primary (for now)
   db_port = 5432
   db_user = odoo_user
   db_password = <odoo-db-password>
   db_filter = ^odoo_ep$
   addons_path = /var/lib/odoo/addons
   log_file = /var/log/odoo/odoo.log
   logfile = /var/log/odoo/odoo.log
   logrotate = True
   EOF
   
   # Set permissions
   chown odoo:odoo /etc/odoo/odoo.conf
   chmod 640 /etc/odoo/odoo.conf
   ```

6. **Create systemd service**
   ```bash
   cat > /etc/systemd/system/odoo.service << 'EOF'
   [Unit]
   Description=Odoo ERP Service
   After=network.target postgresql.service
   
   [Service]
   Type=simple
   User=odoo
   Group=odoo
   ExecStart=/usr/bin/python3 -m odoo --config /etc/odoo/odoo.conf
   Restart=always
   RestartSec=10
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   systemctl enable odoo
   systemctl start odoo
   ```

7. **Verify Odoo is running**
   ```bash
   curl http://localhost:8069/
   # Should return HTML (login page)
   
   systemctl status odoo
   tail -f /var/log/odoo/odoo.log
   ```

**Completion Criteria:**
- [ ] LXC container created (id 300)
- [ ] Odoo dependencies installed
- [ ] Odoo systemd service running
- [ ] Web access at http://192.168.30.30:8069
- [ ] Logs show no errors

**✅ COMPLETED:** _____ (date/time)  
**❌ BLOCKED:** _____ (reason)

---

### Task 2.2: Import Real Easy Pawn Data into Odoo

**Owner:** Eric Bien  
**Duration:** 4-6 hours  
**Status:** 🔴 BLOCKED  
**Prerequisites:** Task 2.1 complete

**Steps:**

1. **Prepare backup of current Easy Pawn data**
   ```bash
   # If Odoo already running elsewhere, export database
   # OR export from Bravo POS / current system
   
   # Example: Export from existing Odoo
   python3 -m odoo export --database=odoo_ep --export-all --output=/tmp/odoo_backup.json
   ```

2. **Create Odoo database**
   ```bash
   pct enter 300
   odoo-bin --database=odoo_ep --init=base,sale,purchase,inventory,account
   ```

3. **Import Easy Pawn data**
   - Import chart of accounts
   - Import products/inventory
   - Import customers
   - Import past transactions (if available)
   
   **Tools:**
   - Odoo import wizard (Data → Import)
   - Custom Python script (if data in CSV/JSON)
   - n8n workflow (when Phase 2 ready)

4. **Verify data integrity**
   ```
   - Check account balances match bank records
   - Verify product inventory counts
   - Spot-check customer records
   - Review cash flow (2025 vs current)
   ```

5. **Test Bravo POS integration**
   - Sync Bravo → Odoo inventory
   - Verify transaction logging
   - Test offline fallback

**Completion Criteria:**
- [ ] Easy Pawn data imported
- [ ] Account balances verified
- [ ] Inventory counts match
- [ ] Bravo POS ↔ Odoo sync working
- [ ] No critical errors in logs

**✅ COMPLETED:** _____ (date/time)  
**❌ BLOCKED:** _____ (reason)

---

## Day 4 (Jan 12): Odoo Testing with Real Data

### Task 4.1: Generate Bank Meeting Reports

**Owner:** Eric Bien  
**Duration:** 4 hours  
**Status:** 🔴 BLOCKED (depends on Task 2.2)  
**Prerequisite:** Odoo running with real Easy Pawn data

**Steps:**

1. **Access Odoo as Admin**
   - Navigate to http://192.168.30.30:8069
   - Login with admin credentials

2. **Generate Financial Reports**
   - Accounting → Reporting → Trial Balance
   - Accounting → Reporting → Balance Sheet
   - Accounting → Reporting → Income Statement
   - Date range: Full year 2025 (or relevant period)

3. **Export to PDF/Excel**
   ```
   File → Export → PDF
   Or: Copy to spreadsheet for presentation
   ```

4. **Review with Accountant / Bank**
   - Are figures accurate?
   - Do they match bank records?
   - Any discrepancies?
   - Document findings

5. **Use for Q1 Bank Meeting**
   - Prepare one-pager for banker
   - Highlight key metrics (revenue, expenses, cash position)
   - Identify any variances

**Completion Criteria:**
- [ ] Financial reports generated from Odoo
- [ ] Figures verified against bank records
- [ ] Reports in PDF format (ready for meeting)
- [ ] No critical discrepancies
- [ ] Banker has confidence in data

**✅ COMPLETED:** _____ (date/time)  
**❌ BLOCKED:** _____ (reason)

---

## Days 5-7 (Jan 13-17): Documentation & Wrap-up

### Task 5.1: Document Current State

**Owner:** Eric Bien  
**Duration:** 2-4 hours  
**Status:** 🔴 BLOCKED (depends on all above)  
**Prerequisite:** All Phase 0 tasks complete

**Steps:**

1. **Update APIARY_INFRASTRUCTURE_MASTER.md**
   - Mark Phase 0 status: COMPLETE
   - Document actual subnet (192.168.30.x)
   - Document VPN tunnel status
   - Document Odoo deployment
   - Update timestamps

2. **Update NETWORK_TOPOLOGY.md**
   - Confirm EP subnet change
   - Document VPN tunnel specs (actual IKE/ESP)
   - Add MTU measurements
   - Document DNS (if applicable)

3. **Update CONTAINER_DEPLOYMENT_MATRIX.md**
   - Mark Odoo (EP) as deployed (Phase 0, not Phase 1)
   - Document actual resource usage (CPU, RAM, storage)
   - Update status from "Phase 1" to "Phase 0 COMPLETE"

4. **Create PHASE_0_COMPLETION_REPORT.md**
   - Summary of what was accomplished
   - What went well
   - What was challenging
   - Lessons learned
   - Recommendations for Phase 1
   - Timeline for next phase

5. **Commit to GitHub**
   ```bash
   cd ~/apiary-infrastructure/
   git add -A
   git commit -m "Phase 0 completion: subnet fixed, VPN tested, Odoo deployed"
   git push origin main
   ```

**Completion Criteria:**
- [ ] All docs updated
- [ ] Phase 0 status marked COMPLETE
- [ ] Reports committed to GitHub
- [ ] Clear next-steps documented for Phase 1
- [ ] Team (if any) has read access to latest docs

**✅ COMPLETED:** _____ (date/time)  
**❌ BLOCKED:** _____ (reason)

---

## Risk & Contingency Plan

### Risk 1: Subnet Change Breaks EP Network

**Risk:** Changing 192.168.2.x to 192.168.30.x causes EP-01-SN or Bravo POS to lose connectivity

**Mitigation:**
- Have rollback plan (revert to 192.168.2.x)
- Do during low-traffic time (early morning or late evening)
- Have EP site contact available (if remote)
- Test one device at a time before full rollout

**Rollback Procedure:**
```bash
# Revert in UniFi controller
Settings → Network → Advanced
Change 192.168.30.0/24 back to 192.168.2.0/24
Reboot EP router
```

---

### Risk 2: S2S VPN Tunnel Won't Establish (BO ↔ EP)

**Risk:** After subnet fix, VPN tunnel still shows "Disconnected"

**Diagnosis:**
```
1. Check UniFi logs: Settings → System → Logs
2. Look for IKE errors ("Phase 1 failed", "PSK mismatch")
3. Verify PSK is identical on both routers
4. Verify local/remote subnets in tunnel config
5. Check WAN connectivity at EP (can reach internet?)
6. Ping UDR7 (EP router) from BO to verify WAN access
```

**Fallback:**
If VPN doesn't work, use Netbird (Phase 1) as overlay mesh

---

### Risk 3: Odoo Can't Connect to PostgreSQL at BO

**Risk:** Odoo (EP) can't reach PostgreSQL (BO) → database errors

**Diagnosis:**
```bash
# From Odoo container
pg_isready -h 192.168.9.20 -p 5432 -U odoo_user
# Should return "accepting connections"

# If failed:
ping 192.168.9.20  # Can reach BO at all?
sudo iptables -L    # Any firewall rules?
psql -h 192.168.9.20 -U odoo_user -d odoo_ep  # Can connect directly?
```

**Fallback:**
Use local SQLite database temporarily (Phase 0)
```bash
# Odoo can use SQLite if PostgreSQL unavailable
# Not ideal (no replication), but works for testing
```

---

## Success Criteria (Repeating)

✅ **Phase 0 COMPLETE when:**

1. EP subnet = 192.168.30.x (no more 192.168.2.x)
2. S2S VPNs working (BH ↔ BO, EP ↔ BO)
3. Ping test successful: all three sites can reach each other
4. Odoo running on EP-01-SN (192.168.30.30:8069)
5. Real Easy Pawn data imported into Odoo
6. Bank meeting reports generated and verified
7. Infrastructure documentation complete on GitHub
8. Estimated cost: $0 (no new hardware, no cloud API tokens)

---

## Daily Progress Log

**Day 1 (Jan 9, Thu)**
- [ ] 9:00 AM - Start Task 1.1 (subnet fix)
- [ ] 10:00 AM - Complete Task 1.1
- [ ] 10:30 AM - Start Task 1.2 (VPN test)
- [ ] 11:00 AM - Complete Task 1.2
- [ ] Notes: _____

**Day 2 (Jan 10, Fri)**
- [ ] Start Task 2.1 (Proxmox + Odoo prep)
- [ ] Notes: _____

**Day 3 (Jan 11, Sat)**
- [ ] Complete Task 2.1
- [ ] Start Task 2.2 (data import)
- [ ] Notes: _____

**Day 4 (Jan 12, Sun)**
- [ ] Complete Task 2.2
- [ ] Start Task 4.1 (reports)
- [ ] Notes: _____

**Days 5-7 (Jan 13-17)**
- [ ] Complete Task 5.1 (documentation)
- [ ] Final review
- [ ] Commit to GitHub
- [ ] Notes: _____

---

**Created:** 2026-01-09 | **Phase:** 0 | **Status:** Ready to Execute
