# KQL Queries for Azure Rationalization

Collection of Azure Resource Graph queries for discovering, analyzing, and monitoring AppID tagging compliance.

---

## Available Queries

### 01-untagged-resources.kql
**Purpose:** Find all resources missing the AppID tag  
**Output:** List of resources with subscription, resource group, type, name, location  
**Use Case:** Initial discovery, weekly compliance audits

**Run it:**
```bash
az graph query -q @kql-queries/01-untagged-resources.kql --output table
```

**Expected Output:**
```
SubscriptionName    ResourceGroupName    ResourceType                ResourceName
Production-East     rg-web-prod          Microsoft.Compute/vm        vm-web-01
Production-East     rg-db-prod           Microsoft.Sql/servers       sql-main
```

---

### 02-appid-coverage.kql
**Purpose:** Calculate tagging compliance percentage by subscription  
**Output:** Compliance metrics per subscription  
**Use Case:** Executive dashboards, KPI tracking

**Run it:**
```bash
az graph query -q @kql-queries/02-appid-coverage.kql --output table
```

**Expected Output:**
```
SubscriptionId    TotalResources    TaggedResources    CompliancePercent    Status
abc-123-def       842               678                80.5                 Good
ghi-789-jkl       500               380                76.0                 Needs Work
```

---

### 03-cost-by-appid.kql
**Purpose:** Aggregate Azure spending by AppID  
**Output:** Monthly cost per application  
**Use Case:** Chargeback, showback, budget analysis

**Run it:**
```bash
az graph query -q @kql-queries/03-cost-by-appid.kql --output table
```

**Expected Output:**
```
AppID             MonthlySpend
CORP-ERP-001      $12,500
CORP-CRM-002      $8,200
DEPT-FIN-001      $4,100
```

**Note:** For large environments (30K+ resources), consider using Azure FinOps Toolkit with Parquet files instead of Cost Management API.

---

### 04-orphaned-detection.kql
**Purpose:** Find resources tagged with decommissioned AppIDs  
**Output:** Resources that should be cleaned up  
**Use Case:** Post-decommission cleanup, cost reduction

**Prerequisites:**
1. Maintain a list of active AppIDs
2. Update the `ActiveAppIDs` array in the query

**Run it:**
```bash
az graph query -q @kql-queries/04-orphaned-detection.kql --output table
```

**Expected Output:**
```
AppID              ResourceCount    MonthlyWaste
LEGACY-APP-001     142              $2,300
RETIRED-CRM-002    68               $1,100
```

---

### 05-resource-distribution.kql
**Purpose:** Understand application footprint across Azure  
**Output:** Resource counts, types, and locations per AppID  
**Use Case:** Architecture reviews, capacity planning

**Run it:**
```bash
az graph query -q @kql-queries/05-resource-distribution.kql --output table
```

**Expected Output:**
```
AppID            TotalResources    TypeCount    SubscriptionCount    LocationCount
CORP-ERP-001     245              12           2                    1
CORP-CRM-002     156              8            1                    2
```

---

## Query Tips

### Best Practices

#### 1. **Start Broad, Then Narrow**
```kql
// Start with this
Resources | where tags !has 'AppID'

// Then filter
Resources 
| where tags !has 'AppID'
| where subscriptionId == 'your-sub-id'
| where resourceGroup startswith 'rg-prod'
```

#### 2. **Use `summarize` for Aggregation**
```kql
Resources
| where tags !has 'AppID'
| summarize Count=count() by type
| order by Count desc
```

#### 3. **Join with Cost Data Carefully**
```kql
// Good: Specific date range
Resources
| join (
    CostManagementResources
    | where properties.date >= ago(30d)
) on $left.id == $right.resourceId

// Bad: No date filter (slow query)
Resources
| join CostManagementResources on id
```

#### 4. **Extract Tags Properly**
```kql
// Correct way
Resources
| extend AppID = tostring(tags.AppID)
| where isnotempty(AppID)

// Wrong way (won't work)
Resources
| where tags.AppID != ""
```

---

## Performance Considerations

### Slow Queries
**Symptom:** Query times out or takes >30 seconds

**Causes:**
- Joining with unfiltered cost data
- No subscription filter on large tenants
- Complex string operations

**Solutions:**
```kql
// Add subscription filter
| where subscriptionId in ('sub1', 'sub2', 'sub3')

// Limit cost data timeframe
| where properties.date >= ago(30d)

// Use summarize instead of project (faster)
| summarize Count=count() by type
```

---

### Large Result Sets
**Symptom:** Query returns 10,000+ rows, truncated results

**Solutions:**
```kql
// Option 1: Aggregate first
Resources
| where tags !has 'AppID'
| summarize Count=count() by subscriptionId, resourceGroup
| order by Count desc

// Option 2: Paginate results
Resources
| where tags !has 'AppID'
| take 1000
```

---

## Advanced Patterns

### Pattern 1: Tag Validation (Regex)
```kql
Resources
| extend AppID = tostring(tags.AppID)
| where isnotempty(AppID)
| where AppID !matches regex "^(CORP|DEPT|SHARED)-[A-Z0-9]+-[0-9]{3}$"
| project name, AppID, resourceGroup
```

**Use Case:** Find malformed AppID values.

---

### Pattern 2: Cross-Subscription Analysis
```kql
Resources
| extend AppID = tostring(tags.AppID)
| where isnotempty(AppID)
| summarize 
    Subscriptions = make_set(subscriptionId),
    ResourceCount = count()
by AppID
| where array_length(Subscriptions) > 3
| project AppID, ResourceCount, SubscriptionCount = array_length(Subscriptions)
```

**Use Case:** Find applications spanning many subscriptions (potential scope creep).

---

### Pattern 3: Tag Drift Detection
```kql
Resources
| extend 
    AppID = tostring(tags.AppID),
    Environment = tostring(tags.Environment),
    Owner = tostring(tags.Owner)
| where isnotempty(AppID)
| summarize 
    Environments = make_set(Environment),
    Owners = make_set(Owner)
by AppID
| where array_length(Environments) > 3 or array_length(Owners) > 1
```

**Use Case:** Find inconsistent tagging within an AppID (e.g., multiple owners).

---

## Exporting Results

### CSV Export
```bash
az graph query -q @kql-queries/01-untagged-resources.kql \
    --query "data" \
    --output json | jq -r '.[] | [.SubscriptionName, .ResourceGroupName, .ResourceName] | @csv' \
    > untagged-resources.csv
```

### JSON Export (for Power BI)
```bash
az graph query -q @kql-queries/02-appid-coverage.kql \
    --query "data" \
    --output json \
    > compliance-data.json
```

### HTML Table
```bash
az graph query -q @kql-queries/01-untagged-resources.kql \
    --output table \
    > report.txt
```

---

## Scheduling Queries

### Weekly Compliance Report (PowerShell)
```powershell
# Save as: Weekly-Compliance-Report.ps1
$date = Get-Date -Format "yyyy-MM-dd"
$output = "compliance-$date.json"

az graph query -q @kql-queries/02-appid-coverage.kql `
    --query "data" `
    --output json > $output

# Email report (requires SMTP setup)
Send-MailMessage `
    -To "governance@company.com" `
    -Subject "Weekly AppID Compliance: $date" `
    -Body "See attached report" `
    -Attachments $output
```

### Azure Automation Runbook
```powershell
# Run this query in Azure Automation
param([string]$SubscriptionId)

Connect-AzAccount -Identity

$query = Get-Content "./kql-queries/02-appid-coverage.kql" -Raw
$results = Search-AzGraph -Query $query -Subscription $SubscriptionId

# Store in Log Analytics or send to webhook
```

---

## Troubleshooting

### Issue 1: "Query syntax error"
**Cause:** KQL is case-sensitive, typo in field names

**Solution:**
```kql
// Wrong
resources | where Tags has 'appid'

// Correct
Resources | where tags has 'AppID'
```

---

### Issue 2: "Empty results (but resources exist)"
**Cause:** Missing subscription scope

**Solution:**
```bash
# Specify subscriptions
az graph query -q @kql-queries/01-untagged-resources.kql \
    --subscriptions "sub1" "sub2" "sub3"
```

---

### Issue 3: "Cost data not appearing"
**Cause:** Cost Management Reader role missing

**Solution:**
```bash
# Assign role
az role assignment create \
    --assignee user@company.com \
    --role "Cost Management Reader" \
    --scope "/subscriptions/your-sub-id"
```

---

## Resources

- **Azure Resource Graph Docs:** https://learn.microsoft.com/azure/governance/resource-graph/
- **KQL Reference:** https://learn.microsoft.com/azure/data-explorer/kusto/query/
- **Cost Management API:** https://learn.microsoft.com/azure/cost-management-billing/costs/cost-analysis-api

---

## Contributing

Found a better query? Optimized performance? Submit a PR!

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.
