# **Okta & Azure AD Logs in Splunk: Enterprise SSO Security Monitoring**

Okta and Azure AD logs provide critical visibility into cloud identity and access management (IAM) events. When analyzed in Splunk, they reveal SSO anomalies, OAuth abuse, suspicious app integrations, and identity federation attacks.

---

## **1. Key Log Sources & Normalization**

### **A. Core Log Sources**
| **Platform**  | **Log Type**          | **Splunk Add-on**                     | **Key Fields**                          |
|--------------|-----------------------|--------------------------------------|----------------------------------------|
| **Okta**     | System Log (API)      | [TA-okta](https://splunkbase.splunk.com/app/3080/) | `actor.alternateId`, `client.ipAddress`, `eventType`, `target` |
| **Azure AD**  | Sign-In Logs         | [TA-microsoft-cloud](https://splunkbase.splunk.com/app/3110/) | `userPrincipalName`, `appDisplayName`, `ipAddress`, `riskState` |

### **B. Field Normalization (CIM Compliance)**
Map to **Splunk's Common Information Model**:
- `user` → `actor.alternateId` (Okta) / `userPrincipalName` (Azure AD)  
- `src_ip` → `client.ipAddress` (Okta) / `ipAddress` (Azure AD)  
- `app` → `client.userAgent.rawUserAgent` (Okta) / `appDisplayName` (Azure AD)  
- `status` → `outcome.result` (Okta) / `status.errorCode` (Azure AD)  

Example transformation:
```spl
index=okta_logs sourcetype=okta:system 
| eval user='actor.alternateId', src_ip='client.ipAddress' 
| stats count by user, src_ip, eventType
```

---

## **2. Critical Detection Use Cases**

### **A. SSO Anomalies (T1556.002)**
#### **Impossible Travel for SSO Sessions**
```spl
index=okta_logs eventType=user.session.start 
| iplocation client.ipAddress 
| stats earliest(_time) as first_login, latest(_time) as last_login by actor.alternateId, Country 
| eval travel_time=last_login-first_login 
| where travel_time < 3600 AND Country!="Unknown"
```

#### **Suspicious SAML Assertions**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| search "authenticationProtocol:saml" 
| stats count by userPrincipalName, ipAddress, appDisplayName 
| sort - count
```

---

### **B. OAuth Token Abuse (T1550.001)**
#### **Excessive OAuth Consent Grants**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| search "grantType:authorization_code" 
| stats dc(appDisplayName) as app_count by userPrincipalName 
| where app_count > 5 
| sort - app_count
```

#### **Overprivileged App Permissions**
```spl
index=okta_logs eventType=application.user_membership.add 
| search "target.alternateId" IN ("*mail.read*", "*files.readwrite*") 
| table _time, actor.alternateId, target.alternateId
```

---

### **C. Identity Federation Attacks (T1484.001)**
#### **Unexpected IdP Changes**
```spl
index=okta_logs eventType=system.idp.* 
| stats count by actor.alternateId, target.alternateId 
| sort - count
```

#### **Suspicious Azure AD Connect Syncs**
```spl
index=azure_logs sourcetype=azure:aad:provisioning 
| search "syncResult:export" 
| stats count by initiatedBy.userPrincipalName, modifiedProperties
```

---

### **D. MFA Bypass Attempts (T1556)**
#### **MFA Fatigue Attacks**
```spl
index=okta_logs eventType=user.mfa.factor* 
| stats count by actor.alternateId, client.ipAddress 
| where count > 3 
| sort - count
```

#### **SMS/Voice Phishing Patterns**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| search "mfaDetail.authMethod:phone" 
| stats count by userPrincipalName, ipAddress 
| where count > 5 
| sort - count
```

---

## **3. Advanced Correlation Techniques**

### **A. Okta + Azure AD Cross-Platform Analysis**
```spl
index=okta_logs eventType=user.session.start 
| join actor.alternateId [search index=azure_logs "userPrincipalName" 
| stats count by userPrincipalName] 
| table _time, actor.alternateId, client.ipAddress
```

### **B. Threat Intel Enrichment**
```spl
index=okta_logs 
| lookup threat_intel_ip client.ipAddress as client.ipAddress OUTPUT is_malicious 
| where is_malicious="true" 
| table _time, actor.alternateId, client.ipAddress, eventType
```

---

## **4. Splunk Dashboard Examples**

### **SSO Security Dashboard**
**Essential Panels:**
1. **Failed Logins by IP** (`eventType=user.authentication.auth* | stats count by client.ipAddress`)
2. **OAuth App Permissions** (`eventType=application.* | top target.alternateId`)
3. **MFA Attempt Trends** (`eventType=user.mfa.* | timechart count`)
4. **Geo-Anomalous Logins** (`iplocation | rare Country`)

### **Azure AD Identity Protection View**
```spl
index=azure_logs sourcetype=azure:aad:signin 
| stats count by riskState, userPrincipalName 
| sort - count
```

---

## **5. Best Practices**

 **Enable All Diagnostic Logs**  
   - Okta: System Log API (90-day retention recommended)  
   - Azure AD: SignIn + Audit + Provisioning logs  

 **Normalize Critical Fields**  
   - Standardize `user`, `src_ip`, `app` across platforms  

 **Baseline Normal Behavior**  
   - Whitelist corporate IP ranges for SSO  
   - Establish typical OAuth app usage patterns  

 **Correlate with Endpoint Logs**  
   - Join SSO events with EDR data (`user`→`username`)  

---

## **Final Thoughts**
Okta/Azure AD logs in Splunk enable detection of:
 **SSO hijacking** (impossible travel, SAML attacks)  
 **OAuth abuse** (overprivileged apps, consent phishing)  
 **Identity federation risks** (IdP configuration changes)  
 **MFA bypass attempts** (fatigue attacks, voice phishing)  
