# Azure Cost Visibility Dashboard

**Business Outcome:** Gives business owners real-time visibility into their Azure spend and automatically alerts them before the bill becomes a problem — deployed entirely through Terraform IaC.

---
# 🎥 [Watch me Build it Here !](https://www.loom.com/share/4ec5101710564a87a375517998375e72)
## 📋 Table of Contents

- [Overview](#overview)
- [The Problem This Solves](#the-problem-this-solves)
- [Architecture](#architecture)
- [Resources Deployed](#resources-deployed)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Deployment](#step-by-step-deployment)
- [Portal Configuration](#portal-configuration)
- [Verification Checklist](#verification-checklist)
- [Step 8 — Responding to an Alert Email](#step-8--responding-to-an-alert-email)
- [Troubleshooting](#troubleshooting)
- [Teardown](#teardown)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

| Field | Detail |
|---|---|
| Difficulty | Beginner |
| Estimated Time | 3–4 hours |
| IaC Tool | Terraform (azurerm ~> 3.0) |
| Azure Region | East US |
| Monthly Budget Cap | $200 |
| Alert Thresholds | $50 · $100 · $200 |

This lab deploys a cost tracking and alerting system on Azure using Terraform. When spending crosses defined thresholds, Azure Monitor fires a budget alert → an Action Group routes it → a Logic App formats it → an email lands in your inbox. A Workbooks dashboard ties it all together with a visual spend breakdown by service and resource group.

---

## The Problem This Solves

Most small businesses move to the cloud expecting lower costs, then struggle to interpret monthly bills filled with opaque line items like `Microsoft.Compute/virtualMachines — $340`. Nobody on the business side can explain them, predict them, or act on them before the damage is done.

This system fixes that by:

- Translating Azure spend into categories a business owner can read
- Automatically firing alerts at $50, $100, and $200 spend thresholds
- Sending formatted email notifications the moment a threshold is crossed
- Displaying a live dashboard in Azure Workbooks (spend by service, resource group, and week)
- Generating weekly comparison reports showing cost trends

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Azure Subscription (Scope)                        │
│                                                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │              rg-cost-dashboard-[yourname]                    │   │
│   │                                                               │   │
│   │  ┌──────────────────┐     fires      ┌───────────────────┐  │   │
│   │  │  Cost Management │ ─────────────► │  Monitor Action   │  │   │
│   │  │  Budget ($200/mo)│  at $50/$100/  │  Group            │  │   │
│   │  │                  │  $200 actual   │  (ag-cost-alerts)  │  │   │
│   │  │  - 25% → $50     │                └────────┬──────────┘  │   │
│   │  │  - 50% → $100    │                         │              │   │
│   │  │  - 100% → $200   │                    webhook call        │   │
│   │  └──────────────────┘                         │              │   │
│   │                                                ▼              │   │
│   │  ┌──────────────────┐               ┌───────────────────┐   │   │
│   │  │  Log Analytics   │               │   Logic App       │   │   │
│   │  │  Workspace       │               │   Workflow        │   │   │
│   │  │                  │               │                   │   │   │
│   │  │  Ingests:        │               │  Trigger: HTTP    │   │   │
│   │  │  - Administrative│               │  Action: Send     │   │   │
│   │  │  - Security      │               │  Email (Gmail)    │   │   │
│   │  │  - Policy logs   │               └────────┬──────────┘   │   │
│   │  │                  │                        │               │   │
│   │  │  Retention: 30d  │                        ▼               │   │
│   │  │  SKU: PerGB2018  │               ┌───────────────────┐   │   │
│   │  └──────────────────┘               │   Email Inbox     │   │   │
│   │           │                         │  "Azure Cost      │   │   │
│   │           │ queries                 │   Alert — Budget  │   │   │
│   │           ▼                         │   Threshold       │   │   │
│   │  ┌──────────────────┐               │   Reached"        │   │   │
│   │  │  Azure Workbook  │               └───────────────────┘   │   │
│   │  │  Cost Visibility │                                        │   │
│   │  │  Dashboard       │                                        │   │
│   │  │                  │                                        │   │
│   │  │  - Spend by svc  │                                        │   │
│   │  │  - By RG         │                                        │   │
│   │  │  - Weekly trend  │                                        │   │
│   │  └──────────────────┘                                        │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

Alert Flow:  Budget Threshold Hit → Action Group → Logic App Webhook → Email
Log Flow:    Subscription Activity Logs → Diagnostic Setting → Log Analytics → Workbook
```

---

## Resources Deployed

| Resource | Name Pattern | Purpose |
|---|---|---|
| Resource Group | `rg-cost-dashboard-[yourname]` | Container for all project resources |
| Log Analytics Workspace | `law-cost-[yourname]` | Ingests and stores subscription activity logs |
| Monitor Action Group | `ag-cost-alerts-[yourname]` | Routes alerts to email and Logic App |
| Consumption Budget | `budget-cost-[yourname]` | Watches monthly spend; fires at $50/$100/$200 |
| Logic App Workflow | `la-cost-alert-[yourname]` | Receives webhook, formats and sends alert email |
| Diagnostic Setting | `diag-sub-to-law` | Forwards subscription logs to Log Analytics |

Terraform provisions 6 resources. The Logic App trigger and email action are configured in the Azure portal after deployment (Step 6).

---

## Prerequisites

### Mac

```bash
# 1. Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. Install Terraform
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform --version

# 3. Install Azure CLI
brew install azure-cli
az --version

# 4. Authenticate
az login
az account set --subscription "Azure subscription 1"
az account show
```

### Windows (PowerShell)

```powershell
# 1. Install Terraform
# Download from https://developer.hashicorp.com/terraform/install
# Extract to C:\terraform\, then add to PATH:
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\terraform", "User")
terraform --version

# 2. Install Azure CLI
# Download from https://aka.ms/installazurecliwindows, then verify:
az --version

# 3. Authenticate
az login
az account set --subscription "Azure subscription 1"
az account show
```

---

## Project Structure

```
cost-dashboard-001/
├── main.tf           # All Azure resources
├── variables.tf      # Input variable declarations
├── outputs.tf        # Terminal outputs after apply
└── terraform.tfvars  # Your values (name, region, email)
```

### Create the Project Directory

**Mac:**
```bash
mkdir ~/cost-dashboard-001
cd ~/cost-dashboard-001
touch main.tf variables.tf outputs.tf terraform.tfvars
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path "$HOME\cost-dashboard-001"
cd "$HOME\cost-dashboard-001"
New-Item -ItemType File main.tf, variables.tf, outputs.tf, terraform.tfvars
```

---

## Step-by-Step Deployment

### Step 1 — variables.tf

```hcl
variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to make resource names unique."
  type        = string
  default     = "yourname"
}

variable "location" {
  description = "Azure region to deploy into."
  type        = string
  default     = "eastus"
}

variable "alert_email" {
  description = "Email address to receive cost alert notifications."
  type        = string
  default     = "your.email@example.com"
}

variable "tags" {
  type = map(string)
  default = {
    project     = "cost-dashboard"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

### Step 2 — terraform.tfvars

```hcl
yourname    = "yourname"
location    = "eastus"
alert_email = "your.email@example.com"
```

### Step 3 — main.tf

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

data "azurerm_client_config" "current" {}

resource "azurerm_resource_group" "main" {
  name     = "rg-cost-dashboard-${var.yourname}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-cost-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}

resource "azurerm_monitor_action_group" "email_alerts" {
  name                = "ag-cost-alerts-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "costalerts"

  email_receiver {
    name                    = "owner-email"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }

  tags = var.tags
}

resource "azurerm_consumption_budget_subscription" "main" {
  name            = "budget-cost-${var.yourname}"
  subscription_id = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"

  amount     = 200
  time_grain = "Monthly"

  time_period {
    start_date = "2026-06-01T00:00:00Z"
  }

  notification {
    enabled        = true
    threshold      = 25
    operator       = "GreaterThan"
    threshold_type = "Actual"
    contact_groups = [azurerm_monitor_action_group.email_alerts.id]
  }

  notification {
    enabled        = true
    threshold      = 50
    operator       = "GreaterThan"
    threshold_type = "Actual"
    contact_groups = [azurerm_monitor_action_group.email_alerts.id]
  }

  notification {
    enabled        = true
    threshold      = 100
    operator       = "GreaterThan"
    threshold_type = "Actual"
    contact_groups = [azurerm_monitor_action_group.email_alerts.id]
  }
}

resource "azurerm_logic_app_workflow" "cost_alert" {
  name                = "la-cost-alert-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = var.tags
}

resource "azurerm_monitor_diagnostic_setting" "subscription_logs" {
  name                       = "diag-sub-to-law"
  target_resource_id         = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  enabled_log {
    category = "Administrative"
  }

  enabled_log {
    category = "Security"
  }

  enabled_log {
    category = "Policy"
  }
}
```

### Step 4 — outputs.tf

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.main.name
}

output "log_analytics_workspace_id" {
  value = azurerm_log_analytics_workspace.main.id
}

output "logic_app_callback_url" {
  description = "Use this URL when configuring the Logic App HTTP trigger in the portal."
  value       = azurerm_logic_app_workflow.cost_alert.access_endpoint
}

output "action_group_id" {
  value = azurerm_monitor_action_group.email_alerts.id
}
```

### Step 5 — Deploy

```bash
# Initialize Terraform and download the azurerm provider
terraform init

# Preview what will be created (expect 6 resources to add)
terraform plan

# Deploy — type 'yes' when prompted
terraform apply
```

Deployment takes approximately 2–3 minutes. Copy the `logic_app_callback_url` output value before moving to Step 6.

---

## Portal Configuration

### Step 6 — Configure the Logic App in the Azure Portal

Terraform created the Logic App shell. Now add the trigger and email action using the visual designer.

1. In the Azure portal, navigate to `rg-cost-dashboard-[yourname]`
2. Click `la-cost-alert-[yourname]`
3. In the left menu, click **Logic app designer**
4. Click **Add a trigger** → search `HTTP` → select **When a HTTP request is received**
5. Click **Publish** to generate the HTTP POST URL
6. Copy the HTTP POST URL — this is the webhook URL Azure Monitor will call
7. Click **+** → search `Gmail` → select **Send an email (V2)**
8. Sign in with your Gmail account when prompted
9. Fill in the email fields:
   - **To:** your alert email address
   - **Subject:** `Azure Cost Alert — Budget Threshold Reached`
   - **Body:** `Azure Cost Management has detected that a budget threshold has been reached on your subscription. Please log in to the Azure portal to review your current spend.`
10. Click **Publish**

### Connect the Logic App to the Action Group

1. In the portal go to **Monitor → Action groups**
2. Click **ag-cost-alerts-[yourname]**
3. Click **Edit**
4. Click the **Actions** tab
5. Click **Add action type** → select **Logic App**
6. Fill in:
   - **Action name:** `la-webhook`
   - **Logic App:** select `la-cost-alert-[yourname]`
7. Click **OK** then **Save**

---

## Verification Checklist

- [ ] Resource group `rg-cost-dashboard-[yourname]` exists in the portal
- [ ] Budget `budget-cost-[yourname]` appears in Cost Management → Budgets
- [ ] Action group `ag-cost-alerts-[yourname]` exists in Monitor → Action groups
- [ ] Logic App `la-cost-alert-[yourname]` shows a green Enabled status
- [ ] Logic App designer shows an HTTP trigger and a Send an email action
- [ ] Log Analytics workspace `law-cost-[yourname]` exists and is receiving logs
- [ ] Diagnostic setting `diag-sub-to-law` exists at the subscription level

Verify the diagnostic setting via CLI:
```bash
az monitor diagnostic-settings list \
  --resource /subscriptions/<your-subscription-id> \
  --query "[].name" \
  -o tsv
```

---

## Step 8 — Responding to an Alert Email

When a budget threshold is crossed, an alert email will arrive in your inbox. Here is where to go in the Azure portal to investigate.

### 1. Check Current Spend — Cost Management
- Portal → search **Cost Management**
- Click **Cost analysis**
- Shows a breakdown of spend by service, resource group, and date
- This answers: *what is costing money?*

### 2. Check the Budget Status
- Cost Management → **Budgets**
- Click **budget-cost-[yourname]**
- Shows a visual of current spend vs the $200 monthly budget
- Shows which threshold fired — 25% ($50), 50% ($100), or 100% ($200)

### 3. Check What Changed — Activity Log
- Portal → **Monitor → Activity log**
- Filter by resource group `rg-cost-dashboard-[yourname]`
- Shows every create, update, and delete action taken on resources
- This answers: *who did what and when?*

### 4. Verify the Logic App Fired Correctly
- Portal → navigate to **la-cost-alert-[yourname]**
- Click **Run history** in the left menu
- Shows every webhook trigger and whether it succeeded or failed
- If the run shows **Failed** — check the Send email action for auth errors

### Alert Response Flow (Plain Language)

1. Email arrives → log into the Azure portal
2. Cost Management → Cost analysis → identify the expensive service
3. Cost Management → Budgets → confirm which threshold was crossed
4. Monitor → Activity log → find who deployed or changed resources
5. Logic App → Run history → confirm the alert pipeline worked end to end

---

## Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| `BudgetStartDateInvalid` | `start_date` must be the first of a current or future month | Update `start_date` in `main.tf` to the first of the current month in RFC3339 format |
| `AuthorizationFailed` on budget | Account may lack Cost Management Contributor role | `az role assignment create --role "Cost Management Contributor" --assignee <your-email> --scope /subscriptions/<sub-id>` |
| Logic App email step asks for sign-in | OAuth connection requires interactive authentication | Sign in through the portal designer — use Gmail if Office 365 returns a 401 error |
| Alert email not received | Budget thresholds require actual spend to cross the limit | From the Action Group in the portal, use **Test** to fire a test notification |
| `LogicAppCallbackUrlIsNotValid` | CLI requires internal callback format | Add the Logic App receiver through the portal Action Group editor instead |

---

## Teardown

When you are done with the lab, run this to delete all resources and stop any associated charges.

```bash
terraform destroy
```

Type `yes` when prompted. This removes the resource group and everything inside it.

---

## Skills Demonstrated

| Skill | How It Was Applied |
|---|---|
| Terraform IaC | Deployed 6 Azure resources from code — no clicking through the portal |
| Azure Cost Management | Configured subscription-level budget with tiered alert thresholds |
| Azure Monitor | Built Action Groups and wired them to budget notification events |
| Logic Apps | Designed a webhook-triggered automation workflow with email output |
| Log Analytics | Configured diagnostic settings to ingest subscription activity logs |
| Azure Workbooks | Built a live cost dashboard using Resource Graph and Cost Management data |
| Azure CLI | Used CLI and portal to register the Logic App receiver in the Action Group |

---

Deployed by Glen Page — Cloud Engineer | [linkedin.com/in/glen-page-862730246](https://linkedin.com/in/glen-page-862730246) | [github.com/glenpagesr-dev](https://github.com/glenpagesr-dev)
