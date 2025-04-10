

### **1. General Splunk Commands**
- Start Splunk:  
  ```bash
  splunk start
  ```
- Stop Splunk:  
  ```bash
  splunk stop
  ```
- Restart Splunk:  
  ```bash
  splunk restart
  ```
- Check Splunk status:  
  ```bash
  splunk status
  ```
- Enable Splunk to start at boot:  
  ```bash
  splunk enable boot-start
  ```

### **2. User & Authentication Management**
- Add a new Splunk user:  
  ```bash
  splunk add user <username> -password <password> -role <role> -full-name "<Full Name>"
  ```
- List all users:  
  ```bash
  splunk list user
  ```
- Edit a user’s role:  
  ```bash
  splunk edit user <username> -role <newrole> -password <newpassword>
  ```
- Delete a user:  
  ```bash
  splunk remove user <username>
  ```

### **3. Index Management**
- List all indexes:  
  ```bash
  splunk list index
  ```
- Create a new index:  
  ```bash
  splunk add index <index_name>
  ```
- Delete an index:  
  ```bash
  splunk remove index <index_name>
  ```

### **4. Forwarder & Deployment Management**
- Deploy a Universal Forwarder (UF):  
  ```bash
  splunk set deploy-poll <deployment_server>:8089
  ```
- Add a forwarder to receive data:  
  ```bash
  splunk add forward-server <host>:9997
  ```
- List forward servers:  
  ```bash
  splunk list forward-server
  ```
- Remove a forward server:  
  ```bash
  splunk remove forward-server <host>:9997
  ```

### **5. License Management**
- Add a license:  
  ```bash
  splunk add licenses <license_file>
  ```
- List licenses:  
  ```bash
  splunk list licenses
  ```
- Remove a license:  
  ```bash
  splunk remove licenses <license_id>
  ```

### **6. Network & Port Configuration**
- Change Splunk’s management port (default: 8089):  
  ```bash
  splunk set splunkd-port <port>
  ```
- Change Splunk web port (default: 8000):  
  ```bash
  splunk set web-port <port>
  ```
- Enable HTTPS for Splunk Web:  
  ```bash
  splunk enable webserver-ssl -auth <admin:password>
  ```

### **7. Distributed Search & Clustering**
- Add a search peer (distributed search):  
  ```bash
  splunk add search-server <host>:8089 -auth <admin:password>
  ```
- List search peers:  
  ```bash
  splunk list search-server
  ```

### **8. Backup & Restore**
- Backup Splunk configuration:  
  ```bash
  splunk export user-config -directory <backup_path>
  ```
- Restore Splunk configuration:  
  ```bash
  splunk import user-config -directory <backup_path>
  ```

### **9. Debugging & Logs**
- Check Splunk logs:  
  ```bash
  tail -f $SPLUNK_HOME/var/log/splunk/splunkd.log
  ```
- Run Splunk in debug mode:  
  ```bash
  splunk start --debug
  ```

These commands help manage Splunk configurations, users, forwarders, licensing, and troubleshooting. Let me know if you need more details on any specific command! 🚀
