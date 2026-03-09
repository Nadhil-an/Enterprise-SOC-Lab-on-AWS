# Setup-04 — Windows Endpoint Deployment & Log Forwarding to Splunk

![AWS](https://img.shields.io/badge/Platform-AWS-FF9900?style=flat-square&logo=amazonaws)
![Windows](https://img.shields.io/badge/OS-Windows%2010%2FServer-0078D6?style=flat-square&logo=windows)
![Splunk](https://img.shields.io/badge/SIEM-Splunk-FF6B35?style=flat-square&logo=splunk)
![Active Directory](https://img.shields.io/badge/Domain-soc.local-purple?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)

---

## 📋 Overview

A Windows endpoint EC2 instance was deployed inside the SOC-VPC and joined to the `soc.local` Active Directory domain. Splunk Universal Forwarder was installed and configured to forward Security, System, and Application event logs to the central Splunk SIEM server. This enables real-time monitoring of all endpoint activity — including logins, process creation, service installation, and privilege escalation — forming the primary attack surface for all endpoint detection labs.

---

## 🏗️ Architecture

```
Attacker Machine
      │
      ▼
Internet Gateway (SOC-IGW)
      │
      ▼
SOC-VPC (10.0.0.0/16)
      ├── Windows Endpoint ──► Splunk Forwarder (port 9997) ──► Splunk SIEM
      └── Active Directory (soc.local) ◄── Domain Join
```

<p align="center">
  <img src="assets/ENDPOINT-ARCHITECTURE.png" width="800">
</p>


---

## ⚙️ EC2 Instance Configuration

| Setting | Value |
|---|---|
| AMI | Windows Server 2019 / Windows 10 |
| Instance Type | t2.micro (Free Tier) |
| Network | SOC-VPC |
| Subnet | SOC-Public-Subnet |
| Auto-assign Public IP | Enabled |
| Storage | 30GB gp2 |

---

## 🔐 Security Group — Inbound Rules

| Port | Protocol | Purpose |
|---|---|---|
| 3389 | TCP | RDP access for administration |
| 9997 | TCP | Splunk Universal Forwarder outbound |

---

## ⚙️ Deployment Steps

### Step 1 — Launch Windows EC2 Instance

Launch a Windows instance inside SOC-VPC. Connect via RDP using the Administrator credentials from the key pair.

<p align="center">
  <img src="assets/ENDPOINT-INSTNACE.png" width="800">
</p>
`

---

### Step 2 — Join Endpoint to Active Directory Domain

1. Press `Win + R` → type `sysdm.cpl` → Enter
2. Click **Change** under Computer Name
3. Select **Domain** and enter:

```
soc.local
```

4. Enter Domain Administrator credentials when prompted
5. Restart the machine

After restart, the endpoint is a member of the `soc.local` domain and will generate domain-level security events.


---

### Step 3 — Install Splunk Universal Forwarder

Download Splunk Universal Forwarder (Windows) from splunk.com and run the installer.

During installation configure:

| Setting | Value |
|---|---|
| Deployment Server | `<Splunk-Server-IP>:8089` |
| Receiving Indexer | `<Splunk-Server-IP>:9997` |

---

### Step 4 — Configure Log Monitoring

Open Command Prompt as Administrator and navigate to the Splunk Forwarder bin directory:

```cmd
cd "C:\Program Files\SplunkUniversalForwarder\bin"
```

Add Windows event log monitors:

```cmd
splunk add monitor WinEventLog:Security
splunk add monitor WinEventLog:System
splunk add monitor WinEventLog:Application
```

Restart the forwarder to apply changes:

```cmd
splunk restart
```

---

### Step 5 — Enable Process Creation Command Line Logging

This is required for detecting encoded PowerShell and malware execution in the endpoint labs.

Open Group Policy Editor:

```
Win + R → gpedit.msc
```

Navigate to:

```
Computer Configuration → Administrative Templates
→ System → Audit Process Creation
→ Include command line in process creation events → Enabled
```

Apply policy:

```cmd
gpupdate /force
```

---

### Step 6 — Enable Advanced Audit Policies

Open Command Prompt as Administrator:

```cmd
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"Registry" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
```

Verify:

```cmd
auditpol /get /category:*
```

---

### Step 7 — Verify Logs in Splunk

On the Splunk server, confirm endpoint logs are being received:

```spl
index=end_point
| stats count by EventCode
| sort - count
```

Expected Event IDs:

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4688 | New process created |
| 4657 | Registry value modified |
| 4720 | User account created |
| 4732 | User added to Administrators |
| 7045 | New service installed |

<p align="center">
  <img src="assets/ENDPOINT-SPLUNK.png" width="800">
</p>

---

## ✅ Deployment Summary

| Component | Status |
|---|---|
| Windows EC2 launched | ✅ |
| Joined to `soc.local` domain | ✅ |
| Splunk Universal Forwarder installed | ✅ |
| Security / System / Application logs forwarding | ✅ |
| Process creation command line logging enabled | ✅ |
| Advanced audit policies configured | ✅ |
| Logs visible in Splunk `index=end_point` | ✅ |

---

## 🔗 Next Steps

- [Setup-05 → Web Server (DVWA) Deployment](05_Web_Server_Setup.md)


---

*SOC Lab Infrastructure | Windows Endpoint | Splunk Universal Forwarder | AWS Free Tier | Author: [Nadil](https://github.com/Nadhil-an)*
