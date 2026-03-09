# Setup-03 — Active Directory Domain Controller Setup on AWS

![AWS](https://img.shields.io/badge/Platform-AWS-FF9900?style=flat-square&logo=amazonaws)
![Windows Server](https://img.shields.io/badge/OS-Windows%20Server%202022-0078D6?style=flat-square&logo=windows)
![Active Directory](https://img.shields.io/badge/Service-Active%20Directory-purple?style=flat-square)
![Splunk](https://img.shields.io/badge/SIEM-Splunk-FF6B35?style=flat-square&logo=splunk)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)

---

## 📋 Overview

A Windows Server EC2 instance was deployed inside the SOC-VPC and configured as an Active Directory Domain Controller. Domain users and security groups were created to simulate a real enterprise environment. Advanced audit policies were enabled to generate security events across all monitored categories. Logs were forwarded to the Splunk SIEM server via Universal Forwarder for centralized monitoring and detection.

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
      ├── Active Directory DC (Windows Server) ──► Splunk Forwarder ──► Splunk SIEM
      ├── Windows Endpoint                      ──► Splunk Forwarder ──┘
      └── Ubuntu Web Server                     ──► Splunk Forwarder ──┘
                                                                        │
                                                                        ▼
                                                                  SOC Analyst
```

<p align="center">
  <img src="assets/ad-architecture.png" width="800">
</p>


---

## ⚙️ EC2 Instance Configuration

| Setting | Value |
|---|---|
| AMI | Windows Server 2022 |
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
| 8000 | TCP | Splunk Web Interface |

---

## ⚙️ Deployment Steps

### Step 1 — Launch Windows Server EC2

Launch a Windows Server 2022 instance inside SOC-VPC with the security group rules above. Connect via Remote Desktop Protocol (RDP) using the key pair password.

<p align="center">
  <img src="assets/ad-instance.png" width="800">
</p>

---

### Step 2 — Install Active Directory Domain Services (AD DS)

1. Open **Server Manager**
2. Click **Add Roles and Features**
3. Select **Active Directory Domain Services (AD DS)**
4. Complete installation
5. Click **Promote this server to a domain controller**
6. Select **Add a new forest**
7. Set root domain name:

```
soc.local
```

8. Set DSRM password
9. Complete wizard and restart server


<p align="center">
  <img src="assets/ad-setup.png" width="800">
</p>

---

### Step 3 — Create Domain Users

1. Open **Active Directory Users and Computers**
2. Expand `soc.local`
3. Right-click **Users** → **New** → **User**
4. Create the following test accounts:

| Username | Role | Purpose |
|---|---|---|
| john.doe | Standard User | Simulate normal user activity |
| jane.admin | Domain Admin | Simulate privileged account |
| svc.splunk | Service Account | Splunk log forwarding |

> **Screenshot — Active Directory Users and Computers:**
> `assets/ad-users.png`

---

### Step 4 — Configure Advanced Audit Policies

Open **Group Policy Management** → Default Domain Policy → Edit

Navigate to:
```
Computer Configuration → Policies → Windows Settings
→ Security Settings → Advanced Audit Policy Configuration
```

Enable the following audit categories:

| Category | Events Generated | Monitored For |
|---|---|---|
| Logon/Logoff — Success | 4624 | Successful logins |
| Logon/Logoff — Failure | 4625 | Failed logins / brute force |
| Account Lockout | 4740 | Account lockout after failures |
| Account Management | 4720, 4727, 4728, 4732 | User/group creation and modification |
| Privilege Use | 4672 | Privileged account usage |
| Process Creation | 4688 | New process execution |
| Directory Service Changes | 5136 | AD object modifications |

Apply policy:

```cmd
gpupdate /force
```

> **Screenshot — Advanced Audit Policy Configuration:**
> `assets/audit-policy.png`

---

### Step 5 — Install Splunk Universal Forwarder

Download Splunk Universal Forwarder from splunk.com and install on the Domain Controller.

During installation configure:

```
Deployment Server: <Splunk-Server-IP>:8089
Receiving Indexer:  <Splunk-Server-IP>:9997
```

After installation, configure inputs to forward Security and System event logs:

```
Settings → Forwarding and Receiving → Forward Data
Index: windows
Log: Security, System, Application
```

> **Screenshot — Universal Forwarder Installed:**
> `assets/uf-installed.png`

---

### Step 6 — Verify Logs in Splunk

On the Splunk server, confirm logs are being received:

```spl
index=windows
| stats count by EventCode
| sort - count
```

Expected Event IDs visible:

```
4624 — Successful Logon
4625 — Failed Logon
4688 — Process Creation
4720 — User Account Created
```

<p align="center">
  <img src="assets/ad-splunk.png" width="800">
</p>

---

## ✅ Deployment Summary

| Component | Status |
|---|---|
| Windows Server EC2 launched | ✅ |
| Active Directory (AD DS) installed | ✅ |
| Domain `soc.local` created | ✅ |
| Domain users created | ✅ |
| Advanced audit policies enabled | ✅ |
| Splunk Universal Forwarder installed | ✅ |
| Logs forwarding to Splunk (index=windows) | ✅ |

---

## 🔗 Next Steps

- [Setup-04 → Web Server (DVWA) Deployment](04_Web_Server_Setup.md)
- [AD-01 → Brute Force Attack Detection (Event ID 4625)](../active-directory-attacks/AD-01_Brute_Force_RDP_4625.md)
- [AD-02 → Account Lockout Detection (Event ID 4740)](../active-directory-attacks/AD-02_Account_Lockout_4740.md)
- [AD-03 → Backdoor User & Group Creation (Event ID 4720)](../active-directory-attacks/AD-03_User_Group_Creation.md)

---

*SOC Lab Infrastructure | Active Directory on Windows Server 2022 | AWS Free Tier | Author: [Nadil](https://github.com/Nadhil-an)*
