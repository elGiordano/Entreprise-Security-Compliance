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

