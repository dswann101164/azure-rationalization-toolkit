# Azure Rationalization Toolkit

> **"Subscriptions are for security. AppID is for truth."**

A practical toolkit for discovering, tagging, and rationalizing Azure resources at enterprise scale. Built on real-world experience managing 44+ subscriptions and 31,000+ resources.

## 🎯 The Problem

You inherit an Azure environment. Resources everywhere. No consistent tagging. Multiple owners. Shadow IT. The CFO asks: *"What applications are we running and what do they cost?"*

Your subscriptions won't tell you. Your resource groups won't tell you. Only **ApplicationID tags** will tell you.

## 💡 The Solution

This toolkit implements **Step Zero** of software rationalization:

1. **Discover** - Find all resources, regardless of subscription structure
2. **Tag** - Enforce ApplicationID tagging via Azure Policy
3. **Report** - Build reliable cost and inventory reports by application
4. **Rationalize** - Make informed decisions about what to keep, consolidate, or kill

## 📦 What's Inside

### KQL Queries (`/queries/`)
- Discover resources missing ApplicationID tags
- Find orphaned resources (no owner, no purpose)
- Identify resources with multiple AppIDs (merge candidates)
- Generate AppID coverage reports across subscriptions

### Azure Policies (`/policies/`)
- Require ApplicationID tag on all new resources
- Audit existing resources for missing tags
- Deny deployments without proper tagging

### PowerShell Scripts (`/scripts/`)
- Bulk tag resources based on resource group patterns
- Export AppID inventory to CSV
- Generate cost allocation reports

### Documentation (`/docs/`)
- Getting started guide
- Step Zero implementation roadmap
- Best practices for tag governance

## 🚀 Quick Start

### Prerequisites
- Azure subscription with Contributor or Owner access
- Azure CLI or PowerShell Az module
- Access to Azure Resource Graph

### 1. Run Discovery Query

Copy `/queries/01-discover-appid-gaps.kql` into Azure Resource Graph Explorer:

```kql
Resources
| where subscriptionId in ('sub1', 'sub2', 'sub3')
| where isempty(tags['ApplicationID'])
| summarize ResourceCount = count() by type, resourceGroup, subscriptionId
| order by ResourceCount desc
```

This shows you where tagging gaps exist.

### 2. Deploy Policy

Use `/policies/require-appid-tag.json` to enforce tagging:

```bash
az policy assignment create \
  --name 'require-appid' \
  --policy 'require-appid-tag' \
  --scope '/subscriptions/YOUR_SUB_ID'
```

### 3. Bulk Tag Existing Resources

Use `/scripts/bulk-tag-resources.ps1`:

```powershell
.\bulk-tag-resources.ps1 -SubscriptionId "YOUR_SUB" -AppID "APP-001"
```

### 4. Generate Reports

Run `/queries/04-appid-coverage-report.kql` to measure progress:

```kql
Resources
| summarize Total = count(), 
            Tagged = countif(isnotempty(tags['ApplicationID'])),
            Coverage = round(100.0 * countif(isnotempty(tags['ApplicationID'])) / count(), 1)
by subscriptionId
```

## 📊 Real-World Results

This approach has been tested at scale:
- **44 subscriptions** across dev, test, prod
- **31,000+ resources** (VMs, databases, storage, networking)
- **13+ months** of cost history tracked by ApplicationID
- **Sub-3-second queries** using Parquet-optimized data lake

## 🏗️ Architecture Philosophy

### Why ApplicationID?

| Dimension | Purpose | Limitation |
|-----------|---------|------------|
| **Subscription** | Security boundary | Changes too often |
| **Resource Group** | Deployment unit | Inconsistent naming |
| **Tags** | Metadata | Only if enforced |
| **ApplicationID** | Source of truth | Requires governance |

**Key insight:** Subscriptions come and go. Applications live for years. Tag by application, report by application, rationalize by application.

## 🎓 Learn More

This toolkit is based on the blog post: [Software Rationalization: Step Zero is NOT What You Think](https://azure-noob.com/blog/software-rationalization-step-zero-devops/)

**Read the post for:**
- Why most rationalization projects fail (they start at the wrong end)
- The 4-stage implementation roadmap
- How to handle "non-production" subscriptions
- Real anonymized numbers from a 44-subscription environment

## 🤝 Contributing

Found a better KQL query? Have a tagging strategy that works? **Pull requests welcome!**

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## 📝 License

MIT License - See [LICENSE](LICENSE) for details.

## ⚠️ Disclaimer

This toolkit contains real patterns from production environments, but:
- All numbers are anonymized or approximated
- Queries may need tuning for your environment
- Policies should be tested in dev/test first
- Always validate with your security team

## 🔗 Links

- **Blog:** [azure-noob.com](https://azure-noob.com)
- **Issues:** Report bugs or request features
- **Discussions:** Share your rationalization war stories

---

**Built by someone who's been there.** No theory. No consultants. Just real Azure at scale.
