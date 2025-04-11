## installing **Splunk Enterprise (Full Stack)** on an **AWS Ubuntu EC2 instance**, covering indexer, search head, and forwarder components:

---

## **Prerequisites**
1. **AWS Account** with an **Ubuntu EC2 Instance** (Recommended: `t3.xlarge` or larger, 50GB+ storage)
2. **Splunk Enterprise License** (Free trial available [here](https://www.splunk.com/en_us/download/splunk-enterprise.html))
3. **SSH Access** to the EC2 instance.

---

## **Step 1: Launch an AWS EC2 Instance (Ubuntu)**
1. **Go to AWS Console** → **EC2** → **Launch Instance**.
   - **AMI**: Ubuntu Server 22.04 LTS (or 20.04)
   - **Instance Type**: `t3.xlarge` (4 vCPUs, 16GB RAM for testing; use `m5.2xlarge` for production)
   - **Storage**: Minimum **50GB GP2/GP3** root volume (increase if storing large logs).
   - **Security Group**: Allow inbound ports:
     - **22 (SSH)**
     - **8000 (Splunk Web)**
     - **8089 (Splunk Management)**
     - **9997 (Forwarding, if used)**
   - **Key Pair**: Attach an existing key or create a new one.

2. **SSH into the Instance**:
   ```bash
   ssh -i "your-key.pem" ubuntu@<EC2_Public_IP>
   ```

---

## **Step 2: Install Splunk Enterprise**
### **1. Update System & Install Dependencies**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget net-tools -y
```

### **2. Download Splunk Enterprise**
```bash
wget -O splunk.tgz "https://download.splunk.com/products/splunk/releases/9.1.2/linux/splunk-9.1.2-b6436bc649ea-Linux-x86_64.tar.gz"
```
*(Replace `9.1.2` with the latest version from [Splunk Downloads](https://www.splunk.com/en_us/download/splunk-enterprise.html))*

### **3. Extract Splunk**
```bash
sudo tar -xvzf splunk.tgz -C /opt
sudo chown -R ubuntu:ubuntu /opt/splunk
```

### **4. Start Splunk & Accept License**
```bash
/opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt
```
- **Set admin username/password** when prompted.

### **5. Enable Splunk at Boot**
```bash
/opt/splunk/bin/splunk enable boot-start -systemd-managed 1
sudo systemctl enable Splunkd
```

### **6. Access Splunk Web**
- Open in browser:
  ```
  http://<EC2_Public_IP>:8000
  ```
- Login with the admin credentials you set.

---

## **Step 3: Configure Splunk Components**
### **1. Open Firewall Ports (AWS Security Group)**
- Edit the **EC2 Security Group** to allow:
  - **8000** (Splunk Web)
  - **8089** (Management)
  - **9997** (Forwarding, if used).

### **2. Set Up Indexer (Data Storage)**
```bash
/opt/splunk/bin/splunk enable listen 9997 -auth admin:yourpassword
```
*(Allows forwarders to send data to this instance.)*

### **3. Add Forwarders (Optional)**
To send data from other servers:
```bash
/opt/splunk/bin/splunk add forward-server <EC2_Private_IP>:9997 -auth admin:yourpassword
```

---

## **Step 4: Secure Splunk**
### **1. Enable HTTPS for Splunk Web**
```bash
/opt/splunk/bin/splunk enable webserver-ssl -auth admin:yourpassword
```
- Restart Splunk:
  ```bash
  /opt/splunk/bin/splunk restart
  ```
- Now access: `https://<EC2_Public_IP>:8000`

### **2. Use Let’s Encrypt (Recommended for Production)**
1. Install Certbot:
   ```bash
   sudo apt install certbot python3-certbot-nginx -y
   ```
2. Get a certificate (replace `yourdomain.com`):
   ```bash
   sudo certbot certonly --standalone -d yourdomain.com
   ```
3. Configure Splunk to use the cert:
   ```bash
   /opt/splunk/bin/splunk edit web-ssl -cert /etc/letsencrypt/live/yourdomain.com/fullchain.pem -privkey /etc/letsencrypt/live/yourdomain.com/privkey.pem -auth admin:yourpassword
   ```

---

## **Step 5: Scale for Production**
### **1. Indexer Clustering (3+ Nodes)**
1. **Master Node**:
   ```bash
   /opt/splunk/bin/splunk edit cluster-config -mode master -replication_factor 2 -search_factor 2 -secret <CLUSTER_SECRET> -auth admin:yourpassword
   ```
2. **Peer Nodes**:
   ```bash
   /opt/splunk/bin/splunk edit cluster-config -mode slave -master_uri https://<MASTER_NODE_IP>:8089 -secret <CLUSTER_SECRET> -auth admin:yourpassword
   ```

### **2. Search Head Clustering (High Availability)**
1. **Deploy 3+ Search Heads**:
   ```bash
   /opt/splunk/bin/splunk init shcluster-config -mgmt_uri https://<SHC_LEADER_IP>:8089 -replication_port 8090 -secret <SHC_SECRET> -auth admin:yourpassword
   ```
2. **Bootstrap Captain**:
   ```bash
   /opt/splunk/bin/splunk bootstrap shcluster-captain -servers_list "https://<SH1_IP>:8089,https://<SH2_IP>:8089" -auth admin:yourpassword
   ```

### **3. AWS-Specific Optimizations**
- **Use EBS Volumes**: Attach additional EBS volumes (`/opt/splunk/var`) for index storage.
- **Load Balancer**: Use **AWS ALB** to distribute traffic to multiple Splunk Web instances.

---

## **Troubleshooting**
| Issue | Solution |
|-------|----------|
| Splunk Web not loading | Check `netstat -tulnp \| grep 8000` and AWS Security Group. |
| High CPU/Memory | Upgrade instance type (`m5.2xlarge`). |
| Data not indexing | Verify forwarders with `./splunk list forward-server`. |

---


## After installing Splunk on AWS, covering data inputs, app deployment, and operational configurations:

---

## **1. Add Data Inputs to Splunk**
### **A. Monitor Files & Directories (e.g., /var/log)**
```bash
# Monitor a directory (recursively)
/opt/splunk/bin/splunk add monitor /var/log/ -index main -sourcetype syslog

# Monitor a specific file
/opt/splunk/bin/splunk add monitor /var/log/auth.log -index security -sourcetype linux_secure
```
- **Verify**:  
  ```bash
  /opt/splunk/bin/splunk list monitor
  ```

### **B. HTTP Event Collector (HEC) for JSON/API Data**
1. **Enable HEC**:
   ```bash
   /opt/splunk/bin/splunk enable http-event-collector -port 8088 -auth admin:yourpassword
   ```
2. **Create a Token**:
   - Web UI: **Settings → Data Inputs → HTTP Event Collector → New Token**
   - Test with `curl`:
     ```bash
     curl -k https://<EC2_IP>:8088/services/collector -H "Authorization: Splunk <HEC_TOKEN>" -d '{"event": "Test event", "sourcetype": "manual"}'
     ```

### **C. Syslog (UDP/TCP)**
```bash
# Enable UDP syslog on port 514
/opt/splunk/bin/splunk enable udp 514 -auth admin:yourpassword -index network -sourcetype syslog

# Enable TCP syslog (optional)
/opt/splunk/bin/splunk enable tcp 514 -auth admin:yourpassword
```
- **AWS Security Group**: Allow inbound UDP/TCP 514.

---

## **2. Install Splunk Apps**
### **A. Core Apps for AWS**
| App | Purpose | Installation Method |
|------|---------|---------------------|
| **AWS Add-on** | Ingest AWS logs (CloudTrail, S3, VPC) | [Splunkbase](https://splunkbase.splunk.com/app/1876/) |
| **Splunk ES** (Enterprise Security) | SIEM/Security Analytics | Requires license |
| **ITSI** | IT Service Monitoring | Requires license |

**Install via CLI**:
```bash
/opt/splunk/bin/splunk install app /path/to/aws-addon.tgz -auth admin:yourpassword
```

### **B. Configure AWS Add-on**
1. **Web UI**: Navigate to **Apps → AWS Add-on → Configuration**.
2. **Add AWS Accounts**:
   - Provide AWS IAM keys with `CloudTrailReadOnly` policy.
   - Enable inputs (CloudTrail, S3, VPC Flow Logs).

### **C. Install Machine Learning Toolkit (Free)**
```bash
/opt/splunk/bin/splunk install app https://splunkbase.splunk.com/app/2890/ -auth admin:yourpassword
```

---

## **3. Set Up Alerts & Dashboards**
### **A. Create Alerts**
1. **Web UI**: **Settings → Alerts → New Alert**.
2. **Example Alert (High CPU)**:
   - Search: `index=_internal sourcetype=splunkd component=Metrics | stats avg(cpu_seconds) as avg_cpu by host`
   - Trigger: `avg_cpu > 50`
   - Action: Send email or trigger AWS Lambda.

### **B. Build Dashboards**
1. **Web UI**: **Dashboards → Create New Dashboard**.
2. **Example AWS Dashboard**:
   - Panel 1: `index=aws* | timechart count by sourcetype`
   - Panel 2: `index=aws_cloudtrail | top user`

### **C. Schedule Reports**
```bash
# CLI example (daily report)
/opt/splunk/bin/splunk add search "index=aws_cloudtrail | stats count by user" -schedule "0 8 * * *" -action email -to admin@example.com
```

---

## **AWS-Specific Optimizations**
### **1. S3 Bucket for Log Ingestion**
1. **Create an S3 Bucket** and enable **S3 Event Notifications**.
2. **Configure AWS Add-on** to poll S3:
   ```bash
   /opt/splunk/bin/splunk add oneshot s3://your-bucket/prefix -index aws -sourcetype aws:s3
   ```

### **2. Use EC2 Instance Roles (Avoid Hardcoding Keys)**
1. **Assign IAM Role** to EC2 with permissions:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": ["cloudtrail:LookupEvents", "s3:GetObject"],
       "Resource": "*"
     }]
   }
   ```
2. **AWS Add-on** will auto-detect role credentials.

### **3. Scale with Auto Scaling**
- **Indexer Cluster**: Deploy peers across AZs using EC2 Auto Scaling.
- **Search Heads**: Use AWS ALB to distribute traffic.

---

## **Troubleshooting Checklist**
| Issue | Debug Command |
|-------|---------------|
| Data not indexing | `./splunk search "index=* \| head 1"` |
| HEC not working | `tail -f /opt/splunk/var/log/splunk/http_event_collector.log` |
| AWS Add-on errors | `./splunk search "index=_internal sourcetype=aws:*"` |

---


## Implementing these next steps for your Splunk deployment on AWS:

---

## **1. Integrate Splunk with AWS Services**
### **A. Kinesis Data Streams**
**Use Case**: Real-time log ingestion from high-volume sources.

**Setup**:
1. **Create Kinesis Data Stream**:
   ```bash
   aws kinesis create-stream --stream-name SplunkLogStream --shard-count 2
   ```

2. **Install AWS Add-on for Splunk**:
   - Navigate to **Splunk Web → Apps → Find More Apps**
   - Install "AWS Add-on for Splunk"

3. **Configure Kinesis Input**:
   ```bash
   /opt/splunk/bin/splunk add kinesis -name MyKinesis -stream SplunkLogStream -region us-east-1 -index aws_logs
   ```

**Verify**:
```bash
aws kinesis put-record --stream-name SplunkLogStream --partition-key 1 --data "Test message"
```

---

### **B. AWS Lambda Integration**
**Use Case**: Process and forward custom events to Splunk.

**Setup**:
1. **Create Lambda Function** (Node.js/Python):
   ```javascript
   const https = require('https');
   
   exports.handler = async (event) => {
     const options = {
       hostname: '<SPLUNK_HEC_ENDPOINT>',
       port: 8088,
       path: '/services/collector',
       method: 'POST',
       headers: {
         'Authorization': 'Splunk <HEC_TOKEN>'
       }
     };
     
     const req = https.request(options);
     req.write(JSON.stringify({ event: event }));
     req.end();
   };
   ```

2. **Trigger Sources**:
   - S3 Bucket: `s3:ObjectCreated:*`
   - CloudWatch Events
   - API Gateway

---

### **C. CloudWatch Logs**
**Use Case**: Centralize AWS service logs.

**Setup**:
1. **Install Splunk Add-on for AWS**:
   ```bash
   /opt/splunk/bin/splunk install app splunk-add-on-for-aws_600.tgz
   ```

2. **Configure CloudWatch Input**:
   ```bash
   /opt/splunk/bin/splunk add cloudwatch -log-group-name /aws/lambda/my-function -region us-east-1 -index aws_cloudwatch
   ```

**Pro Tip**: Use CloudWatch Logs Subscription to push logs directly to Splunk HEC.

---

## **2. Set Up Splunk Universal Forwarders on EC2**
### **A. Install Forwarder**
```bash
wget -O splunkforwarder.tgz "https://download.splunk.com/products/universalforwarder/releases/9.1.2/linux/splunkforwarder-9.1.2-b6436bc649ea-Linux-x86_64.tar.gz"
tar -xvzf splunkforwarder.tgz -C /opt
```

### **B. Configure Forwarding**
```bash
/opt/splunkforwarder/bin/splunk start --accept-license
/opt/splunkforwarder/bin/splunk add forward-server <SPLUNK_INDEXER_IP>:9997 -auth admin:yourpassword
/opt/splunkforwarder/bin/splunk add monitor /var/log/secure -index linux_security
```

### **C. Deploy at Scale**
1. **Create AMI** of configured forwarder
2. **Use EC2 Auto Scaling** to deploy across instances
3. **Central Management** via Deployment Server:
   ```bash
   /opt/splunk/bin/splunk set deploy-poll <DEPLOYMENT_SERVER_IP>:8089
   ```

---

## **3. Splunk AIOps for Anomaly Detection**
### **A. Install Machine Learning Toolkit**
```bash
/opt/splunk/bin/splunk install app https://splunkbase.splunk.com/app/2890/ -auth admin:yourpassword
```

### **B. Sample Use Cases**
1. **Log Anomaly Detection**:
   ```sql
   index=_internal | timechart span=1h count | predict count algorithm=LLP5 future_timespan=5
   ```

2. **AWS Cost Anomalies**:
   ```sql
   index=aws_billing | anomaly_detect CostUSD
   ```

3. **Automated Alerting**:
   ```bash
   /opt/splunk/bin/splunk add alert "index=network | anomaly_detect bytes_out" -action.email.to=admin@example.com
   ```

### **C. Integration with ITSI**
1. Install **IT Service Intelligence (ITSI)**
2. Configure AI-driven service monitoring:
   ```bash
   /opt/splunk/bin/splunk edit itsi -enable_ai 1 -auth admin:yourpassword
   ```

---

## **AWS Optimization Checklist**
| Task | Command/Resource |
|------|------------------|
| Reduce EC2 Costs | Use Spot Instances for indexers |
| Secure Comm. | Enable VPC PrivateLink for Splunk |
| High Availability | Deploy across 3 AZs |
| Backup | Use S3 for Splunk config backups |

---

## **Troubleshooting Tips**
1. **Forwarder Not Sending Data**:
   ```bash
   tail -f /opt/splunkforwarder/var/log/splunk/splunkd.log
   ```

2. **HEC Connection Issues**:
   ```bash
   openssl s_client -connect <SPLUNK_IP>:8088
   ```

3. **AWS Permissions**:
   ```bash
   aws sts get-caller-identity
   ```

---

