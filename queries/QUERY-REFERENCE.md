# Query Reference Guide

All queries are comment-free for maximum compatibility with Azure Resource Graph Explorer.

---

## 01-untagged-resources.kql

**Purpose:** Find all resources missing the AppID tag

**Returns:**
- SubscriptionName
- ResourceGroupName  
- ResourceType
- ResourceName
- Location
- CreatedDate
- ResourceId

**Use Case:** Initial discovery, weekly compliance audits

---

## 02-appid-coverage.kql

**Purpose:** Calculate tagging compliance percentage by subscription

**Returns:**
- SubscriptionId
- TotalResources
- TaggedResources
- UntaggedResources
- CompliancePercent
- ComplianceStatus (Excellent/Good/Needs Work/Critical)

**Use Case:** Executive dashboards, KPI tracking

**Interpretation:**
- 95%+ = Excellent
- 85-94% = Good
- 70-84% = Needs Work
- <70% = Critical

---

## 03-cost-by-appid.kql

**Purpose:** Group resources by AppID and show resource counts

**Returns:**
- AppID
- Environment
- CostCenter
- ResourceCount
- ResourceTypes (array)

**Use Case:** Application inventory, resource counting, initial analysis

**Note:** This is a simplified version that counts resources. For actual cost data, see `03-cost-by-appid-ADVANCED.kql` which requires Cost Management Reader permissions and may need customization for your environment.

---

## 04-orphaned-detection.kql

**Purpose:** Find resources tagged with decommissioned AppIDs

**Returns:**
- AppID
- ResourceType
- ResourceName
- ResourceGroup
- SubscriptionId
- Location
- CreatedDate
- ResourceId

**Prerequisites:**
- Edit the query and replace the example AppIDs with your active application IDs
- Update this line: `| where AppID !in ("CORP-ERP-001", "CORP-CRM-002", ...)`

**Use Case:** Post-decommission cleanup, cost reduction

**How to Customize:**
1. Open the query file
2. Find the line: `| where AppID !in ("CORP-ERP-001", "CORP-CRM-002", "CORP-HR-003", "DEPT-FINANCE-001", "DEPT-SALES-001")`
3. Replace with your active AppIDs: `| where AppID !in ("YOUR-APP-001", "YOUR-APP-002", "YOUR-APP-003")`

---

## 05-resource-distribution.kql

**Purpose:** Understand application footprint across Azure

**Returns:**
- AppID
- TotalResources
- TypeCount (number of different resource types)
- SubscriptionCount (number of subscriptions)
- LocationCount (number of regions)
- ResourceGroupCount (number of resource groups)
- ResourceTypes (array)
- Locations (array)

**Use Case:** Architecture reviews, capacity planning, security assessment

**Interpretation:**
- Simple apps: <20 resources, single subscription/location
- Complex apps: 100+ resources, multi-subscription, multi-region
- Red flags: 1 AppID spanning 10+ subscriptions or 5+ regions

---

## Running Queries

### In Azure Portal
1. Open [Azure Resource Graph Explorer](https://portal.azure.com/#view/HubsExtension/ArgQueryBlade)
2. Copy entire contents of .kql file
3. Paste into query editor
4. Click "Run query"

### Via Azure CLI
```bash
az graph query -q @queries/01-untagged-resources.kql --output table
```

### Via PowerShell
```powershell
$query = Get-Content "queries/01-untagged-resources.kql" -Raw
Search-AzGraph -Query $query | Format-Table
```

---

## Export Results

### To CSV
```bash
az graph query -q @queries/01-untagged-resources.kql --output json | \
    jq -r '.[] | [.SubscriptionName, .ResourceGroupName, .ResourceName] | @csv' > results.csv
```

### To JSON
```bash
az graph query -q @queries/02-appid-coverage.kql --output json > compliance.json
```

---

## Troubleshooting

**Query fails with parser error:**
- Make sure you copied the ENTIRE query
- Check there are no extra characters at beginning/end
- Verify you're in Azure Resource Graph Explorer (not Log Analytics)

**No results returned:**
- Verify you have Reader access to subscriptions
- Check subscription scope in Resource Graph Explorer
- For cost queries, verify Cost Management Reader role

**Cost data missing:**
- Cost queries require Cost Management Reader role
- Cost data may not be available for current day
- Some resource types don't report cost data

---

## Performance Tips

- Add subscription filter for large tenants: `| where subscriptionId in ('sub1', 'sub2')`
- Use `summarize` instead of returning all rows when possible
- Limit cost data timeframe: `| where properties.date >= ago(30d)`

---

## Need Help?

- Azure Resource Graph docs: https://learn.microsoft.com/azure/governance/resource-graph/
- KQL reference: https://learn.microsoft.com/azure/data-explorer/kusto/query/
- Open an issue: https://github.com/dswann101164/azure-rationalization-toolkit/issues
