**emergency access procedures** for our Splunk multi-site architecture, designed for failover scenarios while maintaining security:

---

### **1. Emergency Access Tiers**
#### **Tier 1: Limited DR Operations**
**When**: Primary site unavailable but DR sites functional  
**Access Needed**: Read-only to all data + limited writes to DR indexes  
**Roles Activated**:
```ini
[role:dr_limited]
importRoles = dr_search_head
write_access = [dr_indexes_only]
search_federated = enabled
```

#### **Tier 2: Full DR Failover**  
**When**: Primary site down >4 hours  
**Access Needed**: Full write to DR indexes + SmartStore reconfiguration  
**Roles Activated**:
```ini
[role:dr_full_admin]
importRoles = emergency_admin
replicate = enabled
rebuild = enabled
```

---

### **2. Break-Glass Authentication**
#### **Manual Override (All Sites)**
1. **On any search head**:
   ```bash
   # Activate emergency console (bypasses normal auth)
   splunk enable boot-start -auth admin:emergency_password_2024!
   ```
2. **For SmartStore reconfiguration**:
   ```ini
   # indexes.conf (DR Site B during failover)
   [volume:primary_store]
   path = s3://dr-site-bucket/splunk  # Point to DR bucket
   read_only = false  # Allow writes
   ```

#### **Automated Failover (Pre-configured)**
```bash
#!/bin/bash
# Failover script (runs on Cluster Master heartbeat failure)
if ! ping -c 3 cluster_master_primary; then
  splunk edit user dr_admin -roles emergency_admin -auth $(cat /opt/splunk/.emergency_creds)
  splunk edit cluster-config -mode slave -master_uri https://dr_search_head2:8089
  aws s3 sync s3://dr-site-bucket/splunk s3://primary-site-bucket/splunk --delete
fi
```

---

### **3. Network Isolation Procedures**
**When**: Suspicious activity detected  
**Actions**:
1. **Immediate quarantine**:
   ```bash
   # On Cluster Master
   splunk disable cluster-peers -all
   ```
2. **Activate isolated mode**:
   ```ini
   # server.conf
   [general]
   site = isolated
   [federated_search]
   disabled = true
   ```

---

### **4. SmartStore Emergency Access**
**Scenario**: Primary S3 bucket compromised  
**Procedure**:
1. **Rotate credentials**:
   ```bash
   splunk restart; sleep 30
   splunk edit s3-credentials -volume remote_store -access_key NEW_KEY -secret_key NEW_SECRET
   ```
2. **Failover to DR bucket**:
   ```ini
   # indexes.conf
   [volume:emergency_store]
   path = s3://emergency-backup-bucket/splunk
   ```

---

### **5. Forensic Preservation Steps**
1. **Freeze indexes**:
   ```bash
   splunk freeze -index compromised_data -auth admin:$(cat /etc/emergency.token)
   ```
2. **Create legal hold**:
   ```sql
   | export dump type=raw index=* _time=-24h@h 
     | outputlookup forensic_capture_$(now()).csv
   ```

---

### **6. Post-Recovery Audit**
**Required Checks**:
```sql
| audit action=* emergency_user=* 
| stats count by user, action, _time 
| where count > 5 OR action="delete"
```

**Automatic Reversion**:
```bash
splunk revert-to-good -auth admin:$(cat /opt/splunk/.sync_token)
```

---

### **Key Security Controls**
1. **Physical Safeguards**:
   - Emergency credentials stored in **HSM** or **vault** with 2-person rule
   - All break-glass actions generate **SMS alerts** to CISO

2. **Network Restrictions**:
   ```ini
   # server.conf
   [emergency]
   allowedIP = 10.10.10.0/24  # SOC jumpbox network only
   ```

3. **Time-bound Access**:
   ```bash
   # Auto-disable emergency accounts after 8 hours
   splunk edit user emergency_admin -roles none -expires $(date -d "+8 hours" +%s)
   ```

---

### **DR Drill Test Plan**
1. **Quarterly Failover Test**:
   ```bash
   # Simulate primary outage
   splunk disable server -target primary_cluster_master
   # Validate DR search capability
   curl -k https://dr_search_head:8089/services/search/jobs -d search="| metadata type=hosts"
   ```

2. **Post-Test Validation**:
   ```sql
   | rest /services/server/status/partitions 
   | where isPrimary="false" AND active="true"
   ```
