# **Web Server Logs in Splunk: Monitoring, Analysis & Security Detection**

Web server logs provide critical insights into **traffic patterns, security threats, and performance issues**. Integrating them into **Splunk** enables real-time monitoring, anomaly detection, and incident response. Below is a **comprehensive guide** to collecting, analyzing, and securing web server logs in Splunk.

---

## **1. Key Web Server Log Sources**
| **Log Type**       | **Description**                                                                 | **Security Use Cases**                          |
|--------------------|-------------------------------------------------------------------------------|-----------------------------------------------|
| **Access Logs**    | Records HTTP requests (GET, POST, etc.) with IP, URI, status code, user-agent. | Detect **brute force, SQLi, XSS, scanning**.  |
| **Error Logs**     | Captures server errors (404, 500, etc.) and misconfigurations.                 | Identify **exploit attempts, misconfigurations**. |
| **SSL/TLS Logs**   | Logs encryption handshake details (cipher suites, certificates).               | Detect **weak ciphers, MITM attacks**.        |
| **ModSecurity**    | Web Application Firewall (WAF) logs for blocked attacks.                       | Analyze **blocked malicious payloads**.       |

---

## **2. How to Ingest Web Server Logs into Splunk**
### **A. Direct Splunk Forwarding (Recommended)**
- **For Apache/Nginx**:
  ```bash
  # Install Universal Forwarder on the web server
  /opt/splunkforwarder/bin/splunk add monitor /var/log/apache2/access.log -index web_logs
  ```
- **For IIS (Windows)**:
  - Use **Splunk Add-on for Microsoft IIS** ([Splunkbase](https://splunkbase.splunk.com/app/742)).

### **B. Syslog Forwarding**
- Configure **rsyslog/syslog-ng** to forward logs to Splunk.
  ```bash
  # Example rsyslog config
  *.* @@splunk-server:514
  ```

### **C. Splunk HTTP Event Collector (HEC)**
- Send logs via **HTTP POST** (useful for cloud-hosted servers).
  ```bash
  curl -k https://splunk-server:8088/services/collector -H "Authorization: Splunk <HEC_TOKEN>" -d '{"event":"<LOG_DATA>"}'
  ```

---

## **3. Critical Security Detection Use Cases**
### **1. Web Attacks (OWASP Top 10)**
#### **SQL Injection (SQLi)**
```splunk
index=web_logs sourcetype=apache:access 
| search 
  (uri="*SELECT*" OR uri="*UNION*" OR uri="*1=1*" OR uri="*DROP TABLE*") 
  OR 
  (form_data="*' OR 1=1*")
| stats count by src_ip, uri, user_agent
```

#### **Cross-Site Scripting (XSS)**
```splunk
index=web_logs sourcetype=apache:access 
| search 
  (uri="*<script>*" OR uri="*javascript:*") 
  OR 
  (form_data="*alert(*" OR form_data="*onerror=*")
| table _time, src_ip, uri, user_agent
```

#### **Directory Traversal**
```splunk
index=web_logs sourcetype=apache:access 
| search 
  uri="*../*" OR uri="*/etc/passwd*" 
| stats count by src_ip, uri
```

### **2. Brute Force & Credential Stuffing**
```splunk
index=web_logs sourcetype=apache:access 
  (status=401 OR status=403) 
| stats 
  count as failed_attempts, 
  dc(uri) as endpoints_targeted 
  by src_ip 
| where failed_attempts > 10 
| sort - failed_attempts
```

### **3. Scanner & Bot Detection**
```splunk
index=web_logs sourcetype=apache:access 
| regex user_agent="(nmap|sqlmap|wget|curl|nikto|burp)" 
| stats count by src_ip, user_agent
```

### **4. Data Exfiltration (Large File Downloads)**
```splunk
index=web_logs sourcetype=apache:access 
  status=200 
| eval size_MB = bytes/1024/1024 
| where size_MB > 50 
| table _time, src_ip, uri, size_MB
```

### **5. Suspicious User Agents**
```splunk
index=web_logs sourcetype=apache:access 
| search 
  user_agent="*-" OR user_agent="" OR user_agent="python-requests*" 
| stats count by src_ip, user_agent
```

---

## **4. Performance & Operational Monitoring**
### **High Error Rates (5xx Status Codes)**
```splunk
index=web_logs sourcetype=apache:access 
  status>=500 
| timechart span=1h count as errors
```

### **Slow Requests (Latency Analysis)**
```splunk
index=web_logs sourcetype=apache:access 
| eval response_time_sec = request_time/1000 
| where response_time_sec > 5 
| stats avg(response_time_sec) as avg_latency by uri
```

### **Top Requested URLs**
```splunk
index=web_logs sourcetype=apache:access 
| top limit=20 uri
```

---

## **5. Advanced Correlation & Threat Hunting**
### **Web Logs + Firewall Logs (Blocked Attacks)**
```splunk
index=web_logs sourcetype=apache:access 
| join type=left src_ip 
  [ search index=firewall action=block ] 
| stats count by src_ip, uri, action
```

### **Web Logs + Endpoint Logs (Post-Exploitation Activity)**
```splunk
index=web_logs sourcetype=apache:access 
| join type=inner src_ip 
  [ search index=endpoint_logs event_type=malware_execution ] 
| table _time, src_ip, uri, process_name
```

---

## **6. Best Practices for Web Log Analysis**
 **Normalize Fields** (Extract `src_ip`, `uri`, `user_agent`, `status`).  
 **Enrich with GeoIP** (`iplocation src_ip`).  
 **Use CIM Compliance** (Splunk Common Information Model).  
 **Set Up Alerts** (Brute force, SQLi, XSS).  
 **Retain Logs for Forensics** (90+ days recommended).  

---
