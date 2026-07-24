---
layout: series
title: Reconnaissance
order: 1
series_title: My Red Team Notebook
---

Purpose: Establish the model for the entire AD attack lifecycle.

Prerequisites for the Overall attack

| Starting Access Level | What You Can Do | Typical Entry Methods |
| --- | --- | --- |
| **True** **External** | OSINT credential gathering. Password spraying against public portals (VPN, OWA, O365) | External penetratoin tests, public credential breaches. |
| **Unauthenticated Internal Network (On LAN, No Credentials)** | AS-REP roasting (if DC port 88 is reachable), LLMNR/NBT-NS poisoning (e.g Responder) | Physical network drop, rogue Wi-Fi, NAC bypass. |
| **Domain user / Machine Account (no local admin)** | AD enumeration (LDAP queries, BloodHound). Kerberoasting. Limited lateral movement to other systems with basic user rights. | Phishing, password spray, breach pivot |
| **Local admin on 1+ machines** | Credential harvesting (LSASS dumping, SAM Dumping). Token theft and impersonation. Pass-the-hash to other systems. Disabling local antivirus/EDR. | Local privilege escalation, local misconfigurations, or client-side exploits. |
| **Physical access** | Boot to USB/Linux, disable defender offline, dump SAM | On-prem physical assessment |
| **SYSTEM on 1+ machines** | Full credential extraction, Accessing the network under the machine account's context (MachineName$). | Local admin → UAC bypass → SYSTEM |
| **Domain Admin / DC Access** | DCSync (extracting password hashes of all domain users). Skeleton key (implanting backdoors in LSASS on the DC). Creating Golden and Silver Tickets. | Exploiting AD certificate Services, abusing delegation or migrating sessions of domain administrators. |

\*\* Minimum starting position: Domain user credentials (username, password). Valid Domain User credentials (username/password) OR a compromised Machine Account (MachineName$). Without authenticated access, you cannot fully enumerate or perform authenticated queries against LDAP, SMB, or Kerberos.

#### The overall attack chain

```markdown
Phase 1: Initial Access 
         (Phishing, External Vulnerabilities, or Password Spraying)
               │
               ▼
Phase 2: Enumeration & Discovery 
         (BloodHound, LDAP mapping, scanning for misconfigurations)
               │
               ▼
Phase 3: Local Privilege Escalation 
         (Standard User ──> Local Admin / SYSTEM on the host)
               │
               ▼
Phase 4: Credential Access 
         (Dumping LSASS/SAM, Harvesting cleartext passwords/tokens)
               │
               ▼
Phase 5: Lateral Movement (LAT) 
         (Using credentials/hashes to jump to other machines via WinRM/RDP)
               │
               ▼
Phase 6: Domain Privilege Escalation (ESC) 
         (Abusing ADCS, Delegation, or GPOs to become Domain Admin)
               │
               ▼
Phase 7: Persistence 
         (Golden Tickets, DC backdoors, or rogue ACL modifications)
               │
               ▼
Phase 8: Action on Objectives 
         (Data exfiltration, ransomware deployment, or specific assessment goals)
```