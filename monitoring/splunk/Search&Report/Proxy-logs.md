# **Proxy Logs in Splunk: Enterprise Security Deep Dive**

Proxy logs provide critical visibility into web traffic, enabling detection of malware downloads, C2 communications, data exfiltration, and policy violations. Here's how to maximize their value in Splunk for security monitoring.

---

## **1. Key Proxy Log Sources & Normalization**

### **A. Common Enterprise Proxy Solutions**
| **Vendor**       | **Log Type**          | **Splunk Add-on**                     | **Key Fields**                          |
|------------------|-----------------------|--------------------------------------|----------------------------------------|
| **Blue Coat**    | Web Proxy             | [TA-bluecoat](https://splunkbase.splunk.com/app/1377/) | `src_ip`, `dest_host`, `url`, `action`, `user` |
| **Zscaler**      | ZIA Web Logs          | [TA-zscaler](https://splunkbase.splunk.com/app/4216/) | `client_ip`, `server_ip`, `url`, `action` |
| **Squid**        | Access Logs           | Built-in parsing                     | `src_ip`, `dest_domain`, `http_status`, `bytes` |
| **Microsoft WAF**| Azure Front Door      | [TA-microsoft-cloud](https://splunkbase.splunk.com/app/3110/) | `clientIP`, `host`, `requestUri`, `rule` |

### **B. Field Normalization (CIM Compliance)**
For consistent analysis, map proxy logs to **Splunk's Common Information Model**:
- `src_ip` → Originating client IP  
- `dest_host` → Destination domain/IP  
- `url` → Full HTTP request  
- `http_method` → GET/POST/PUT  
- `user_agent` → Client browser/software  
- `bytes_out` → Data exfil volume  

Example transformation:
```spl
index=proxy_logs sourcetype=bluecoat:proxylog 
| eval src_ip=client_ip, dest_host=cs_host, url=cs_uri, http_method=cs_method 
| stats sum(sc_bytes) as bytes_out by src_ip, dest_host, url
```

---

## **2. Critical Proxy Detection Use Cases**

### **A. Malware Downloads (T1105)**
#### **Detect Executable Downloads from Untrusted Domains**
```spl
index=proxy_logs url=* AND (url=*.exe OR url=*.ps1 OR url=*.dll) 
| lookup threat_intel_domains dest_host as dest_host OUTPUT is_malicious 
| where is_malicious="true" OR dest_host IN ("pastebin.com","anonfiles.com") 
| table _time, src_ip, user, dest_host, url, bytes_out
```
 **Triggers on**: Downloads of executables from known malicious sites.

#### **Find Office Docs with Macros (Potential Phishing)**
```spl
index=proxy_logs url=* AND (url=*.doc* OR url=*.xls*) 
| search url=*macro* OR url=*vba* 
| stats count by src_ip, user, dest_host, url
```

---

### **B. Command & Control (C2) Traffic (T1071)**
#### **Beaconing to Rare/DGA Domains**
```spl
index=proxy_logs 
| stats count by src_ip, dest_host 
| eventstats avg(count) as avg, stdev(count) as stdev by src_ip 
| eval lower_threshold=(avg-(stdev*3)), upper_threshold=(avg+(stdev*3)) 
| where count < lower_threshold OR count > upper_threshold 
| sort - count
```
 **Detects**: Beaconing via low/high request counts (statistical anomaly).

#### **HTTP POST to Unknown IPs (Data Exfiltration)**
```spl
index=proxy_logs http_method=POST dest_host!=*.* 
| lookup threat_intel_ip dest_host as dest_host OUTPUT is_malicious 
| where is_malicious="true" OR cidrmatch("192.168.0.0/16", dest_host)=false 
| table _time, src_ip, user, dest_host, url, bytes_out
```

---

### **C. Data Exfiltration (T1041)**
#### **Large File Uploads to Cloud Storage**
```spl
index=proxy_logs http_method=POST 
| search (url=*drive.google.com* OR url=*dropbox.com*) 
| stats sum(bytes_out) as total_upload by src_ip, user, dest_host 
| where total_upload > 100000000 
| sort - total_upload
```
 **Flags**: 100MB+ uploads to cloud services (potential data theft).

#### **Base64-Encoded Exfil in URLs**
```spl
index=proxy_logs url=*%3D* 
| rex field=url ".*?(?<b64_content>[A-Za-z0-9+/%]{20,}=*)" 
| where isnotnull(b64_content) 
| table _time, src_ip, dest_host, url
```

---

### **D. Policy Violations & Shadow IT**
#### **Unauthorized Cloud Apps**
```spl
index=proxy_logs 
| search dest_host=* 
| eval domain=lower(mvindex(split(dest_host,"."),-2).".".lower(mvindex(split(dest_host,"."),-1) 
| lookup allowed_domains_lookup domain as domain OUTPUT is_allowed 
| where is_allowed!="true" 
| stats count by src_ip, user, domain
```

#### **TOR/Anonymizer Usage**
```spl
index=proxy_logs 
| lookup tor_exit_nodes dest_host as dest_host OUTPUT is_tor 
| where is_tor="true" 
| table _time, src_ip, user, dest_host, url
```

---

## **3. Advanced Correlation Techniques**

### **A. Combine Proxy + EDR Logs (Malware Execution Chain)**
```spl
index=proxy_logs url=*.exe 
| join src_ip [search index=endpoint_logs process=*.exe 
| stats count by src_ip, process] 
| table _time, src_ip, user, url, process
```

### **B. User Agent Anomaly Detection**
```spl
index=proxy_logs 
| stats dc(user_agent) as unique_agents by src_ip 
| where unique_agents > 3 
| sort - unique_agents
```

---

## **4. Splunk Dashboard Examples**

### **Proxy Security Monitoring Dashboard**
**Essential Panels:**
1. **Top Malicious Domains** (`lookup threat_intel_domains | stats count by dest_host`)
2. **Data Exfiltration Risks** (`http_method=POST | stats sum(bytes_out) by src_ip`)
3. **User Agent Anomalies** (`rare user_agent values by department`)
4. **Policy Violations** (`unauthorized domains/apps`)

### **Splunk XML Snippet**
```xml
<panel>
  <title>Recent Malware Downloads</title>
  <search>
    <query>index=proxy_logs url=* AND (url=*.exe OR url=*.ps1) 
    | lookup threat_intel_domains dest_host as dest_host OUTPUT is_malicious 
    | where is_malicious="true" 
    | stats count by src_ip, user, dest_host, url</query>
  </search>
  <option name="count">10</option>
</panel>
```

---

## **5. Best Practices**
 **Log Full URLs** (Include query strings for investigation)  
 **Normalize Key Fields** (Use CIM-compliant `src_ip`, `dest_host`)  
 **Enrich with Threat Intel** (Domain/IP reputation lookups)  
 **Baseline Normal Traffic** (Reduce false positives)  

---

## **Final Thoughts**
Proxy logs in Splunk enable detection of:
 **Malware distribution** (drive-by downloads)  
 **C2 communications** (beaconing, exfil)  
 **Insider threats** (data theft, policy violations)  

