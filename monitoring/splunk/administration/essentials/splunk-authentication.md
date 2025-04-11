## authentication method and hardening technique:

---

## **Splunk Authentication Methods and Hardening**

### **1. Built-in Authentication**
**Config File**: `$SPLUNK_HOME/etc/system/local/authentication.conf`  
**CLI Alternative**:
```bash
splunk add user <username> -password <password> -role <role> -auth admin:<password>
```

### **2. LDAP/Active Directory Integration**
**Config File**: `$SPLUNK_HOME/etc/system/local/authentication.conf`  
**CLI Alternative**:
```bash
splunk edit user <username> -auth-type LDAP -roles <roles> -auth admin:<password>
```

### **3. SAML (Enterprise)**
**Config File**: `$SPLUNK_HOME/etc/system/local/authentication.conf`  
**CLI Alternative** (limited):
```bash
splunk http-auth -auth-type saml -saml-entityid <entityID> -auth admin:<password>
```

### **4. RADIUS Authentication**
**Config File**: `$SPLUNK_HOME/etc/system/local/authentication.conf`  
**CLI Alternative**:
```bash
splunk edit authentication -auth-type RADIUS -radius-server <server:port> -auth admin:<password>
```

### **5. PKI/Certificate-Based**
**Config File**: `$SPLUNK_HOME/etc/system/local/server.conf`  
**CLI Alternative**:
```bash
splunk edit ssl-client-auth -enable 1 -cert-path <path> -auth admin:<password>
```

---

## **Authentication Hardening Guide**

### **1. Password Policies**
**Config File**: `$SPLUNK_HOME/etc/system/local/authentication.conf`  
**CLI Alternative**:
```bash
splunk edit authentication -password-policy strict -min-password-length 12 -auth admin:<password>
```

### **2. Session Management**
**Config File**: `$SPLUNK_HOME/etc/system/local/web.conf`  
**CLI Alternative**:
```bash
splunk edit web -session-timeout 4h -secure-cookie true -auth admin:<password>
```

### **3. Network Restrictions**
**Config File**: `$SPLUNK_HOME/etc/system/local/web.conf`  
**CLI Alternative**:
```bash
splunk edit web -allow-basic-auth false -trusted-ip "10.0.0.0/8,192.168.1.100" -auth admin:<password>
```



### **5. Audit Logging**
**Config File**: `$SPLUNK_HOME/etc/system/local/audit.conf`  
**CLI Alternative**:
```bash
splunk edit audit -enable 1 -audit-trail $SPLUNK_HOME/var/log/splunk/audit.log -auth admin:<password>
```

---

## **Enterprise Security Add-ons**

### **1. Multi-Factor Authentication (MFA)**
**Config File**: `$SPLUNK_HOME/etc/apps/<MFA_app>/local/authentication.conf`  
**CLI Alternative** (app-specific):
```bash
splunk install app <mfa_app.tgz> -auth admin:<password>
```

### **2. Certificate Pinning**
**Config File**: `$SPLUNK_HOME/etc/system/local/web.conf`  
**CLI Alternative**:
```bash
splunk edit web -cert-fingerprint "SHA256:xx:xx..." -auth admin:<password>
```

---

## **Key File Locations**
| Purpose | Primary Location | Secondary Location |
|---------|------------------|--------------------|
| Authentication | `$SPLUNK_HOME/etc/system/local/authentication.conf` | `$SPLUNK_HOME/etc/apps/<app>/local/authentication.conf` |
| Web Settings | `$SPLUNK_HOME/etc/system/local/web.conf` | `$SPLUNK_HOME/etc/apps/<app>/local/web.conf` |
| Authorization | `$SPLUNK_HOME/etc/system/local/authorize.conf` | `$SPLUNK_HOME/etc/users/<user>/local/authorize.conf` |
| Audit Logs | `$SPLUNK_HOME/etc/system/local/audit.conf` | `$SPLUNK_HOME/var/log/splunk/audit.log` |

---

## **Important Notes**
1. **Configuration Precedence**:
   - App-specific configs override system defaults
   - Changes via CLI automatically update corresponding config files

2. **Backup Configs**:
   ```bash
   tar -czvf splunk_auth_configs.tar.gz $SPLUNK_HOME/etc/system/local/{authentication,web,authorize}.conf
   ```

3. **Apply Changes**:
   ```bash
   splunk restart -auth admin:<password>
   ```

4. **Verification**:
   ```bash
   splunk btool authentication list --debug
   splunk btool web list --debug
   ```

