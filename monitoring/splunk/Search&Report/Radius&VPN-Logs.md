# **RADIUS & VPN Logs in Splunk: Enterprise Security Monitoring Guide**

RADIUS and VPN logs provide critical visibility into remote access authentication and network entry points. When properly analyzed in Splunk, they reveal brute force attacks, credential stuffing, MFA bypass attempts, and suspicious geographic logins.

---

## **1. Key Log Sources & Normalization**

### **A. Common VPN/RADIUS Solutions**
| **Vendor**       | **Log Type**          | **Splunk Add-on**                     | **Key Fields**                          |
|------------------|-----------------------|--------------------------------------|----------------------------------------|
| **Cisco AnyConnect** | ASA/PIX logs     | [TA-cisco-asa](https://splunkbase.splunk.com/app/1627/) | `src_ip`, `user`, `vpn_group`, `duration` |
| **Palo Alto GlobalProtect** | Gateway logs | Built-in PAN add-on                 | `src_ip`, `user`, `auth_method`, `reason` |
| **Fortinet SSL-VPN** | FortiGate logs   | [TA-fortinet](https://splunkbase.splunk.com/app/1629/) | `srcip`, `user`, `realm`, `status`     |
| **FreeRADIUS**   | Radius logs          | Custom parsing                      | `Calling-Station-Id`, `User-Name`, `Acct-Status-Type` |

### **B. Field Normalization (CIM Compliance)**
Map to **Splunk's Common Information Model**:
- `src_ip` → Client public IP  
- `user` → Authenticating username  
- `auth_method` → PAP/CHAP/EAP-TLS  
- `status` → Accept/Reject  
- `duration` → Session length  

Example transformation:
```spl
index=vpn_logs sourcetype=cisco:asa 
| eval src_ip=c_ip, user=username, status=action 
| stats count by src_ip, user, status 
| where status="failed"
```

---

## **2. Critical Detection Use Cases**

### **A. Credential Attacks (T1110)**
#### **VPN Brute Force Detection**
```spl
index=vpn_logs status=failed 
| stats count by src_ip, user 
| where count > 5 
| sort - count
```
 **Triggers on**: 5+ failed auth attempts from single IP.

#### **Password Spraying**
```spl
index=vpn_logs status=failed 
| stats dc(user) as failed_users by src_ip 
| where failed_users > 10 
| sort - failed_users
```

---

### **B. MFA Bypass Attempts (T1556)**
#### **MFA Fatigue Attacks**
```spl
index=radius_logs "State=Challenge" 
| stats count by src_ip, user 
| where count > 3 
| sort - count
```

#### **Time-Based MFA Bypass**
```spl
index=radius_logs "Service-Type=Authenticate-Only" 
| stats count by src_ip, user 
| eval rate=count/3600 
| where rate > 0.5
```

---

### **C. Geographic Anomalies (T1078)**
#### **Impossible Travel**
```spl
index=vpn_logs status=success 
| iplocation src_ip 
| stats earliest(_time) as first_login, latest(_time) as last_login by user, Country 
| eval travel_time=last_login-first_login 
| where travel_time < 3600 AND Country!="Unknown"
```

#### **Rare Country Logins**
```spl
index=vpn_logs status=success 
| iplocation src_ip 
| lookup allowed_countries.csv Country as Country OUTPUT is_allowed 
| where is_allowed!="true" 
| table _time, user, src_ip, Country
```

---

### **D. Suspicious Session Patterns (T1133)**
#### **Long VPN Sessions**
```spl
index=vpn_logs status=success 
| eval duration=session_end_time-session_start_time 
| where duration > 86400 
| table user, src_ip, duration
```

#### **Concurrent Sessions**
```spl
index=vpn_logs status=success 
| stats dc(src_ip) as concurrent_sessions by user 
| where concurrent_sessions > 2 
| sort - concurrent_sessions
```

---

## **3. Advanced Correlation Techniques**

### **A. VPN + EDR Correlation**
```spl
index=vpn_logs status=success 
| join user [search index=edr_logs event_type=process 
| stats count by user, process] 
| table _time, user, src_ip, process
```

### **B. Threat Intel Enrichment**
```spl
index=vpn_logs 
| lookup threat_intel_ip src_ip as src_ip OUTPUT is_malicious 
| where is_malicious="true" 
| table _time, user, src_ip, status
```

---

## **4. Splunk Dashboard Examples**

### **VPN Security Dashboard**
**Essential Panels:**
1. **Failed Logins by IP** (`status=failed | stats count by src_ip`)
2. **MFA Attempt Trends** (`State=Challenge | timechart count`)
3. **Geographic Anomalies** (`iplocation | rare Country`)
4. **Session Duration Outliers** (`eval duration | outlier action=trim`)

### **RADIUS Authentication Dashboard**
```spl
index=radius_logs 
| stats count by "User-Name", "Acct-Status-Type" 
| sort - count
```

---

## **5. Best Practices**

 **Enable Verbose Logging**  
   - Log both success/failure events  
   - Capture MFA challenge states  

 **Normalize Authentication Methods**  
   - Tag EAP-TLS vs PAP/CHAP  

 **Baseline Normal Behavior**  
   - Whitelist corporate IP ranges  
   - Establish typical session durations  

 **Correlate with Other Logs**  
   - Combine with EDR for post-auth activity  

---

## **Final Thoughts**
RADIUS/VPN logs in Splunk enable detection of:
 **Credential attacks** (brute force, spraying)  
 **MFA bypass attempts** (fatigue attacks)  
 **Suspicious access patterns** (impossible travel)  
 **Session hijacking** (concurrent sessions)  
