# **NetFlow & PCAP Analysis in Splunk: Network Traffic Intelligence**

NetFlow and PCAP data provide critical network visibility that complements firewall and IDS logs. When analyzed in Splunk, they enable detection of lateral movement, data exfiltration, and suspicious traffic patterns that evade traditional security tools.

---

## **1. Key Data Sources & Collection Methods**

### **A. NetFlow (Metadata)**
| **Type**       | **Description**                          | **Splunk Add-on**                     |
|---------------|-----------------------------------------|--------------------------------------|
| NetFlow v5/v9 | Cisco standard (source/dest IP, ports, bytes) | [TA-netflow](https://splunkbase.splunk.com/app/1847/) |
| IPFIX         | Extended NetFlow (custom fields)        | [TA-ipfix](https://splunkbase.splunk.com/app/4345/) |
| sFlow         | Sampling-based flow data               | [TA-sflow](https://splunkbase.splunk.com/app/4346/) |

### **B. PCAP (Full Packet Capture)**
| **Method**              | **Splunk Integration**                 |
|-------------------------|--------------------------------------|
| Zeek/Bro logs           | [TA-zeek](https://splunkbase.splunk.com/app/1917/) |
| Arkime (Moloch) metadata | Splunk HEC integration              |
| Custom PCAP processing  | Splunk Stream (deprecated)           |

---

## **2. Critical Detection Use Cases**

### **A. Data Exfiltration (T1048)**
#### **Unusual Outbound Data Transfers**
```spl
index=netflow dest_ip!=10.0.0.0/8 
| stats sum(bytes) as total_bytes by src_ip, dest_ip 
| where total_bytes > 100000000 
| sort - total_bytes
```
 **Detects**: 100MB+ transfers to external IPs (potential data theft)

#### **Scheduled Data Transfers (Beaconing)**
```spl
index=netflow 
| bin _time span=1h 
| stats sum(bytes) as hourly_bytes by src_ip, dest_ip, _time 
| eventstats avg(hourly_bytes) as avg, stdev(hourly_bytes) as stdev by src_ip, dest_ip 
| where hourly_bytes > (avg+(stdev*3))
```

---

### **B. Lateral Movement (T1021)**
#### **Internal SMB Scanning**
```spl
index=netflow src_ip=10.0.0.0/8 dest_ip=10.0.0.0/8 dest_port=445 
| stats dc(dest_ip) as scanned_hosts by src_ip 
| where scanned_hosts > 20 
| sort - scanned_hosts
```

#### **RDP Protocol Anomalies**
```spl
index=netflow dest_port=3389 
| stats sum(bytes) as total_rdp by src_ip, dest_ip 
| where total_rdp > 50000000 
| sort - total_rdp
```

---

### **C. Command & Control (T1071)**
#### **DNS Tunneling Detection**
```spl
index=zeek dns 
| stats dc(query) as unique_queries by src_ip 
| where unique_queries > 100 
| sort - unique_queries
```

#### **Low-and-Slow C2 Channels**
```spl
index=netflow 
| stats count, sum(bytes) as total_bytes by src_ip, dest_ip 
| eval bytes_per_flow=total_bytes/count 
| where bytes_per_flow < 500 AND count > 50
```

---

## **3. Advanced Analysis Techniques**

### **A. Protocol Anomaly Detection**
```spl
index=netflow 
| stats dc(dest_port) as unique_ports by src_ip 
| eventstats avg(unique_ports) as avg, stdev(unique_ports) as stdev 
| where unique_ports > (avg+(stdev*3))
```

### **B. Time-Based Beaconing Detection**
```spl
index=netflow 
| timechart span=1h count by src_ip, dest_ip 
| anomalydetection count
```

### **C. Zeek File Extraction Monitoring**
```spl
index=zeek files 
| search "mime_type=application/x-dosexec" 
| stats count by src_ip, dest_ip, filename 
| sort - count
```

---

## **4. Splunk Visualization & Dashboards**

### **Network Traffic Dashboard**
**Essential Panels:**
1. **Top Talkers** (`stats sum(bytes) by src_ip | sort - sum(bytes)`)
2. **Protocol Distribution** (`stats count by proto | sort - count`)
3. **Internal-External Traffic** (`cidrmatch("10.0.0.0/8",src_ip) | stats sum(bytes)`)
4. **Port Scan Detection** (`stats dc(dest_port) by src_ip | where dc>50`)

### **PCAP Analysis Dashboard**
1. **DNS Query Analysis** (`index=zeek dns | top query`)
2. **HTTP Suspicious Requests** (`index=zeek http | search "User-Agent=*curl*"`)
3. **SSL/TLS Anomalies** (`index=zeek ssl | search "validation_status=*"`)

---

## **5. Implementation Best Practices**

 **Sampling Strategy**  
   - NetFlow: 1:1 sampling for critical networks  
   - sFlow: 1:1000 sufficient for large networks  

 **Field Extraction**  
   - Normalize `src_ip`, `dest_ip`, `bytes`, `duration`  
   - Enrich with asset DB (owner, location)  

 **Storage Optimization**  
   - NetFlow: ~1GB/day per 10Gbps network  
   - PCAP: Store only metadata (Zeek logs) or critical sessions  

 **Correlation**  
   - Combine with IDS/IPS alerts (`join src_ip,dest_ip`)  
   - Enrich with threat intel (`lookup threat_intel_ip`)  

---

## **Final Thoughts**

NetFlow/PCAP in Splunk provides:
 **East-West traffic visibility** (internal lateral movement)  
 **Data loss prevention** (exfiltration detection)  
 **Threat hunting** (protocol anomalies, C2 patterns)  

