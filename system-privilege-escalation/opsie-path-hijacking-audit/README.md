# Security Assessment & Privilege Escalation Audit: Local SUID & PATH Hijacking

## 📌 Executive Summary
This technical report documents an end-to-end security assessment involving web application reconnaissance, configuration file credential harvesting, local group enumeration, and local privilege escalation. The evaluation highlights critical risks associated with insecure relative path execution inside SUID-enabled binaries, demonstrating how environment variable manipulation (`PATH` hijacking) leads to full root system compromise.

---

## 🛠️ Methodology & Tooling
* **Reconnaissance & Port Scanning:** High-speed network enumeration and service profiling using `nmap`.
* **Web Directory Fuzzing:** Content discovery via `gobuster` using structured wordlists to uncover hidden web application assets.
* **Credential Harvesting:** Reviewing source code repositories, configuration files, and internal web directories for leaked authentication secrets.
* **Privilege & Access Enumeration:** Analyzing user context, group memberships (`id`), and system-wide file permissions (`find`) to locate misconfigured SUID binaries.
* **Environment Exploitation (PATH Hijacking):** Intercepting un-pathed command calls within privileged executables by manipulating the shell's `PATH` variable.

---

## 🔍 Technical Attack Path & Findings

### Phase 1: Reconnaissance & Web Enumeration
Initial network mapping identifies active communication ports and web services. When applications rely on virtual hosting, mapping the target IP address to a designated domain name within local configuration files (`/etc/hosts`) ensures correct routing for subsequent web content discovery and directory fuzzing phases.

### Phase 2: Credential Harvesting & Initial Access
Web servers frequently store configuration files, source code backups, or database connection strings within accessible internal directory structures. Systematic inspection of web application files allows security auditors to harvest valid credentials (such as service accounts or user passwords), which can be leveraged for remote interactive access (e.g., SSH).

### Phase 3: Local Privilege Enumeration
Upon securing an initial low-privileged session on the target system, enumeration focuses on identifying local vectors for escalation:
* **Group Memberships:** Checking auxiliary system groups to determine if the user has special read/execute privileges over proprietary software or administrative utilities.
* **SUID Binary Discovery:** Scanning the filesystem for binaries with the SetUID (SUID) permission bit set (`find / -group <group> 2>/dev/null`), allowing execution with the owner's privileges (typically root).

### Phase 4: Exploitation via PATH Hijacking
If a compiled SUID binary executes system commands or external utilities (such as text viewers or helper scripts) using **relative paths** instead of absolute paths (e.g., calling `cat` rather than `/bin/cat`):
1. **Command Interception:** An attacker with write access to a temporary directory (like `/tmp`) can create a malicious script or binary bearing the exact same name as the expected utility.
2. **Environment Manipulation:** Prepending the writable directory to the system's `PATH` environment variable forces the shell to look in that directory first when the SUID binary invokes the command.
3. **Privilege Inheritance:** Executing the vulnerable SUID binary triggers the internal command call, which resolves to the malicious script and executes it with root privileges, successfully spawning an administrative shell session.

---

### 💻 Quick Reference: Technical Commands & Workflow

```bash
# 1. Host Discovery & Comprehensive Port Scanning
ping -c 3 <TARGET_IP>
nmap -p- -n -Pn -sS --min-rate 5000 -vvv <TARGET_IP> -oG allPorts
nmap -p22,80 -sCV <TARGET_IP> -oN targeted

# 2. Virtual Host Mapping & Web Fuzzing
echo "<TARGET_IP> <DOMAIN>" | sudo tee -a /etc/hosts
gobuster dir -u http://<TARGET_IP>/ -w /path/to/wordlist.txt -t 150

# 3. Catching Reverse Shell & Upgrading TTY
nc -lvnp <ATTACKER_PORT>
script /dev/null -c bash
# (Press Ctrl+Z, then run:)
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash

# 4. Local Enumeration & SUID Target Discovery
id
find / -group <target_group> 2>/dev/null
ls -la /path/to/suid_binary

# 5. PATH Hijacking Exploitation
cd /tmp
echo '#!/bin/sh' > <command_name>
echo '/bin/sh' >> <command_name>
chmod +x <command_name>
export PATH=/tmp:$PATH
/path/to/suid_binary <argument>