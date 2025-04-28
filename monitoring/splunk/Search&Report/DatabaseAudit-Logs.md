# **Database Audit Logs in Splunk: Comprehensive Security Monitoring Guide**

Database audit logs are critical for detecting **unauthorized access, data breaches, and compliance violations**. Integrating them into **Splunk** enables real-time monitoring, threat detection, and forensic investigations. Below is a **complete guide** to analyzing **Oracle, SQL Server, MySQL, PostgreSQL, and NoSQL** audit logs in Splunk.

---

## **1. Key Database Log Sources**
| **Database**   | **Audit Log Location**                     | **Key Fields**                     |
|---------------|-------------------------------------------|-----------------------------------|
| **Oracle**    | `AUDIT_TRAIL` (Database/OS files)         | `OS_USERNAME, DB_USER, ACTION_NAME, OBJECT_NAME` |
| **SQL Server**| SQL Server Audit Logs / Windows Event Log | `session_id, server_principal_name, object_name` |
| **MySQL**     | General Query Log / Enterprise Audit Plugin | `user_host, command_type, argument` |
| **PostgreSQL**| `pg_log` directory                        | `user_name, database_name, query` |
| **MongoDB**   | Audit Log (JSON format)                   | `atype, param.command, users.roles` |

---

## **2. Critical Detection Use Cases**
### **1. Unauthorized Access & Privilege Escalation**
#### **Failed Login Attempts (Brute Force)**
```splunk
index=db_logs action=LOGON status=FAILED 
| stats count by user, client_ip 
| where count > 5 
| sort - count
```

#### **Privilege Escalation (Granting Admin Rights)**
```splunk
index=db_logs action=GRANT 
| search granted_role="*DBA*" OR granted_role="*ADMIN*" 
| table _time, user, granted_role, grantor
```

### **2. Sensitive Data Access & Exfiltration**
#### **Access to Payment/Customer Data (PCI/GDPR)**
```splunk
index=db_logs 
| search object_name="*credit_card*" OR object_name="*customer*" 
| stats count by user, object_name 
| sort - count
```

#### **Large Data Exports (Potential Data Theft)**
```splunk
index=db_logs action=SELECT 
| eval row_count = if(match(query, "LIMIT \d+"), null(), 1000) 
| where row_count > 10000 
| table _time, user, query, row_count
```

### **3. SQL Injection & Malicious Queries**
#### **Suspicious SQL Patterns (SQLi)**
```splunk
index=db_logs 
| regex query=".*(SELECT\s.*FROM\s.*WHERE\s.*\d+=\d+|UNION\sSELECT|xp_cmdshell).*" 
| table _time, user, client_ip, query
```

#### **Database Schema Tampering (DROP/ALTER)**
```splunk
index=db_logs 
| search action IN (DROP, ALTER, TRUNCATE) 
| stats count by user, action, object_name
```

### **4. Database Misconfigurations & Compliance Risks**
#### **Audit Logging Disabled (Compliance Violation)**
```splunk
index=db_logs action="AUDIT_CONFIG" 
| search new_value="OFF" 
| table _time, user, action, new_value
```

#### **Default/Weak Credentials Usage**
```splunk
index=db_logs action=LOGON user IN (admin, sa, root) 
| stats count by user, client_ip
```

---

## **3. Advanced Correlation & Anomaly Detection**
### **1. Impossible Travel (Database Access from Multiple Locations)**
```splunk
index=db_logs action=LOGON status=SUCCESS 
| iplocation client_ip 
| stats 
  earliest(_time) as first_login, 
  latest(_time) as last_login, 
  values(country) as countries 
  by user 
| eval time_diff = (last_login - first_login)/3600 
| where time_diff < 24 AND mvcount(countries) > 1
```

### **2. Unusual Query Timing (After-Hours Activity)**
```splunk
index=db_logs 
| eval hour = strftime(_time, "%H") 
| where hour > 20 OR hour < 6 
| stats count by user, action 
| sort - count
```

### **3. Data Integrity Attacks (Bulk Deletions/Updates)**
```splunk
index=db_logs action IN (DELETE, UPDATE) 
| stats count by user, action 
| where count > 1000 
| sort - count
```

---

## **4. Best Practices for Database Log Analysis**
 **Enable Full Audit Logging** (DDL, DML, DCL, LOGON events)  
 **Normalize Fields** (e.g., `user`, `action`, `object_name`)  
 **Enrich with Threat Intel** (e.g., known malicious IPs)  
 **Use CIM (Common Information Model)** for consistency  
 **Store in a Dedicated Index** (`db_logs`)  

---

## **5. Sample Splunk Dashboard Panels**
| **Panel**                     | **SPL Query**                          |
|-------------------------------|---------------------------------------|
| **Top Database Users**        | `index=db_logs | top user`             |
| **Failed Logins Trend**       | `index=db_logs action=LOGON status=FAILED | timechart count` |
| **Sensitive Data Access**    | `index=db_logs object_name="*password*" | stats count by user` |
| **Unusual Query Volume**     | `index=db_logs | anomalydetect count by user` |

---
