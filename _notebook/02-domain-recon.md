---
layout: series
title: Domain Recon
order: 2
series_title: My Red Team Notebook
---

>  Purpose:** Map the domain comprehensively before choosing an attack path. This is **Phase 1** of the overarching attack chain.

## WHEN Table: Enumeration Tool Selection

| Tool                                    | Required Privileges                                                | Detection Risk (Noise)                                                                                                                       | When to Use                                                                                                                                            | When to Avoid                                                                                                                                                                                                                                        |
| --------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PowerView (PowerSploit)**             | Domain user (normal)                                               | **HIGH** — every cmdlet is signatured; ScriptBlock logging captures all output; AMSI scans every script block                                | Interactive ops where you have AMSI + logging bypassed AND time is not critical AND you need full PowerView cmdlet coverage                            | Any environment with EDR, AMSI active, ScriptBlock logging, or CLM ConstrainedLanguage                                                                                                                                                               |
| **ADModule (Microsoft)**                | Domain user                                                        | **LOW** — signed Microsoft module; cmdlets like `Get-ADUser` are legitimate admin operations                                                 | Environments where you blend in with legitimate IT activity; when you have RSAT installed on a jump box; when you can install ADModule silently        | When RSAT/ADModule not installed (requires `Install-WindowsFeature RSAT-AD-PowerShell`); missing cmdlets for ACL abuse (no `Get-DomainObjectAcl` equivalent)                                                                                         |
| **SharpView (.NET)**                    | Domain user                                                        | **MEDIUM** — .NET binary, not signatured like PowerView, but .NET execution monitored by EDR                                                 | CLM ConstrainedLanguage (PowerView blocked); when you can execute .NET binaries but not PowerShell scripts                                             | When .NET execution is also blocked (AppLocker, WDAC); when no .NET runtime available                                                                                                                                                                |
| **BloodHound (SharpHound collector)**   | Domain user (collector); Neo4j + BloodHound UI on attacker machine | **MEDIUM-HIGH** — SharpHound generates large volumes of LDAP queries; anomaly detection may trigger on sudden LDAP query spike from one host | When you need automated attack path analysis; when the domain is large (>500 objects); when you need to identify complex ACL chains                    | When LDAP traffic volume is monitored (SIEM alert on >100 LDAP queries/min); when you cannot install Neo4j; when the domain is small (<100 objects — manual is faster); when you are time-constrained and don't have time to load + query BloodHound |
| **BloodHound.py (Linux)**               | Domain user                                                        | **MEDIUM** — same LDAP volume concern; fewer collection methods (no session collection from Linux)                                           | When you operate from Linux and need BloodHound graph; when you cannot execute binaries on target                                                      | Same LDAP noise concerns as SharpHound; cannot collect sessions, local admin, or logged-on data                                                                                                                                                      |
| **ADExplorer (Sysinternals)**           | Domain user                                                        | **MEDIUM** — legitimate Microsoft tool, but taking full AD snapshot is anomalous                                                             | When you need an offline AD snapshot for manual analysis; when LDAP queries per-minute are rate-limited but a full snapshot is acceptable in one burst | When network bandwidth is constrained (AD snapshot can be 50 MB-1 GB+)                                                                                                                                                                               |
| **ldapsearch / ldapdomaindump (Linux)** | Domain user (or anonymous if allowed)                              | **LOW-MEDIUM** — standard LDAP queries; blends with directory sync tools                                                                     | Linux-only ops; automated HTML/JSON report generation                                                                                                  | No Windows environment available; when you need SMB/RPC-based enumeration                                                                                                                                                                            |
| **CrackMapExec (CME)**                  | Domain user (for SMB enumeration)                                  | **MEDIUM** — SMB connections to every machine are anomalous                                                                                  | Enumerate live hosts, SMB shares, login test, Kerberoasting all from one tool                                                                          | Heavy EDR environment (CME sequential SMB connections trigger lateral movement alerts)                                                                                                                                                               |
| **enum4linux-ng**                       | anonymous or domain user                                           | **HIGH** (NULL session attempts and RPC enumeration are heavily signatured)                                                                  | External assessment; initial foothold; when no domain creds available                                                                                  | Modern Windows environments (NULL sessions disabled by default since Server 2003 SP1)                                                                                                                                                                |

## PHASE Workflow 

### BASIC DOMAIN RECON — No Special Permissions Required

**Prerequisite:** Domain user credentials (username + password + FQDN of domain).

**Goal:** Understand the domain structure, identify high-value targets, detect defensive posture.

**Commands to run FIRST, in this exact order:**

#### 1.1 — Resolve Domain Identity

```powershell
# Confirm domain name, DC, and functional level
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Output: Name, DomainControllers, DomainMode (functional level)
# Functional level matters: 2016+ functional level enables new Kerberos features
```

#### 1.2 — Get Domain SID

```powershell
Get-DomainSID
# OR (via .NET):
(New-Object System.DirectoryServices.DirectoryEntry("LDAP://RootDSE")).objectSid
```

**Why first:** The domain SID is required to construct `S-1-5-21-<DOMAIN>-500` (Administrator), `-512` (Domain Admins), `-519` (Enterprise Admins), etc. Without the domain SID, you cannot identify these groups.

#### 1.3 — List Domain Controllers

```powershell
Get-DomainController | select Name, IPAddress, SiteName, IsGlobalCatalog, ForestName
```

**Why:** You need to know which DCs exist, which site they're in, and whether any are global catalogs (contain partial replicas of all domains in the forest). Site info tells you network segmentation.

#### 1.4 — List Domain Admins

```powershell
Get-DomainGroupMember -Identity "Domain Admins" | select MemberName, MemberDistinguishedName
```

**Why:** Identifies the primary targets. You need DA names for credential targeting and session hunting.

#### 1.5 — Enumerate All Domain Users (Basic)

```powershell
Get-DomainUser -Properties samaccountname, displayname, memberof, useraccountcontrol, pwdlastset, lastlogon, description
```

**Why:** `useraccountcontrol` reveals disabled accounts, accounts that don't require preauth (AS-REP roastable), accounts with password-never-expires. `description` often contains passwords or sensitive notes. `pwdlastset` identifies stale accounts (easier to target without triggering user alerting). `lastlogon` (not `lastlogontimestamp`) requires querying each DC individually but reveals accurate last login.

#### 1.6 — Identify Kerberoastable Users

```powershell
Get-DomainUser -SPN | select samaccountname, serviceprincipalname, memberof
```

**Why:** Users with SPNs are Kerberoastable. Their TGS tickets can be requested by any domain user and cracked offline. This is the first escalation vector to check.

#### 1.7 — List All Computers and Operating Systems

```powershell
Get-DomainComputer -Properties dnshostname, operatingsystem, operatingsystemservicepack, lastlogontimestamp
```

**Why:** Identify legacy systems (Server 2003, 2008, XP, 7) which may have known exploits. Identify DCs (operatingsystem contains "Server" + high `lastlogontimestamp`). Identify which machines are active.

#### 1.8 — List Domain Groups

```powershell
Get-DomainGroup -Properties name, samaccountname, admincount, memberof | Where-Object { $_.admincount -eq 1 }
```

**Why:** `admincount=1` marks protected groups (Domain Admins, Enterprise Admins, Schema Admins, Administrators, Account Operators, Server Operators, etc.). Members of these groups have their ACLs regularly reset by SDProp. Essential to know which groups are "protected."

#### 1.9 — Check Password Policy

```powershell
Get-DomainPolicy -Policy PasswordPolicy
```

**Why:** Determines minimum password length, complexity requirements, and lockout threshold. Critical for password spraying (you need to stay UNDER the lockout threshold). A lockout threshold of 0 means NO lockout (rare but devastating).

#### 1.10 — Identify Exchange Servers and SQL Servers

```powershell
# Exchange
Get-DomainGroupMember -Identity "Exchange Windows Permissions" | select MemberName
Get-DomainGroupMember -Identity "Organization Management"

# SQL (via SPN)
Get-DomainUser -SPN "*MSSQL*"
```

**Why:** Exchange has the `WriteDacl` privilege on the domain root (Exchange Windows Permissions group). This is a well-known escalation path to DA. SQL service accounts often have high privileges or interesting data.

#### 1.11 — Check LAPS

```powershell
Get-DomainComputer -Properties ms-mcs-admpwd, ms-mcs-admpwdexpirationtime | Where-Object { $_.'ms-mcs-admpwd' -ne $null }
```

**Why:** If LAPS is deployed, the local admin password is stored in `ms-mcs-admpwd` and readable by users with `AllExtendedRights` on that computer. If YOU have that right, you have local admin on that machine. If not, LAPS still tells you the environment has centralized local admin management (harder lateral movement).

#### 1.12 — Map Group Policy Objects (GPOs)

```powershell
Get-DomainGPO | select displayname, gpcfilepath
```

**Why:** GPOs control security settings across the domain. Identifying GPOs with "Firewall", "AppLocker", "AV", "Defender" in the name reveals defensive GPOs. Identifying GPOs that are applied to DCs reveals DC hardening level.

## ACL, GPO, and Trust Mapping

**Prerequisite:** Phase 1 complete. You have identified attack candidates and need detailed exploitation paths.

#### 2.1 — ACL Enumeration

```powershell
# Find interesting ACEs where a non-privileged user/group has rights over a privileged object
Find-InterestingDomainAcl -ResolveGUIDs | select IdentityReferenceName, ObjectDN, ActiveDirectoryRights

# Find all ACEs for a specific user
Get-DomainObjectAcl -Identity "TargetUser" -ResolveGUIDs | Where-Object {
    $_.SecurityIdentifier -eq (ConvertTo-SID "SourceUser")
}

# Find all principals that can DCSync
Get-DomainObjectAcl -SearchBase "DC=domain,DC=local" -ResolveGUIDs | Where-Object {
    $_.ObjectAceType -eq 'DS-Replication-Get-Changes' -or
    $_.ObjectAceType -eq 'DS-Replication-Get-Changes-All'
}
```

**Key ACLs to hunt (in priority order):**

| ACE Right | GUID | What It Allows | Priority |
|---|---|---|---|
| `DS-Replication-Get-Changes` | `1131f6aa-...` | DCSync | **CRITICAL** |
| `DS-Replication-Get-Changes-All` | `1131f6ad-...` | DCSync (including protected) | **CRITICAL** |
| `GenericAll` on a user | — | Full control (reset password, change group membership) | HIGH |
| `GenericAll` on a group | — | Add self to group (e.g., Domain Admins) | HIGH |
| `GenericAll` on a computer | — | RBCD attack | HIGH |
| `WriteDacl` on domain root | — | Grant self DCSync rights | **CRITICAL** |
| `WriteOwner` on a group | — | Become owner → modify ACL → add self | HIGH |
| `Self` with `Self-Membership` on group | `bf9679c0-...` | Add self to group | HIGH |
| `WriteProperty` on `ServicePrincipalName` | — | Targeted Kerberoasting (set SPN on user, then Kerberoast) | MEDIUM |
| `WriteProperty` on `msDS-KeyCredentialLink` | — | Shadow Credentials attack (add key material for PKINIT) | HIGH |
| `ForceChangePassword` | `00299570-...` | Reset user's password (target DA) | CRITICAL |
| `ExtendedRight` on computer (LAPS) | — | Read LAPS password → local admin | HIGH |

#### 2.2 — GPO Analysis

```powershell
# Find GPOs applied to specific OU (e.g., Domain Controllers OU)
Get-DomainOU | Get-DomainObjectAcl -ResolveGUIDs | Where-Object {
    $_.ObjectAceType -eq 'GP-Link'
}

# Find GPOs that modify local admin group
Get-DomainGPO | ForEach-Object {
    $gpo = $_
    Get-DomainGPOComputerLocalGroupMapping -GPO $gpo | Where-Object {
        $_.GroupName -eq 'Administrators'
    }
}

# Check if any non-privileged user can modify GPOs
Get-DomainGPO | Get-DomainObjectAcl -ResolveGUIDs | Where-Object {
    $_.ActiveDirectoryRights -match 'WriteProperty|GenericAll|WriteDacl|WriteOwner'
}
```

**Why GPO analysis matters:**

- A GPO applied to the Domain Controllers OU that grants a user local admin on DCs = immediate Domain Admin.
- A GPO that can be edited by a non-DA user with `WriteProperty` = you can inject a scheduled task as SYSTEM.
- GPOs that deploy software (MSI installers) = you can replace the MSI on the distribution point.
- GPOs that configure services, firewall rules, or registry keys = you can abuse each.

#### 2.3 — Trust Mapping (Full Flow)

See Section 5 below for the complete step-by-step trust enumeration flow.

#### 2.4 — Delegation Enumeration

```powershell
# Unconstrained delegation
Get-DomainComputer -Unconstrained | select dnshostname, samaccountname

# Constrained delegation (with protocol transition)
Get-DomainUser -TrustedToAuth | select samaccountname, msds-allowedtodelegateto

# Resource-Based Constrained Delegation (RBCD)
Get-DomainComputer | Get-DomainObjectAcl -ResolveGUIDs | Where-Object {
    $_.ObjectAceType -eq 'ms-DS-Allowed-To-Act-On-Behalf-Of-Other-Identity'
}
```

## USER HUNTING

**Prerequisite:**

- You have local admin on at least one machine in the domain (enables session collection).
- OR you have a machine account context (SYSTEM on a domain-joined machine).
- Phase 1–3 has identified high-value targets (DA users, privileged groups).

**If you are a normal domain user (no local admin):** You CANNOT do user hunting via session enumeration (requires SMB/admin access to target machines). Skip to credential hunting (Kerberoasting, AS-REP roasting) or ACL abuse.

#### 3.1 — Find Machines Where Domain Admins Are Logged In

```powershell
# PowerView — run from a machine where you have local admin
Find-DomainUserLocation

# Check sessions on specific machine
Get-NetSession -ComputerName "workstation01.domain.local"
```

**How it works:** Queries the `srvsvc` named pipe (SMB) on each target machine to enumerate active sessions. Requires local admin on the machine being queried.

#### 3.2 — Find Domain Admin Sessions on Current Machine

```powershell
# List all sessions on current machine
Get-NetSession -ComputerName localhost

# List logged-on users on current machine
Get-NetLoggedon -ComputerName localhost
```

#### 3.3 — Session Correlation

```powershell
# Match Domain Admins to machines where they have sessions
$da_users = Get-DomainGroupMember -Identity "Domain Admins" | % { $_.MemberName }
$sessions = Find-DomainUserLocation | Where-Object { $_.UserName -in $da_users }
$sessions | select UserName, ComputerName, SessionFrom
```

**Why this is the end-game of enumeration:**

- If you have local admin on WORKSTATION01, and a DA has a session there, you can:
    1. Steal the DA's token (incognito).
    2. Dump LSASS on WORKSTATION01 (may contain DA credentials).
    3. Key-log the DA session (non-operationally safe, but illustrates the point).
- This is the highest-value Phase 4 input.

---

## Comparison: PowerView vs. ADModule vs. SharpView

| Criterion | PowerView | ADModule | SharpView |
|---|---|---|---|
| **CLM Compatibility** | NO — Blocked in ConstrainedLanguage | PARTIAL — Some cmdlets work; advanced features fail | YES — Runs as .NET binary, not PowerShell script |
| **Detection Risk** | HIGH — Every cmdlet signatured; ScriptBlock logging captures full output | LOW — Legitimate Microsoft module; blend with IT admin activity | MEDIUM — .NET binary execution monitored; less signatured than PS scripts |
| **Output Completeness** | VERY HIGH — 200+ cmdlets covering every AD enumeration scenario | MEDIUM — ~50 cmdlets; no ACL-specific cmdlets; no session enumeration; no SPN-user hunting | HIGH — ~95% of PowerView ported; missing some edge-case cmdlets |
| **Ease of Delivery** | LOW — Must bypass AMSI + CLM + AppLocker + ScriptBlock logging; heavy obfuscation required | HIGH — Can be installed on a jump box via `Install-WindowsFeature`; or loaded from disk | MEDIUM — Must compile (File 03); NetLoader delivery; requires .NET runtime on target |
| **Accepts Pipeline Input** | YES — Strong pipeline support between cmdlets | YES — Standard PowerShell pipeline | Manual — Only takes arguments; no pipeline between commands |
| **Offline AD Snapshot** | NO — Requires live LDAP | NO — Requires live LDAP | NO — Requires live LDAP |
| **Cross-Domain Enumeration** | YES — Supports trust traversal natively | PARTIAL — Supports `-Server` parameter; no automated trust traversal | YES — Supports `--domain` parameter for each domain; manual stitching |
| **Verbose ACL Output** | YES — Resolves GUIDs to human-readable ACE types | NO — `Get-ACL` on AD objects returns raw GUIDs; must resolve manually | YES — GUID resolution built-in |
| **GPO Analysis** | YES — Full GPO local group mapping, GPP parsing | PARTIAL — `Get-GPO` exists in GroupPolicy module; lacks local group mapping | YES — Full GPO mapping |
| **Session Enumeration** | YES — `Get-NetSession`, `Find-DomainUserLocation` | NO — Not available | YES — `Get-NetSession` |
| **Trust Enumeration** | YES — `Get-DomainTrust`, `Get-ForestTrust` | YES — `Get-ADTrust` | YES — `Get-DomainTrust`, `Get-ForestTrust` |
| **SPN Scanning (port scan via SPN)** | NO — Not built-in (use `Get-SPN` from Discover-PSInterestingServices) | NO | NO |

### Decision Rule

```
IF (CLM == FullLanguage AND AMSI bypass available AND interactive session AND need for maximum feature coverage)
    → PowerView (with heavy obfuscation per File 02)
ELSE IF (CLM == ConstrainedLanguage OR need to blend with legit admin activity)
    → SharpView (compile + obfuscate per File 03)
ELSE IF (on a legitimate admin jump box with RSAT installed AND blend-in is critical)
    → ADModule (legitimate admin tool usage)
ELSE IF (Linux attacker machine)
    → BloodHound.py + ldapsearch + CrackMapExec
```

## TRUST ENUMERATION FLOW

Trust enumeration is the pathway to cross-domain escalation (File 09). Understanding trusts is essential before Phase 4 escalations that involve cross-boundary movement.

### Step-by-Step Trust Enumeration

#### Step 1: Discover Domain Trusts

**Purpose:** Identify all trusted domains and the direction/type of trust.

```powershell
# PowerView
Get-DomainTrust

# ADModule
Get-ADTrust -Filter *

# .NET (C#)
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().GetAllTrustRelationships()
```

**Output columns to capture:**

| Column | Meaning | Why Important |
|---|---|---|
| `SourceName` | Current domain | — |
| `TargetName` | Trusted domain | The domain you can potentially move to |
| `TrustDirection` | Inbound, Outbound, Bidirectional | OUTBOUND = you (source) trust them. INBOUND = they trust you. Bidirectional = both. |
| `TrustType` | ParentChild, TreeRoot, External, Forest, Kerberos | Dictates attack path: ParentChild = possible EA escalation. Forest = possible SID filtering bypass. External = NT4 non-transitive; limited. |
| `TrustAttributes` | Bitmask (e.g., `FOREST_TRANSITIVE`, `WITHIN_FOREST`, `QUARANTINED_DOMAIN`) | `FOREST_TRANSITIVE` = trust extends to all domains in the forest. `QUARANTINED_DOMAIN` = SID filtering enabled (blocks SIDHistory abuse). |

#### Step 2: Discover Forest Trusts

**Purpose:** When the current domain is in a forest, discover all forest-level trusts (cross-forest to other forests).

```powershell
# PowerView
Get-ForestTrust

# ADModule
Get-ADTrust -Filter {ForestTransitive -eq $true}

# .NET (C#)
[System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest().GetAllTrustRelationships()
```

**Why forest trusts matter:** A forest trust that is bidirectional and has SID filtering disabled allows a DA in Forest A to forge the Enterprise Admin SID of Forest B. This is the Holy Grail of cross-domain escalation.

#### Step 3: Map Trust Transitivity

**Purpose:** Understand the reach of each trust. A transitive trust means the trust extends to all child domains.

```powershell
# For each trust, check if it's transitive
Get-DomainTrust | ForEach-Object {
    $trust = $_
    $isTransitive = ($trust.TrustAttributes -band 2) -eq 2  # 0x2 = TRUST_ATTRIBUTE_NON_TRANSITIVE?
    Write-Host "$($trust.TargetName): Transitive=$(-not $isTransitive)"
}
```

#### Step 4: Enumerate Trusted Domain Objects (TDOs)

**Purpose:** Get the full LDAP object for each trust (contains additional attributes like trust account password change interval).

```powershell
Get-DomainObject -LDAPFilter "(objectClass=trustedDomain)" -Properties trustPartner, trustDirection, trustType, trustAttributes, flatName, trustAuthIncoming, trustAuthOutgoing
```

#### Step 5: Check SID Filtering Status

**Purpose:** SID filtering removes non-domain SIDs from authentication tokens across trusts. If disabled, SIDHistory injection works.

```powershell
# Check for QUARANTINED_DOMAIN (0x4) attribute
Get-DomainTrust | ForEach-Object {
    $sidFilteringEnabled = ($_.TrustAttributes -band 4) -eq 4
    Write-Host "$($_.TargetName): SID Filtering = $sidFilteringEnabled"
    if (-not $sidFilteringEnabled) {
        Write-Host "  [!] SID FILTERING DISABLED — SIDHistory abuse possible!"
    }
}
```

#### Step 6: Enumerate Across the Trust (If Credentials Allow)

**Purpose:** If you have credentials valid in the trusted domain (e.g., same username/password, or you've cracked credentials), start Phase 1 enumeration in the trusted domain.

```powershell
# PowerView — enumerate users in trusted domain
Get-DomainUser -Domain "trusted.local" -Properties samaccountname, memberof

# ADModule
Get-ADUser -Server "dc.trusted.local" -Filter *

# SharpHound — collect from trusted domain
.\SharpHound.exe -c DCOnly -d trusted.local

# BloodHound.py
bloodhound-python -u 'user@trusted.local' -p 'pass' -d trusted.local -dc dc.trusted.local -c DCOnly
```

#### Step 7: Check for Foreign Group Membership

**Purpose:** A user in Domain A that is a member of Domain Admins in Domain B = cross-trust privilege escalation candidate.

```powershell
Get-DomainGroup -Domain "target.local" | ForEach-Object {
    $group = $_
    Get-DomainGroupMember -Identity $group.distinguishedname -Domain "target.local" | Where-Object {
        $_.MemberDomain -ne "target.local"
    }
}
```

#### Trust Enumeration Summary: What to Do With Each Trust Type

| Trust Type | Direction | What It Means | Attack Path |
|---|---|---|---|
| **Parent-Child** | Bidirectional (implicit) | Current domain is child of parent | DA in child = potential DA in parent (EA if enterprise-level privilege). Use SIDHistory injection if SID filtering disabled. |
| **Tree-Root** | Bidirectional | Different namespace trees in same forest | Same as Parent-Child within the forest. |
| **External (NT4)** | Directional | Trust between domains in DIFFERENT forests (non-transitive). Does NOT extend to child domains. | SID filtering is ON by default. Use credential-based movement only (Pass-the-Hash, golden ticket in trusted domain). |
| **Forest** | Directional | Trust between entire FORESTS. Transitive (extends to all domains in both forests). | If SID filtering disabled: SIDHistory injection across entire forest. If enabled: credential-based movement within forest. |
| **Realm** | Directional | Trust between AD and MIT Kerberos realm (e.g., Linux) | Kerberos ticket conversion; trust path for Kerberos attacks. |
| **Shortcut** | Directional | Direct trust between two child domains to shortcut referral path | Same attack paths as Parent-Child; just shorter referral chain. |


