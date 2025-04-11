### **Install Full-Stack Splunk on Google Cloud (Ubuntu Instance)**
This guide covers setting up **Splunk Enterprise** (search head, indexer, and universal forwarder if needed) on a **Google Cloud Platform (GCP) Ubuntu instance**.

---

## **Prerequisites**
1. **Google Cloud Account** (with a running Ubuntu VM)
   - Minimum recommended machine type: **n2-standard-4 (4 vCPUs, 16GB RAM)**
   - Storage: **At least 50GB disk space**
2. **Splunk Enterprise Download** (free trial or license)
   - Download from [Splunk Official Site](https://www.splunk.com/en_us/download/splunk-enterprise.html)
3. **SSH Access** to the GCP instance.

---

## **Step 1: Set Up a Google Cloud VM (Ubuntu)**
1. **Create a new VM instance**:
   - Go to **Compute Engine** → **VM Instances** → **Create Instance**.
   - Select **Ubuntu 22.04 LTS** (or 20.04).
   - Choose a machine type (e.g., `n2-standard-4`).
   - Increase boot disk size to **50GB** (for logs and indexes).
   - Allow **HTTP/HTTPS traffic** (for Splunk Web).
   - Click **Create**.

2. **SSH into the VM**:
   ```bash
   gcloud compute ssh <instance-name> --zone <zone>
   ```
   (Replace `<instance-name>` and `<zone>` with your VM details.)

---

## **Step 2: Install Splunk Enterprise**
### **1. Update & Install Dependencies**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget net-tools -y
```

### **2. Download Splunk Enterprise**
```bash
wget -O splunk.tgz "https://download.splunk.com/products/splunk/releases/<version>/linux/splunk-<version>-linux-x86_64.tar.gz"
```
(Replace `<version>` with the latest, e.g., `9.1.2`)

### **3. Extract & Install Splunk**
```bash
tar -xvzf splunk.tgz -C /opt
sudo chown -R $USER:$USER /opt/splunk
```

### **4. Start Splunk & Accept License**
```bash
/opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt
```
- **Set admin credentials** when prompted.

### **5. Enable Splunk at Boot**
```bash
/opt/splunk/bin/splunk enable boot-start -systemd-managed 1
sudo systemctl enable Splunkd
```

### **6. Open Splunk Web Interface**
- **Check the IP of your GCP VM**:
  ```bash
  curl ifconfig.me
  ```
- Access Splunk Web at:
  ```
  http://<GCP_VM_External_IP>:8000
  ```
- Login with the admin credentials you set.

---

## **Step 3: Configure Splunk (Indexer & Search Head)**
### **1. Open Necessary Ports (GCP Firewall)**
- Go to **VPC Network** → **Firewall** → **Create Firewall Rule**:
  - Allow **TCP:8000** (Splunk Web)
  - Allow **TCP:8089** (Splunk Management)
  - Allow **TCP:9997** (Forwarding)

### **2. Configure Splunk as Indexer**
```bash
/opt/splunk/bin/splunk enable listen 9997 -auth admin:yourpassword
```

### **3. Add Forwarders (Optional)**
If you want to send logs from other machines:
```bash
/opt/splunk/bin/splunk add forward-server <GCP_VM_IP>:9997 -auth admin:yourpassword
```

---

## **Step 4: Secure Splunk (HTTPS & Authentication)**
### **1. Enable HTTPS for Splunk Web**
```bash
/opt/splunk/bin/splunk enable webserver-ssl -auth admin:yourpassword
```
- Restart Splunk:
  ```bash
  /opt/splunk/bin/splunk restart
  ```
- Now access: `https://<GCP_VM_IP>:8000`

### **2. Configure Firewall & Reverse Proxy (Optional)**
For better security, use **Nginx/Apache as a reverse proxy**:
```bash
sudo apt install nginx -y
```
Configure `/etc/nginx/sites-available/splunk` with SSL (Let’s Encrypt).

---

## **Step 5: Verify Installation**
- Check Splunk status:
  ```bash
  /opt/splunk/bin/splunk status
  ```
- Check logs:
  ```bash
  tail -f /opt/splunk/var/log/splunk/splunkd.log
  ```

---
---

## **Next Steps** : 


---

## **📊 1. Add Data Inputs to Splunk**
### **A. Monitor Files & Directories**
```bash
# Add a file input (e.g., /var/log/syslog)
/opt/splunk/bin/splunk add monitor /var/log/syslog -index main -sourcetype syslog
```
- **Verify**:  
  ```bash
  /opt/splunk/bin/splunk list monitor
  ```

### **B. HTTP Event Collector (HEC) for APIs/JSON**
1. **Enable HEC**:
   ```bash
   /opt/splunk/bin/splunk enable http-event-collector -port 8088 -auth admin:yourpassword
   ```
2. **Create a Token**:
   - Go to **Settings → Data Inputs → HTTP Event Collector → New Token**.
   - Use `curl` to test:
     ```bash
     curl -k https://<GCP_IP>:8088/services/collector -H "Authorization: Splunk <HEC_TOKEN>" -d '{"event": "Hello, Splunk!"}'
     ```

### **C. Syslog (UDP/TCP)**
```bash
# Enable UDP syslog on port 514
/opt/splunk/bin/splunk enable udp 514 -auth admin:yourpassword -index main -sourcetype syslog
```
- **GCP Firewall**: Allow `udp:514`/`tcp:514`.

---

## **🛠️ 2. Install Splunk Apps**
### **A. Install via CLI**
```bash
/opt/splunk/bin/splunk install app /path/to/app.tgz -auth admin:yourpassword
```
- **Popular Apps**:
  - **Splunk Enterprise Security (ES)** → SIEM
  - **ITSI** → IT monitoring
  - **AWS/Azure Add-ons** → Cloud logs

### **B. Install from Splunkbase**
1. Download from [Splunkbase](https://splunkbase.splunk.com/).
2. Upload via Splunk Web (**Apps → Manage Apps → Install from file**).

### **C. Configure Apps**
- **ES**: Set up correlation searches, data models.
- **ITSI**: Configure services, KPIs.

---

## **🚀 3. Scale for Production**
### **A. Indexer Clustering (Horizontal Scaling)**
1. **Deploy 3+ Nodes** (1 Master, 2 Peers):
   ```bash
   # On Master Node:
   /opt/splunk/bin/splunk edit cluster-config -mode master -replication_factor 2 -search_factor 2 -secret <CLUSTER_SECRET> -auth admin:yourpassword

   # On Peer Nodes:
   /opt/splunk/bin/splunk edit cluster-config -mode slave -master_uri https://<MASTER_NODE_IP>:8089 -secret <CLUSTER_SECRET> -auth admin:yourpassword
   ```
2. **Verify**:
   ```bash
   /opt/splunk/bin/splunk show cluster-status -auth admin:yourpassword
   ```

### **B. Search Head Clustering (High Availability)**
1. **Deploy 3+ Search Heads**:
   ```bash
   /opt/splunk/bin/splunk init shcluster-config -mgmt_uri https://<SHC_LEADER_IP>:8089 -replication_port 8090 -secret <SHC_SECRET> -auth admin:yourpassword
   ```
2. **Captain Election**:
   ```bash
   /opt/splunk/bin/splunk bootstrap shcluster-captain -servers_list "https://<SH1_IP>:8089,https://<SH2_IP>:8089" -auth admin:yourpassword
   ```

### **C. Load Balancing (GCP-Specific)**
1. **Use GCP Load Balancer**:
   - Forward `:8000` (Splunk Web) and `:8089` (management) to multiple instances.
2. **Configure Health Checks**:
   - Path: `/services/server/info` (HTTP 200 = healthy).

---

## **🔐 4. Security Hardening**
### **A. Enable Role-Based Access (RBAC)**
```bash
# Create a read-only role
/opt/splunk/bin/splunk add role read_only -imported user -auth admin:yourpassword
```

### **B. Audit Logs**
```bash
/opt/splunk/bin/splunk enable audit -auth admin:yourpassword
```
- Logs stored in `_audit` index.

### **C. Network Isolation**
- Place Splunk in a **GCP Private Subnet**.
- Use **Cloud NAT** for outbound traffic.

---

## **📈 5. Monitoring Splunk Itself**
### **A. Built-In Dashboards**
- **Splunk Monitoring Console**:  
  `https://<GCP_IP>:8000/en-GB/manager/search/monitoringconsole`

### **B. Forward Splunk Logs to Another Indexer**
```bash
/opt/splunk/bin/splunk add forward-server <BACKUP_INDEXER_IP>:9997 -auth admin:yourpassword
```

---

---

### **Troubleshooting Checklist**
| Issue | Command |
|-------|---------|
| Data not indexing | `./splunk search "index=* \| head 1"` |
| High CPU | `./splunk list processes` |
| Forwarder issues | `./splunk list forward-server` |

---


---



---

## **🔧 Troubleshooting Splunk on GCP (Ubuntu)**
### **1. Splunk Fails to Start**
#### **Symptoms**:
- `splunk status` shows errors.
- Splunk Web (`:8000`) is unreachable.

#### **Solutions**:
✅ **Check Splunk logs**:
```bash
tail -f /opt/splunk/var/log/splunk/splunkd.log
```
- Look for `ERROR` or `FATAL` messages.

✅ **Verify disk space**:
```bash
df -h  # Check if `/opt/splunk` has free space.
```
- If full, clear logs or expand the disk.

✅ **Check if Splunk is already running**:
```bash
ps aux | grep splunk
```
- If stuck, kill the process and restart:
  ```bash
  sudo pkill splunkd
  /opt/splunk/bin/splunk restart
  ```

---

### **2. Splunk Web (Port 8000) Not Accessible**
#### **Symptoms**:
- `Connection refused` when accessing `http://<GCP_IP>:8000`.

#### **Solutions**:
✅ **Check if Splunk Web is listening**:
```bash
netstat -tulnp | grep 8000
```
- If not, restart Splunk:
  ```bash
  /opt/splunk/bin/splunk restart
  ```

✅ **Verify GCP Firewall Rules**:
- Go to **GCP Console → VPC Network → Firewall**.
- Ensure **Ingress rule** allows `tcp:8000` (source: `0.0.0.0/0` or your IP).

✅ **Check Ubuntu’s UFW (if enabled)**:
```bash
sudo ufw status
sudo ufw allow 8000/tcp  # If firewall is active.
```

---

### **3. Forwarding Data Fails (Port 9997)**
#### **Symptoms**:
- Forwarders can’t send data to the indexer.
- `splunk list forward-server` shows errors.

#### **Solutions**:
✅ **Check if Splunk is listening on 9997**:
```bash
netstat -tulnp | grep 9997
```
- If not, enable receiving:
  ```bash
  /opt/splunk/bin/splunk enable listen 9997 -auth admin:yourpassword
  ```

✅ **Verify GCP Firewall for 9997**:
- Add **Ingress rule** for `tcp:9997`.

✅ **Test connectivity from forwarder**:
```bash
telnet <GCP_IP> 9997
```
- If blocked, check **GCP firewall** or **NACLs**.

---

### **4. High CPU/Memory Usage**
#### **Symptoms**:
- Splunk is slow or crashes.
- `top` shows `splunkd` consuming high resources.

#### **Solutions**:
✅ **Increase GCP VM size**:
- Upgrade to `n2-standard-8` (8 vCPUs + 32GB RAM).

✅ **Optimize Splunk settings**:
- Reduce indexing volume (adjust `inputs.conf`).
- Limit concurrent searches (`limits.conf`).

✅ **Check for stuck processes**:
```bash
/opt/splunk/bin/splunk list processes
```
- Kill unresponsive processes:
  ```bash
  kill -9 <PID>
  ```

---

### **5. Disk Space Running Out**
#### **Symptoms**:
- Splunk crashes with "No space left" errors.

#### **Solutions**:
✅ **Extend GCP Disk**:
1. Resize the disk in **GCP Console → Compute Engine → Disks**.
2. Expand filesystem:
   ```bash
   sudo growpart /dev/sda 1
   sudo resize2fs /dev/sda1
   ```

✅ **Clean up old indexes**:
```bash
/opt/splunk/bin/splunk clean eventdata -index <index_name>
```

✅ **Add a new persistent disk**:
1. Attach a new disk in GCP Console.
2. Mount it to `/opt/splunk/var/lib/splunk`:
   ```bash
   sudo mkdir -p /mnt/splunkdata
   sudo mount /dev/sdb1 /mnt/splunkdata
   sudo chown -R splunk:splunk /mnt/splunkdata
   ```
3. Update `indexes.conf` to use the new path.

---

### **6. SSL Certificate Errors (HTTPS)**
#### **Symptoms**:
- Browser warns about insecure connection.
- Splunk Web HTTPS (`:8000`) fails.

#### **Solutions**:
✅ **Regenerate Splunk Web cert**:
```bash
/opt/splunk/bin/splunk createssl web-cert -auth admin:yourpassword
```

✅ **Use Let’s Encrypt (Recommended)**:
1. Install Certbot:
   ```bash
   sudo apt install certbot python3-certbot-nginx -y
   ```
2. Get a certificate:
   ```bash
   sudo certbot certonly --standalone -d yourdomain.com
   ```
3. Configure Splunk to use the cert:
   ```bash
   /opt/splunk/bin/splunk edit web-ssl -cert /etc/letsencrypt/live/yourdomain.com/fullchain.pem -privkey /etc/letsencrypt/live/yourdomain.com/privkey.pem -auth admin:yourpassword
   ```

---

## **📌 Pro Tips**
- **Backup Splunk Configs**:
  ```bash
  tar -czvf splunk_backup.tar.gz /opt/splunk/etc
  ```
- **Monitor Splunk Health**:
  ```bash
  /opt/splunk/bin/splunk btool check
  ```

---




