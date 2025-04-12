# Splunk Custom Index: Creation and Management

## What is a Custom Index in Splunk?

A custom index in Splunk is a user-defined container for storing and organizing specific types of data separately from the default indexes (`main`, `_internal`, `_audit`, etc.). Creating custom indexes allows for:

- Better data segregation and organization
- Fine-grained access control
- Customized retention policies
- Improved search performance for specific data sets

## Creating a Custom Index

### Method 1: Using Splunk Web UI

1. Navigate to **Settings** → **Indexes**
2. Click **New Index**
3. Configure index properties:
   - **Index name**: Follow naming conventions (lowercase, no spaces)
   - **Data type**: Select appropriate type
   - **Storage settings**: Configure max size and retention
   - **Permissions**: Set role-based access

### Method 2: Using indexes.conf

1. Create or edit `$SPLUNK_HOME/etc/system/local/indexes.conf` or create an app-specific version
2. Add your index definition:

```
[<your_index_name>]
homePath = $SPLUNK_DB/<your_index_name>/db
coldPath = $SPLUNK_DB/<your_index_name>/colddb
thawedPath = $SPLUNK_DB/<your_index_name>/thaweddb
maxTotalDataSizeMB = <size_in_MB>
frozenTimePeriodInSecs = <retention_period_in_seconds>
```

3. Restart Splunk for changes to take effect

## Best Practices for Custom Indexes

1. **Naming conventions**: Use lowercase, descriptive names (e.g., `web_logs`, `firewall_events`)
2. **Retention policies**: Set appropriate data retention based on:
   - Compliance requirements
   - Storage constraints
   - Business needs
3. **Access control**: Use Splunk roles to restrict access to sensitive indexes
4. **Performance considerations**:
   - Distribute high-volume data across multiple indexes
   - Consider index size limits to prevent storage issues
5. **Documentation**: Maintain documentation of all custom indexes and their purposes

## Managing Custom Indexes

- **Monitor index sizes**: Use `| dbinspect` or Splunk Monitoring Console
- **Adjust settings**: Modify retention or size limits as needed
- **Archive/delete**: Implement processes for handling aged-out data
- **Reindexing**: Consider reindexing if schema changes are needed

## Common Commands for Working with Custom Indexes

```
# Search a specific index
index=<your_index_name> <search_query>

# List all indexes
| eventcount summarize=false index=* | dedup index | table index

# Check index data volume
| dbinspect index=<your_index_name>
```

