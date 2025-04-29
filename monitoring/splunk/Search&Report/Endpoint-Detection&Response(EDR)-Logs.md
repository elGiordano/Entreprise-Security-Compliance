# **Endpoint Detection & Response (EDR) Logs in Splunk: Comprehensive Security Monitoring Guide**

## **1. Introduction to EDR Logs in Splunk**
Endpoint Detection and Response (EDR) solutions generate critical security telemetry from workstations, servers, and mobile devices. Integrating these logs with Splunk enables:
- **Real-time threat detection** (malware, ransomware, lateral movement)
- **Incident investigation** (process trees, file modifications)
- **Threat hunting** (IOC searches, behavioral analytics)
- **Compliance reporting** (audit trails, configuration checks)

### **Key EDR Data Sources**
| **Data Type**       | **Description**                          | **Example Fields**                     |
|---------------------|------------------------------------------|----------------------------------------|
| **Process Events**  | Process creation/termination             | `parent_process, command_line, hash`   |
| **File Events**     | File creation/modification/deletion      | `file_path, action, user`             |
| **Network Events**  | Connections and DNS lookups              | `src_ip, dest_ip, port, domain`       |
| **Registry Events** | Windows registry modifications          | `key_path, value_name, action`        |
| **Alert Events**    | EDR-generated detections                 | `alert_name, severity, iocs`          |

---

## **2. Critical Detection Use Cases**

### **1. Malware Execution & Persistence**
#### **Suspicious Process Creation**
```splunk
index=edr_logs event_type=process_create 
| search 
    [| inputlookup suspicious_processes | table process_name] 
    OR 
    [| inputlookup malicious_hashes | table hash] 
| stats count by host, process_name, hash
```

#### **Persistence Mechanisms**
```splunk
index=edr_logs event_type=registry_mod 
| search 
    key_path="*\\Run\\*" 
    OR key_path="*\\Services\\*" 
| table _time, host, user, key_path
```

### **2. Lateral Movement & Ransomware**
#### **PsExec/WMI Abuse**
```splunk
index=edr_logs event_type=process_create 
| search 
    (process_name="psexec*" OR process_name="wmic*") 
    AND parent_process!="C:\\Windows\\System32\\*" 
| stats count by host, user, parent_process
```

#### **Mass File Encryption (Ransomware)**
```splunk
index=edr_logs event_type=file_mod 
| stats 
    dc(file_path) as unique_files, 
    count by host, process_name 
| where unique_files > 100 
| sort - unique_files
```

### **3. Credential Access & Theft**
#### **LSASS Memory Dumping**
```splunk
index=edr_logs event_type=process_create 
| search 
    process_name IN ("procdump*", "mimikatz*") 
    AND parent_process="lsass.exe" 
| table _time, host, user, command_line
```

#### **SAM Database Access**
```splunk
index=edr_logs event_type=file_access 
| search file_path="C:\\Windows\\System32\\config\\SAM" 
| stats count by host, process_name
```

### **4. Defense Evasion**
#### **Process Hollowing**
```splunk
index=edr_logs event_type=process_create 
| search 
    process_name IN ("svchost.exe", "explorer.exe") 
    AND image_path!="C:\\Windows\\System32\\*" 
| table _time, host, image_path, command_line
```

#### **Timestomping**
```splunk
index=edr_logs event_type=file_mod 
| search 
    action="timestamp_modified" 
    AND file_path="*.exe" 
| stats count by host, process_name
```

---

## **3. Advanced Correlation Techniques**

### **1. Process Chain Analysis**
```splunk
index=edr_logs event_type=process_create 
| transaction host process_id maxspan=5s 
| search 
    process_name="cmd.exe" 
    AND parent_process="explorer.exe" 
    AND child_process="powershell.exe" 
| table _time, host, process_chain
```

### **2. Beaconing Detection (Jitter Analysis)**
```splunk
index=edr_logs event_type=network_conn 
| bin _time span=1m 
| stats 
    count as conn_count, 
    stdev(_time) as time_stdev 
    by host, dest_ip 
| where 
    conn_count > 50 
    AND time_stdev < 10 
| sort - conn_count
```

### **3. Threat Hunting with MITRE ATT&CK**
```splunk
index=edr_logs 
| lookup mitre_techniques technique_id OUTPUT technique_name 
| search technique_name="*Credential Dumping*" 
| stats count by host, technique_name
```

---

## **4. Best Practices for EDR Log Analysis**

 **Normalize fields** (use CIM-compliant field names)  
 **Enrich with threat intel** (malicious hashes, IPs, domains)  
 **Correlate with other logs** (firewall, proxy, authentication)  
 **Store in dedicated indexes** (`edr_logs`, `windows_events`)  
 **Implement baselining** (anomaly detection for process behavior)  

---

## **5. Sample Splunk Dashboard Panels**

| **Panel**                     | **SPL Query**                          |
|-------------------------------|---------------------------------------|
| **Top Malicious Processes**   | `index=edr_logs event_type=alert | top process_name` |
| **Lateral Movement Attempts** | `index=edr_logs event_type=network_conn dest_ip=10.0.0.0/8 | timechart count` |
| **Fileless Attack Detection** | `index=edr_logs event_type=process_create image_path=memory | stats count by host` |
| **EDR Alert Trends**          | `index=edr_logs event_type=alert | timechart count by alert_name` |

---

## **6. Next Steps**
- **Deploy Splunk Universal Forwarder** to collect EDR logs  
- **Integrate with Splunk Enterprise Security (ES)** for advanced detections  
- **Automate response** (quarantine hosts via Splunk Phantom)  
