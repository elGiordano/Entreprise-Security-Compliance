# **Sysmon Logs in Splunk: Deep Dive for Threat Detection**

Sysmon (System Monitor) is a powerful Windows system service that provides detailed logging of process creations, network connections, file changes, and more. When ingested into Splunk, Sysmon logs become a **goldmine for threat hunting, detection engineering, and incident response**.

---

## **1. Key Sysmon Event IDs & Their Security Relevance**
Sysmon logs events with **Event IDs 1-255**. Below are the most critical ones for security monitoring:

| **Event ID** | **Description**                          | **MITRE ATT&CK Mapping**               | **Splunk Use Case**                     |
|-------------|-----------------------------------------|----------------------------------------|-----------------------------------------|
| **1**       | Process creation                        | T1059 (Command-Line Interface)         | Detect malicious processes (e.g., `powershell -enc`) |
| **3**       | Network connection                      | T1043 (Commonly Used Port)             | Detect C2 beacons (e.g., odd IP:port combos) |
| **7**       | Image loaded (DLL injection)            | T1055 (Process Injection)              | Find malware loading malicious DLLs     |
| **8**       | CreateRemoteThread (Code injection)     | T1055 (Process Injection)              | Detect process hollowing                |
| **10**      | ProcessAccess (LSASS dumping)           | T1003 (Credential Dumping)             | Detect Mimikatz-like activity          |
| **11**      | File creation                           | T1112 (Modify Registry)                | Detect webshells (`aspx` in `TEMP`)     |
| **12-13**   | Registry key/value modifications        | T1112 (Modify Registry)                | Detect persistence via `Run` keys       |
| **17-18**   | PipeEvent (Named pipe communication)    | T1021.002 (SMB/Admin Shares)           | Detect lateral movement via pipes       |
| **22**      | DNS query                               | T1071.004 (DNS C2)                     | Detect malware resolving C2 domains     |

---

## **2. Critical Splunk Queries for Sysmon Threat Hunting**
### **A. Detect Suspicious Process Execution (Event ID 1)**
```spl
index=sysmon EventID=1 
| search (ParentImage="*\\cmd.exe" OR ParentImage="*\\powershell.exe") 
  AND (Image="*\\whoami.exe" OR CommandLine="*net user*") 
| table _time, host, User, ParentImage, Image, CommandLine
```
 **Use Case**: Detects **living-off-the-land (LOLBin)** abuse (e.g., `whoami` or `net user` executed from PowerShell).

---

### **B. Detect LSASS Memory Dumping (Event ID 10)**
```spl
index=sysmon EventID=10 TargetImage="*\\lsass.exe" 
| stats count by host, SourceImage, GrantedAccess 
| where count > 3 AND like(GrantedAccess,"%0x1FFFFF%") 
| sort - count
```
 **Use Case**: Finds **Mimikatz-like credential dumping** (common in ransomware attacks).

---

### **C. Detect DNS-Based C2 (Event ID 22)**
```spl
index=sysmon EventID=22 
| lookup threat_intel_domains domain as QueryName OUTPUT IsMalicious 
| where IsMalicious="true" 
| table _time, host, QueryName, IsMalicious
```
 **Use Case**: Detects **malware beaconing** via known-bad domains (requires threat intel lookup).

---

### **D. Detect Process Injection (Event ID 8)**
```spl
index=sysmon EventID=8 
| search (TargetImage="*\\explorer.exe" OR TargetImage="*\\svchost.exe") 
| stats count by host, SourceImage, TargetImage 
| sort - count
```
 **Use Case**: Finds **malware injecting into trusted processes** (e.g., `explorer.exe`).

---

### **E. Detect Persistence via Registry (Event ID 12-13)**
```spl
index=sysmon EventID=13 
| search TargetObject="*\\Run\\*" OR TargetObject="*\\RunOnce\\*" 
| table _time, host, User, TargetObject, Details
```
 **Use Case**: Detects **malware persistence via `Run` keys**.

---

## **3. Best Practices for Sysmon in Splunk**
### ** Deploy Sysmon with a Strong Config**
- Use **SwiftOnSecurity’s Sysmon config** or **MITRE’s Sysmon Modular** for optimal logging.
- Log **critical events** (e.g., `EventID 1,3,8,10,22`) without noise.

### ** Normalize Sysmon Logs in Splunk**
- Use the **Splunk Add-on for Sysmon** (`TA-microsoft-sysmon`) for proper field extraction.
- Map fields to **CIM (Common Information Model)** for consistency.

### ** Correlate Sysmon with Other Logs**
- Combine with **Windows Security logs** (`EventID 4688`, `4624`) for better context.
- Enrich with **EDR data** (e.g., CrowdStrike, SentinelOne).

### ** Build Splunk Dashboards for Sysmon**
- **Threat Hunting Dashboard**: Track suspicious process trees, network connections.
- **Incident Response Dashboard**: Focus on credential dumping, lateral movement.

---

## **4. Example: Sysmon Dashboard in Splunk**
### **Panels to Include**
1. **Top Malicious Processes** (`EventID=1 | stats count by Image`)
2. **Unusual Network Connections** (`EventID=3 | where dest_port > 49152`)
3. **LSASS Access Attempts** (`EventID=10 | timechart count`)
4. **DNS Anomalies** (`EventID=22 | top QueryName limit=10`)

### **Splunk XML Snippet (For Dashboard)**
```xml
<panel>
  <title>Top Suspicious Processes</title>
  <search>
    <query>index=sysmon EventID=1 | stats count by Image | sort - count | head 10</query>
  </search>
  <option name="count">10</option>
</panel>
```

---

## **5. Advanced: Splunk + Sysmon + MITRE ATT&CK**
Use **Splunk’s `mitre_attack_lookup`** to tag Sysmon events with ATT&CK techniques:
```spl
index=sysmon EventID=1 
| `mitre_attack_lookup("T1059")` 
| table _time, host, Image, CommandLine, mitre_attack_technique
```

---

## **Final Thoughts**
Sysmon + Splunk = **Enterprise-grade detection**.  
**Deploy a strong Sysmon config** (minimize noise).  
**Use these Splunk queries** for threat hunting.  
**Correlate with other logs** (EDR, firewall).  
