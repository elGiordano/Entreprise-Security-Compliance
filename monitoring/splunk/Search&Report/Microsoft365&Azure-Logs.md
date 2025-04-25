# **Microsoft 365 & Azure Logs in Splunk: Comprehensive Security Monitoring Guide**

Microsoft 365 and Azure logs provide critical visibility into cloud productivity and infrastructure activities. When properly analyzed in Splunk, they reveal email compromises, suspicious SharePoint access, Azure AD anomalies, and cloud resource misconfigurations.

---

## **1. Key Log Sources & Normalization**

### **A. Essential Microsoft 365/Azure Log Types**
| **Log Type**               | **Security Relevance**                                                                 | **Splunk Add-on**                     |
|----------------------------|--------------------------------------------------------------------------------------|--------------------------------------|
| **Azure AD Sign-ins**       | Authentication events (success/failures, MFA, risky sign-ins)                       | [TA-microsoft-cloud](https://splunkbase.splunk.com/app/3110/) |
| **Office 365 Audit**        | Exchange Online, SharePoint, Teams activities                                      | Requires Office 365 API setup        |
| **Microsoft Defender ATP** | Endpoint detection and response events                                             | [TA-microsoft-defender](https://splunkbase.splunk.com/app/5367/) |
| **Azure Activity Logs**    | ARM API calls (resource changes, RBAC modifications)                               | Built-in Azure Monitor integration   |

### **B. Field Normalization (CIM Compliance)**
Map to **Splunk's Common Information Model**:
- `user` → `userPrincipalName` (Azure AD) / `UserId` (Office 365)  
- `src_ip` → `ipAddress` (Azure AD) / `ClientIP` (Office 365)  
- `action` → `operationName` (Azure) / `Operation` (Office 365)  
- `resource` → `targetResources.name`  

Example transformation:
```spl
index=azure_logs sourcetype=azure:aad:signin 
| eval user=userPrincipalName, src_ip=ipAddress, action=appDisplayName 
| stats count by user, src_ip, action, status.errorCode
```

---

## **2. Critical Detection Use Cases**

### **A. Email Compromise (T1534)**
#### **Suspicious Mailbox Rules**
```spl
index=o365_logs Operation="New-InboxRule" 
| search Parameters LIKE "%forward%" OR Parameters LIKE "%redirect%" 
| table _time, UserId, ClientIP, Parameters
```

#### **Mass Email Exfiltration**
```spl
index=o365_logs Operation="Send" 
| stats dc(Parameters.To) as recipients by UserId 
| where recipients > 50 
| sort - recipients
```

---

### **B. SharePoint Data Exfiltration (T1213)**
#### **Bulk File Downloads**
```spl
index=o365_logs Operation="FileDownloaded" 
| stats count by UserId, SiteUrl, SourceFileName 
| where count > 100 
| sort - count
```

#### **Sensitive File Access**
```spl
index=o365_logs Operation="FileAccessed" 
| search SourceFileName="*.xlsx" OR SourceFileName="*.docx" 
| lookup sensitive_files.csv SourceFileName as SourceFileName OUTPUT is_sensitive 
| where is_sensitive="true"
```

---

### **C. Azure AD Anomalies (T1078)**
#### **Impossible Travel**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| iplocation ipAddress 
| stats earliest(_time) as first_login, latest(_time) as last_login by userPrincipalName, Country 
| eval travel_time=last_login-first_login 
| where travel_time < 3600 AND Country!="Unknown"
```

#### **Risky Sign-ins**
```spl
index=azure_logs sourcetype=azure:aad:signin riskState="atRisk" 
| table _time, userPrincipalName, riskDetail, ipAddress
```

---

### **D. Azure Resource Misconfigurations (T1530)**
#### **Public Storage Accounts**
```spl
index=azure_logs sourcetype=azure:activity operationName="Microsoft.Storage/storageAccounts/write" 
| search requestBody.properties.networkAcls.defaultAction="Allow" 
| table _time, caller, requestBody.name
```

#### **Overprivileged Service Principals**
```spl
index=azure_logs sourcetype=azure:activity operationName="Microsoft.Authorization/roleAssignments/write" 
| search requestBody.properties.roleDefinitionId="*Owner*" 
| table _time, caller, requestBody.principalId
```

---

## **3. Advanced Correlation Techniques**

### **A. Cross-Platform Threat Hunting**
```spl
index=azure_logs sourcetype=azure:aad:signin riskState="atRisk" 
| join userPrincipalName 
    [search index=o365_logs Operation="Send" 
    | stats count by UserId] 
| table _time, userPrincipalName, riskDetail, count
```

### **B. Threat Intel Enrichment**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| lookup threat_intel_ip ipAddress as ipAddress OUTPUT is_malicious 
| where is_malicious="true" 
| table _time, userPrincipalName, appDisplayName, ipAddress
```

### **C. Time-Based Anomaly Detection**
```spl
index=o365_logs Operation="FileDownloaded" 
| timechart span=1h count by UserId 
| anomalydetection count 
| where 'anomaly_detection(count)' > 0
```

---

## **4. Splunk Dashboard Examples**

### **Microsoft 365 Security Dashboard**
**Essential Panels:**
1. **Risky Sign-ins** (`riskState="atRisk" | stats count by riskDetail`)
2. **Email Forwarding Rules** (`Operation="Set-Mailbox" | top Parameters`)
3. **SharePoint File Activity** (`Operation="File*" | timechart count`)
4. **Azure RBAC Changes** (`operationName=*roleAssignment* | stats count`)

### **Azure AD Protection View**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| stats count by riskState, userPrincipalName 
| sort - count
```

---

## **5. Best Practices**

 **Enable All Diagnostic Settings**  
   - Azure AD: SignIn + Audit logs  
   - Office 365: Unified Audit Log  

 **Normalize Key Fields**  
   - Standardize `user`, `src_ip` across Azure and Office 365  

 **Baseline Normal Behavior**  
   - Whitelist corporate IP ranges  
   - Establish typical file access patterns  

 **Correlate with Endpoint Logs**  
   - Join Azure AD sign-ins with Defender ATP alerts  

---

## **Final Thoughts**
Microsoft 365/Azure logs in Splunk enable detection of:
 **Email compromises** (malicious rules, mass sending)  
 **Cloud data exfiltration** (SharePoint, OneDrive)  
 **Identity attacks** (risky sign-ins, impossible travel)  
 **Cloud misconfigurations** (public storage, excessive permissions)  
