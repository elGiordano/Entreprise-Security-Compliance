# **Container & Kubernetes Logs in Splunk: Comprehensive Security Monitoring Guide**

Containerized environments generate massive amounts of operational and security data. By ingesting **Kubernetes, Docker, and container runtime logs** into Splunk, you gain critical visibility into your cloud-native infrastructure. This guide covers **collection methods, critical security use cases, SPL queries, and best practices**.

---

## **1. Key Log Sources in Container Environments**

| **Component**       | **Log Location**                          | **Critical Data**                          |
|---------------------|------------------------------------------|-------------------------------------------|
| **Kubernetes API**  | `/var/log/kube-apiserver.log`            | Audit logs (user actions, auth attempts)   |
| **Kubelet**         | `/var/log/kubelet.log`                   | Node/container operations                 |
| **Docker**          | `/var/log/docker.log` (JSON format)      | Container lifecycle events                |
| **Pods**            | Stdout/Stderr (via kubectl logs)         | Application-specific logs                 |
| **Cloud Providers** | AWS EKS/Azure AKS/GCP GKE control plane  | Managed service audit trails              |

---

## **2. Critical Security Detection Use Cases**

### **1. Privilege Escalation & Unauthorized Access**
#### **Kubernetes RBAC Bypass Attempts**
```splunk
index=k8s_audit verb=create 
| search objectRef.resource="pods/exec" OR objectRef.resource="pods/attach" 
| stats count by user.username, objectRef.namespace
```

#### **Privileged Container Creation**
```splunk
index=k8s_audit verb=create objectRef.resource=pods 
| search requestObject.spec.containers.securityContext.privileged=true 
| table _time, user.username, requestObject.metadata.name
```

### **2. Suspicious Container Behavior**
#### **Cryptomining Containers (High CPU)**
```splunk
index=container_metrics container_name=* 
| stats avg(cpu_usage) as avg_cpu by container_name, pod_name 
| where avg_cpu > 80  // Threshold for investigation
```

#### **Unusual Network Connections (C2 Traffic)**
```splunk
index=k8s_networking 
| lookup threat_intel_ip dest_ip OUTPUT is_malicious 
| where is_malicious=true 
| table _time, pod_name, dest_ip, dest_port
```

### **3. Configuration Risks & Compliance**
#### **Missing Pod Security Policies**
```splunk
index=k8s_audit verb=create objectRef.resource=pods 
| search NOT requestObject.metadata.annotations."pod-security.kubernetes.io/enforce"=* 
| stats count by user.username, requestObject.metadata.namespace
```

#### **Default Service Account Usage**
```splunk
index=k8s_audit 
| search user.username="system:serviceaccount:default:default" 
| stats count by verb, objectRef.resource
```

### **4. Runtime Threats**
#### **Container Breakout Attempts**
```splunk
index=docker_logs 
| search "docker.sock" OR "/var/run/docker.sock" 
| stats count by container_name, image
```

#### **Sensitive Mounts (/etc, /var/run)**
```splunk
index=k8s_audit verb=create objectRef.resource=pods 
| search requestObject.spec.volumes.hostPath.path IN ("/etc", "/var/run") 
| table _time, user.username, requestObject.metadata.name
```

---

## **3. Advanced Correlation Techniques**

### **1. Anomalous Pod Creation (Deviation from Baseline)**
```splunk
index=k8s_audit verb=create objectRef.resource=pods 
| anomalydetect method=avg count by user.username 
| where isoutlier=1 
| table _time, user.username, requestObject.metadata.name
```

### **2. Impossible Location (Node Geolocation Mismatch)**
```splunk
index=k8s_audit 
| iplocation user.ip 
| stats values(node_name) as nodes, values(country) as countries by user.username 
| where mvcount(countries) > 1
```

### **3. Suspicious CronJob Execution**
```splunk
index=k8s_audit objectRef.resource=cronjobs 
| search responseStatus.code=200 
| stats count by user.username, objectRef.name 
| where count > 5  // Unusually frequent executions
```

---

## **4. Log Collection Methods**

### **1. Direct Collection (Recommended)**
- **Splunk Connect for Kubernetes** (Helm chart)
  ```bash
  helm install splunk-connect -f values.yaml splunk/splunk-connect-for-kubernetes
  ```
- **Fluentd/Fluent Bit DaemonSet**
  ```yaml
  output-elasticsearch.conf: |
    [OUTPUT]
    Name  splunk
    Host  splunk.example.com
    Port  8088
  ```

### **2. Cloud-Specific Solutions**
- **AWS EKS → CloudWatch → Splunk HTTP Event Collector**
- **GCP GKE → Pub/Sub → Splunk Dataflow**

---

## **5. Best Practices**

 **Tag logs with Kubernetes metadata** (namespace, pod_name, labels)  
 **Normalize fields** using Splunk CIM (Common Information Model)  
 **Implement multi-line log handling** for stack traces  
 **Store in dedicated indexes** (`k8s_audit`, `container_logs`)  
 **Correlate with host/network logs** for full context  

---

## **6. Sample Splunk Dashboard Panels**

| **Panel**                     | **SPL Query**                          |
|-------------------------------|---------------------------------------|
| **Top Pod Errors**           | `index=container_logs log_level=ERROR | top pod_name` |
| **RBAC Changes**             | `index=k8s_audit objectRef.resource=roles | timechart count` |
| **Privileged Containers**    | `index=k8s_audit search "privileged=true" | stats count by namespace` |
| **Anomalous Exec Sessions**  | `index=k8s_audit verb=exec | anomalydetect count by user` |

---


