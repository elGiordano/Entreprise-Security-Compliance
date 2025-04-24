# **Linux & Mac OS Audit Logs in Splunk: Enterprise Security Deep Dive**

Linux (`auditd`) and Mac OS (`osquery/OpenBSM`) audit logs provide critical security visibility comparable to Windows Event Logs. When ingested into Splunk, they enable detection of **privilege escalation, file tampering, unauthorized access, and malicious process execution**.

---

## **1. Key Audit Log Sources & Files**
### **Linux (`auditd`) Log Locations**
| **Log File**               | **Purpose**                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| `/var/log/audit/audit.log` | Primary auditd logs (processes, file access, auth events)                  |
| `/var/log/auth.log`        | Authentication logs (SSH, sudo, PAM)                                       |
| `/var/log/secure`          | RedHat/CentOS equivalent of `auth.log`                                     |
| `/var/log/syslog`          | System-level events (useful if auditd not installed)                       |

### **Mac OS Audit Logs**
| **Source**                 | **Purpose**                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| **OpenBSM** (`/var/audit/`)| macOS native audit logs (processes, file access)                           |
| **osquery**                | Cross-platform endpoint visibility (can forward to Splunk)                 |
| **Unified Logging** (`log stream`) | Modern macOS logging (requires `log` CLI tool)                     |

---

## **2. Critical Linux/Mac Audit Events & MITRE ATT&CK Mapping**
### **A. Privilege Escalation (T1548)**
#### **1. Sudo Abuse (Linux)**
```spl
index=linux sourcetype=linux_secure "sudo: session opened" 
| stats count by user, command 
| where count > 5
```
 **Detects**: Excessive `sudo` usage (potential privilege abuse).

#### **2. Setuid/Setgid Execution (Linux)**
```spl
index=linux sourcetype=auditd type=EXECVE a0="*chmod*" a1="*+s*" 
| table _time, host, user, exe, key
```
 **Detects**: `chmod +s` (setting SUID/GUID bits for persistence).

---

### **B. Credential Access (T1003)**
#### **1. Access to `/etc/shadow` (Linux)**
```spl
index=linux sourcetype=auditd type=SYSCALL arch=b64 success=no syscall=openat path="/etc/shadow" 
| stats count by host, user, exe
```
 **Detects**: Failed attempts to read hashed passwords.

#### **2. Keychain Dumping (Mac)**
```spl
index=macos sourcetype=osquery name="process_events" 
| search path="/usr/bin/security" cmdline="*dump-keychain*" 
| table _time, host, uid, cmdline
```
 **Detects**: Mac keychain extraction (e.g., `security dump-keychain`).

---

### **C. Persistence (T1547, T1053)**
#### **1. Cron Job Modifications (Linux)**
```spl
index=linux sourcetype=auditd type=SYSCALL arch=b64 syscall=openat path="/etc/cron*" 
| stats count by host, user, exe, path
```
 **Detects**: Malicious cron jobs (common in web shell attacks).

#### **2. LaunchAgent/LaunchDaemon (Mac)**
```spl
index=macos sourcetype=osquery name="file_events" 
| search target_path="/Library/Launch*/*.plist" 
| table _time, host, uid, target_path
```
 **Detects**: Persistence via macOS LaunchDaemons.

---

### **D. Lateral Movement (T1021)**
#### **1. SSH Logins (Linux)**
```spl
index=linux sourcetype=linux_secure "sshd" ("Accepted password" OR "Failed password") 
| stats count by src_ip, user 
| where count > 3
```
 **Detects**: SSH brute force or credential stuffing.

#### **2. ARP Cache Poisoning (Linux)**
```spl
index=linux sourcetype=auditd type=NETFILTER_PKT 
| search saddr!=orig_saddr 
| table _time, host, saddr, orig_saddr
```
 **Detects**: MITM attacks (requires `auditd` network rules).

---

### **E. Defense Evasion (T1070, T1562)**
#### **1. Log Deletion (Linux)**
```spl
index=linux sourcetype=auditd type=SYSCALL arch=b64 syscall=unlinkat path="/var/log/*" 
| stats count by host, user, exe
```
 **Detects**: Attackers wiping logs (`rm -rf /var/log/secure`).

#### **2. Auditd Configuration Tampering (Linux)**
```spl
index=linux sourcetype=auditd type=CONFIG_CHANGE 
| search "auditctl" AND ("-D" OR "--delete") 
| table _time, host, user, proctitle
```
 **Detects**: Disabling auditd rules (`auditctl -D`).

---

## **3. Splunk Best Practices for Linux/Mac Audit Logs**
### ** Normalization**
- Use **Splunk Common Information Model (CIM)** fields:
  - `user`, `src_ip`, `process` (for consistency with Windows logs).
- Example:
  ```spl
  index=linux sourcetype=auditd 
  | eval process=exe 
  | eval user=acct 
  | table _time, host, user, process, key
  ```

### ** Correlation**
- Combine with **network logs** (e.g., `index=netfw`) to track lateral movement.
- Example:
  ```spl
  index=linux sourcetype=linux_secure "Accepted password" 
  | join src_ip [search index=netfw dest_ip=* action=deny] 
  | table _time, host, user, src_ip, dest_port
  ```

### ** Threat Hunting**
- Hunt for **rare parent-child process relationships**:
  ```spl
  index=linux sourcetype=auditd type=EXECVE 
  | stats count by ppid, exe 
  | sort - count 
  | head 10
  ```

---

## **4. Example Splunk Dashboard for Linux/Mac Audit Logs**
### **Panels to Include**
1. **Top Sudo Commands** (`sourcetype=linux_secure "sudo" | stats count by command`)
2. **Failed SSH Logins** (`sourcetype=linux_secure "Failed password" | timechart count`)
3. **File Integrity Monitoring** (`sourcetype=auditd path="/etc/passwd" OR path="/etc/shadow"`)
4. **Unusual Process Execution** (`sourcetype=auditd exe="/tmp/*" | stats count by exe`)

### **Splunk Snippet for Dashboard**
```xml
<panel>
  <title>Failed SSH Logins by IP</title>
  <search>
    <query>index=linux sourcetype=linux_secure "Failed password" | stats count by src_ip | sort - count</query>
  </search>
  <option name="count">10</option>
</panel>
```

---

## **5. Advanced: osquery + Splunk for Mac/Linux**
osquery provides **real-time SQL queries** for endpoint visibility. Example:
```spl
index=osquery sourcetype=osquery:results 
| search name="processes" 
| search path="/Library/*.app/Contents/MacOS/*" 
| table _time, host, name, path
```
 **Use Case**: Detect unsigned macOS apps running.

---

## **Final Recommendations**
 **Deploy `auditd` on Linux** with rules for:
   - File integrity (`-w /etc/passwd -p wa -k passwd_changes`)
   - Process execution (`-a always,exit -F arch=b64 -S execve`).  
 **On Mac**, use **osquery** or **OpenBSM** for granular logging.  
 **Correlate with Splunk ES** for ATT&CK coverage.  

