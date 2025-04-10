
### **1. Indexer Cluster Configuration**

| File | Path | Node Type |
|------|------|-----------|
| `server.conf` (Cluster Master) | `$SPLUNK_HOME/etc/system/local/server.conf` | Cluster Master |
| `server.conf` (Peer Nodes) | `$SPLUNK_HOME/etc/system/local/server.conf` | All Indexer Peers |
| `indexes.conf` (Global) | `$SPLUNK_HOME/etc/master-apps/_cluster/local/indexes.conf` | Cluster Master |

---

### **2. Search Head Cluster Configuration**

| File | Path | Node Type |
|------|------|-----------|
| `server.conf` (SHC Members) | `$SPLUNK_HOME/etc/system/local/server.conf` | All Search Heads |
| `deploymentclient.conf` | `$SPLUNK_HOME/etc/system/local/deploymentclient.conf` | SHC Deployer |
| `distsearch.conf` | `$SPLUNK_HOME/etc/system/local/distsearch.conf` | All Search Heads |

---

### **3. Forwarder Configurations**

| File | Path | Node Type |
|------|------|-----------|
| `inputs.conf` | `$SPLUNK_HOME/etc/system/local/inputs.conf` | All Forwarders |
| `outputs.conf` | `$SPLUNK_HOME/etc/system/local/outputs.conf` | All Forwarders |
| `props.conf` (Heavy Forwarders) | `$SPLUNK_HOME/etc/apps/TA_custom/local/props.conf` | Heavy Forwarders Only |

---

### **4. Security Configurations**

| File | Path | Node Type |
|------|------|-----------|
| `authentication.conf` | `$SPLUNK_HOME/etc/system/local/authentication.conf` | All Nodes |
| `authorize.conf` | `$SPLUNK_HOME/etc/system/local/authorize.conf` | All Nodes |
| `web.conf` (SSL) | `$SPLUNK_HOME/etc/system/local/web.conf` | All Web-Enabled Nodes |

---

### **5. Monitoring & Alerting**

| File | Path | Node Type |
|------|------|-----------|
| `alert_actions.conf` | `$SPLUNK_HOME/etc/system/local/alert_actions.conf` | Monitoring Console |
| `savedsearches.conf` | `$SPLUNK_HOME/etc/apps/health_check/local/savedsearches.conf` | Search Heads |

---

### **6. App Configurations**

| File | Path | Purpose |
|------|------|---------|
| `serverclass.conf` | `$SPLUNK_HOME/etc/system/local/serverclass.conf` | Deployment Server |
| `limits.conf` | `$SPLUNK_HOME/etc/system/local/limits.conf` | All Nodes (Tuning) |

---

### **Key Path Notes:**
1. **`$SPLUNK_HOME`** typically resolves to:
   - `/opt/splunk` (Linux)
   - `C:\Program Files\Splunk` (Windows)

2. **Cluster Master** maintains authoritative copies in:
   ```
   $SPLUNK_HOME/etc/master-apps/
   $SPLUNK_HOME/etc/slave-apps/
   ```

3. **Forwarder configurations** are pushed from:
   ```
   $SPLUNK_HOME/etc/deployment-apps/  (On Deployment Server)
   ```

4. **Site-specific configs** use naming convention:
   ```
   $SPLUNK_HOME/etc/apps/site1_indexes/local/
   $SPLUNK_HOME/etc/apps/site2_indexes/local/
   ```

---

### **Example Directory Tree**
```
/opt/splunk/
├── etc/
│   ├── system/
│   │   └── local/          # Node-specific configs
│   ├── master-apps/        # Cluster Master managed
│   ├── slave-apps/         # Peer-replicated configs 
│   ├── apps/
│   │   ├── search_head/    # SHC-specific
│   │   └── site1_indexes/  # Site-aware configs
│   └── deployment-apps/    # Forwarder configs
└── var/
    └── lib/splunk/         # Index data storage
```

---

### **Critical Maintenance Paths**
1. **Certificate Store**:
   ```
   $SPLUNK_HOME/etc/auth/
   ```

2. **License File**:
   ```
   $SPLUNK_HOME/etc/licenses/enterprise/
   ```

3. **KV Store Backups**:
   ```
   $SPLUNK_HOME/var/lib/splunk/kvstorebackup/
   ```

This structure ensures proper separation of concerns while maintaining Splunk's operational requirements for multi-site deployments. Always validate paths against your specific Splunk version using:
```bash
splunk btool --list --debug
```
