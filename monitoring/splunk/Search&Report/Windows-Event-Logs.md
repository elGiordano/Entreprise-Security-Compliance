### ** Windows Event Logs in Splunk for Enterprise Security**  

Windows Event Logs are a **critical** source of security telemetry in Splunk, providing visibility into authentication, process execution, lateral movement, and more. Below is a structured breakdown of key **Event IDs**, their **security relevance**, and how to analyze them in Splunk.

---

## **1. Key Windows Event Log Channels**
Windows generates logs in several channels, but the most important for security are:

| **Log Channel**       | **Relevance**                                                                 |
|-----------------------|------------------------------------------------------------------------------|
| **Security**          | Authentication, account changes, privilege escalation (Most critical for SOC) |
| **System**            | Service crashes, driver failures, boot logs                                  |
| **Application**       | Software errors, installation logs                                           |
| **Microsoft-Windows-Sysmon/Operational** | Enhanced process tracking, file/registry changes (If Sysmon is installed) |
| **PowerShell/Operational** | Logs PowerShell commands (critical for detecting malicious scripts)       |
| **TaskScheduler/Operational** | Scheduled tasks (often abused by attackers for persistence)              |

---

## **2. Critical Security Event IDs & MITRE ATT&CK Mapping**
These are the **most important Event IDs** to monitor in Splunk:

### **A. Account Logon & Authentication (T1110, T1078)**
| **Event ID** | **Description**                          | **MITRE ATT&CK Technique**       |
|-------------|-----------------------------------------|----------------------------------|
| **4624**    | Successful logon (Track user logins)    | T1078 (Valid Accounts)           |
| **4625**    | Failed logon (Brute force attacks)      | T1110 (Brute Force)              |
| **4771**    | Kerberos pre-authentication failed      | T1558 (Steal or Forge Kerberos Tickets) |
| **4768-4769** | Kerberos TGT & service ticket requests | T1558 (Golden/Silver Ticket Attacks) |

#### **Splunk Query Example: Detect Brute Force Attacks**
```spl
index=win_events EventCode=4625 
| stats count by src_ip, user 
| where count > 5 
| sort - count
```

---

### **B. Process Execution & Persistence (T1059, T1053, T1547)**
| **Event ID** | **Description**                          | **MITRE ATT&CK Technique**       |
|-------------|-----------------------------------------|----------------------------------|
| **4688**    | New process creation (Cmd, PowerShell)  | T1059 (Command-Line Interface)   |
| **4697**    | Service installation (Malware persistence) | T1543 (Create or Modify System Process) |
| **7045**    | Service binary path modified (Persistence) | T1547 (Boot or Logon Autostart Execution) |

#### **Splunk Query Example: Detect Unusual Process Execution**
```spl
index=win_events EventCode=4688 
| search ParentProcessName="*explorer.exe*" AND (Process="*powershell.exe*" OR Process="*cmd.exe*") 
| table _time, host, user, ParentProcessName, Process, CommandLine
```

---

### **C. Lateral Movement & Remote Execution (T1021, T1076)**
| **Event ID** | **Description**                          | **MITRE ATT&CK Technique**       |
|-------------|-----------------------------------------|----------------------------------|
| **4624** (Logon Type 3) | Network logon (SMB, RDP, PsExec) | T1021 (Remote Services) |
| **5140**    | Network share access (SMB)              | T1021.002 (SMB/Admin Shares)     |
| **4672**    | Admin logon (Privileged account usage)  | T1078 (Valid Accounts)           |

#### **Splunk Query Example: Detect PsExec Lateral Movement**
```spl
index=win_events EventCode=4688 
| search Process="*PsExec*" OR Process="*psexesvc*" 
| table _time, host, user, Process, CommandLine
```

---

### **D. Privilege Escalation (T1548, T1134)**
| **Event ID** | **Description**                          | **MITRE ATT&CK Technique**       |
|-------------|-----------------------------------------|----------------------------------|
| **4672**    | Admin privileges assigned               | T1548 (Abuse Elevation Control Mechanism) |
| **4103**    | PowerShell script block logging (Malicious modules) | T1059.001 (PowerShell) |

#### **Splunk Query Example: Detect UAC Bypass Attempts**
```spl
index=win_events EventCode=4688 
| search CommandLine="*bypassuac*" OR CommandLine="*eventvwr*" 
| table _time, host, user, CommandLine
```

---

## **3. Best Practices for Windows Event Logs in Splunk**
**Enable Advanced Logging** (Sysmon, PowerShell logging, WEF for centralized collection).  
**Use Splunk Add-ons** (`TA-windows`, `Splunk Add-on for Microsoft Sysmon`).  
**Normalize Fields** (Use CIM-compliant fields like `user`, `src_ip`, `process`).  
**Correlate Events** (Combine Event IDs to detect attack chains, e.g., `4624 → 4688 → 5140`).  

---

## **4. Example Splunk Dashboard for Windows Events**
Create a **Windows Security Dashboard** in Splunk with:
- **Failed Logons Over Time** (`EventCode=4625`)  
- **Top Processes Spawned by Explorer** (`EventCode=4688 ParentProcessName=explorer.exe`)  
- **Suspicious PowerShell Commands** (`EventCode=4103`)  

```spl
index=win_events 
| timechart count by EventCode 
| rename EventCode as "Event ID"
```

---

### **Final Thoughts**
Windows Event Logs are **goldmines** for detecting attacks. By focusing on **critical Event IDs** and mapping them to **MITRE ATT&CK**, Splunk becomes a powerful tool for SOC analysts.  

