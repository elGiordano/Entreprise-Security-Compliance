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

# **Web Server Logs in Splunk: Security Monitoring & Threat Detection Guide**

Web server logs provide critical insights into **web traffic, attacks, and user behavior**. When ingested into **Splunk**, they enable **real-time security monitoring, anomaly detection, and incident response**. Below is a **comprehensive guide** to analyzing **Apache, Nginx, IIS, and other web server logs** in Splunk for security purposes.

---

## **1. Key Web Server Log Sources**
| **Web Server** | **Default Log Location**          | **Key Fields**                     |
|---------------|----------------------------------|-----------------------------------|
| **Apache**    | `/var/log/apache2/access.log`    | `client_ip, status_code, user_agent, request` |
| **Nginx**     | `/var/log/nginx/access.log`      | `remote_addr, http_referer, request` |
| **IIS**       | `%SystemDrive%\inetpub\logs\LogFiles\` | `c-ip, cs-method, cs-uri-stem, sc-status` |
| **Cloud (AWS ALB, Cloudflare)** | Varies (API/S3) | `http.request.method, http.user_agent, src_ip` |

---

## **2. Critical Security Detection Use Cases**
### **1. Web Attacks (OWASP Top 10)**
#### **SQL Injection (SQLi)**
```splunk
index=web_logs (http_method=POST OR http_method=GET)
| regex request=".*([';]+\s*(SELECT|UNION|DROP|INSERT|DELETE|UPDATE|EXEC).*)" 
| stats count by client_ip, request, user_agent
```

#### **Cross-Site Scripting (XSS)**
```splunk
index=web_logs 
| regex request=".*(<script|javascript:|onerror=|alert\().*"
| table _time, client_ip, request, user_agent
```

#### **Local/Remote File Inclusion (LFI/RFI)**
```splunk
index=web_logs 
| regex request=".*(\.\./|etc/passwd|php://input|http://).*"
| stats count by client_ip, request
```

### **2. Brute Force & Credential Stuffing**
#### **Login Page Attacks**
```splunk
index=web_logs uri_path="/login" 
| stats 
  count(eval(status_code=401)) as failed_logins, 
  count(eval(status_code=200)) as success_logins 
  by client_ip 
| where failed_logins > 5 
| sort - failed_logins
```

#### **WordPress XML-RPC Abuse**
```splunk
index=web_logs uri_path="/xmlrpc.php" 
| stats count by client_ip 
| where count > 20  // Common in brute force attacks
```

### **3. Suspicious Bots & Scanners**
#### **Bad User Agents (Automated Scanners)**
```splunk
index=web_logs 
| search user_agent="*sqlmap*" OR user_agent="*nikto*" OR user_agent="*hydra*" 
| stats count by client_ip, user_agent
```

#### **Web Vulnerability Scanners**
```splunk
index=web_logs 
| regex user_agent="(Acunetix|Nessus|Burp Suite|OWASP ZAP)" 
| table _time, client_ip, user_agent, request
```

### **4. Data Exfiltration & Unusual Traffic**
#### **Large File Downloads (Data Theft)**
```splunk
index=web_logs status_code=200 
| eval response_size_kb = bytes/1024 
| where response_size_kb > 10240  // 10MB+ files
| stats sum(response_size_kb) as total_downloaded by client_ip
```

#### **Unusual Outbound Connections (C2 Traffic)**
```splunk
index=web_logs 
| lookup threat_intel_ip client_ip OUTPUT threat_desc 
| where isnotnull(threat_desc) 
| table _time, client_ip, threat_desc, request
```

---

## **3. Advanced Correlation & Anomaly Detection**
### **1. Impossible Travel (Session Hijacking)**
```splunk
index=web_logs 
| iplocation client_ip 
| stats 
  earliest(_time) as first_request, 
  latest(_time) as last_request, 
  values(country) as countries 
  by session_id 
| eval time_diff = (last_request - first_request)/3600 
| where time_diff < 2 AND mvcount(countries) > 1
```

### **2. Zero-Day Exploit Detection (Anomalous Requests)**
```splunk
index=web_logs 
| anomalydetect method=avg request_length by client_ip 
| where isoutlier=1 
| table _time, client_ip, request, status_code
```

### **3. DDoS & Rate Limiting**
```splunk
index=web_logs 
| bin _time span=1m 
| stats count by client_ip, _time 
| where count > 1000  // 1000+ requests/min
| sort - count
```

---

## **4. Best Practices for Web Log Analysis**
✅ **Normalize Fields** (e.g., `client_ip`, `uri_path`, `user_agent`)  
✅ **Enrich with Threat Intel** (e.g., known malicious IPs, TOR exit nodes)  
✅ **Use CIM (Common Information Model)** for consistent field mapping  
✅ **Store in a Dedicated Index** (`web_logs`) for retention & performance  
✅ **Automate Blocking** (via Splunk + Firewall/WAF integration)  

---

## **5. Sample Splunk Dashboard Panels**
| **Panel**                     | **SPL Query**                          |
|-------------------------------|---------------------------------------|
| **Top Attackers**            | `index=web_logs | top client_ip`       |
| **Failed Login Trends**      | `index=web_logs status_code=401 | timechart count` |
| **Malicious User Agents**    | `index=web_logs | regex user_agent="(sqlmap|nikto)"` |
| **Unusual HTTP Methods**     | `index=web_logs | rare http_method`    |

---
