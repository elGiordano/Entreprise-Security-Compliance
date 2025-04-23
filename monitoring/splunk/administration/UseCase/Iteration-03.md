 integrate Splunk SmartStore into your multi-site architecture with federated search:

# Splunk Architecture with SmartStore and Federated Search

```
                   Site A (Primary)                     Site B (DR)                     Site C (DR)
                  ┌─────────────────┐                ┌─────────────────┐                ┌─────────────────┐
                  │                 │                │                 │                │                 │
                  │  Indexer Peer 1 │◄───REPLICATION─┤  Indexer Peer 2 │◄───REPLICATION─┤  Indexer Peer 3 │
                  │  Search Head 1  │                │  Search Head 2  │                │  Search Head 3  │
                  │  Cluster Master │                │                 │                │                 │
                  │  Deployment Srvr│                │                 │                │                 │
                  │  Federated Srvr │                │                 │                │                 │
                  └────────┬────────┘                └────────┬────────┘                └────────┬────────┘
                           │                                  │                                  │
                           ▼                                  ▼                                  ▼
                  ┌─────────────────┐                ┌─────────────────┐                ┌─────────────────┐
                  │  Shared Object  │                │  Shared Object  │                │  Shared Object  │
                  │  Storage (S3)  │◄─────DR SYNC───┤  Storage (S3)  │◄─────DR SYNC───┤  Storage (S3)  │
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

                  FEDERATED SEARCH CONNECTIONS:
                  Site A Search Head ──────► Can query data in Site B and Site C
                  Site B Search Head ──────► Can query data in Site A (read-only)
                  Site C Search Head ──────► Can query data in Site A (read-only)
```

## SmartStore Implementation Details

1. **Storage Architecture**:
   - Each site has its own S3-compatible object storage bucket
   - DR sync between storage buckets (configurable frequency)
   - Local cache on each indexer (typically 10-30% of total data)

2. **Key Configuration (indexes.conf)**:
```ini
[default]
remotePath = volume:remote_store/$_index_name
repFactor = auto
homePath = $SPLUNK_DB/$_index_name/db
coldPath = $SPLUNK_DB/$_index_name/colddb
thawedPath = $SPLUNK_DB/$_index_name/thaweddb

[volume:remote_store]
storageType = remote
path = s3://your-bucket-name/splunk
remote.s3.access_key = YOUR_ACCESS_KEY
remote.s3.secret_key = YOUR_SECRET_KEY
remote.s3.endpoint = https://s3-endpoint.example.com
```

3. **Disaster Recovery Setup**:
   - Primary site bucket is authoritative
   - DR sites can either:
     a) Use their local bucket copies during outage
     b) Failover to DR bucket replica if primary unavailable
   - Bucket sync frequency based on RPO requirements

4. **Performance Considerations**:
   - Configure cache sizing based on working set:
     ```ini
     [smartstore]
     cache_size = 20GB  # per indexer
     ```
   - Enable SSD caching if available:
     ```ini
     cachePath = /mnt/ssd_cache/$_index_name
     ```

5. **Federated Search Integration**:
   - SmartStore buckets appear as normal indexes to federated search
   - Search heads don't need special SmartStore configuration
   - Queries automatically resolve to cached or remote data

## Recommended Deployment Strategy

1. **Phased Rollout**:
   - Start with non-critical indexes in SmartStore
   - Monitor cache hit rates and query performance
   - Gradually migrate more indexes

2. **Monitoring**:
   - Track cache efficiency metrics
   - Monitor S3 API costs and latency
   - Alert on cache misses impacting searches

3. **Maintenance**:
   - Regular bucket lifecycle management
   - Cache warming for predictable workloads
   - Periodic integrity checks

---

Here are specific configuration examples and optimizations for our Splunk SmartStore implementation with federated search:

---

### **1. SmartStore Detailed Configuration Examples**

#### **indexes.conf (Primary Site)**
```ini
# Global SmartStore Settings
[default]
remotePath = volume:primary_store/$_index_name
repFactor = auto
homePath = $SPLUNK_DB/$_index_name/db
coldPath = $SPLUNK_DB/$_index_name/colddb
thawedPath = $SPLUNK_DB/$_index_name/thaweddb
maxHotBuckets = 10
maxDataSize = auto_high_volume

# S3 Storage Volume Definition
[volume:primary_store]
storageType = remote
path = s3://primary-site-bucket/splunk
remote.s3.access_key = AKIAXXXXXXXXXXXXXXXX
remote.s3.secret_key = ****************************************
remote.s3.endpoint = https://s3-us-west-2.amazonaws.com
remote.s3.ssl = true
```

#### **server.conf (Cache Configuration)**
```ini
[smartstore]
cache_size = 30GB  # 25-30% of total data volume
cachePath = /opt/splunk/cache/$_index_name
max_cache_data_size = 10GB  # per bucket
```

---

### **2. DR Site SmartStore Configuration**

#### **indexes.conf (DR Site B)**
```ini
[volume:dr_store]
storageType = remote
path = s3://dr-site-bucket/splunk
remote.s3.access_key = AKIAXXXXXXXXXXXXXXXX
remote.s3.secret_key = ****************************************
remote.s3.endpoint = https://s3-eu-west-1.amazonaws.com
remote.s3.ssl = true
```

#### **DR Sync Automation (AWS CLI Example)**
```bash
# Daily bucket sync from Primary to DR sites
aws s3 sync s3://primary-site-bucket/splunk s3://dr-site-bucket/splunk \
  --delete \
  --exclude "*.hotbucket" \
  --profile splunk_sync
```

---

### **3. Federated Search Optimization with SmartStore**

#### **federated_provider.conf (On Search Head 1)**
```ini
[federated_provider:site_b]
host = search_head_2.site_b.example.com
port = 8089
authType = basic
username = federated_user
password = $7$XXXXXXXXXXXXXXXXXXXXXXXXXX
remoteIndexes = main,security,network
autoDiscover = false

# SmartStore-specific timeout settings
searchTimeout = 120
connectionTimeout = 30
maxParallelSearches = 5
```

#### **SmartStore-Aware Federated Search SPL Example**
```sql
index=*:site_b:security OR index=*:site_c:network 
| stats count by src_ip, dest_port
| where count > 100
```

---

### **4. Performance Tuning Recommendations**

1. **Cache Optimization**:
   ```ini
   [smartstore]
   hotlist_recency_interval = 3600  # Keep recent 1 hour data always cached
   hotlist_bloom_filter = true      # Reduce cache lookup overhead
   ```

2. **Network Optimization**:
   ```ini
   [splunktcp-ssl]
   maxReceiveKB = 8192
   [http]
   max_connections = 50
   ```

3. **Indexer Performance**:
   ```ini
   [indexing]
   mempool_max = 512MB
   max_concurrent_optimizes = 6
   ```

---

### **5. Monitoring & Maintenance Scripts**

#### **SmartStore Health Check (run on each indexer)**
```bash
#!/bin/bash
# Check cache utilization
curl -s -k -u admin:password https://localhost:8089/services/admin/cacheman?output_mode=json | jq '.entry[].content'

# Verify bucket integrity
splunk check-integrity -index main -bucket 202405
```

#### **Federated Search Monitoring (run on search heads)**
```sql
| rest /services/search/jobs 
| search is_federated=1 
| stats avg(run_time) as avg_time, count by remote_host 
| where avg_time > 30
```

---

### **6. Disaster Recovery Procedures**

**Failover Scenario (When Primary Site Goes Down):**
1. Promote DR Site B's bucket to primary:
   ```bash
   aws s3 cp s3://dr-site-bucket/splunk/ s3://primary-site-bucket/splunk/ --recursive
   ```
2. Update federated search priorities:
   ```ini
   [federated_provider:site_a]
   priority = 2  # Demote primary site
   [federated_provider:site_b] 
   priority = 1  # Promote DR site
   ```

---

### **Key Recommendations**
1. **For Cold Data**: Set higher `remotePath` priority for older indexes
   ```ini
   [volume:archive_store]
   priority = 100
   ```

2. **For Security Data**: Keep hot buckets local longer
   ```ini
   [security]
   frozenTimePeriodInSecs = 2592000  # 30 days
   ```

3. **For High-Value Indexes**: Implement tiered storage
   ```ini
   [financial_data]
   remotePath = volume:premium_store/$_index_name
   [volume:premium_store]
   storageClass = INTELLIGENT_TIERING  # AWS-specific
   ```

