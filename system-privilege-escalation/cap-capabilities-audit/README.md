# Security Audit Report: Cleartext Credential Capture & Linux Capabilities Escalation (Cap - HTB)

## 📌 Executive Summary
This document outlines the security audit and penetration testing process conducted against the **Cap** machine (`10.129.126.65`)[cite: 22]. The attack chain includes port reconnaissance[cite: 22], cleartext credential extraction via network traffic capture analysis (`.pcap`)[cite: 22], initial SSH access[cite: 22], and privilege escalation to `root` via misconfigured Linux capabilities on a Python binary[cite: 23].

## 🛠️ Technical Scope & Methodology
* **Analyzed Services:** FTP (Port 21), SSH (Port 22), and HTTP (Port 80)[cite: 22].
* **Attack Vectors:** Network traffic analysis with `tshark`[cite: 22], credential reuse[cite: 22], and Linux capability abuse on system binaries (`python3.8`)[cite: 23].
* **Tools Used:** Nmap, tshark, FTP client, SSH client[cite: 22, 23].

---

## 🔍 Attack Chain & Findings

### Phase 1: Reconnaissance & Port Scanning
* Performed a full TCP port scan using Nmap to identify active services on the target IP[cite: 22].
* Executed a targeted version scan on primary ports (FTP, SSH, HTTP) to fingerprint underlying software[cite: 22].
* Configured local name resolution by mapping the domain `cap.htb` to the target IP[cite: 22].

### Phase 2: Traffic Analysis & Credential Extraction
* Inspected network packet capture files (`0.pcap`) using `tshark` filtered for FTP traffic[cite: 22].
* The analysis exposed an unencrypted FTP session revealing plaintext credentials for user `nathan` with password `Buck3tH4TF0RM3!`.

### Phase 3: Initial Access (SSH)
* Validated the recovered credentials via FTP connection attempts[cite: 22].
* Successfully established an interactive remote shell session on the target system via SSH under the `nathan` user context[cite: 22].

### Phase 4: Privilege Escalation to Root
* Audited the system for binaries with enabled Linux file capabilities using the `getcap` command[cite: 23].
* Identified that `/usr/bin/python3.8` possessed the `cap_setuid` capability[cite: 23].
* Leveraged this capability to execute a Python script that sets the user ID to `0` (root) and spawns an interactive root shell[cite: 23].
* Navigated to the `/root/` administrative directory and successfully retrieved the root flag (`908cce380d3d54206d2cfb624216323e`)[cite: 23].

---

## 🛡️ Remediation & Defensive Hardening
1. **Enforce Encrypted Communications:** Mandate secure protocols (such as SFTP or FTPS) instead of legacy FTP to prevent cleartext credential interception[cite: 22].
2. **Audit Linux Capabilities:** Regularly review and restrict file capabilities assigned to system binaries and language interpreters to prevent unauthorized privilege escalation vectors[cite: 23].
3. **Principle of Least Privilege:** Ensure that standard users lack advanced UID-switching permissions or capabilities over interpreted binaries[cite: 23].
