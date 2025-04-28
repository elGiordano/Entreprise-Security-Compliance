
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

--- 

# **Critical Detection Use Cases for Google Workspace Logs in Splunk**

Monitoring **Google Workspace (GSuite)** logs in **Splunk** is essential for detecting security threats, insider risks, and compliance violations. Below are the **most critical detection use cases**, along with **Splunk SPL queries** and **best practices** for implementation.

---

## **1. Account Takeover & Credential Compromise**
### **Detection Scenarios**
- **Brute Force Attacks** (Multiple failed logins followed by success)
- **Impossible Travel** (Logins from geographically distant locations in a short time)
- **Suspicious IPs** (Logins from Tor, VPNs, or known malicious IPs)
- **Phishing Success** (Logins from phishing-prone regions)

### **SPL Queries**
#### **Brute Force Detection**
```splunk
index=google_workspace log_type=login 
| stats 
  count(eval(status="FAILED")) as failed_logins, 
  count(eval(status="SUCCESS")) as successful_logins, 
  earliest(_time) as first_attempt, 
  latest(_time) as last_attempt 
  by user, src_ip 
| where failed_logins > 5 AND successful_logins > 0 
| eval time_window = (last_attempt - first_attempt)/60 
| where time_window < 30  // 30-minute window
```

#### **Impossible Travel**
```splunk
index=google_workspace log_type=login status=SUCCESS 
| iplocation client_ip 
| stats 
  earliest(_time) as first_login, 
  latest(_time) as last_login, 
  values(city) as cities, 
  values(country) as countries 
  by user 
| eval time_diff = (last_login - first_login)/3600 
| where time_diff < 24 
| eval distance = haversine(first_login, last_login) 
| where distance > 500  // 500 km in less than 24h
```

#### **Suspicious IPs (Threat Intel Lookup)**
```splunk
index=google_workspace log_type=login 
| lookup threat_intel_ip client_ip OUTPUT threat_desc 
| where isnotnull(threat_desc) 
| table _time, user, client_ip, threat_desc
```

---

## **2. Insider Threats & Data Exfiltration**
### **Detection Scenarios**
- **Mass File Downloads/Deletions** (Potential data theft)
- **Unauthorized Sharing of Sensitive Files** (External sharing of confidential docs)
- **Unusual Access Patterns** (Accessing files outside job role)
- **After-Hours Activity** (Unusual logins or file access at odd hours)

### **SPL Queries**
#### **Mass File Downloads**
```splunk
index=google_workspace log_type=drive event_type=download 
| stats count by user, file_name 
| where count > 50  // Threshold for excessive downloads
```

#### **Sensitive File Shared Externally**
```splunk
index=google_workspace log_type=drive event_type=share 
| search shared_with_domain != "yourcompany.com" 
| table _time, user, file_name, shared_with_email
```

#### **After-Hours Activity**
```splunk
index=google_workspace log_type=login OR log_type=drive 
| eval hour = strftime(_time, "%H") 
| where hour > 20 OR hour < 6  // 8 PM - 6 AM
| stats count by user, event_type
```

---

## **3. Privilege Escalation & Admin Abuse**
### **Detection Scenarios**
- **Unauthorized Admin Role Changes** (Sudden privilege escalation)
- **Super Admin Logins from Unusual Locations** (Compromised admin accounts)
- **Excessive Permission Grants** (Bulk user permission changes)

### **SPL Queries**
#### **Admin Role Changes**
```splunk
index=google_workspace log_type=admin event_type=assign_role 
| table _time, admin_user, target_user, new_role
```

#### **Super Admin Logins from New Countries**
```splunk
index=google_workspace log_type=login status=SUCCESS user=*admin* 
| iplocation client_ip 
| stats count by user, country 
| where count < 5  // Rare country for this admin
```

#### **Bulk Permission Changes**
```splunk
index=google_workspace log_type=admin event_type=modify_permissions 
| stats dc(target_user) as users_affected by admin_user 
| where users_affected > 10
```

---

## **4. Compliance Violations & Audit Failures**
### **Detection Scenarios**
- **Disabled Audit Logging** (Attempts to turn off logging)
- **GDPR Violations** (Personal data shared externally)
- **Failed Compliance Checks** (Unauthorized access to regulated data)

### **SPL Queries**
#### **Disabled Audit Logging**
```splunk
index=google_workspace log_type=admin event_type=change_setting 
| search setting_name="audit_logging" new_value="disabled" 
| table _time, admin_user, setting_name, new_value
```

#### **External Sharing of Sensitive Files (GDPR)**
```splunk
index=google_workspace log_type=drive event_type=share 
| search file_name="*confidential*" AND shared_with_domain!="yourcompany.com" 
| table _time, user, file_name, shared_with_email
```

---

## **5. Threat Correlation with Other Log Sources**
### **Detection Scenarios**
- **Google Workspace + Endpoint Logs** (Malware detected after suspicious login)
- **Google Workspace + VPN Logs** (Impossible travel verification)
- **Google Workspace + Email Logs** (Phishing link clicked before account compromise)

### **SPL Query Example (Malware + Suspicious Login)**
```splunk
index=google_workspace log_type=login status=SUCCESS 
| join type=inner user 
  [ search index=endpoint_logs event_type=malware_detected ] 
| table _time, user, client_ip, malware_name
```
