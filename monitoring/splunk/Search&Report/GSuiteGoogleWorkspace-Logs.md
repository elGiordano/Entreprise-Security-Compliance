# **GSuite / Google Workspace Logs in Splunk: Comprehensive Security Monitoring Guide**

Integrating **Google Workspace (formerly GSuite)** logs into **Splunk** enables organizations to monitor user activities, detect security threats, and ensure compliance. This guide covers best practices for collecting, parsing, and analyzing Google Workspace logs in Splunk for security monitoring.

---

## **1. Why Monitor Google Workspace Logs in Splunk?**
Google Workspace logs provide visibility into:
- **User activities** (logins, file access, admin actions)
- **Data exfiltration risks** (unusual file downloads/sharing)
- **Phishing & account compromise** (suspicious logins)
- **Compliance auditing** (GDPR, HIPAA, SOX)

Splunk enhances this by:
- **Correlating** logs with other security data (EDR, firewall, etc.)
- **Alerting** on anomalies (impossible travel, bulk deletions)
- **Automating investigations** (playbooks, dashboards)

---

## **2. Methods to Ingest Google Workspace Logs into Splunk**
### **A. Google Workspace Admin Audit & Vault API (Recommended)**
- **Supported Logs**:
  - Admin Activity
  - Login Activity
  - Drive Activity
  - Groups Activity
  - SAML Activity
- **Setup Steps**:
  1. **Enable API Access** in Google Workspace Admin Console.
  2. **Create a Service Account** with proper permissions.
  3. **Use Splunk Add-ons**:
     - **[Google Workspace Add-on for Splunk](https://splunkbase.splunk.com/app/3084/)**
     - **[Google Cloud Platform Add-on for Splunk](https://splunkbase.splunk.com/app/2634/)**
  4. **Configure HTTP Event Collector (HEC)** for log ingestion.

### **B. Syslog Forwarding (Legacy)**
- Configure **Google Workspace to send logs to a syslog server**.
- Use **Splunk Universal Forwarder** to collect logs.

### **C. Manual Export & Ingest (Ad-Hoc)**
- Export logs via **Google Workspace Admin Console** or **BigQuery**.
- Upload to Splunk via **CSV/JSON inputs**.

---

## **3. Key Google Workspace Log Sources**
| **Log Type**          | **Description**                                      | **Security Use Cases** |
|-----------------------|-----------------------------------------------------|------------------------|
| **Admin Activity**    | Changes by admins (user creation, role changes)     | Detect privilege abuse |
| **Login Activity**    | User sign-ins (success/failure, IP, device)         | Detect brute force, phishing |
| **Drive Activity**    | File access, sharing, deletions                     | Detect data exfiltration |
| **SAML Activity**     | SSO logins & failures                               | Detect SSO hijacking |
| **Groups Activity**   | Changes to Google Groups                            | Detect unauthorized access |

---

## **4. Parsing & Enriching Logs in Splunk**
### **A. SPL Queries for Common Security Scenarios**
#### **1. Suspicious Logins (Impossible Travel)**
```splunk
index=google_workspace log_type=login 
| eval time_diff = (now() - _time)/3600 
| stats earliest(_time) as first_login, latest(_time) as last_login, values(ip_address) by user 
| where time_diff < 24 
| eval travel_speed = (geo_distance(first_login, last_login)/time_diff) 
| where travel_speed > 500  // Threshold in km/h
```

#### **2. Unusual File Downloads (Data Exfiltration)**
```splunk
index=google_workspace log_type=drive event_type=download 
| stats count by user, file_name 
| where count > 50  // Threshold for excessive downloads
```

#### **3. Admin Privilege Escalation**
```splunk
index=google_workspace log_type=admin event_type=assign_role 
| table _time, admin_user, target_user, new_role
```

### **B. Enrichment with Threat Intelligence**
- Use **Splunk ES** or **lookups** to match IPs against known malicious lists.
- Example:
  ```splunk
  | lookup threat_intel ip as client_ip OUTPUT threat_desc
  | where isnotnull(threat_desc)
  ```

---

## **5. Building Security Dashboards**
### **A. Google Workspace Security Overview**
- **Panels**:
  - Top Failed Logins
  - Suspicious Admin Activities
  - Unusual File Sharing Trends
  - Geo-Map of Login Attempts

### **B. Insider Threat Detection**
- Track **mass deletions, unauthorized sharing, after-hours logins**.

### **C. Compliance Reporting**
- Generate **audit reports** for **GDPR, HIPAA, SOX**.

---

## **6. Automating Alerts & Responses**
### **A. Splunk Alerts for Critical Events**
- **Example Alert: "Multiple Failed Logins Followed by Success"**
  ```splunk
  index=google_workspace log_type=login 
  | stats count(eval(status="FAIL")) as fails, count(eval(status="SUCCESS")) as success by user, ip 
  | where fails > 5 AND success > 0
  ```

### **B. SOAR Integration (Splunk Phantom, Palo Alto XSOAR)**
- **Automated Responses**:
  - Disable compromised accounts via **Google Admin API**.
  - Trigger **ticket creation** in ITSM tools.

---

## **7. Best Practices**
 **Enable all relevant log sources** in Google Workspace.  
 **Normalize fields** (e.g., `user`, `ip`, `event_type`) for correlation.  
 **Store logs in a dedicated Splunk index** (`google_workspace`).  
 **Regularly review dashboards & fine-tune alerts**.  
 **Integrate with SIEM (Splunk ES) for advanced analytics**.  

---

## **Conclusion**
By ingesting **Google Workspace logs into Splunk**, security teams gain **real-time visibility** into user activities, detect **insider threats & external attacks**, and meet **compliance requirements**. Follow this guide to set up a robust monitoring framework.

**Next Steps:**
- [Download the Splunk Google Workspace Add-on](https://splunkbase.splunk.com/app/3084/)
- [Explore Splunk ES for advanced detection](https://www.splunk.com/en_us/software/enterprise-security.html)
