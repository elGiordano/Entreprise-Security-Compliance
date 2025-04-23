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

