# **Firewall Logs in Splunk: Deep Dive for Palo Alto, Cisco ASA & Fortinet**

Firewall logs are the **first line of defense** for network security monitoring in Splunk. When properly analyzed, they reveal brute force attacks, C2 communications, data exfiltration, and lateral movement. Here's how to leverage them effectively.

---

## **1. Key Firewall Log Sources & Normalization**
### **A. Log Sources & Splunk Integration**
| **Vendor**    | **Log Type**           | **Splunk Add-on**                     | **Key Fields**                          |
|--------------|-----------------------|--------------------------------------|----------------------------------------|
| **Palo Alto** | Traffic, Threat, URL  | [TA-paloalto](https://splunkbase.splunk.com/app/2757/) | `action`, `src_ip`, `dest_ip`, `app`, `rule` |
| **Cisco ASA** | Connection, ACL       | [TA-cisco-asa](https://splunkbase.splunk.com/app/1627/) | `src_ip`, `dest_ip`, `src_port`, `dest_port`, `action` |
| **Fortinet**  | Traffic, UTM, VPN     | [TA-fortinet](https://splunkbase.splunk.com/app/1629/) | `srcip`, `dstip`, `service`, `action`, `sentbyte` |

### **B. Normalization (CIM Compliance)**
For consistent analysis, map firewall logs to **Splunk’s Common Information Model (CIM)**:
- `src_ip` → Source host  
- `dest_ip` → Destination host  
- `dest_port` → Destination port  
- `action` (`allow`/`deny`)  
- `bytes_in`/`bytes_out` → Data transfer volume  

Example:
```spl
index=pan_logs sourcetype=pan:traffic 
| eval src_ip=src, dest_ip=dst, dest_port=dport 
| stats sum(bytes) as total_bytes by src_ip, dest_ip, dest_port, action
```

---

## **2. Critical Firewall Detection Use Cases**
### **A. Brute Force Attacks (T1110)**
#### **Palo Alto (Repeated Denied Connections)**
```spl
index=pan_logs sourcetype=pan:traffic action=deny 
| stats count by src_ip, dest_ip, dest_port 
| where count > 10 
| sort - count
```
 **Detects**: SSH/RDP brute force (e.g., `dest_port=3389`).  

#### **Cisco ASA (Failed VPN Logins)**
```spl
index=cisco_asa "AAA authentication rejected" 
| stats count by src_ip, user 
| where count > 5 
| sort - count
```
 **Detects**: VPN credential stuffing.  

---

### **B. Command & Control (C2) Traffic (T1071)**
#### **Fortinet (Beaconing to Rare Domains)**
```spl
index=fortinet_logs sourcetype=fortigate_utm dstip IN ("1.1.1.1", "8.8.8.8") 
| stats count by srcip, dstip, service 
| where count > 50 
| sort - count
```
 **Detects**: DNS/HTTP beaconing (use threat intel lookups for better accuracy).  

#### **Palo Alto (Unusual Outbound Ports)**
```spl
index=pan_logs sourcetype=pan:traffic action=allow dest_port < 1024 NOT dest_port IN (80, 443, 53) 
| stats count by src_ip, dest_ip, dest_port 
| sort - count
```
 **Detects**: Reverse shells (e.g., `dest_port=4444`).  

---

### **C. Data Exfiltration (T1048)**
#### **Cisco ASA (Large Data Transfers)**
```spl
index=cisco_asa action=permit 
| stats sum(bytes_out) as total_outbound by src_ip, dest_ip 
| where total_outbound > 100000000 
| sort - total_outbound
```
 **Detects**: Data theft (e.g., 100MB+ transfers to external IPs).  

#### **Palo Alto (RDP with High Byte Count)**
```spl
index=pan_logs sourcetype=pan:traffic app=ms-rdp bytes > 10000000 
| table _time, src_ip, dest_ip, user, bytes
```
 **Detects**: RDP abuse for data exfiltration.  

---

### **D. Lateral Movement (T1021, T1210)**
#### **Fortinet (Internal SMB Scanning)**
```spl
index=fortinet_logs sourcetype=fortigate_traffic service=SMB action=accept 
| stats count by srcip, dstip 
| where count > 20 
| sort - count
```
 **Detects**: Internal worm-like spreading (e.g., EternalBlue).  

#### **Cisco ASA (Unexpected Internal Connections)**
```spl
index=cisco_asa src_ip=10.0.0.0/8 dest_ip=10.0.0.0/8 action=permit 
| stats count by src_ip, dest_ip, dest_port 
| where count > 50 
| sort - count
```
 **Detects**: Internal reconnaissance (e.g., `dest_port=445`).  

---

## **3. Advanced Correlation Techniques**
### **A. Combine Firewall + Endpoint Logs**
```spl
index=pan_logs sourcetype=pan:traffic action=allow dest_port=443 
| join dest_ip [search index=win_events EventCode=3 dest_ip=*] 
| table _time, src_ip, dest_ip, dest_port, user, process
```
 **Finds**: Allowed HTTPS traffic tied to malicious processes.  

### **B. Threat Intel Enrichment**
```spl
index=pan_logs sourcetype=pan:traffic 
| lookup threat_intel_ip src_ip as src_ip OUTPUT is_malicious 
| where is_malicious="true" 
| table _time, src_ip, dest_ip, action
```
 **Flags**: Traffic to known-bad IPs (e.g., C2 servers).  

---

## **4. Splunk Dashboard Examples**
### **Firewall Security Overview Dashboard**
**Panels to Include:**
1. **Top Denied IPs** (`action=deny | stats count by src_ip`)
2. **Data Transfer Anomalies** (`bytes > 1GB | stats sum(bytes) by src_ip`)
3. **Unusual Port Activity** (`dest_port NOT IN (80,443,22) | timechart count`)
4. **Threat Intel Matches** (`lookup threat_intel_ip src_ip`)

### **Splunk XML Snippet**
```xml
<panel>
  <title>Top Denied Source IPs</title>
  <search>
    <query>index=pan_logs sourcetype=pan:traffic action=deny | stats count by src_ip | sort - count | head 10</query>
  </search>
  <option name="count">10</option>
</panel>
```

---

## **5. Best Practices**
 **Enable Logging for Critical Rules**  
   - Log both `allow` and `deny` actions for investigation.  
 **Normalize Fields** (Use CIM-compliant `src_ip`, `dest_ip`)  
 **Tune for False Positives** (Adjust thresholds like `count > 10`)  
 **Correlate with Other Logs** (e.g., EDR + Firewall = Better detection)  

---

## **Final Thoughts**
Firewall logs in Splunk are **essential for detecting**:  
 **External attacks** (brute force, C2)  
 **Insider threats** (lateral movement, data theft)  
 **Policy violations** (unauthorized apps/ports)  

