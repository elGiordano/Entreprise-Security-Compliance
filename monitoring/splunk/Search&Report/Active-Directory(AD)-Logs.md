# **Active Directory (AD) Logs in Splunk: Comprehensive Security Monitoring Guide**

Active Directory logs are the crown jewels for detecting identity-based attacks in Windows environments. When properly analyzed in Splunk, they reveal brute force attempts, privilege escalation, Golden Ticket attacks, and lateral movement. Here's how to leverage them effectively.

---

## **1. Critical AD Log Sources & Event IDs**

### **A. Essential Windows Event Log Channels**
| **Log Channel**       | **Security Relevance**                                                                 | **Key Event IDs**                          |
|-----------------------|--------------------------------------------------------------------------------------|------------------------------------------|
| **Security**          | Authentication, account changes, privilege escalation                                | 4624, 4625, 4768-4771, 4672              |
| **Directory Service** | AD object modifications (users, groups, OUs)                                         | 5136, 4662, 4928-4929                    |
| **DNS Server**        | AD-integrated DNS queries (often abused for C2)                                      | 5168, 5176                               |
| **File Replication**  | DC synchronization (detects DCSync attacks)                                          | 4928, 4929                               |

### **B. Must-Monitor Event IDs**
| **Event ID** | **Attack Technique**               | **Splunk Detection Use Case**                     |
|-------------|-----------------------------------|------------------------------------------------|
| **4624**    | Successful logon                  | Baseline normal activity                       |
| **4625**    | Failed logon                      | Brute force attacks (T1110)                    |
| **4672**    | Admin logon                       | Privileged account usage (T1078)               |
| **4768**    | Kerberos TGT request              | Golden Ticket detection (T1558.001)            |
| **4776**    | NTLM authentication               | Pass-the-Hash attempts (T1550.002)             |
| **5136**    | AD object modification            | Suspicious user/group changes (T1098)          |
| **4662**    | Directory Service Access          | DCSync attack detection (T1003.003)            |

---

## **2. Critical Detection Use Cases**

### **A. Credential Attacks (T1110)**
#### **Brute Force Detection**
```spl
index=win_events EventCode=4625 
| stats count by src_ip, user 
| where count > 5 
| sort - count
```
 **Triggers on**: 5+ failed logins from single IP.

#### **Password Spraying**
```spl
index=win_events EventCode=4625 
| stats dc(user) as failed_users by src_ip 
| where failed_users > 10 
| sort - failed_users
```

---

### **B. Privilege Escalation (T1068, T1078)**
#### **Admin Group Modifications**
```spl
index=win_events EventCode=4728 OR EventCode=4732 
| search "Domain Admins" OR "Enterprise Admins" 
| table _time, host, user, member, target_user
```

#### **Kerberos Golden Ticket (T1558.001)**
```spl
index=win_events EventCode=4768 
| search TicketOptions="0x40810000" 
| stats count by src_ip, user 
| sort - count
```

---

### **C. Lateral Movement (T1021)**
#### **Pass-the-Hash (NTLM)**
```spl
index=win_events EventCode=4776 
| stats count by src_ip, user 
| where count > 3 
| sort - count
```

#### **Overpass-the-Hash (Kerberos)**
```spl
index=win_events EventCode=4768 
| search ServiceName="krbtgt" AND TicketOptions="0x40810000" 
| table _time, src_ip, user
```

---

### **D. Persistence (T1098)**
#### **Skeleton Key Attack**
```spl
index=win_events EventCode=4611 
| search ProcessName="lsass.exe" 
| table _time, host, user, ProcessName
```

#### **AdminSDHolder Modification**
```spl
index=win_events EventCode=5136 
| search ObjectClass="groupPolicyContainer" AttributeLDAPDisplayName="adminCount" 
| table _time, host, user, ObjectDN
```

---

## **3. Advanced Correlation Techniques**

### **A. DCShadow Attack Detection**
```spl
index=win_events (EventCode=4928 OR EventCode=4929) 
| stats count by src_ip, user 
| where count > 3 
| sort - count
```

### **B. Honey Account Monitoring**
```spl
index=win_events EventCode=4624 user=honey* 
| table _time, src_ip, user, LogonType
```

### **C. Time Anomaly Detection (Kerberos)**
```spl
index=win_events EventCode=4773 
| eval time_diff=abs(ClientTime-_time) 
| where time_diff > 300 
| table _time, host, user, ClientTime, time_diff
```

---

## **4. Splunk Dashboard Examples**

### **AD Security Dashboard**
**Essential Panels:**
1. **Failed Logons by IP** (`EventCode=4625 | stats count by src_ip`)
2. **Privileged Account Usage** (`EventCode=4672 | timechart count`)
3. **Kerberos Anomalies** (`EventCode IN (4768,4769) | stats count by ServiceName`)
4. **AD Object Changes** (`EventCode=5136 | top ObjectDN`)

### **Golden Ticket Detection Panel**
```spl
index=win_events EventCode=4768 TicketOptions="0x40810000" 
| stats count by src_ip, user 
| sort - count
```

---

## **5. Best Practices**

 **Enable Advanced Auditing** (Audit Policy → "Object Access" = Success/Failure)  
 **Collect All Domain Controllers** (Ensure complete visibility)  
 **Normalize Fields** (`user`, `src_ip`, `LogonType` per CIM)  
 **Tune for False Positives** (Exclude known admin workstations)  

---

## **Final Thoughts**
AD logs in Splunk enable detection of:
 **Credential compromise** (brute force, spraying)  
 **Privilege escalation** (Golden Tickets, group changes)  
 **Persistence mechanisms** (Skeleton Keys, AdminSDHolder)  
 **Lateral movement** (Pass-the-Hash, Overpass-the-Hash)  
