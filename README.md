# Wazuh-SOC-Project
# 🛡️ SOC Home Lab — Wazuh SIEM, Detection Engineering & Incident Response

![Wazuh](https://img.shields.io/badge/Wazuh-v4.x-3C8CBE?style=for-the-badge&logo=wazuh&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Windows](https://img.shields.io/badge/Windows_10-Pro-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-7.x-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK_Mapped-red?style=for-the-badge)

> A fully functional Security Operations Center (SOC) home lab built to simulate real-world attack scenarios, generate endpoint and authentication telemetry, engineer custom detection rules, and execute an end-to-end incident response lifecycle — all monitored through a self-hosted Wazuh SIEM.

---

## 📑 Table of Contents

- [Project Overview](#-project-overview)
- [Lab Architecture](#-lab-architecture)
- [Environment & Tools](#-environment--tools)
- [Phase 1 — SIEM Deployment & Agent Enrollment](#phase-1--siem-deployment--agent-enrollment)
- [Phase 2 — Telemetry Generation (Attack Simulation)](#phase-2--telemetry-generation-attack-simulation)
- [Phase 3 — Dashboard Engineering](#phase-3--dashboard-engineering)
- [Phase 4 — Custom Detection Rules](#phase-4--custom-detection-rules)
- [Phase 5 — Active Response](#phase-5--active-response)
- [Phase 6 — Incident Investigation Report](#phase-6--incident-investigation-report)
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
4. **Respond** automatically using Wazuh's Active Response module.
5. **Document** the entire incident through a formal investigation report.

### Detection Engineering Lifecycle

```
Threat Simulation → Log Ingestion → Detection Rule → Alert → Triage → Active Response → Report
```

---

## 🏗️ Lab Architecture

```
                    ┌──────────────────────────────────────────┐
                    │         HOST MACHINE (Hypervisor)        │
                    │              VirtualBox                  │
                    └──────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
        ┌───────────────────┐ ┌───────────────┐ ┌───────────────────┐
        │  Ubuntu Server    │ │ Windows 10 VM │ │ Ubuntu Desktop VM │
        │  (Wazuh Manager   │ │ (Victim /     │ │ (Attacker /       │
        │   + Indexer +     │ │  Endpoint)    │ │  Red Team)        │
        │   Dashboard)      │ │               │ │                   │
        │  192.168.56.10    │ │ 192.168.56.20 │ │ 192.168.56.30     │
        └───────────────────┘ └───────────────┘ └───────────────────┘
                 ▲                    │                  │
                 │                    │                  │
                 └──── Wazuh Agent ───┘                  │
                 └──── Wazuh Agent ──────────────────────┘
                          (Log forwarding on port 1514/1515)
```

**Network:** Internal VirtualBox network `192.168.56.0/24` (host-only + NAT for updates).

![Lab Architecture](docs/screenshots/00-lab-architecture.png)

---

## 🧰 Environment & Tools

| Component | Role | Version / Specs |
|---|---|---|
| **Wazuh Manager** | SIEM, rule engine, active response | v4.x on Ubuntu Server 22.04 |
| **Wazuh Indexer** | Log storage & search | Bundled with Wazuh |
| **Wazuh Dashboard** | Visualization & alert triage | Bundled with Wazuh |
| **Ubuntu Server VM** | Hosts Wazuh stack | 4 vCPU / 8 GB RAM / 60 GB disk |
| **Windows 10 VM** | Monitored endpoint | 2 vCPU / 4 GB RAM |
| **Ubuntu Desktop VM** | Attacker + monitored endpoint | 2 vCPU / 4 GB RAM |
| **Sysmon** | Enhanced Windows telemetry | v15.x (SwiftOnSecurity config) |
| **VirtualBox** | Hypervisor | v7.x |

**Frameworks referenced:** MITRE ATT&CK, NIST SP 800-61, Cyber Kill Chain.

---

## Phase 1 — SIEM Deployment & Agent Enrollment

### 1.1 Wazuh Server Setup

Deployed the Wazuh all-in-one stack on Ubuntu Server using the official installer:

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

**Ubuntu endpoints (Linux agent):**

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
| **MITRE ATT&CK Technique Breakdown** | Coverage at a glance |
| **Active Response Trigger Count** | Track automated remediations |
| **Alert Severity Distribution** | Rule level 0–15 histogram |

### Design Decisions

- **Pivot-friendly**: each viz uses `agent.name` as a global filter.
- **Time-bucketed at 5 min** to make brute-force bursts visually obvious.
- **Saved searches** for `rule.id:5710 OR rule.id:5760` and `win.eventdata.eventID:4720`.

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

## Phase 5 — Active Response

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

## Phase 6 — Incident Investigation Report

Authored a formal incident report (see [`docs/INC-2024-001-Investigation.pdf`](docs/INC-2024-001-Investigation.pdf)) following NIST SP 800-61 structure.

### Executive Summary

> The SOC home lab detected a coordinated attack involving **SSH brute-force from `192.168.56.30`**, followed by **unauthorized local account creation and privilege escalation** on the Windows 10 endpoint. Custom Wazuh detection rules triggered, and Active Response automatically blocked the attacker IP. No persistence remained after cleanup. **MTTD: ~45 seconds. MTTR: ~2 seconds (automated).**

### Report Structure

| Section | Content |
|---|---|
| **1. Incident Metadata** | ID, severity, classification, status, analyst |
| **2. Timeline of Events** | Timestamped log excerpts (UTC) |
| **3. MITRE ATT&CK Mapping** | T1110.001, T1136.001, T1078.001 |
| **4. Indicators of Compromise** | Source IP, usernames, Event IDs, hashes |
| **5. Impact Assessment** | Affected hosts, data at risk, lateral movement |
| **6. Detection & Response** | Rule IDs fired, Active Response executed |
| **7. Root Cause Analysis** | Weak SSH policy; no MFA on local accounts |
| **8. Recommendations** | Key-based SSH auth, disable Guest, enable MFA |
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

![Report Cover](docs/screenshots/11-report-cover.png)
![Timeline View](docs/screenshots/12-timeline-view.png)

**✅ Achievement:** Produced an audit-ready report demonstrating written communication — a skill cited in nearly every SOC job description.

---

## 🧠 Skills Demonstrated

| Domain | Evidence |
|---|---|
| **SIEM Administration** | Deployed & tuned Wazuh Manager, Indexer, Dashboard |
| **Log Source Integration** | Windows (Sysmon + Security), Linux (auditd, auth.log) |
| **Detection Engineering** | Custom XML rules with thresholds & MITRE mapping |
| **Threat Simulation** | ATT&CK-mapped TTPs (T1110.001, T1136.001, T1078.001) |
| **Incident Response** | NIST SP 800-61 lifecycle; auto-containment via Active Response |
| **Dashboarding** | Pivot-ready visualizations in Wazuh Dashboard |
| **Documentation** | Formal incident report with timeline & IOCs |
| **Networking** | Isolated host-only network for safe attack simulation |

---

## 📚 Lessons Learned

1. **Noise reduction is 80% of rule writing.** Default rules fired constantly; scoping by user and Event ID dropped false positives dramatically.
2. **Time synchronization matters.** Mismatched clocks broke timeline correlation — fixed with NTP on all agents.
3. **Active Response needs guardrails.** Always include `<timeout>` to prevent permanent lockouts.
4. **Sysmon is essential on Windows.** Native Security logs alone missed process-level context.
5. **Reports win interviews.** The investigation report is what gets discussed in interviews.
6. **Version pinning saves rebuilds.** Wazuh component mismatches broke ingestion once.

---

## 🚀 Roadmap / Future Work

- [ ] Add **Sysmon for Linux** for process-level telemetry on Ubuntu endpoints.
- [ ] Integrate **VirusTotal** and **AbuseIPDB** APIs for IOC enrichment.
- [ ] Deploy **TheHive + Cortex** for case management.
- [ ] Add **Atomic Red Team** tests for broader ATT&CK coverage.
- [ ] Build a **Sigma → Wazuh XML** conversion pipeline.
- [ ] Add a **pfSense** VM for network-level controls and firewall telemetry.

---

## 🔗 References

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [NIST SP 800-61 Rev. 2](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)

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
│       └── 12-timeline-view.png
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
