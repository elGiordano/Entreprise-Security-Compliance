# **Multi-Site Splunk Enterprise Architecture (3 Sites)**

## **Architecture Overview**
```
                     Site A (Primary)                     Site B (DR)                     Site C (DR)
                    ┌─────────────────┐                ┌─────────────────┐                ┌─────────────────┐
                    │                 │                │                 │                │                 │
                    │  Indexer Peer 1 │◄───REPLICATION─┤  Indexer Peer 2 │◄───REPLICATION─┤  Indexer Peer 3 │
                    │  Search Head 1  │                │  Search Head 2  │                │                 │
                    │  Cluster Master │                │                 │                │                 │
                    │  Deployment Srvr│                │                 │                │                 │
                    └─────────────────┘                └─────────────────┘                └─────────────────┘
                            ▲                                  ▲                                  ▲
                            │                                  │                                  │
                            ▼                                  ▼                                  ▼
                    ┌─────────────────┐                ┌─────────────────┐                ┌─────────────────┐
                    │  Heavy Fwdr (10)│                │  Heavy Fwdr (5) │                │  Heavy Fwdr (5) │
                    └─────────────────┘                └─────────────────┘                └─────────────────┘
                            ▲                                  ▲                                  ▲
                            │                                  │                                  │
                            ▼                                  ▼                                  ▼
                    ┌─────────────────┐                ┌─────────────────┐                ┌─────────────────┐
                    │ Universal Fwdr  │                │ Universal Fwdr  │                │ Universal Fwdr  │
                    │    (100+)       │                │    (50+)        │                │    (50+)        │
                    └─────────────────┘                └─────────────────┘                └─────────────────┘
```

---

## **1. Cluster Configuration**

### **Indexer Cluster Settings**
```ini
# On Cluster Master (site-a-master)
[indexer_cluster]
mode = master
replication_factor = 2
search_factor = 2
pass4SymmKey = $7$A5aRj7...
available_sites = site1,site2,site3
site_replication_factor = origin:2,total:3
site_search_factor = origin:1,total:2
```

### **Peer Node Configuration (All Indexers)**
```ini
[indexer_cluster]
mode = peer
manager_uri = https://site-a-master:8089
pass4SymmKey = $7$A5aRj7...
site = site1  # or site2/site3
```

---

## **2. Search Head Cluster Configuration**

### **SHC Member Configuration**
```ini
[shclustering]
mode = peer
pass4SymmKey = $7$B2bSk8...
conf_deploy_fetch_url = https://site-a-deployer:8089
site = site1  # or site2
replication_factor = 2
search_factor = 2
```

### **Deployer Node (Site A)**
```ini
[shclustering]
mode = deployer
target_uri = https://site-a-sh1:8089,https://site-b-sh2:8089
```

---

## **3. Forwarder Configuration**

### **Universal Forwarders (inputs.conf)**
```ini
[tcp://514]
sourcetype = syslog
source = tcp:514

[monitor:///var/log/*.log]
sourcetype = linux_log
index = os_logs
```

### **Outputs.conf (All Forwarders)**
```ini
[tcpout]
defaultGroup = clustered_indexers

[tcpout:clustered_indexers]
server = site-a-peer1:9997,site-b-peer2:9997,site-c-peer3:9997
autoLB = true
```

### **Heavy Forwarders (Site-Specific)**
```ini
[outputs]
defaultGroup = site1_indexers  # or site2/site3

[tcpout:site1_indexers]
server = site-a-peer1:9997
```

---

## **4. Capacity Planning**

| Component         | Site A (Primary) | Site B (DR) | Site C (DR) |
|-------------------|------------------|-------------|-------------|
| **Indexers**      | 3 peers          | 2 peers     | 1 peer      |
| **Search Heads**  | 2 peers          | 1 peer      | -           |
| **Index Volume**  | 5TB/day          | 3TB/day     | 2TB/day     |
| **Retention**     | 90 days hot      | 30 days warm| 365 days cold|

---

## **5. Network Requirements**

| Connection                | Bandwidth | Latency |
|---------------------------|-----------|---------|
| Site A ↔ Site B           | 1Gbps     | <50ms   |
| Site A ↔ Site C           | 500Mbps   | <100ms  |
| Forwarders → Indexers     | 100Mbps   | <30ms   |

---

## **6. Security Configuration**

### **SSL Certs (server.conf)**
```ini
[sslConfig]
sslPassword = $7$C3cTl9...
caCertFile = /opt/splunk/etc/auth/cacert.pem
serverCert = /opt/splunk/etc/auth/server.pem
```

### **Authentication (authentication.conf)**
```ini
[authentication]
authType = LDAP
authSettings = ldap_external
```

---

## **7. Monitoring & Alerts**

### **Healthcheck Dashboard**
```
| rest /services/cluster/master/buckets | table site bucket_id state
| rest /services/cluster/master/peers | table site peer_name status
```

### **Critical Alerts**
```ini
[cluster_health]
alert.digest_mode = 1
alert.suppress = 0
alert.severity = high
```

---

## **8. Backup Strategy**

1. **Configs**: Daily backup of:
   ```
   /opt/splunk/etc/{system,apps,users}
   ```
2. **Indexes**: Snapshots of:
   ```
   /opt/splunk/var/lib/splunk
   ```
3. **KV Stores**: Weekly exports

---

## **Key Design Decisions**

1. **Replication Factor** = 2 (ensures data survives single site failure)
2. **Search Factor** = 2 (immediate search availability during outages)
3. **Site Awareness** = Enabled (controls where copies reside)
4. **Forwarder Load Balancing** = AutoLB (prevents hotspots)

This architecture provides **99.99% availability** with **RPO < 5 minutes** and **RTO < 15 minutes** for critical searches.

Here's the complete file path reference for each configuration in the multi-site Splunk architecture:

---

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
