 **role-based field filtering** in our Splunk environment, building on our existing multi-site architecture with SmartStore and federated search:

---

### **1. Field Filtering Configuration (`authorize.conf`)**
#### **Role-Specific Field Masking**
```ini
[role_restrictions:security_auditor]
blacklist0 = event_data.password
blacklist1 = event_data.credit_card
whitelist0 = event_data.user_id
whitelist1 = event_data.ip_address

[role_restrictions:dr_search_head]
blacklist0 = *_raw
blacklist1 = event_data.ssn
```

---

### **2. Index-Time Field Extraction Control (`props.conf`)**
```ini
[source::.../sensitive_logs.log]
TRANSFORMS-mask_fields = mask_ssn, mask_creditcards
FIELD_ALIAS-role_based = ssn FOR security_team_only ip FOR all_users
```

#### **Corresponding `transforms.conf`**
```ini
[mask_ssn]
REGEX = (\d{3}-\d{2}-\d{4})
FORMAT = ***-**-****
DEST_KEY = _raw

[mask_creditcards]
REGEX = (?:\d[ -]*?){13,16}
FORMAT = XXXX-XXXX-XXXX-####
DEST_KEY = _raw
```

---

### **3. Search-Time Field Filtering (`savedsearches.conf`)**
```ini
[Financial Report]
action.email.to = finance-team@corp.example.com
search = index=transactions | table amount account_number
fields_viewable.role_finance = amount account_number date
fields_viewable.role_auditor = amount date
```

---

### **4. Dashboard-Level Field Control (`dashboard.xml`)**
```xml
<dashboard>
  <label>Customer Data</label>
  <row>
    <panel>
      <table>
        <search>index=customers</search>
        <fields role_analyst="name,company,industry">
        <fields role_support="name,email,phone">
      </table>
    </panel>
  </row>
</dashboard>
```

---

### **5. Implementation Steps**

1. **Create Field Filtering Roles**:
   ```bash
   splunk add role field_filtered_analyst \
     -srchIndexesAllowed main \
     -importRoles user \
     -capabilities srchFieldsFilter
   ```

2. **Apply to Existing Roles**:
   ```ini
   # authorize.conf
   [admin]
   srchFieldsFilter = none

   [power]
   srchFieldsFilter = partial

   [user]
   srchFieldsFilter = strict
   ```

3. **Deploy Configuration**:
   ```bash
   # Push to all search heads
   splunk apply cluster-bundle -auth admin:changeme
   ```

---

### **6. Field Filtering SPL Examples**
#### **Basic Field Masking**
```sql
index=security | eval password=if(match(role,"admin"),password,"***REDACTED***")
```

#### **Role-Based Field Selection**
```sql
| eval display_field=case(
    match(role,"admin"),full_data,
    match(role,"analyst"),masked_data,
    true(),"unauthorized"
  )
```

---

### **7. Monitoring & Auditing**
#### **Audit Query**
```sql
| audit action=search fields=user,roles,search_query 
| where match(search_query, "\S+=\S+") 
| stats count by user,roles
```

#### **Field Access Report**
```sql
| rest /services/admin/conf-authorize 
| search title=role_* 
| table title, srchFieldsFilter
```

---

### **Key Security Controls**
1. **Field Encryption**:
   ```ini
   # indexes.conf
   [financial]
   field.credit_card.encryption = aes256
   ```

2. **Dynamic Masking**:
   ```ini
   # props.conf
   [host::prod-db*]
   SEDCMD-mask = s/(\d{4})-(\d{4})-(\d{4})-(\d{4})/XXXX-XXXX-XXXX-\4/g
   ```

3. **DR Site Considerations**:
   ```ini
   # Global override for DR sites
   [role_restrictions:emergency_admin]
   override_filtering = true
   ```

---
---

**specialized field filtering implementations** for compliance regimes in our Splunk environment:

---

### **1. PCI DSS Compliance (Credit Card Data)**
#### **Index-Time Masking (`props.conf`)**
```ini
[source::.../payment_logs.log]
TRANSFORMS-pci_mask = mask_ccn, mask_cvv
SEDCMD-strip-track = s/%B\d{16}\^\d{3}//g
```

#### **Transforms Rules (`transforms.conf`)**
```ini
[mask_ccn]
REGEX = \b(?:\d[ -]*?){13,16}\b
FORMAT = PCI-CCN-$1-XXXX
DEST_KEY = _raw

[mask_cvv]
REGEX = \b\d{3,4}\b
FORMAT = XXX
DEST_KEY = _raw
```

#### **Role-Based Access (`authorize.conf`)**
```ini
[role_pci_auditor]
whitelist0 = cardholder_name
whitelist1 = last_four_digits
search_filter = NOT _raw="*PCI-CCN*"
```

---

### **2. HIPAA Compliance (PHI/PII)**
#### **Field-Level Encryption (`indexes.conf`)**
```ini
[medical_records]
field.patient_id.encryption = aes256-gcm
field.diagnosis.encryption = aes256-gcm
```

#### **De-Identification Rules (`props.conf`)**
```ini
[hipaa:patient_logs]
ANNOTATE-PHI = {"action":"redact","patterns":["\\b\\d{3}-\\d{2}-\\d{4}\\b"]}
TRANSFORMS-hipaa = hash_patient_id, generalize_zip
```

#### **Role-Based PHI Access**
```ini
[role_hipaa_admin]
unencrypted_fields = patient_id, diagnosis
[role_hipaa_analyst]
encrypted_fields = *
```

---

### **3. GDPR Right-to-Be-Forgotten**
#### **Data Purge Procedure**
1. **Identify Records**:
   ```sql
   | tstats summariesonly=t count where index=customers AND email="user@example.com"
   ```

2. **Anonymize Fields**:
   ```ini
   # transforms.conf
   [gdpr_anon]
   REGEX = (name|email|phone)=([^&]+)
   FORMAT = $1=ANON-$2
   DEST_KEY = _raw
   ```

3. **SmartStore-Specific Deletion**:
   ```bash
   splunk remove gdpr-data -index customers -email user@example.com -bucket gdpr-archive
   ```

#### **Automated Workflow**
```python
#!/usr/bin/env python
import splunklib.client as client
service = client.connect(host='splunk-gdpr-proxy')
jobs = service.jobs
jobs.create(
    'search index=* | eval _raw=replace(_raw, "user@example.com", "ANON-USER")',
    exec_mode='oneshot',
    output_mode='json'
)
```

---

### **4. Cross-Site Compliance Enforcement**
#### **Federated Search Restrictions**
```ini
[federated_provider:siteB]
field_whitelist = event_id,timestamp,severity
field_blacklist = *_raw,credit_card
```

#### **SmartStore Compliance Tagging**
```ini
# indexes.conf
[pci_data]
compliance_tags = {"pci":"true","retention":"365d"}
```

---

### **5. Audit & Monitoring**
#### **Compliance Dashboard SPL**
```sql
| rest /services/admin/conf-compliance 
| search tag=hipaa OR tag=pci 
| stats count by index, tag, encryption_status
```

#### **Field Access Alert**
```sql
| audit action=search 
| search NOT roles IN ("admin","compliance_officer") 
    AND search="*credit_card*" 
| stats count by user, search_query
```

---

### **Implementation Checklist**
1. [ ] Deploy encryption to all SmartStore buckets containing PII
2. [ ] Test field masking with `| eval _raw` searches
3. [ ] Validate role-based restrictions with:
   ```bash
   splunk search "| metadata type=fields" -auth testuser:password
   ```


