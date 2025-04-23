 **LDAP authentication** for our Splunk users while maintaining the role-based access control in our multi-site architecture:

---

### **1. LDAP Server Configuration** (`authentication.conf`)
```ini
[authentication]
authType = LDAP
authSettings = ldap_splunk_prod

[ldap_splunk_prod]
host = ldap.corp.example.com
port = 636
SSLEnabled = 1
bindDN = cn=splunk-auth,ou=service-accounts,dc=corp,dc=example,dc=com
bindDNpassword = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
userBaseDN = ou=users,dc=corp,dc=example,dc=com
userBaseFilter = (objectClass=person)
userNameAttribute = sAMAccountName
groupBaseDN = ou=splunk-groups,dc=corp,dc=example,dc=com
groupBaseFilter = (objectClass=group)
groupMemberAttribute = member
groupNameAttribute = cn
```

---

### **2. Role Mapping** (`authorize.conf`)
#### **Primary Site Mapping**
```ini
[roleMapping_ldap_splunk_prod]
group_Splunk_Cluster_Admins = cluster_admin
group_Splunk_SOC = security_auditor
group_Splunk_Federated_Admins = federated_admin
```

#### **DR Site Mapping**
```ini
group_Splunk_DR_Admins = emergency_admin
group_Splunk_DR_Users = dr_search_head
```

---

### **3. Service Account Fallback** (`passwords.conf`)
```ini
[user:svc_splunk_federated]
password = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
roles = federated_admin
ldapExclude = true  # Always use local auth for this account

[user:breakglass_admin]
password = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
roles = emergency_admin
ldapExclude = true
```

---

### **4. Multi-Site LDAP Deployment**
#### **Site A (Primary)**
```ini
[ldap_splunk_prod:siteA]
siteAffinity = primary
failoverServers = ldap-dr1.corp.example.com:636,ldap-dr2.corp.example.com:636
```

#### **Site B/C (DR)**
```ini
[ldap_splunk_prod:siteB]
siteAffinity = secondary
primaryServer = ldap.corp.example.com:636
cacheTTL = 3600  # Cache credentials for 1 hour if LDAP unavailable
```

---

### **5. LDAP-Enabled Federated Search**
```ini
[federated_provider:siteB]
authType = LDAP
ldapServer = ldap_splunk_prod
requireGroup = group_Splunk_CrossSite_Users
```

---

### **6. Verification Commands**
```bash
# Test LDAP connectivity
splunk search '| ldapsearch server="ldap_splunk_prod" search="(cn=admin-user)"' -auth admin:changeme

# Check user-role mapping
splunk list user-role -auth admin -user jdoe@corp.example.com
```

---

### **7. Security Hardening**
1. **Certificate Pinning**:
   ```ini
   [ldap_splunk_prod]
   sslCertPath = /opt/splunk/etc/auth/ldap_ca.pem
   sslVerify = true
   ```

2. **Rate Limiting**:
   ```ini
   [ldapSettings]
   maxFailedLogins = 3
   lockoutThreshold = 5
   ```

3. **Audit Logging**:
   ```sql
   | audit action=login authType=LDAP | stats count by user, src_ip, status
   ```

---

### **8. Emergency Bypass Procedure**
When LDAP is down:
1. **Activate local auth**:
   ```bash
   splunk edit authentication -authType Splunk -roleMap_ldap_splunk_prod::disabled 1
   ```
2. **Use breakglass account**:
   ```bash
   splunk login -auth breakglass_admin:<password>
   ```

---

### **Implementation Checklist**
1. [ ] Deploy identical `authentication.conf` to all search heads
2. [ ] Sync LDAP CA certs to all Splunk instances
3. [ ] Test failover with `ldapsearch` commands
4. [ ] Configure monitoring for LDAP health:
   ```sql
   | rest /services/authentication/providers | search title="ldap_splunk_prod"
   ```

---

 **Active Directory-specific configurations** for our Splunk LDAP integration, optimized for our multi-site architecture:

---

### **1. Active Directory Schema Preparation**

#### **Required AD Attributes**
```powershell
# PowerShell: Verify attributes exist
Get-ADObject -SearchBase (Get-ADRootDSE).schemaNamingContext -LDAPFilter "(name=user)" | Select-Object -ExpandProperty mayContain
```
Ensure these attributes are populated:
- `sAMAccountName` (username)
- `memberOf` (group membership)
- `userPrincipalName` (for UPN login)
- `department` (optional for role mapping)

---

### **2. AD Group Structure Example**
```ldif
dn: ou=splunk-groups,ou=security,dc=corp,dc=example,dc=com
objectClass: organizationalUnit

dn: cn=splunk_cluster_admins,ou=splunk-groups,ou=security,dc=corp,dc=example,dc=com
objectClass: group
member: cn=splunk-svc,ou=service-accounts,dc=corp,dc=example,dc=com
member: cn=jdoe,ou=users,dc=corp,dc=example,dc=com

dn: cn=splunk_dr_analysts,ou=splunk-groups,ou=security,dc=corp,dc=example,dc=com
objectClass: group
member: cn=bsmith,ou=users,dc=corp,dc=example,dc=com
```

---

### **3. Splunk-Specific AD Configuration**
#### **authentication.conf (AD-optimized)**
```ini
[ldap_splunk_prod]
# Connection
host = dc1.corp.example.com
port = 636
SSLEnabled = 1

# Service Account
bindDN = "CN=splunk-auth,OU=Service-Accounts,DC=corp,DC=example,DC=com"
bindDNpassword = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# User Lookup
userBaseDN = "OU=Users,DC=corp,DC=example,DC=com"
userNameAttribute = sAMAccountName
userBaseFilter = "(&(objectCategory=Person)(objectClass=User)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"

# Group Mapping
groupBaseDN = "OU=Splunk-Groups,OU=Security,DC=corp,DC=example,DC=com"
groupMemberAttribute = member
groupNameAttribute = cn
nestedGroups = 1  # Enable nested group resolution
```

---

### **4. Nested Group Handling**
For nested AD groups (e.g., GroupA → GroupB → User):
```ini
[ldap_splunk_prod]
nestedGroups = 1
maxNestingLevel = 5
expandNestedGroups = 1
```

---

### **5. Site-Specific AD Queries**
#### **Primary Site (Site A)**
```ini
[roleMapping_ldap_splunk_prod:primary]
group_Splunk_Cluster_Admins = cluster_admin
group_Splunk_Security_Team = security_auditor
```

#### **DR Site (Site B)**
```ini
[roleMapping_ldap_splunk_prod:dr]
group_Splunk_DR_Team = dr_search_head
group_Splunk_DR_Admins = emergency_admin
```

---

### **6. AD Certificate Management**
1. **Export CA Certificate**:
   ```powershell
   Export-Certificate -Cert (Get-ChildItem -Path Cert:\LocalMachine\Root\<CA_THUMBPRINT>) -FilePath C:\splunk_ldap_ca.cer
   ```
2. **Configure in Splunk**:
   ```ini
   [ldap_splunk_prod]
   sslCertPath = /opt/splunk/etc/auth/ldap_ca.pem
   sslVerify = 2  # Strict validation
   ```

---

### **7. AD User Login Formats**
Support multiple login styles:
```ini
[authentication]
usernameAttribute = sAMAccountName,userPrincipalName
```

Example logins:
- `jdoe` (sAMAccountName)
- `jdoe@corp.example.com` (UPN)
- `CORP\jdoe` (NetBIOS)

---

### **8. AD Account Lockout Prevention**
```ini
[ldap_splunk_prod]
passwordExpirationWarning = 15d
accountLockoutThreshold = 0  # Let AD handle lockouts
```

---

### **9. Verification Commands (AD-specific)**
```powershell
# Verify service account can read groups
Get-ADGroup -Identity "splunk_cluster_admins" -Properties member | Select-Object -ExpandProperty member

# Test user attributes
Get-ADUser -Identity jdoe -Properties memberOf | Select-Object sAMAccountName,userPrincipalName,memberOf
```

---

### **10. Emergency Access with AD**
```ini
# passwords.conf
[user:breakglass_ad_admin]
password = $7$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
roles = emergency_admin
ldapExclude = true
adExclude = true  # Bypass even if AD is configured
```

---

### **Key AD Best Practices**
1. **Use a dedicated OU** for Splunk service accounts and groups
2. **Enable AD auditing** for the bindDN account
3. **Set password policies**:
   ```powershell
   Set-ADDefaultDomainPasswordPolicy -Identity corp.example.com -LockoutThreshold 5 -LockoutDuration 00:30:00
   ```
4. **Site-aware LDAP servers**:
   ```ini
   [ldap_splunk_prod:siteA]
   host = dc1.corp.example.com  # Primary DC
   [ldap_splunk_prod:siteB] 
   host = dc2.dr.corp.example.com  # DR site DC
   ```
