# **Fabric Lakehouse Consolidation Tool**


###### This notebook provides an automated solution for creating and managing consolidated lakehouses in Microsoft Fabric.
###### It allows you to create a unified lakehouse with shortcuts to multiple source lakehouses, organized by schemas, with intelligent filtering and refresh capabilities.

--- 
## What this tool does

###### This tool automates the creation of a **consolidated lakehouse** that acts as a centralized view of multiple source lakehouses (e.g., Bronze, Silver, Gold layers in a medallion architecture). Instead of copying data, it creates **shortcuts** (symbolic links) to tables in source lakehouses, organizing them into schemas for easy navigation.

---


## 📋 Table of Contents

- [Example Architecture]¡
- [Key Features]
- [Prerequisites]
- [Configuration]
- [Usage Scenarios]
- [Best Practices]
- [Troubleshooting]
- [Support]

## 🏗️ Example Architecture

```
Consolidated Lakehouse
├── bronze_schema/
│   ├── raw_customers (shortcut → Bronze LH)
│   ├── raw_orders (shortcut → Bronze LH)
│   └── raw_products (shortcut → Bronze LH)
├── silver_schema/
│   ├── cleaned_customers (shortcut → Silver LH)
│   ├── cleaned_orders (shortcut → Silver LH)
│   └── cleaned_products (shortcut → Silver LH)
└── gold_schema/
    ├── dim_customer (shortcut → Gold LH)
    ├── dim_product (shortcut → Gold LH)
    └── fact_sales (shortcut → Gold LH)
```

### Benefits

✅ Single point of access to all data layers  
✅ No data duplication (shortcuts only)  
✅ Organized by business domains/schemas  
✅ Easy to maintain and refresh  
✅ Source lakehouses can be in different workspaces  

## ✨ Key Features

### 1. Initial Setup
Create a new consolidated lakehouse with shortcuts to all specified source lakehouses.

### 2. Intelligent Refresh
Automatically detect and create shortcuts for new tables that have been added to source lakehouses since the last run. Existing shortcuts are skipped gracefully (no errors).

### 3. Add New Sources
Dynamically add new data sources to an existing consolidated lakehouse without rebuilding everything.

### 4. Table Filtering
Control exactly which tables are included from each source:
- **Include all tables:** `"table_filter": []`
- **Include specific tables:** `"table_filter": ["table1", "table2", "table3"]`

### 5. Cross-Workspace Support
Source lakehouses can be located in different workspaces. The tool handles authentication and access automatically.

### 6. Error Handling
- Gracefully handles existing shortcuts (no duplicate errors)
- Validates lakehouse access before processing
- Detailed error reporting for failed operations
- Continues processing even if individual tables fail

## 📦 Prerequisites

### Required Permissions

Your account needs:
- **Read** permissions on all source workspace(s) and lakehouse(s)
- **Write** permissions on the target workspace where the consolidated lakehouse will be created
- **Contributor** or **Admin** role is recommended

### Required Libraries

The notebook uses:
- `requests` - HTTP library (pre-installed)
- `mssparkutils` - Fabric utilities (pre-installed)
- `json`, `time`, `typing` - Standard Python libraries

## ⚙️ Configuration

### Step 1: Define Target Workspace

```python
# Workspace where the consolidated lakehouse will be created
target_workspace_id = "your-target-workspace-id"
```

**How to find Workspace ID:**
1. Navigate to your workspace in Fabric
2. Look at the URL: `https://app.fabric.microsoft.com/groups/{workspace-id}/...`
3. Copy the GUID between `/groups/` and the next `/`

### Step 2: Configure Source Lakehouses

```python
sources = {
    "bronze": {
        "workspace_id": "workspace-id-where-bronze-lives",
        "lakehouse_id": "bronze-lakehouse-id",
        "schema_name": "bronze_schema",
        "table_filter": []  # Empty = all tables
    },
    "silver": {
        "workspace_id": "workspace-id-where-silver-lives",
        "lakehouse_id": "silver-lakehouse-id",
        "schema_name": "silver_schema",
        "table_filter": ["customers", "orders"]  # Only these tables
    },
    "gold": {
        "workspace_id": "workspace-id-where-gold-lives",
        "lakehouse_id": "gold-lakehouse-id",
        "schema_name": "gold_schema",
        "table_filter": []  # All tables
    }
}
```

**Configuration Parameters:**
- `workspace_id`: Workspace GUID where the source lakehouse is located
- `lakehouse_id`: Lakehouse GUID of the source
- `schema_name`: Name for the schema in the consolidated lakehouse (appears as folder)
- `table_filter`: List of table names to include (empty list = all tables)

**How to find Lakehouse ID:**
1. Open the lakehouse in Fabric
2. Look at the URL: `https://app.fabric.microsoft.com/groups/{workspace-id}/lakehouses/{lakehouse-id}`
3. Copy the GUID after `/lakehouses/`

## 🚀 Usage Scenarios

### Scenario 1: Initial Setup (First Time)

**When to use:** You're creating a consolidated lakehouse for the first time.

```python
# Define your sources (see Configuration section)
sources = {
    "bronze": {...},
    "silver": {...},
    "gold": {...}
}

# Create the consolidated lakehouse
consolidated_lh_id = setup_consolidated_lakehouse(
    target_workspace_id=target_workspace_id,
    new_lakehouse_name="My_Consolidated_Lakehouse",
    sources_config=sources
)

print(f"✅ Consolidated Lakehouse created with ID: {consolidated_lh_id}")
```

**What happens:**
- Creates a new lakehouse in the target workspace
- Connects to each source lakehouse
- Creates shortcuts for all tables (respecting filters)
- Organizes shortcuts into schemas
- Provides detailed statistics

<details>
<summary><strong>Expected Output</strong></summary>

```
======================================================================
Starting consolidated lakehouse setup: My_Consolidated_Lakehouse
======================================================================

[STEP 1] Creating target lakehouse...
✅ Lakehouse created successfully
   ID: abc-123-def-456
   Workspace: xyz-789-workspace

======================================================================
[SOURCE] BRONZE
======================================================================
  Workspace ID: bronze-workspace-123
  Lakehouse ID: bronze-lh-456
  Target Schema: bronze_schema
  Table Filter: All tables

  🔍 Verifying access to source lakehouse...
  ✅ Lakehouse accessible

  📋 Getting table list...
  ✅ 25 tables found

  🔗 Creating shortcuts...
     [  1/25] customers... ✅
     [  2/25] orders... ✅
     ...

  📊 Summary for bronze:
     ✅ Successfully created: 25
     ❌ Failed: 0

[Processing continues for silver and gold...]

======================================================================
FINAL SUMMARY
======================================================================
  Lakehouse: My_Consolidated_Lakehouse
  ID: abc-123-def-456
  Workspace: xyz-789-workspace

  📊 Statistics:
     Total tables processed: 75
     ✅ Shortcuts created: 75
     ❌ Shortcuts failed: 0
     📈 Success rate: 100.0%
======================================================================
```
</details>

### Scenario 2: Refresh (Add New Tables)

**When to use:** New tables have been added to your source lakehouses and you want to add them to the consolidated lakehouse.

```python
# Use the same sources configuration as initial setup
sources = {
    "bronze": {...},
    "silver": {...},
    "gold": {...}
}

# Refresh to catch new tables
refresh_stats = refresh_consolidated_lakehouse(
    target_workspace_id=target_workspace_id,
    target_lakehouse_id="your-existing-consolidated-lakehouse-id",
    sources_config=sources
)
```

**What happens:**
- Connects to each source lakehouse
- Gets current list of tables
- Compares with existing shortcuts
- Creates shortcuts ONLY for new tables
- Skips existing shortcuts (no errors)

**Use Cases:**
- Daily/weekly scheduled refresh to catch new tables
- After ETL processes add new tables to source lakehouses
- Maintenance after data pipeline updates

### Scenario 3: Add a New Source

**When to use:** You want to add a completely new data source (e.g., adding a "Platinum" layer or external data) to your existing consolidated lakehouse.

```python
# Define the new source
platinum_source = {
    "workspace_id": "platinum-workspace-id",
    "lakehouse_id": "platinum-lakehouse-id",
    "schema_name": "platinum_schema",
    "table_filter": []  # All tables, or specify specific ones
}

# Add to existing consolidated lakehouse
add_stats = add_source_to_lakehouse(
    target_workspace_id=target_workspace_id,
    target_lakehouse_id="your-existing-consolidated-lakehouse-id",
    source_name="platinum",
    source_config=platinum_source
)
```

### Scenario 4: Update Table Filters

**When to use:** You want to add more tables from an existing source that was previously filtered.

**Example:** You originally only included `["customers", "orders"]` from Silver, but now you want to add "products" and "inventory".

```python
# Original configuration
original_silver = {
    "workspace_id": "silver-workspace",
    "lakehouse_id": "silver-lh",
    "schema_name": "silver_schema",
    "table_filter": ["customers", "orders"]
}

# Updated configuration - add more tables
updated_silver = {
    "workspace_id": "silver-workspace",
    "lakehouse_id": "silver-lh",
    "schema_name": "silver_schema",
    "table_filter": ["customers", "orders", "products", "inventory"]  # Added 2 tables
}

# Refresh with updated configuration
refresh_stats = refresh_consolidated_lakehouse(
    target_workspace_id=target_workspace_id,
    target_lakehouse_id="your-consolidated-lakehouse-id",
    sources_config={"silver": updated_silver}  # Only refresh silver
)
```

### Scenario 5: Scheduled Maintenance

**When to use:** You want to run a regular job (daily/weekly) to keep your consolidated lakehouse up to date.

```python
def daily_refresh_job():
    """
    Scheduled job to refresh consolidated lakehouse
    Can be triggered by Fabric scheduling or orchestration
    """
    import datetime
    
    print(f"Starting scheduled refresh: {datetime.datetime.now()}")
    
    # Your standard configuration
    sources = {
        "bronze": {...},
        "silver": {...},
        "gold": {...}
    }
    
    # Refresh
    refresh_stats = refresh_consolidated_lakehouse(
        target_workspace_id=target_workspace_id,
        target_lakehouse_id="your-consolidated-lakehouse-id",
        sources_config=sources
    )
    
    # Log results
    if refresh_stats:
        print(f"✅ Refresh completed:")
        print(f"   New shortcuts: {refresh_stats['success']}")
        print(f"   Unchanged: {refresh_stats['skipped']}")
        print(f"   Failed: {refresh_stats['failed']}")
        
        # Alert if failures
        if refresh_stats['failed'] > 0:
            print("⚠️  WARNING: Some shortcuts failed - review logs")
    
    return refresh_stats

# Run the job
daily_refresh_job()
```

**Scheduling Options:**
- **Fabric Pipeline:** Create a pipeline with a Notebook activity
- **Cron Job:** If using external orchestration
- **Manual:** Run on-demand when needed

## 📖 Best Practices

### Keep a Record of Your Configuration

```python
# Configuration Documentation
"""
Consolidated Lakehouse: Analytics_Hub
Created: 2026-01-15
Owner: Data Team
Purpose: Unified view of medallion architecture

Sources:
- Bronze: Raw data from all systems (120 tables)
- Silver: Cleaned and validated (85 tables, filtered)
- Gold: Business-ready aggregations (45 tables)

Refresh Schedule: Daily at 2 AM UTC
Last Updated: 2026-01-15
"""

sources = {
    # ... your configuration
}
```

## 🔧 Troubleshooting

### Common Issues and Solutions

#### Issue 1: "Cannot access lakehouse"

```
❌ Cannot access lakehouse bronze
   Check permissions and verify the ID is correct
```

**Solutions:**
- Verify the lakehouse ID is correct
  - Open lakehouse in browser
  - Check URL for correct GUID
- Check workspace permissions (you need at least Read access)
- Verify the lakehouse hasn't been deleted or renamed

**How to verify:**

```python
# Test lakehouse access manually
test_url = f"{base_url}/workspaces/{workspace_id}/lakehouses/{lakehouse_id}"
response = requests.get(test_url, headers=headers)
print(f"Status: {response.status_code}")
print(f"Response: {response.text}")
```

#### Issue 2: "No tables found"

```
⚠️  No tables found in bronze
```

**Solutions:**
- Check if source lakehouse actually has tables
  - Open lakehouse in Fabric UI
  - Verify tables exist in Tables section
- Tables might be in Files section (not supported - shortcuts only work with Delta tables)
- Check if tables are still loading

#### Issue 3: Filtered tables not found

```
⚠️  No tables match the filter criteria
```

**Solutions:**
- Check spelling of table names in filter (table names are case-sensitive)
- Remove filter temporarily to see all available tables:

```python
# Temporarily set filter to empty to see all tables
"table_filter": []
```

#### Issue 4: Some shortcuts fail

```
❌ Shortcuts failed: 3
```

**Solutions:**
- Check the detailed error messages in output
- Common causes:
  - Source table was deleted
  - Permission changes
  - Network issues
- Run refresh again - transient errors often resolve
- Check specific table:

```python
# Debug specific table
tables = get_lakehouse_tables(workspace_id, lakehouse_id)
print([t['name'] for t in tables])
```

#### Issue 5: Rate limiting errors

```
Error: 429 - Too Many Requests
```

**Solutions:**
- The tool includes automatic delays (`time.sleep(0.5)`)
- If still occurring, increase delay:

```python
# In process_source function, increase sleep time
time.sleep(1.0)  # Instead of 0.5
```

- Process sources in smaller batches

#### Issue 6: Authentication errors

```
Error: 401 - Unauthorized
```

**Solutions:**
- Token may have expired - rerun the notebook
- Verify you're using the correct authentication:

```python
token = mssparkutils.credentials.getToken("pbi")
```

- Check if workspace/lakehouse access was revoked

### Debug Mode

Enable detailed logging for troubleshooting:

```python
# Add at the top of your notebook
DEBUG = True

# Modify functions to include debug output
if DEBUG:
    print(f"DEBUG: Attempting to create shortcut")
    print(f"  Target: {target_lakehouse_id}")
    print(f"  Source: {source_lakehouse_id}")
    print(f"  Table: {table_name}")
```

## 📚 Useful Links

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [OneLake Shortcuts Documentation](https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts)
- [Fabric REST API Reference](https://learn.microsoft.com/en-us/rest/api/fabric/)

## 📝 Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-15 | Initial release |

## 💬 Support and Feedback

For questions, issues, or suggestions:

- **Contact:** eneko.egiguren.gomez@gmail.com
- **LinkedIn:** [Eneko Egiguren](https://www.linkedin.com/in/enekoegiguren/)

---

**⚠️ Important Note:** This tool creates shortcuts, not copies. Changes in source lakehouses are immediately reflected in the consolidated lakehouse. There is no data duplication, which keeps storage costs low but means source lakehouses must remain accessible.
