# 🛡️ SOC Home Lab — Wazuh SIEM, Detection Engineering, FIM & Incident Response

![Wazuh](https://img.shields.io/badge/Wazuh-v4.x-3C8CBE?style=for-the-badge&logo=wazuh&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Windows](https://img.shields.io/badge/Windows_10-Pro-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![VMware](https://img.shields.io/badge/VMware_Workstation-17.x-607078?style=for-the-badge&logo=vmware&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK_Mapped-red?style=for-the-badge)

> A fully functional Security Operations Center (SOC) home lab built to simulate real-world attack scenarios, generate endpoint and authentication telemetry, engineer custom detection rules, monitor file integrity, and execute an end-to-end incident response lifecycle — all monitored through a self-hosted Wazuh SIEM.

---

## 📑 Table of Contents

- [Project Overview](#-project-overview)
- [Lab Architecture](#-lab-architecture)
- [Environment & Tools](#-environment--tools)
- [Phase 1 — SIEM Deployment & Agent Enrollment](#phase-1--siem-deployment--agent-enrollment)
- [Phase 2 — Telemetry Generation (Attack Simulation)](#phase-2--telemetry-generation-attack-simulation)
- [Phase 3 — Dashboard Engineering](#phase-3--dashboard-engineering)
- [Phase 4 — Custom Detection Rules](#phase-4--custom-detection-rules)
- [Phase 5 — File Integrity Monitoring (FIM)](#phase-5--file-integrity-monitoring-fim)
- [Phase 6 — Active Response](#phase-6--active-response)
- [Phase 7 — Incident Investigation Report](#phase-7--incident-investigation-report)
- [Skills Demonstrated](#-skills-demonstrated)
- [Lessons Learned](#-lessons-learned)
- [Roadmap](#-roadmap--future-work)
- [References](#-references)
- [About Me](#-about-me)

---

## 🎯 Project Overview

The goal of this lab was to build a **realistic, hands-on SOC environment** that mirrors the day-to-day workflow of a Tier 1 / Tier 2 SOC analyst:

1. **Ingest** logs from heterogeneous endpoints (Windows 10, Ubuntu Desktop, Ubuntu Server) into a centralized SIEM.
2. **Simulate** real attacker behaviors mapped to the MITRE ATT&CK framework.
3. **Detect** those behaviors using custom, tuned detection rules — not just out-of-the-box defaults.
4. **Monitor** critical files for unauthorized changes using Wazuh's File Integrity Monitoring (FIM).
5. **Respond** automatically using Wazuh's Active Response module.
6. **Document** the entire incident through a formal investigation report.

### Detection Engineering Lifecycle

```
Threat Simulation → Log Ingestion → Detection Rule → Alert → Triage → Active Response → Report
                                        ▲
                                        │
                             File Integrity Monitoring (FIM)
```

---

## 🏗️ Lab Architecture

```
                    ┌──────────────────────────────────────────┐
                    │         HOST MACHINE (Hypervisor)        │
                    │        VMware Workstation Pro 17         │
                    └──────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
        ┌───────────────────┐ ┌───────────────┐ ┌───────────────────┐
        │  Ubuntu Server    │ │ Windows 10 VM │ │ Ubuntu Desktop VM │
        │  ⭐ WAZUH SERVER ⭐│ │ (Victim /     │ │ (Attacker /       │
        │  (Wazuh Manager + │ │  Endpoint)    │ │  Red Team)        │
        │   Indexer +       │ │               │ │                   │
        │   Dashboard)      │ │               │ │                   │
        │  192.168.56.10    │ │ 192.168.56.20 │ │ 192.168.56.30     │
        └───────────────────┘ └───────────────┘ └───────────────────┘
                 ▲                    │                  │
                 │                    │                  │
                 │  Wazuh Agent ──────┘                  │
                 │  Wazuh Agent ─────────────────────────┘
                 │  (Log + FIM forwarding on port 1514/1515)
                 │
        ⭐ = Hosts the entire Wazuh SIEM stack
             (Manager, Indexer, Dashboard, plus FIM monitoring
              of its own critical system files)
```

**Network:** VMware **VMnet1 (Host-only)** network on `192.168.56.0/24`, plus **VMnet8 (NAT)** for internet access to install packages.

**VMware Networking note:** Configured via **VMware Virtual Network Editor** (`Edit → Virtual Network Editor`). Each VM was given two adapters:
- **Adapter 1:** Host-only (VMnet1) → isolated `192.168.56.0/24` lab traffic
- **Adapter 2:** NAT (VMnet8) → outbound internet for updates and package installation

![Lab Architecture](docs/screenshots/00-lab-architecture.png)

---

## 🧰 Environment & Tools

| Component | Role | Version / Specs |
|---|---|---|
| **Wazuh Manager** | SIEM, rule engine, active response, FIM server | v4.x on Ubuntu Server 22.04 |
| **Wazuh Indexer** | Log storage & search | Bundled with Wazuh |
| **Wazuh Dashboard** | Visualization & alert triage | Bundled with Wazuh |
| **Ubuntu Server VM** | ⭐ **Hosts the Wazuh stack** + monitored endpoint | 4 vCPU / 8 GB RAM / 60 GB disk |
| **Windows 10 VM** | Monitored endpoint | 2 vCPU / 4 GB RAM |
| **Ubuntu Desktop VM** | Attacker + monitored endpoint | 2 vCPU / 4 GB RAM |
| **Sysmon** | Enhanced Windows telemetry | v15.x (SwiftOnSecurity config) |
| **Auditd** | Linux file/process auditing (Linux FIM backend) | Default Ubuntu package |
| **VMware Workstation Pro** | Hypervisor | v17.x |

**Frameworks referenced:** MITRE ATT&CK, NIST SP 800-61, Cyber Kill Chain.

---

## Phase 1 — SIEM Deployment & Agent Enrollment

### 1.1 Wazuh Server Setup (on Ubuntu Server VM)

Deployed the Wazuh all-in-one stack on **Ubuntu Server** using the official installer:

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Verified services:

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### 1.2 Agent Deployment

**Ubuntu endpoints (Linux agent — includes Ubuntu Server itself for FIM):**

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.x_amd64.deb
sudo WAZUH_MANAGER='192.168.56.10' dpkg -i ./wazuh-agent_4.x_amd64.deb
sudo systemctl enable --now wazuh-agent
```

**Windows 10 endpoint:**

- Installed the MSI with `WAZUH_MANAGER='192.168.56.10'` parameter.
- Installed **Sysmon** for process creation, network, and registry telemetry:

```powershell
Sysmon64.exe -accepteula -i sysmonconfig.xml
```

- Configured `ossec.conf` to forward the Sysmon operational log channel.

![Agents Active](docs/screenshots/01-agents-active.png)

**✅ Achievement:** Enrolled **3 heterogeneous agents** (Windows + 2× Linux) into a single SIEM with a stable cross-OS log pipeline.

---

## Phase 2 — Telemetry Generation (Attack Simulation)

Simulated attacker behaviors from the **Ubuntu Desktop VM** ("attacker") targeting the **Windows 10 VM** and **Ubuntu Server** ("victims").

### 2.1 Failed SSH Logon (Brute Force Simulation)

**Technique:** `T1110.001 – Brute Force: Password Guessing`

```bash
for i in $(seq 1 10); do
  sshpass -p "wrongpass$i" ssh victim@192.168.56.10
done
```

**Telemetry produced** (`/var/log/auth.log`):

```
Failed password for invalid user victim from 192.168.56.30 port 51xxx ssh2
```

![Failed SSH Alerts](docs/screenshots/02-failed-ssh-alerts.png)

---

### 2.2 Guest Account Created & Deleted

**Technique:** `T1136.001 – Create Account: Local Account`

On Windows 10 (elevated prompt):

```cmd
net user GuestUser P@ssw0rd123 /add
net localgroup Administrators GuestUser /add
```

Simulated cleanup:

```cmd
net user GuestUser /delete
```

**Telemetry produced:** Windows Security Event IDs **4720**, **4732**, **4726**, plus **Sysmon Event ID 1** (`net.exe` process creation).

![Guest Account Events](docs/screenshots/03-guest-account-events.png)

---

### 2.3 Successful Logon

**Technique:** `T1078 – Valid Accounts`

Generated an interactive logon to Windows 10 with the newly created account, producing:

- **Event ID 4624** (Logon Type 10 / 2)
- **Event ID 4672** (special privileges assigned)

![Successful Logon](docs/screenshots/04-successful-logon.png)

**✅ Achievement:** Produced a **complete attack chain** in the SIEM, with every action mapped to a MITRE ATT&CK technique.

---

## Phase 3 — Dashboard Engineering

Built a custom Wazuh dashboard titled **"SOC Home Lab — Threat Overview"** containing:

| Visualization | Purpose |
|---|---|
| **Authentication Failures (Time Series)** | Spot brute-force spikes by source IP |
| **Top 10 Source IPs (Failed Auth)** | Identify aggressive attackers |
| **Windows Account Management Events** | 4720 / 4726 / 4732 heatmap |
| **File Integrity Monitoring Events** | FIM alerts by agent & file path |
| **MITRE ATT&CK Technique Breakdown** | Coverage at a glance |
| **Active Response Trigger Count** | Track automated remediations |
| **Alert Severity Distribution** | Rule level 0–15 histogram |

### Design Decisions

- **Pivot-friendly**: each viz uses `agent.name` as a global filter.
- **Time-bucketed at 5 min** to make brute-force bursts visually obvious.
- **Saved searches** for `rule.id:5710 OR rule.id:5760`, `win.eventdata.eventID:4720`, and `rule.groups:syscheck` (FIM).

![Dashboard Overview](docs/screenshots/05-dashboard-overview.png)
![Dashboard Drill-down](docs/screenshots/06-dashboard-drilldown.png)

**✅ Achievement:** Translated raw log noise into decision-support visualizations a Tier 1 analyst could triage in seconds.

---

## Phase 4 — Custom Detection Rules

Out-of-the-box rules were noisy and missed several simulated behaviors. I wrote **custom Wazuh rules** in `/var/ossec/etc/rules/local_rules.xml`.

### Rule 1 — SSH Brute Force (Threshold-Based)

```xml
<group name="syslog,sshd,attack,">
  <rule id="100100" level="10" frequency="5" timeframe="120">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip />
    <description>Possible SSH brute force — 5 failed logins from same IP in 120s</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
  </rule>
</group>
```

### Rule 2 — Suspicious Local Account Creation

```xml
<group name="windows,account_management,">
  <rule id="100200" level="12">
    <if_sid>60106</if_sid>
    <field name="win.system.eventID">^4720$</field>
    <description>Windows local account created: $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1136.001</id>
    </mitre>
  </rule>
</group>
```

### Rule 3 — Account Added to Administrators Group

```xml
<group name="windows,privilege_escalation,">
  <rule id="100201" level="13">
    <if_sid>60106</if_sid>
    <field name="win.system.eventID">^4732$</field>
    <field name="win.eventdata.targetUserName">GuestUser</field>
    <description>User added to Administrators group — possible privilege escalation</description>
    <mitre>
      <id>T1078.001</id>
    </mitre>
  </rule>
</group>
```

### Rule 4 — Guest Account Reactivation

```xml
<group name="windows,persistence,">
  <rule id="100202" level="12">
    <if_sid>60106</if_sid>
    <field name="win.system.eventID">^4722$</field>
    <description>Previously disabled Guest account was enabled</description>
    <mitre>
      <id>T1078.001</id>
    </mitre>
  </rule>
</group>
```

### Rule Validation

```bash
sudo /var/ossec/bin/wazuh-logtest
```

![Rule Test](docs/screenshots/07-rule-test.png)
![Custom Alerts](docs/screenshots/08-custom-alerts.png)

**✅ Achievement:** Wrote production-style XML rules with thresholds, field matching, and MITRE tagging; reduced false positives by scoping to specific Event IDs and target users.

---

## Phase 5 — File Integrity Monitoring (FIM)

Configured Wazuh's **File Integrity Monitoring** (`syscheck`) to detect unauthorized changes to critical system and application files on the **Ubuntu Server** (which hosts Wazuh) and the **Ubuntu Desktop** endpoint. FIM generates hashes (MD5, SHA1, SHA256) for monitored files, and Wazuh alerts on **creation, modification, or deletion**.

### 5.1 FIM Configuration — Ubuntu Server (`/var/ossec/etc/ossec.conf`)

Tuned the `<syscheck>` block to monitor security-critical directories:

```xml
<syscheck>
  <disabled>no</disabled>

  <!-- Frequency that syscheck runs (in seconds). 300 = every 5 min -->
  <frequency>300</frequency>

  <!-- Directories to monitor -->
  <directories check_all="yes" realtime="yes" report_changes="yes">/etc</directories>
  <directories check_all="yes" realtime="yes" report_changes="yes">/usr/bin</directories>
  <directories check_all="yes" realtime="yes" report_changes="yes">/usr/sbin</directories>
  <directories check_all="yes" realtime="yes" report_changes="yes">/bin</directories>
  <directories check_all="yes" realtime="yes" report_changes="yes">/sbin</directories>

  <!-- Wazuh's own critical files -->
  <directories check_all="yes" realtime="yes">/var/ossec/etc</directories>
  <directories check_all="yes" realtime="yes">/var/ossec/ruleset</directories>

  <!-- Ignore noisy paths -->
  <ignore>/etc/mtab</ignore>
  <ignore>/etc/random-seed</ignore>
  <ignore type="sregex">.log$|.swp$</ignore>

  <!-- Whodata: capture the user/process that modified the file (uses auditd) -->
  <whodata>
    <disabled>no</disabled>
  </whodata>
</syscheck>
```

**Key options explained:**
| Option | Purpose |
|---|---|
| `realtime="yes"` | Alerts within seconds of a change (via inotify) |
| `report_changes="yes"` | Diffs the file content — shows *what* changed, not just *that* it changed |
| `check_all="yes"` | Hashes + permissions + owner + timestamps + size |
| `whodata` | Uses Linux **auditd** to attribute changes to a specific user & process |

### 5.2 FIM Configuration — Windows 10 (`ossec.conf`)

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>300</frequency>

  <!-- Windows critical paths -->
  <directories check_all="yes" realtime="yes" report_changes="yes">C:\Windows\System32\drivers\etc</directories>
  <directories check_all="yes" realtime="yes" report_changes="yes">C:\Windows\System32\config</directories>
  <directories check_all="yes" realtime="yes">C:\Users\Public</directories>

  <!-- Registry monitoring (Windows-only) -->
  <windows_registry check_all="yes" realtime="yes">HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run</windows_registry>
  <windows_registry check_all="yes" realtime="yes">HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce</windows_registry>
</syscheck>
```

### 5.3 Simulated Attack — FIM Trigger

**Technique:** `T1543 – Create or Modify System Process` / `T1037 – Boot or Logon Initialization Scripts`

From an attacker foothold on the Ubuntu Server, simulated persistence by modifying `/etc/passwd` and dropping a file into `/etc/init.d`:

```bash
# Simulate tampering with a critical system file
echo "# backdoor" | sudo tee -a /etc/passwd

# Simulate dropping a persistence script
sudo touch /etc/init.d/backdoor.sh
sudo chmod +x /etc/init.d/backdoor.sh
sudo sh -c 'echo "#!/bin/bash" > /etc/init.d/backdoor.sh'

# Verify a new file
sudo touch /etc/suspicious.conf
```

On Windows 10, simulated persistence via the **Run** registry key:

```cmd
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v Backdoor /t REG_SZ /d "C:\Users\Public\evil.exe" /f
```

### 5.4 FIM Alerts Generated

Wazuh fired alerts under rule group **`syscheck`**:

| Rule ID | Level | Description |
|---|---|---|
| `550` | 7 | Integrity checksum changed |
| `553` | 7 | File deleted |
| `554` | 7 | File added to the system |
| `594` | 5 | Registry entry added (Windows) |
| `597` | 7 | Registry entry modified (Windows) |

**Sample alert fields:**

```
rule.id:      550
agent.name:   ubuntu-server
file:         /etc/passwd
md5_before:   <hash>
md5_after:    <hash>
sha256_after: <hash>
size_before:  2734
size_after:   2748
uname_after:  root
```

### 5.5 FIM Dashboard Integration

Added the **"File Integrity Monitoring"** visualizations from the Wazuh default dashboard pack, plus a custom **"Recent FIM Events (Last 24h)"** table filtered on `rule.groups:syscheck` to display:

- Agent name
- Monitored path
- Change type (added / modified / deleted)
- Timestamp
- User & process responsible (thanks to whodata)

### 5.6 FIM Rule Tuning

FIM is notoriously noisy on `/etc` (log rotations, temp files, package updates). Tuned the configuration to:

- **Ignore** volatile paths: `/etc/mtab`, `*.log`, `*.swp`, `*.tmp`
- **Baseline** the file tree after updates so package-manager changes don't trigger false positives
- **Whitelist** known-good changes by IP/user in a custom rule:

```xml
<rule id="100300" level="0">
  <if_sid>550</if_sid>
  <field name="file">^/etc/mtab$</field>
  <description>Ignored: /etc/mtab changes (kernel)</description>
</rule>
```

**✅ Achievement:**
- Deployed **real-time FIM** across Linux and Windows endpoints, including **Windows registry monitoring**.
- Enabled **whodata** to attribute changes to a specific user/process.
- Tuned the configuration to reduce false positives from routine system activity.
- Mapped FIM coverage to MITRE ATT&CK persistence techniques (T1543, T1037).

![FIM Config](docs/screenshots/13-fim-config.png)
![FIM Alerts](docs/screenshots/14-fim-alerts.png)
![FIM Diff](docs/screenshots/15-fim-file-diff.png)
![FIM Registry](docs/screenshots/16-fim-registry-windows.png)
![FIM Dashboard](docs/screenshots/17-fim-dashboard.png)

---

## Phase 6 — Active Response

Configured Wazuh's **Active Response** module for automated mitigation of brute-force attempts.

### Configuration (`/var/ossec/etc/ossec.conf` on the manager)

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100100</rules_id>
  <timeout>600</timeout>
</active-response>
```

- **Trigger:** Rule `100100` (SSH brute force) fires.
- **Action:** Wazuh runs `firewall-drop` on the target Linux host.
- **Effect:** Attacker IP (`192.168.56.30`) blocked via `iptables` for **600 seconds**.
- **Timeout:** Automatic rollback after 10 minutes.

### Verified Behavior

```bash
sudo iptables -L -n | grep 192.168.56.30
# DROP  all  --  192.168.56.30  0.0.0.0/0
```

Confirmed the attacker VM's subsequent SSH attempts **timed out** until the block expired.

![Active Response Log](docs/screenshots/09-active-response-log.png)
![iptables Block](docs/screenshots/10-iptables-block.png)

**✅ Achievement:** Implemented automated containment with timeout rollback — a core SOC Tier 1/2 responsibility.

---

## Phase 7 — Incident Investigation Report

Authored a formal incident report (see [`docs/INC-2024-001-Investigation.pdf`](docs/INC-2024-001-Investigation.pdf)) following NIST SP 800-61 structure.

### Executive Summary

> The SOC home lab detected a coordinated attack involving **SSH brute-force from `192.168.56.30`**, followed by **unauthorized local account creation and privilege escalation** on the Windows 10 endpoint, and **unauthorized file modifications** on the Ubuntu Server (`/etc/passwd`, `/etc/init.d/`) detected by Wazuh's File Integrity Monitoring. Custom detection rules triggered, and Active Response automatically blocked the attacker IP. No persistence remained after cleanup. **MTTD: ~45 seconds. MTTR: ~2 seconds (automated).**

### Report Structure

| Section | Content |
|---|---|
| **1. Incident Metadata** | ID, severity, classification, status, analyst |
| **2. Timeline of Events** | Timestamped log excerpts (UTC) |
| **3. MITRE ATT&CK Mapping** | T1110.001, T1136.001, T1078.001, T1543, T1037 |
| **4. Indicators of Compromise** | Source IP, usernames, Event IDs, hashes, modified files |
| **5. Impact Assessment** | Affected hosts, data at risk, lateral movement |
| **6. Detection & Response** | Rule IDs fired (incl. FIM rules 550/553/554/597), Active Response executed |
| **7. Root Cause Analysis** | Weak SSH policy; no MFA on local accounts; no FIM alerting policy |
| **8. Recommendations** | Key-based SSH auth, disable Guest, enable MFA, FIM baseline tuning |
| **9. Lessons Learned** | Rule tuning, time sync, dashboard improvements |

### Sample Timeline Excerpt

| Time (UTC) | Host | Event | Rule ID |
|---|---|---|---|
| 14:02:11 | ubuntu-server | Failed SSH password (×1) | 5760 |
| 14:02:29 | ubuntu-server | Failed SSH password (×5) | **100100** |
| 14:02:31 | ubuntu-server | Active Response: IP blocked | 651 |
| 14:05:47 | win10 | Local account created (4720) | **100200** |
| 14:05:52 | win10 | Added to Administrators (4732) | **100201** |
| 14:06:10 | win10 | Successful logon (4624) | 60106 |
| 14:07:33 | win10 | Account deleted (4726) | 60106 |
| 14:09:12 | ubuntu-server | FIM: `/etc/passwd` modified | **550** |
| 14:09:14 | ubuntu-server | FIM: `/etc/init.d/backdoor.sh` added | **554** |
| 14:10:01 | win10 | FIM: Registry Run key modified | **597** |

![Report Cover](docs/screenshots/11-report-cover.png)
![Timeline View](docs/screenshots/12-timeline-view.png)

**✅ Achievement:** Produced an audit-ready report demonstrating written communication — a skill cited in nearly every SOC job description.

---

## 🧠 Skills Demonstrated

| Domain | Evidence |
|---|---|
| **SIEM Administration** | Deployed & tuned Wazuh Manager, Indexer, Dashboard on Ubuntu Server |
| **Log Source Integration** | Windows (Sysmon + Security), Linux (auditd, auth.log) |
| **Detection Engineering** | Custom XML rules with thresholds & MITRE mapping |
| **File Integrity Monitoring (FIM)** | Real-time Linux FIM + Windows registry monitoring, whodata attribution, false-positive tuning |
| **Threat Simulation** | ATT&CK-mapped TTPs (T1110.001, T1136.001, T1078.001, T1543, T1037) |
| **Incident Response** | NIST SP 800-61 lifecycle; auto-containment via Active Response |
| **Dashboarding** | Pivot-ready visualizations in Wazuh Dashboard |
| **Documentation** | Formal incident report with timeline & IOCs |
| **Virtualization** | Designed & deployed multi-VM lab with VMware Workstation, custom VMnet segmentation |

---

## 📚 Lessons Learned

1. **Noise reduction is 80% of rule writing.** Default rules fired constantly; scoping by user and Event ID dropped false positives dramatically.
2. **FIM is powerful but noisy.** `/etc` changes constantly (log rotations, package updates). Ignoring volatile paths and running a post-update baseline is essential.
3. **Whodata turns FIM into an attribution tool.** Knowing *who* and *what process* touched a file is far more actionable than just knowing *which file* changed.
4. **Windows registry is a prime persistence location.** Monitoring `Run`/`RunOnce` keys catches a huge class of real-world malware.
5. **Time synchronization matters.** Mismatched clocks broke timeline correlation — fixed with NTP on all agents.
6. **Active Response needs guardrails.** Always include `<timeout>` to prevent permanent lockouts.
7. **Sysmon is essential on Windows.** Native Security logs alone missed process-level context.
8. **VMware VMnet segmentation is clean.** Using VMnet1 (host-only) for lab traffic and VMnet8 (NAT) for package installs kept the lab isolated while still allowing updates.
9. **Reports win interviews.** The investigation report is what gets discussed in interviews.
10. **Version pinning saves rebuilds.** Wazuh component mismatches broke ingestion once.

---

## 🚀 Roadmap / Future Work

- [ ] Add **Sysmon for Linux** for process-level telemetry on Ubuntu endpoints.
- [ ] Integrate **VirusTotal** and **AbuseIPDB** APIs for IOC enrichment.
- [ ] Deploy **TheHive + Cortex** for case management.
- [ ] Add **Atomic Red Team** tests for broader ATT&CK coverage.
- [ ] Build a **Sigma → Wazuh XML** conversion pipeline.
- [ ] Add a **pfSense** VM for network-level controls and firewall telemetry.
- [ ] Extend FIM to monitor **SSH authorized_keys** and **sudoers** for persistence detection.
- [ ] Take **VMware snapshots** before each attack phase for repeatable rollback.

---

## 🔗 References

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Wazuh FIM Documentation](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [NIST SP 800-61 Rev. 2](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)
- [VMware Workstation Documentation](https://docs.vmware.com/en/VMware-Workstation-Pro/index.html)

---

## 📂 Repository Structure

```
soc-home-lab/
├── README.md
├── docs/
│   ├── INC-2024-001-Investigation.pdf
│   └── screenshots/
│       ├── 00-lab-architecture.png
│       ├── 01-agents-active.png
│       ├── 02-failed-ssh-alerts.png
│       ├── 03-guest-account-events.png
│       ├── 04-successful-logon.png
│       ├── 05-dashboard-overview.png
│       ├── 06-dashboard-drilldown.png
│       ├── 07-rule-test.png
│       ├── 08-custom-alerts.png
│       ├── 09-active-response-log.png
│       ├── 10-iptables-block.png
│       ├── 11-report-cover.png
│       ├── 12-timeline-view.png
│       ├── 13-fim-config.png
│       ├── 14-fim-alerts.png
│       ├── 15-fim-file-diff.png
│       ├── 16-fim-registry-windows.png
│       └── 17-fim-dashboard.png
├── configs/
│   ├── local_rules.xml
│   ├── ossec-manager.conf
│   ├── ossec-windows-agent.conf
│   └── ossec-linux-agent.conf
└── scripts/
    └── ssh-brute-force-sim.sh
```

---

## 👤 About Me

I'm an aspiring **SOC Analyst** actively seeking entry-level or internship opportunities. This lab reflects my commitment to learning the defensive side of security through hands-on engineering — not just theory.

- 📧 **Email:** your.email@example.com
- 💼 **LinkedIn:** [linkedin.com/in/yourhandle](https://linkedin.com/in/yourhandle)
- 🐙 **GitHub:** [github.com/yourhandle](https://github.com/yourhandle)

> 💡 *If you're a hiring manager and would like a walkthrough of this lab, I'd be happy to give a live demo. Just reach out.*

---

⭐ **If this project helped you build your own SOC lab, drop a star — and feel free to open an issue with questions.**
