# **IDS/IPS Logs in Splunk: Enterprise Security Deep Dive**

Intrusion Detection and Prevention System (IDS/IPS) logs are critical for identifying malicious network activity. When properly analyzed in Splunk, they reveal attack patterns, exploit attempts, and network intrusions. Here's how to maximize their value for security monitoring.

---

## **1. Key IDS/IPS Log Sources & Normalization**

### **A. Common Enterprise IDS/IPS Solutions**
| **Solution**      | **Log Type**          | **Splunk Add-on**                     | **Key Fields**                          |
|------------------|-----------------------|--------------------------------------|----------------------------------------|
| **Suricata**     | EVE JSON logs         | [TA-suricata](https://splunkbase.splunk.com/app/4917/) | `src_ip`, `dest_ip`, `alert.signature`, `proto` |
| **Snort**        | Unified2/alert logs   | [TA-snort](https://splunkbase.splunk.com/app/2676/) | `src_ip`, `dst_ip`, `sig_msg`, `sig_id` |
| **Cisco Firepower** | NGFW logs         | [TA-firepower](https://splunkbase.splunk.com/app/1910/) | `sourceAddress`, `destinationAddress`, `impactFlag`, `signature` |
| **Palo Alto IPS** | Threat logs       | Built-in PAN add-on                 | `src_ip`, `dest_ip`, `threat_id`, `subtype` |

### **B. Field Normalization (CIM Compliance)**
Map IDS/IPS logs to **Splunk's Common Information Model**:
- `src_ip` → Attacker IP  
- `dest_ip` → Target IP  
- `signature` → Rule/alert name  
- `severity` → Alert priority  
- `action` → Blocked/Detected  

Example transformation:
```spl
index=ids_logs sourcetype=suricata:json 
| eval src_ip=src_ip, dest_ip=dest_ip, signature=alert.signature 
| stats count by src_ip, dest_ip, signature, alert.severity
```

---

## **2. Critical IDS/IPS Detection Use Cases**

### **A. Exploit Attempts (T1190)**
#### **Detect Common Vulnerability Exploits**
```spl
index=ids_logs signature="*Apache Struts*" OR signature="*Log4j*" OR signature="*ProxyShell*" 
| stats count by src_ip, dest_ip, signature 
| sort - count
```
 **Triggers on**: Known exploit patterns (CVE-based signatures).

#### **Find Zero-Day Exploit Patterns**
```spl
index=ids_logs severity=1 action=blocked 
| lookup cve_lookup signature as signature OUTPUT cve_id 
| where isnotnull(cve_id) 
| table _time, src_ip, dest_ip, signature, cve_id
```

---

### **B. Network Scanning (T1046)**
#### **Port Scanning Detection**
```spl
index=ids_logs signature="*port scan*" OR signature="*NMAP*" 
| stats count by src_ip, signature 
| where count > 5 
| sort - count
```
 **Detects**: Reconnaissance activity.

#### **Horizontal Scanning (Internal Network)**
```spl
index=ids_logs src_ip IN (10.0.0.0/8) dest_ip IN (10.0.0.0/8) signature="*scan*" 
| stats dc(dest_ip) as scanned_hosts by src_ip 
| where scanned_hosts > 10 
| sort - scanned_hosts
```

---

### **C. Command & Control (C2) Traffic (T1071)**
#### **Known Malware C2 Channels**
```spl
index=ids_logs [inputlookup c2_servers.csv | table ip] 
| stats count by src_ip, dest_ip, signature 
| sort - count
```
 **Detects**: Traffic to known C2 IPs (threat intel enrichment).

#### **DNS Tunneling Attempts**
```spl
index=ids_logs signature="*DNS tunneling*" OR signature="*DNS exfiltration*" 
| stats count by src_ip, dest_ip, query 
| sort - count
```

---

### **D. Lateral Movement (T1021)**
#### **Detect SMB Exploits (EternalBlue)**
```spl
index=ids_logs signature="*EternalBlue*" OR signature="*SMB exploit*" 
| stats count by src_ip, dest_ip 
| sort - count
```
 **Flags**: Internal worm propagation attempts.

#### **RDP Bruteforce Attempts**
```spl
index=ids_logs signature="*RDP brute force*" 
| stats count by src_ip, dest_ip 
| where count > 3 
| sort - count
```

---

## **3. Advanced Correlation Techniques**

### **A. Combine IDS + Firewall Logs (Evasion Detection)**
```spl
index=ids_logs action=detected 
| join src_ip,dest_ip [search index=firewall_logs action=allow 
| stats count by src_ip, dest_ip] 
| table _time, src_ip, dest_ip, signature
```
 **Finds**: IDS alerts that weren't blocked by firewall.

### **B. Threat Intel Enrichment**
```spl
index=ids_logs 
| lookup threat_intel_ip src_ip as src_ip OUTPUT is_malicious 
| where is_malicious="true" 
| table _time, src_ip, dest_ip, signature
```

---

## **4. Splunk Dashboard Examples**

### **IDS/IPS Security Monitoring Dashboard**
**Essential Panels:**
1. **Top Attack Signatures** (`stats count by signature | sort - count`)
2. **Exploit Attempts Timeline** (`timechart count by signature`)
3. **Internal Scanning Activity** (`src_ip IN (RFC1918) | stats dc(dest_ip)`)
4. **C2 Communication Alerts** (`lookup c2_servers | stats count`)

### **Splunk XML Snippet**
```xml
<panel>
  <title>Recent Exploit Attempts</title>
  <search>
    <query>index=ids_logs severity IN ("high","critical") 
    | stats count by src_ip, dest_ip, signature 
    | sort - count | head 10</query>
  </search>
  <option name="count">10</option>
</panel>
```

---

## **5. Best Practices**
 **Enable Full Packet Capture** (For critical alerts)  
 **Tune False Positives** (Adjust thresholds/suppressions)  
 **Correlate with EDR** (Endpoint detection validation)  
 **Prioritize by Business Impact** (Focus on critical assets)  

---

## **Final Thoughts**
IDS/IPS logs in Splunk enable detection of:
 **External attacks** (exploits, scans)  
 **Internal threats** (lateral movement)  
 **C2 communications** (malware activity)  
