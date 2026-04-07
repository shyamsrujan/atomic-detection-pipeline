# 🛡️ Atomic Detection Pipeline — MITRE ATT&CK Emulation & SIEM Detection

> **End-to-end threat detection lab:** Emulated a real-world MITRE ATT&CK technique on a Windows host and proved detection through a Wazuh SIEM — from adversary simulation to dashboard alert.



---

## 📌 What This Project Proves

Most security portfolio projects stop at *"I installed a SIEM."* This one goes further:

1. **Attacker perspective** — Executed a real MITRE ATT&CK technique using Atomic Red Team
2. **Telemetry layer** — Captured process-level detail via Sysmon that Windows Event Logs alone miss
3. **Detection layer** — Verified the alert surfaced in Wazuh with correct MITRE ATT&CK mapping
4. **AI augmentation** — Extended the pipeline with an AI-powered alert summarizer that mirrors Microsoft Copilot for Security

---

## 🧱 Lab Architecture

```
Atomic Red Team  →  Sysmon (Event Capture)  →  Wazuh Agent  →  Wazuh Manager  →  Alert + MITRE Mapping
    (Attacker)           (Telemetry)              (Shipper)        (SIEM/Rules)       (Dashboard)
```

### Components

| Component | Tool / Version | Role |
|-----------|---------------|------|
| SIEM / Manager | Wazuh v4.14.3 OVA | Collects logs, applies detection rules, generates alerts |
| Windows Victim | Windows 10 x64 (VirtualBox) | Target machine where the attack was emulated |
| Attack Emulation | Atomic Red Team v2.1.0 | Executes MITRE ATT&CK mapped test cases safely |
| Telemetry | Sysmon (SwiftOnSecurity config) | Deep process, network, and file event logging |
| Log Forwarder | ossec-agent (Wazuh 4.x) | Ships Sysmon & Windows logs to the Wazuh manager |

---

## ⚔️ Technique Emulated

**T1059.001 — Command and Scripting Interpreter: PowerShell**

| Field | Detail |
|-------|--------|
| MITRE Technique ID | T1059.001 |
| Tactic | Execution |
| Sub-technique | PowerShell |
| Atomic Test | Test #1 (`Invoke-AtomicTest T1059.001 -TestNumbers 1`) |
| Target Machine | `DESKTOP-JRU2NP1` (Windows 10) |

**Why PowerShell?**  
Threat actors abuse PowerShell for execution, lateral movement, and post-exploitation because it is built into every modern Windows system and provides native access to the .NET framework and Windows APIs. It is consistently one of the most abused techniques across APT groups (see: FIN7, APT29).

**Execution Command:**
```powershell
Import-Module C:\AtomicRedTeam\invoke-atomicredteam\InvokeAtomicRedTeam.psd1 -Force
Invoke-AtomicTest T1059.001 -TestNumbers 1
```

---

## 🔍 Detection Evidence

### Sysmon Configuration (ossec.conf)

The critical dependency that made detection possible — without this `localfile` block, Wazuh receives **zero** Sysmon data:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Sysmon **Event ID 1 (Process Creation)** captured the PowerShell execution with full command-line arguments — data that standard Windows Security Event Logs (4688) would partially miss without additional audit policy configuration.

### Wazuh Dashboard Results

| Metric | Value |
|--------|-------|
| Total Sysmon Events Detected | 30 |
| High Severity Alerts (Level 12+) | 1 |
| Reporting Agent | DESKTOP-JRU2NP1 |
| MITRE Technique Confirmed | PowerShell (T1059.001) |
| MITRE Tactic Confirmed | Execution |

### Screenshots

**Screenshot 1 — Wazuh Threat Hunting Dashboard: MITRE ATT&CK Wheel showing PowerShell detection**

> *(Add `screenshots/wazuh-mitre-wheel.png` here)*

**Screenshot 2 — Wazuh Events Tab: 432 security events from DESKTOP-JRU2NP1**

> *(Add `screenshots/wazuh-events-tab.png` here)*

---

## 🤖 Extension: AI-Powered Alert Summarizer

**Live Demo:** https://wazuhai-36zejwby.manus.space

To reduce analyst triage time, an AI-powered web app was built on top of the detection pipeline. A SOC analyst can paste any raw Wazuh alert JSON into the tool and receive an instant structured incident report.

**Output includes:**
- Severity rating
- MITRE ATT&CK technique and tactic tags
- Plain-English threat analysis
- Numbered list of recommended response actions

**Stack:** Manus AI (app generation) + Google Gemini 2.0 Flash API

**Why this matters:** This mirrors capabilities found in enterprise tools like **Microsoft Copilot for Security** — built entirely with free tools. It demonstrates LLM-assisted SOC workflow augmentation, a skill increasingly relevant to modern blue team roles.

---

## 🔑 Key Findings

1. **Full end-to-end detection confirmed** — T1059.001 was emulated, captured, shipped, and alerted on within 30–60 seconds of execution.
2. **Sysmon is non-negotiable** — Basic Windows Event Logs lacked the process-level telemetry needed for confident detection. Sysmon's full command-line capture was the difference.
3. **MITRE mapping works out of the box** — Wazuh's ATT&CK module correctly tagged the alert in the Threat Hunting dashboard with no manual rule customization.
4. **ossec.conf localfile block is a critical dependency** — This single config element determines whether Sysmon data flows at all. A misconfigured or missing block = blind SIEM.

---

## 📂 Repo Structure

```
atomic-detection-pipeline/
├── README.md
├── report/
│   └── Atomic_Detection_Pipeline_Report.pdf
├── screenshots/
│   ├── wazuh-mitre-wheel.png
│   ├── wazuh-events-tab.png
│   ├── ai-alert-input.png
│   └── ai-alert-results.png
├── config/
│   └── ossec-localfile-sysmon.xml      # The critical Wazuh agent config block
└── notes/
    └── setup-notes.md                  # Lab setup steps, gotchas, lessons learned
```

---

## 🛠️ How to Replicate This Lab

### Prerequisites
- VirtualBox (free)
- Wazuh OVA v4.x (free download from wazuh.com)
- Windows 10 VM
- Atomic Red Team (PowerShell install)
- Sysmon + SwiftOnSecurity config

### Steps (High Level)
1. Deploy Wazuh OVA and note the manager IP
2. Install Wazuh agent on Windows VM, point to manager IP
3. Install Sysmon with SwiftOnSecurity config
4. Add the `localfile` block to `ossec.conf` (see `config/` folder)
5. Restart Wazuh agent service
6. Install Atomic Red Team on Windows VM
7. Execute `Invoke-AtomicTest T1059.001 -TestNumbers 1`
8. Check Wazuh Threat Hunting dashboard after ~60 seconds

---

## 🎯 Skills Demonstrated

- **SIEM deployment & configuration** (Wazuh)
- **Threat detection engineering** (custom log source integration)
- **MITRE ATT&CK framework** (technique mapping, tactic context)
- **Adversary simulation** (Atomic Red Team)
- **Windows telemetry** (Sysmon Event ID 1, process creation)
- **AI-augmented SOC workflows** (LLM-assisted alert triage)

---

## 👤 Author

**Shyam Srujan Mukkamala**  
M.S. Computer Science — Illinois Institute of Technology  
ISC2 CC | SC-900 | AZ-900 | Splunk Core Certified  
[LinkedIn](https://linkedin.com/in/shyamsrujan) | [Portfolio](https://portfolio-rose-ten-95.vercel.app)
