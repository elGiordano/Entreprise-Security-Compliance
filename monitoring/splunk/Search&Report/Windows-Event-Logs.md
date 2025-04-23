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

----

### **Splunk Saved Searches for Critical Windows Attack Techniques**  
 **ready-to-use Splunk saved searches** mapped to **MITRE ATT&CK techniques** for detecting common Windows-based attacks.  

---

### **1. Credential Dumping (T1003)**
**Technique**: Attackers dump credentials from LSASS memory using tools like Mimikatz.  
**Relevant Event IDs**:  
- **10** (ProcessAccess from Sysmon) – LSASS access  
- **4688** (New process) – Suspicious process calling LSASS  

#### **Splunk Query (Sysmon + Security Logs)**  
```spl
index=win_events (EventCode=10 TargetImage="*lsass.exe*" AND GrantedAccess="0x1FFFFF") 
OR (EventCode=4688 Process="*mimikatz*" OR CommandLine="*sekurlsa::logonpasswords*") 
| stats count by host, user, Process, CommandLine 
| sort - count
```

---

### **2. PowerShell Execution (T1059.001)**
**Technique**: Malicious PowerShell scripts (obfuscation, bypassing logging).  
**Relevant Event IDs**:  
- **4104** (PowerShell Script Block Logging)  
- **4688** (Process creation)  

#### **Splunk Query (Detect Obfuscated PowerShell)**  
```spl
index=win_events (EventCode=4104 OR (EventCode=4688 Process="*powershell*")) 
| search CommandLine="* -nop -exec bypass *" OR CommandLine="* -EncodedCommand *" OR CommandLine="*Invoke-*" 
| table _time, host, user, CommandLine
```

---

### **3. Lateral Movement via PsExec (T1021.002)**
**Technique**: Attackers use PsExec for lateral movement.  
**Relevant Event IDs**:  
- **4688** (PsExec process creation)  
- **5140** (Network share access)  

#### **Splunk Query (Detect PsExec Usage)**  
```spl
index=win_events (EventCode=4688 (Process="*PsExec*" OR Process="*psexesvc*")) 
OR (EventCode=5140 RelativeTargetName="*ADMIN$*") 
| stats count by host, user, Process, RelativeTargetName 
| sort - count
```

---

### **4. Persistence via Scheduled Tasks (T1053.005)**
**Technique**: Malicious scheduled tasks for persistence.  
**Relevant Event IDs**:  
- **106** (Scheduled task created)  
- **4698** (Scheduled task modified)  

#### **Splunk Query (Detect Malicious Tasks)**  
```spl
index=win_events (EventCode=106 OR EventCode=4698) 
| search TaskName="*Update*" OR Author="*SYSTEM*" 
| table _time, host, TaskName, Author, Command
```

---

### **5. RDP Brute Force (T1110.003)**
**Technique**: Attackers brute-force RDP logins.  
**Relevant Event IDs**:  
- **4625** (Failed logon)  
- **4776** (NTLM authentication failure)  

#### **Splunk Query (Detect RDP Attacks)**  
```spl
index=win_events (EventCode=4625 LogonType=10) OR EventCode=4776 
| stats count by src_ip, user 
| where count > 5 
| sort - count
```

---

### **6. Registry Persistence (T1547.001)**
**Technique**: Modifying `Run` keys for persistence.  
**Relevant Event IDs**:  
- **4657** (Registry value modified) – Sysmon  
- **13** (RegistryEvent) – Sysmon  

#### **Splunk Query (Detect Run Key Modifications)**  
```spl
index=win_events (EventCode=13 OR EventCode=4657) 
| search TargetObject="*\\Run\\*" 
| table _time, host, user, TargetObject, Details
```

---

### **7. Windows Defender Tampering (T1562.001)**
**Technique**: Disabling AV for evasion.  
**Relevant Event IDs**:  
- **5001** (Windows Defender disabled)  
- **4688** (Process stopping AV services)  

#### **Splunk Query (Detect AV Tampering)**  
```spl
index=win_events (EventCode=5001) OR (EventCode=4688 (Process="*MsMpEng.exe*" OR CommandLine="*sc stop WinDefend*")) 
| table _time, host, user, Process, CommandLine
```

---

### **How to Deploy These in Splunk**  
1. **Save as Alerts**:  
   - Go to **Splunk Search** → **Save As** → **Alert**.  
   - Set a **cron schedule** (e.g., every 15 mins).  
   - Trigger actions (email, ITSM integration).  

2. **Add to Enterprise Security (ES)**:  
   - Use **Correlation Searches** in Splunk ES.  
   - Map to MITRE ATT&CK via `mitre_attack_lookup`.  

3. **Visualize in Dashboards**:  
   - Create a **Windows Threat Hunting Dashboard** with these queries.  

---

### **Final Tips**  
🔹 **Tune for False Positives** (Adjust thresholds like `count > 5`).  
🔹 **Enrich with Threat Intel** (Use `lookup` to match IOCs).  
🔹 **Combine with Sysmon** (For deeper visibility).  
