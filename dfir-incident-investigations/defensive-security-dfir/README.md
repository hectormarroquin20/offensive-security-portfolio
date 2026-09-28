# Incident Investigation Report: Unauthorized Credential Usage & Policy Violation (Enterprise SIEM & Forensics)

## 📌 Executive Summary
This report documents an internal security investigation conducted within an enterprise environment utilizing **Splunk** as a centralized SIEM platform for aggregating firewall, endpoint, and network telemetry. The investigation uncovered a critical internal policy violation where an individual utilized another employee's credentials (specifically, a manager using a subordinate's account on a dedicated workstation). Following the detection, a formal chain of custody was established, physical security was coordinated to secure the asset, and administrative remediation was enforced.

## 🛠️ Technical Scope & Methodology
* **Core Tools:** Splunk Enterprise (SIEM), Windows Security Event Logs, Network Firewall & Router Syslogs.
* **Core Vectors:** Credential sharing/impersonation, non-repudiation analysis, physical asset custody, and internal compliance enforcement.
* **Methodology:** Centralized log correlation, Windows logon session analysis (Event IDs 4624/4625), digital forensics chain of custody, and collaboration with physical security.

---

## 🔍 Investigation Chain & Findings

### Phase 1: Centralized Log Collection & Telemetry Architecture
* Telemetry from diverse infrastructure components—including corporate firewalls, core routers, and Windows endpoints—was centrally ingested into **Splunk** for continuous monitoring and audit compliance.
* Windows Event Forwarding (WEF) and Syslog collectors ensured real-time visibility into authentication events, session creations, and asset bindings.

### Phase 2: Anomaly Detection & Windows Log Analysis
* Following an internal report, security analysts initiated a targeted query in Splunk to investigate user authentication patterns on a specific corporate laptop.
* Analysis of Windows Security logs revealed a discrepancy: the active user operating the physical workstation did not match the authenticated user session context. Specifically, the individual logged into the system was utilizing the credentials belonging to the laptop's primary owner (who held a managerial position over the operator).
* Splunk correlation queries confirmed multiple interactive logons (Logon Type 2/10) where credentials of the subordinate/owner were entered by another user, violating non-repudiation principles and corporate security policies.

### Phase 3: Chain of Custody & Physical Security Coordination
* Given the integrity breach and potential data/liability implications, an immediate incident response workflow was triggered.
* A formal **Chain of Custody** protocol was initiated to document the seizure of the physical laptop.
* **Physical Security** was engaged to accompany the security team during the retrieval of the hardware, ensuring proper documentation, witness verification, and secure transfer of the asset to the forensic lab for preservation.

### Phase 4: Policy Enforcement & Administrative Resolution
* Forensic review confirmed unauthorized credential usage, establishing that the user had bypassed individual accountability controls.
* Even though the user involved held a supervisory/managerial role relative to the credential owner, corporate policy strictly prohibits credential sharing or delegation.
* Formal notification and disciplinary review were conducted in coordination with management and HR, establishing clear boundaries on authentication hygiene and system access accountability.

---

### 💻 Quick Reference: Technical Commands & Splunk SPL Queries

```splunk
# 1. Splunk SPL Query: Correlating Windows Logon Events on a Specific Host
index=windows sourcetype="WinEventLog:Security" EventCode=4624 ComputerName="WORKSTATION_NAME" 
| table _time, ComputerName, TargetUserName, AccountName, IpAddress, LogonType

# 2. Splunk SPL Query: Detecting Multi-User Logons on a Single Endpoint (Credential Sharing Indicators)
index=windows sourcetype="WinEventLog:Security" EventCode=4624 ComputerName="WORKSTATION_NAME" 
| stats distinct_count(TargetUserName) as UniqueUsers by ComputerName, _time 
| where UniqueUsers > 1

# 3. Chain of Custody Documentation Checklist (Linux/Local Shell CLI documentation)
echo "Case ID: INC-2026-0492" > /evidence/case_meta.txt
sha256sum /path/to/extracted_evidence.img >> /evidence/case_meta.txt
```