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


