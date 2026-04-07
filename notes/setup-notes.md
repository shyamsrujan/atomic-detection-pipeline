# Lab Setup Notes — Atomic Detection Pipeline

> Personal notes from building the MITRE ATT&CK detection pipeline.  
> Written for my future self and anyone replicating this lab from scratch.

---

## Environment

- **Host machine:** Single physical host running all VMs via VirtualBox
- **Network mode:** Internal network — Windows VM and Wazuh OVA can communicate; no internet exposure needed for the core pipeline
- **Wazuh version:** v4.14.3 OVA (imported directly into VirtualBox)
- **Windows VM:** Windows 10 x64

---

## Step-by-Step Setup

### 1. Deploy Wazuh OVA

- Download the Wazuh OVA from [wazuh.com/install](https://wazuh.com/install/)
- Import into VirtualBox: `File → Import Appliance`
- Note the Wazuh manager IP after boot — you need this for agent enrollment
- Default credentials are in the Wazuh documentation (change them)

### 2. Enroll the Windows Agent

On the Windows VM, download and install the Wazuh agent MSI.  
During install, set the **Manager IP** to your Wazuh OVA's IP address.

Verify the agent is connected in the Wazuh dashboard under `Agents`.

### 3. Install Sysmon

Download Sysmon from Microsoft Sysinternals.  
Use the SwiftOnSecurity config (the default Sysmon config misses too much):

```powershell
# Download SwiftOnSecurity config
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile sysmonconfig.xml

# Install Sysmon with the config
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Verify Sysmon is running:
```powershell
Get-Service Sysmon64
```

### 4. Configure ossec.conf — THE CRITICAL STEP

This is the single most important config change. Without it, **Wazuh receives zero Sysmon data** even though Sysmon is running and logging perfectly.

Location of ossec.conf on Windows agent:
```
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Add this block inside the `<ossec_config>` section:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

After saving, restart the Wazuh agent service:
```powershell
Restart-Service -Name WazuhSvc
```

**Why this matters:**  
Wazuh doesn't automatically ingest every Windows event channel. You have to explicitly tell the agent which channels to forward. The Sysmon channel (`Microsoft-Windows-Sysmon/Operational`) is not included by default. If you skip this step and wonder why no Sysmon events appear in the dashboard — this is why.

### 5. Install Atomic Red Team

```powershell
# Run as Administrator
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
```

Verify install:
```powershell
Import-Module C:\AtomicRedTeam\invoke-atomicredteam\InvokeAtomicRedTeam.psd1 -Force
Invoke-AtomicTest T1059.001 -ShowDetails
```

### 6. Execute the Technique

```powershell
Import-Module C:\AtomicRedTeam\invoke-atomicredteam\InvokeAtomicRedTeam.psd1 -Force
Invoke-AtomicTest T1059.001 -TestNumbers 1
```

Wait **30–60 seconds** for log shipping to complete, then check the Wazuh dashboard.

---

## Verifying Detection in Wazuh

1. Go to **Threat Hunting** in the Wazuh dashboard
2. Filter by agent: `DESKTOP-JRU2NP1` (or your machine name)
3. Filter by log source: `Microsoft-Windows-Sysmon/Operational`
4. Look for **MITRE ATT&CK** wheel — T1059.001 / Execution tactic should appear
5. Check **Alerts** tab for Level 12+ severity hits

---

## Lessons Learned / Gotchas

### ❌ Gotcha 1: ossec.conf missing Sysmon localfile block
**Symptom:** Sysmon is running, you can see events in Windows Event Viewer, but nothing appears in Wazuh.  
**Root cause:** Wazuh agent was never told to read that event channel.  
**Fix:** Add the `<localfile>` block above and restart `WazuhSvc`.

### ❌ Gotcha 2: Windows Defender blocking Atomic Red Team
**Symptom:** `Invoke-AtomicTest` runs but the actual technique doesn't execute, or files get quarantined.  
**Fix:** Add an exclusion for `C:\AtomicRedTeam\` in Windows Defender settings. This is a lab machine — acceptable for a sandboxed VM.

### ❌ Gotcha 3: VirtualBox network misconfiguration
**Symptom:** Agent shows as disconnected in Wazuh dashboard even after install.  
**Fix:** Both VMs must be on the same VirtualBox internal network. Check `Settings → Network → Adapter` on each VM. Using NAT on both prevents them from seeing each other.

### ❌ Gotcha 4: Wazuh agent not restarted after ossec.conf edit
**Symptom:** You added the localfile block but still no Sysmon events.  
**Fix:** Config changes require a service restart. Run `Restart-Service -Name WazuhSvc` after every ossec.conf edit.

### ✅ What Sysmon Event ID 1 gives you that Windows Security logs don't
Standard Windows Security Event 4688 (Process Creation) requires explicit audit policy configuration to capture command-line arguments, and even then the data is less structured. Sysmon Event ID 1 captures:
- Full command-line with arguments
- Parent process name and PID
- Process GUID (for correlation)
- File hashes (MD5, SHA256)
- Current working directory

This is the difference between "PowerShell ran" and "PowerShell ran this exact command from this parent process with these hashes."

---

## Detection Rule Notes

Wazuh's built-in rules mapped T1059.001 without any custom rule writing. The relevant Wazuh rule file is:

```
/var/ossec/ruleset/rules/0800-sysmon_id_1.xml
```

If you want to write custom rules (e.g., alert on specific PowerShell flags like `-EncodedCommand` or `-WindowStyle Hidden`), that's the file to extend.

---

## Time to Detection

From `Invoke-AtomicTest` execution to alert appearing in Wazuh dashboard: **~30–60 seconds**

This lag is log shipping + rule processing time, not detection latency in a real agent-based deployment.

---

## What I'd Add Next

- [ ] Test additional techniques: T1003.001 (LSASS dump), T1055 (Process Injection)
- [ ] Write a custom Wazuh rule for encoded PowerShell (`-EncodedCommand`)
- [ ] Add Wazuh active response to automatically isolate the agent on a Level 12+ alert
- [ ] Integrate with a ticketing system (TheHive or JIRA) for full SOC workflow simulation
- [ ] Swap Gemini for Claude API in the AI summarizer

---

*Author: Shyam Srujan Mukkamala | Lab Date: February 2026*
