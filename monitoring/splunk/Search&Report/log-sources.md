In an enterprise environment, **Splunk** ingests logs from various sources to enable security monitoring, threat detection, and operational analytics. Below is a list of **common log sources** that organizations typically collect and analyze in Splunk for security and compliance purposes.

---

### **1. Endpoint Logs (EDR / Host-Based Logging)**
- **Windows Event Logs** (`WinEventLog`)  
  - Security (Event IDs: 4624, 4625, 4688, 4697, 4700, 5140)  
  - System (Event IDs: 7034, 7045)  
  - PowerShell (Event IDs: 4103, 4104)  
- **Sysmon Logs** (Enhanced process tracking, network connections)  
  - Process creation (Event ID 1)  
  - Named pipe communications (Event ID 17-18)  
- **Linux/Mac OS Audit Logs** (`auditd`, `syslog`)  
  - Command execution (`/var/log/auth.log`, `/var/log/secure`)  
  - File integrity monitoring (`auditd` rules)  

---

### **2. Network Security Logs**
- **Firewall Logs** (Palo Alto, Cisco ASA, Fortinet)  
  - Allowed/denied connections (`denied src_ip=... dst_ip=...`)  
  - Port scanning, brute force attempts  
- **Proxy Logs** (Blue Coat, Squid, Zscaler)  
  - URL access, blocked domains, user-agent strings  
- **IDS/IPS Logs** (Suricata, Snort, Cisco Firepower)  
  - Alerts on exploit attempts (CVE-based signatures)  
- **NetFlow / PCAP Data** (Network traffic metadata)  
  - Anomalous data transfers (large outbound flows)  

---

### **3. Authentication & Identity Logs**
- **Active Directory (AD) Logs**  
  - Failed logins (Event ID 4771, 4776)  
  - Account lockouts (Event ID 4740)  
  - Golden Ticket attacks (Kerberos Event ID 4769)  
- **RADIUS / VPN Logs** (Cisco AnyConnect, Palo Alto GlobalProtect)  
  - Multi-factor authentication (MFA) failures  
  - Unusual geolocation logins  
- **Okta / Azure AD Logs** (SAML, OAuth events)  
  - Suspicious SSO attempts  

---

### **4. Cloud & SaaS Logs**
- **AWS CloudTrail** (API activity)  
  - IAM role changes, S3 bucket access  
  - Unauthorized `AssumeRole` calls  
- **Microsoft 365 / Azure Logs**  
  - Exchange Online mailbox access  
  - SharePoint file downloads  
- **GSuite / Google Workspace Logs**  
  - Admin activity, Drive file sharing  

---

### **5. Application & Database Logs**
- **Web Server Logs** (Apache, Nginx, IIS)  
  - SQLi, XSS attacks (`/var/log/nginx/access.log`)  
- **Database Audit Logs** (Oracle, MSSQL, MySQL)  
  - Unusual queries (`SELECT * FROM users;`)  
  - Privilege escalations (`GRANT` commands)  
- **Container/Kubernetes Logs** (Docker, EKS, AKS)  
  - Unauthorized pod creation  

---

### **6. Endpoint Detection & Response (EDR) Logs**
- **CrowdStrike, Carbon Black, SentinelOne**  
  - Process injection attempts  
  - Ransomware file encryption patterns  

---

### **7. Email Security Logs**
- **Microsoft Exchange / Proofpoint / Mimecast**  
  - Phishing emails (malicious attachments/links)  
  - Business Email Compromise (BEC) patterns  

---

### **8. Industrial & OT Logs (ICS/SCADA)**
- **Siemens, Rockwell, Schneider Electric**  
  - Unauthorized PLC modifications  

---

### **Why These Logs Matter in Splunk?**
- **Correlation**: Combining firewall + AD logs detects lateral movement.  
- **Threat Hunting**: Sysmon + EDR logs reveal process hollowing (T1055).  
- **Compliance**: PCI-DSS, HIPAA require specific log retention.  

---

### **Example Splunk Queries for Common Logs**
1. **Detect Brute Force (T1110)**  
   ```spl
   index=win_events EventCode=4625 
   | stats count by src_ip 
   | where count > 5
   ```
2. **Find PowerShell Execution (T1059)**  
   ```spl
   index=win_events EventCode=4104 
   | search "PowerShell -nop -exec bypass"
   ```
3. **Proxy Logs for C2 Traffic**  
   ```spl
   index=proxy_logs url=*pastebin.com* 
   | table src_ip, user, url
   ```

---

### **Best Practices for Log Collection in Splunk**
 **Normalize logs** (use CIM-compliant fields like `src_ip`, `user`).  
 **Filter noise** (exclude irrelevant events to save license costs).  
 **Use Splunk TA (Technology Add-ons)** for parsing (e.g., `TA-microsoft-sysmon`).  
