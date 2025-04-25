# **AWS CloudTrail Logs in Splunk: Comprehensive Security Monitoring Guide**

AWS CloudTrail logs provide critical visibility into API activity across your AWS environment. When properly analyzed in Splunk, they reveal unauthorized access, configuration changes, data exfiltration attempts, and compliance violations.

---

## **1. Key CloudTrail Log Sources & Normalization**

### **A. Essential CloudTrail Log Types**
| **Log Type**       | **Security Relevance**                                                                 | **Splunk Add-on**                     |
|--------------------|--------------------------------------------------------------------------------------|--------------------------------------|
| **Management Events** | All AWS management API calls (create, modify, delete actions)                     | [AWS Add-on](https://splunkbase.splunk.com/app/1876/) |
| **Data Events**      | S3 object-level activity and Lambda execution logs                                | Requires explicit enablement         |
| **Insights Events**  | Automated detection of unusual API activity patterns                              | Available in Splunk ES                |

### **B. Field Normalization (CIM Compliance)**
Map to **Splunk's Common Information Model**:
- `user` → `userIdentity.arn`  
- `src_ip` → `sourceIPAddress`  
- `action` → `eventName` + `eventSource`  
- `resource` → `resources.ARN`  
- `region` → `awsRegion`  

Example transformation:
```spl
index=aws_logs sourcetype=aws:cloudtrail 
| eval user=userIdentity.arn, src_ip=sourceIPAddress, action=eventName.":".eventSource 
| stats count by user, src_ip, action 
| sort - count
```

---

## **2. Critical Detection Use Cases**

### **A. Privilege Escalation (T1098)**
#### **IAM Policy Changes**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName IN ("PutUserPolicy", "PutRolePolicy", "AttachUserPolicy") 
| table _time, userIdentity.arn, eventName, requestParameters.policyDocument
```

#### **New Admin Users**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName="CreateUser" 
| join requestParameters.userName 
    [search index=aws_logs sourcetype=aws:cloudtrail eventName="AttachUserPolicy" 
    | search requestParameters.policyArn="*AdministratorAccess*"] 
| table _time, userIdentity.arn, requestParameters.userName
```

---

### **B. Data Exfiltration (T1530)**
#### **S3 Bucket Data Downloads**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventSource="s3.amazonaws.com" eventName IN ("GetObject", "ListBucket") 
| stats sum(requestParameters.bytes) as total_bytes by userIdentity.arn, requestParameters.bucketName 
| where total_bytes > 100000000 
| sort - total_bytes
```

#### **RDS Database Snapshots Exported**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName="StartExportTask" 
| search requestParameters.exportSourceType="SNAPSHOT" 
| table _time, userIdentity.arn, requestParameters.s3BucketName
```

---

### **C. Defense Evasion (T1562)**
#### **CloudTrail Logging Disabled**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName IN ("StopLogging", "DeleteTrail") 
| table _time, userIdentity.arn, eventName, awsRegion
```

#### **Security Group Modifications**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName="AuthorizeSecurityGroupIngress" 
| search requestParameters.ipPermissions.items.*.ipRanges.items.*.cidrIp="0.0.0.0/0" 
| table _time, userIdentity.arn, requestParameters.groupId
```

---

### **D. Lateral Movement (T1578)**
#### **EC2 Assume Role Abuse**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName="AssumeRole" 
| stats count by userIdentity.arn, requestParameters.roleArn 
| where count > 5 
| sort - count
```

#### **VPC Peering Connections**
```spl
index=aws_logs sourcetype=aws:cloudtrail eventName="CreateVpcPeeringConnection" 
| search requestParameters.peerVpcId!="vpc-12345678" 
| table _time, userIdentity.arn, requestParameters.vpcId, requestParameters.peerVpcId
```

---

## **3. Advanced Correlation Techniques**

### **A. Anomalous API Activity Detection**
```spl
index=aws_logs sourcetype=aws:cloudtrail 
| stats dc(eventName) as unique_apis by userIdentity.arn 
| eventstats avg(unique_apis) as avg, stdev(unique_apis) as stdev 
| where unique_apis > (avg+(stdev*3)) 
| sort - unique_apis
```

### **B. Threat Intel Enrichment**
```spl
index=aws_logs sourcetype=aws:cloudtrail 
| lookup threat_intel_ip sourceIPAddress as sourceIPAddress OUTPUT is_malicious 
| where is_malicious="true" 
| table _time, userIdentity.arn, eventName, sourceIPAddress
```

### **C. Cross-Account Activity Monitoring**
```spl
index=aws_logs sourcetype=aws:cloudtrail 
| rex field=userIdentity.arn ":account:(?<account_id>\d+):" 
| stats count by account_id, eventName 
| sort - count
```

---

## **4. Splunk Dashboard Examples**

### **AWS Security Dashboard**
**Essential Panels:**
1. **Top API Actions** (`stats count by eventName | sort - count`)
2. **Unauthorized Attempts** (`errorCode=AccessDenied | timechart count`)
3. **Data Transfer Anomalies** (`eventName=GetObject | stats sum(bytes)`)
4. **IAM Changes Timeline** (`eventName IN ("CreateUser","Put*Policy") | timechart count`)

### **CloudTrail Insights View**
```spl
index=aws_logs sourcetype=aws:cloudtrail:insights 
| table _time, eventName, insightType, insightContext.anomalousApi
```

---

## **5. Best Practices**

 **Enable All Regions**  
   - Configure CloudTrail in all AWS regions (including GovCloud)  

 **Enable Data Events**  
   - Critical S3 buckets and Lambda functions  

 **Normalize Key Fields**  
   - Standardize `user`, `action`, `resource` across AWS services  

 **Correlate with GuardDuty**  
   - Join CloudTrail logs with GuardDuty findings  

 **Monitor Root Account**  
   - Alert on any root account activity  

---

## **Final Thoughts**
AWS CloudTrail logs in Splunk enable detection of:
 **Privilege escalation** (IAM changes, new admin users)  
 **Data exfiltration** (S3 downloads, RDS exports)  
 **Defense evasion** (logging disabled, security group changes)  
 **Lateral movement** (role assumption, VPC peering)  

