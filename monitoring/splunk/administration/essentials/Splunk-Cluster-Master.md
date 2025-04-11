## **Splunk Cluster Master** configuration and management:

---

## **Splunk Indexer Cluster Master: Core Architecture**

### **1. Cluster Master Roles**
| Role | Purpose | Key Processes |
|------|---------|--------------|
| **Manager Node** | Coordinates peer nodes | `clustering` |
| **License Master** | Distributes licenses | `license_owner` |
| **Monitoring Console** | Cluster health dashboard | `splunkd` |

### **2. Election Process**
- **Automatic failover** (when using multiple managers)
- **Raft consensus protocol** for leader election
- **Quorum** = (N/2)+1 (e.g., 3-node cluster needs 2 healthy nodes)

---

## **Configuration Steps**

### **1. Initialize Cluster Master**
```bash
# On designated master node
/opt/splunk/bin/splunk edit cluster-config -mode master \
-replication_factor 2 \
-search_factor 2 \
-secret your_cluster_secret \
-auth admin:yourpassword
```

### **2. Configure Peer Nodes**
```bash
# On each peer node
/opt/splunk/bin/splunk edit cluster-config -mode peer \
-manager_uri https://<cluster_master>:8089 \
-secret your_cluster_secret \
-auth admin:yourpassword
```

### **3. Set Replication Factor**
```ini
# In $SPLUNK_HOME/etc/system/local/server.conf
[indexer_cluster]
replication_factor = 2
search_factor = 2
```

---

## **Key Administrative Commands**

### **1. Cluster Status Checks**
```bash
# Show cluster health
/opt/splunk/bin/splunk show cluster-status -auth admin:yourpassword

# List all peers
/opt/splunk/bin/splunk list cluster-peers -auth admin:yourpassword

# Check bucket replication
/opt/splunk/bin/splunk list cluster-buckets -auth admin:yourpassword
```

### **2. Maintenance Operations**
```bash
# Decommission a peer
/opt/splunk/bin/splunk offline cluster-peer -peer <peer_name>:8089 -auth admin:yourpassword

# Force bucket repair
/opt/splunk/bin/splunk repair cluster-buckets -auth admin:yourpassword
```

---

## **Advanced Configuration**

### **1. Multi-Site Clustering**
```ini
# In server.conf on master
[general]
site = site1

[clustering]
multisite = true
available_sites = site1,site2
site_replication_factor = origin:2,total:3
```

### **2. Storage Optimization**
```ini
# SmartStore configuration (S3-backed)
[indexer_cluster]
smartstore = s3
remote.s3.access_key = AKIA...
remote.s3.secret_key = ...
remote.s3.bucket = your-splunk-bucket
```

### **3. Security Hardening**
```bash
# Enable TLS for intra-cluster comms
/opt/splunk/bin/splunk edit cluster-config -ssl_cert_path /opt/splunk/etc/auth/cert.pem \
-ssl_password yourpassword \
-auth admin:yourpassword
```

---

## **Troubleshooting Guide**

### **1. Common Issues**
| Symptom | Diagnostic Command | Solution |
|---------|--------------------|----------|
| Peer offline | `splunk search 'index=_internal "peer heartbeat failed"'` | Check network/firewall |
| Bucket imbalance | `splunk list cluster-buckets -status` | Run repair operation |
| License violations | `splunk list license-usage` | Add more peer nodes |

### **2. Critical Logs**
```
/opt/splunk/var/log/splunk/clustering.log
/opt/splunk/var/log/splunk/splunkd.log
```

### **3. Monitoring Metrics**
```sql
index=_internal sourcetype=splunkd component=clustering
| timechart span=1h count by log_level
```

---

## **Best Practices**
1. **Hardware Sizing**:
   - Minimum: 8 CPU cores, 16GB RAM
   - Recommended: 16+ CPU cores, 32GB RAM for production

2. **Network Configuration**:
   - 10Gbps+ between peers
   - <1ms latency for optimal performance

3. **Backup Strategy**:
   ```bash
   # Backup cluster configs
   tar -czvf cluster_backup.tar.gz $SPLUNK_HOME/etc/system/local/{server,indexes}.conf
   ```

4. **Disaster Recovery**:
   - Maintain **hot standby master**
   - Regular `bucketfly` checks for data integrity

---

## **Cluster Master vs. Search Head Cluster**
| Feature | Indexer Cluster Master | Search Head Cluster |
|---------|------------------------|---------------------|
| **Purpose** | Data storage/replication | Search coordination |
| **Failover** | Automatic peer promotion | Captain election |
| **Config** | `server.conf` | `serverclass.conf` |
| **Scale Limit** | 100+ peers | 19 search heads max |

🛠️
