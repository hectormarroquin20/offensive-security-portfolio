# Security Audit Report: LLMNR Poisoning & Credential Recovery (Responder - HTB)

## 📌 Executive Summary
This assessment documents the security evaluation of the Responder environment. By exploiting insecure name-resolution protocols (LLMNR/NBT-NS) and capturing Net-NTLMv2 authentication handshakes via Responder, an attacker can extract and crack administrative password hashes offline, leading to total infrastructure compromise through Windows Remote Management (WinRM).

## 🛠️ Technical Scope & Methodology
- **Domain:** `unika.htb`
- **Exposed Services:** HTTP (Port 80), WinRM (Port 5985), Pando-pub (Port 7680)
- **Core Vectors:** LLMNR/NBT-NS Poisoning, NTLMv2 Hash Interception, Offline Dictionary Attacks, WinRM Session Abuse.
- **Tools Utilized:** Nmap, Responder, John the Ripper, Evil-WinRM.

---

## 🔍 Attack Chain & Findings

### Phase 1: Reconnaissance & Port Scanning
Comprehensive TCP port scanning via `nmap` identified three active services on the target system[cite: 1]:
- **Port 80:** HTTP web service hosting the primary application domain (`unika.htb`).
- **Port 5985:** WinRM (Windows Remote Management), standard administrative interface.
- **Port 7680:** Pando-pub service.

### Phase 2: Traffic Interception via Responder
Because Windows environments broadcast name-resolution requests (LLMNR and NBT-NS) when resolving non-existent local network resources, launching **Responder** on the attack interface (`tun0`) allows an operator to spoof responses and trick client applications into authenticating against the attacker's machine:
```bash
sudo responder -I tun0
```

This action successfully intercepted and recorded a Net-NTLMv2 hash corresponding to the domain/local Administrator account.

### Phase 3: Offline Hash Cracking

The captured NTLMv2 challenge-response hash (hash.txt) was processed offline using John the Ripper combined with the comprehensive rockyou.txt wordlist:

```bash
john -w=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt hash.txt
```

The dictionary attack successfully recovered the plaintext password linked to the administrative profile.

### Phase 4: Administrative Access via WinRM

Armed with valid administrative credentials, interactive remote access was established directly through the WinRM service on port 5985:

```bash
evil-winrm -i 10.129.223.101 -P 5985 -u administrator
```

Authenticating successfully granted an interactive shell with full administrative privileges over the target host.
🛡️ Remediation & Defensive Hardening
- Disable LLMNR and NBT-NS: Explicitly disable Link-Local Multicast Name Resolution (LLMNR) and NetBIOS over TCP/IP (NBT-NS) via Group Policy objects (GPO) across the enterprise network to prevent spoofing attacks.
- Enforce SMB Signing: Require SMB packet signing enterprise-wide to mitigate SMB relay attacks.
- Strong Password Policies: Enforce high-entropy, complex passwords to protect against offline dictionary/brute-force attacks even if hashes are intercepted.
- Network Segmentation: Implement strict network segmentation to isolate sensitive broadcast domains and limit internal exposure.
