 **SAML authentication** for our Splunk users while maintaining compatibility with our existing LDAP setup and multi-site architecture:

---

### **1. Prerequisites**
1. **Identity Provider (IdP) Metadata** (e.g., ADFS, Okta, Azure AD)
2. **SSL Certificates** for Splunk (same across all sites)
3. **Service Provider (SP) Entity ID**: `https://splunk.corp.example.com/saml/acs`

---

### **2. SAML Configuration (`authentication.conf`)**
#### **Primary Site (Site A)**
```ini
[authentication]
authType = SAML
saml_settings = saml_okta

[saml_okta]
entityId = https://splunk-primary.corp.example.com/saml/acs
idpSSOUrl = https://corp.okta.com/app/splunkprod/sso/saml
idpSLOUrl = https://corp.okta.com/app/splunkprod/slo/saml
issuerId = http://www.okta.com/123456
certPath = /opt/splunk/etc/auth/saml/okta_cert.pem
signatureAlgorithm = RSA-SHA256
signedAssertion = true
nameIDPolicyFormat = urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
samlResponseTimeout = 120
```

#### **DR Sites (Sites B & C)**
```ini
[saml_okta:dr]
entityId = https://splunk-dr.corp.example.com/saml/acs
idpSSOUrl = https://corp.okta.com/app/splunkdr/sso/saml
```

---

### **3. Role Mapping (`authorize.conf`)**
```ini
[roleMap_saml_okta]
group_Splunk_Admins = admin
group_Splunk_Security = security_auditor
group_Splunk_DR = dr_search_head

# Attribute from SAML assertion to map
attributeName = http://schemas.xmlsoap.org/claims/Group
```

---

### **4. IdP Configuration (Example: Azure AD)**
#### **Required Claims**
| Claim Type | Value |
|------------|-------|
| `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` | user.email |
| `http://schemas.xmlsoap.org/claims/Group` | user.groups |

#### **Azure AD Manifest Snippet**
```json
"requiredResourceAccess": [
    {
        "resourceAppId": "00000003-0000-0ff1-ce00-000000000000",
        "resourceAccess": [
            {
                "id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d",
                "type": "Scope"
            }
        ]
    }
]
```

---

### **5. Multi-Site SAML Considerations**
#### **Service Provider Settings**
```ini
[saml_okta:site_affinity]
siteA_entityId = https://splunk-primary.corp.example.com/saml/acs
siteB_entityId = https://splunk-dr1.corp.example.com/saml/acs
siteC_entityId = https://splunk-dr2.corp.example.com/saml/acs
sharedSecret = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

#### **DR Failover Procedure**
1. Update IdP metadata to point to DR site:
   ```bash
   splunk edit saml-config -authType SAML -idpSSOUrl https://corp.okta.com/app/splunkdr/sso/saml
   ```

---

### **6. Certificate Management**
```bash
# Generate SP certificate (run on primary)
openssl req -x509 -newkey rsa:2048 -keyout sp_key.pem -out sp_cert.pem -days 365 -nodes -subj "/CN=splunk.corp.example.com"

# Deploy to all sites
scp sp_*.pem splunk@dr-site1:/opt/splunk/etc/auth/saml/
```

---

### **7. SAML-Enabled Federated Search**
```ini
[federated_provider:siteB]
authType = SAML
samlIdP = saml_okta
requireGroup = group_Splunk_CrossSite_Users
```

---

### **8. Verification Commands**
```bash
# Test SAML config
splunk check saml-config -authType SAML

# Validate metadata
curl -k https://splunk-primary.corp.example.com/saml/metadata -o sp_metadata.xml
```

---

### **9. Emergency Bypass**
```ini
# passwords.conf
[user:breakglass_saml_admin]
password = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
roles = admin
samlExclude = true
```

---

### **10. Monitoring SPL Queries**
```sql
| audit action=login authType=SAML 
| stats count by user, src_ip, status 
| where status="failure"
```

---

### **Key Recommendations**
1. **Use Persistent NameID**: Ensures consistent user mapping during failovers
2. **Enable SLO**: Configure Single Logout in `web.conf`:
   ```ini
   [saml]
   sloEnabled = true
   ```
3. **DR Site Testing**: Validate SAML with:
   ```bash
   splunk test saml-auth -username testuser@corp.example.com -authMgr saml_okta
   ```

---

 **IdP-specific configuration guides** for SAML authentication in our Splunk environment, covering Okta, Azure AD, and ADFS:

---

### **1. Okta Configuration**
#### **Step 1: Create Splunk App in Okta**
1. Navigate to **Applications → Add Application → Create New App**
2. Select:
   - **Platform**: Web
   - **Sign-on method**: SAML 2.0

#### **Step 2: Configure SAML Settings**
```xml
<!-- Okta SAML Settings -->
Single sign-on URL: https://splunk-primary.corp.example.com/saml/acs
Audience URI (SP Entity ID): https://splunk.corp.example.com/saml/acs
Name ID Format: EmailAddress
Attribute Statements:
  - Name: groups
    Value: user.groups
    Filter: Regex (.*)
```

#### **Step 3: Assign Groups**
```powershell
# PowerShell to verify Okta group assignments
Get-OktaGroup -Name "Splunk_*" | Get-OktaGroupMember
```

#### **Step 4: Download Metadata**
```bash
wget https://corp.okta.com/app/splunkprod/sso/saml/metadata -O okta_metadata.xml
```

---

### **2. Azure AD Configuration**
#### **Step 1: Enterprise App Registration
1. Azure Portal → **Enterprise Applications → New application → Non-gallery application**
2. Set **Name**: Splunk Production

#### **Step 2: Configure SAML**
```xml
<!-- Azure AD SAML Settings -->
Identifier (Entity ID): https://splunk.corp.example.com/saml/acs
Reply URL: https://splunk-primary.corp.example.com/saml/acs
Logout URL: https://splunk-primary.corp.example.com/saml/logout
Claims:
  - Unique User Identifier: user.mail
  - Groups: user.groups
```

#### **Step 3: Group Claims**
```json
// Azure AD Manifest Addition
"groupMembershipClaims": "SecurityGroup",
"optionalClaims": {
    "idToken": [
        {
            "name": "groups",
            "source": null,
            "essential": false,
            "additionalProperties": []
        }
    ]
}
```

#### **Step 4: Certificate Management**
```powershell
# Renew SP certificate
New-AzureADApplicationKeyCredential -ObjectId <app_id> -Type AsymmetricX509Cert -Usage Verify -Value $(Get-Content -Encoding byte -Path sp_cert.pem)
```

---

### **3. ADFS Configuration**
#### **Step 1: Relying Party Trust**
```powershell
# PowerShell to create trust
Add-AdfsRelyingPartyTrust `
    -Name "Splunk Production" `
    -Identifier "https://splunk.corp.example.com/saml/acs" `
    -WSFedEndpoint "https://splunk-primary.corp.example.com/saml/acs" `
    -IssuanceTransformRulesFile 'C:\config\splunk_claims_rules.txt'
```

#### **Step 2: Claim Rules**
```text
@RuleTemplate = "LdapClaims"
@RuleName = "Splunk Groups"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
=> issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/claims/Group"), query = "tokenGroups(!(objectClass=computer));", param = "http://schemas.xmlsoap.org/claims/Group");
```

#### **Step 3: Metadata Export**
```powershell
Export-AdfsAuthenticationProviderConfiguration -Name "Splunk" -FilePath "C:\metadata\splunk_adfs_metadata.xml"
```

---

### **4. Splunk Configuration for All IdPs**
#### **authentication.conf**
```ini
[saml_common]
entityId = https://splunk.corp.example.com/saml/acs
signedAssertion = true
signatureAlgorithm = RSA-SHA256
nameIDPolicyFormat = urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
```

#### **web.conf**
```ini
[saml]
spNameIdFormat = urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
sloEnabled = true
strictAudienceCheck = true
```

---

### **5. IdP-Specific Troubleshooting**
#### **Okta Debugging**
```bash
splunk cmd diag --saml --idp okta --debug 3
```

#### **Azure AD Logs**
```kusto
AzureADAuditLogs
| where AppDisplayName == "Splunk Production"
| project TimeGenerated, ResultType, ResultDescription
```

#### **ADFS Event Logs**
```powershell
Get-WinEvent -LogName "AD FS/Admin" -MaxEvents 100 | Where-Object {$_.Message -like "*Splunk*"}
```

---

### **6. DR Site Failover Procedures**
#### **Okta**
1. Update alternate SP URLs in Okta app:
   ```text
   https://splunk-dr1.corp.example.com/saml/acs
   https://splunk-dr2.corp.example.com/saml/acs
   ```

#### **Azure AD**
```powershell
Set-AzureADApplication -ObjectId <app_id> `
    -ReplyUrls @("https://splunk-primary.corp.example.com/saml/acs",
                 "https://splunk-dr1.corp.example.com/saml/acs")
```

#### **ADFS**
```powershell
Set-AdfsRelyingPartyTrust -TargetName "Splunk Production" `
    -AdditionalWSFedEndpoint "https://splunk-dr1.corp.example.com/saml/acs"
```

---

### **7. Recommended Monitoring**
#### **SAML Login Dashboard**
```sql
index=_internal source=*saml* 
| stats count by user, idp, status 
| timechart span=1h count by status
```

#### **Certificate Expiry Alert**
```powershell
# PowerShell for cert checks
$cert = Get-PfxCertificate -FilePath C:\saml\sp_cert.pem
if ($cert.NotAfter -lt (Get-Date).AddDays(30)) {
    Send-MailMessage -To "splunk-admin@corp.example.com" -Subject "SAML Cert Expiry Warning"
}
```

---

### **Key Security Recommendations**
1. **Enable AlwaysRequireAuthentication** in `web.conf`:
   ```ini
   [saml]
   forceAuthn = true
   ```
2. **Implement IP-based restrictions** at IdP level
3. **Rotate certificates** every 6 months


---

