
# Azure Cost Visibility Dashboard

> **Business Outcome:** Gives business owners real-time visibility into their Azure spend and automatically alerts them before the bill becomes a problem — deployed entirely through Terraform IaC.

---

## 📋 Table of Contents

- [Overview](#overview)
- [The Problem This Solves](#the-problem-this-solves)
- [Architecture](#architecture)
- [Resources Deployed](#resources-deployed)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Deployment](#step-by-step-deployment)
- [Portal Configuration](#portal-configuration)
- [Dashboard Setup](#dashboard-setup)
- [Verification Checklist](#verification-checklist)
- [Troubleshooting](#troubleshooting)
- [Teardown](#teardown)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

| | |
|---|---|
| **Difficulty** | Beginner |
| **Estimated Time** | 3–4 hours |
| **IaC Tool** | Terraform (azurerm ~> 3.0) |
| **Azure Region** | East US |
| **Monthly Budget Cap** | $200 |
| **Alert Thresholds** | $50 · $100 · $200 |

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
│   │  │  - Security      │               │  Email (O365)     │   │   │
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

### How the Alert Flow Works (Plain Language)

1. **Azure Cost Management** watches your subscription spend against a $200 monthly budget
2. When actual spend crosses 25%, 50%, or 100% of that budget, it fires a notification
3. The **Action Group** receives the signal and routes it to two places: directly to email, and to the Logic App via webhook
4. The **Logic App** formats the raw alert payload into a readable email and sends it via Office 365
5. The **Log Analytics Workspace** independently ingests subscription activity logs for querying
6. The **Azure Workbook** reads from Log Analytics and Cost Management to render the visual dashboard

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

> **Terraform provisions 6 resources.** The Logic App trigger and email action are configured in the Azure portal after deployment (Step 6).

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
├── main.tf           # All Azure resources (provider, budget, alerts, Logic App, diagnostics)
├── variables.tf      # Input variable declarations
├── outputs.tf        # Terminal outputs after apply (resource names, callback URL)
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

Declares the input variables Terraform expects. These keep your configuration reusable — no hardcoded names or emails in your resource blocks.

```hcl
variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to make resource names unique."
  type        = string
}

variable "location" {
  description = "Azure region to deploy into."
  type        = string
  default     = "East US"
}

variable "alert_email" {
  description = "Email address to receive cost alert notifications."
  type        = string
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

---

### Step 2 — terraform.tfvars

Supplies your actual values. Replace the email with your own before running `terraform apply`.

```hcl
yourname    = "yourname"
location    = "East US"
alert_email = "your.email@example.com"
```

---

### Step 3 — main.tf

Paste each block in order. Each section is explained so you know *why* it is written the way it is.

#### Provider and Data Sources

The `azurerm` provider is the Terraform plugin that communicates with Azure. `azurerm_client_config` reads your active `az login` session to retrieve your subscription and tenant IDs — required for scoping the budget and diagnostic settings.

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
```

#### Resource Group

Think of this as a project folder. Everything in this lab lives here. Deleting it cleans up all resources at once.

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-cost-dashboard-${var.yourname}"
  location = var.location
  tags     = var.tags
}
```

#### Log Analytics Workspace

Azure's central logging and query engine. `PerGB2018` means you pay only for data ingested — at lab scale this is negligible. 30-day retention is the minimum and keeps costs low.

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-cost-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}
```

#### Monitor Action Group

Defines *what happens* when an alert fires. It is reusable — attach it to as many alert rules as you need. `use_common_alert_schema = true` ensures a consistent email format across all Azure alert types.

```hcl
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
```

#### Subscription Budget with Alert Thresholds

This is the core of the lab. `amount = 200` sets the monthly ceiling. The three `notification` blocks fire at 25% ($50), 50% ($100), and 100% ($200) of that amount. Each one is linked to the Action Group above.

> **Important:** Update `start_date` to the first day of the current month before deploying.

```hcl
resource "azurerm_consumption_budget_subscription" "main" {
  name            = "budget-cost-${var.yourname}"
  subscription_id = data.azurerm_client_config.current.subscription_id

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
```

#### Logic App Workflow

Terraform creates the Logic App container here. The trigger and action steps (HTTP webhook → Send Email) are added through the portal visual designer in Step 6 — this is by design, since interactive OAuth connections cannot be automated by Terraform.

```hcl
resource "azurerm_logic_app_workflow" "cost_alert" {
  name                = "la-cost-alert-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = var.tags
}
```

#### Diagnostic Settings

Tells Azure to forward subscription-level activity logs into Log Analytics. Without this, the workspace has nothing to query and the Workbook has no data to display.

```hcl
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

---

### Step 4 — outputs.tf

These values print to your terminal after `terraform apply` completes. The `logic_app_callback_url` is needed in Step 6 to connect the Logic App to the Action Group.

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

---

### Step 5 — Deploy

These commands are identical on Mac and Windows PowerShell.

```bash
# Initialize Terraform and download the azurerm provider
terraform init

# Preview what will be created (expect 6 resources to add)
terraform plan

# Deploy — type 'yes' when prompted
terraform apply
```

> Deployment takes approximately 2–3 minutes. Copy the `logic_app_callback_url` output value before moving to Step 6.

---

## Portal Configuration

### Step 6 — Configure the Logic App in the Azure Portal

Terraform created the Logic App shell. Now add the trigger and email action using the visual designer.

1. In the Azure portal, navigate to **`rg-cost-dashboard-[yourname]`**
2. Click **`la-cost-alert-[yourname]`**
3. In the left menu, click **Logic app designer**
4. Click **Add a trigger** → search `HTTP` → select **When a HTTP request is received**
5. Copy the **HTTP POST URL** — this is the webhook URL Azure Monitor will call when a budget alert fires
6. Click **+ New step** → search `Office 365 Outlook` → select **Send an email (V2)**
7. Sign in with your Microsoft account when prompted
8. Fill in the email fields:
   - **To:** your alert email address
   - **Subject:** `Azure Cost Alert — Budget Threshold Reached`
   - **Body:** Click **Add dynamic content** → select the **Body** field from the HTTP trigger
9. Click **Save**

### Connect the Logic App to the Action Group

After saving the Logic App, register it as a receiver in the Action Group via CLI.

**Mac:**
```bash
az monitor action-group update \
  --name ag-cost-alerts-[yourname] \
  --resource-group rg-cost-dashboard-[yourname] \
  --add-action logicapp la-webhook la-cost-alert-[yourname] \
    /subscriptions/<sub-id>/resourceGroups/rg-cost-dashboard-[yourname]/providers/Microsoft.Logic/workflows/la-cost-alert-[yourname] \
    <logic-app-callback-url>
```

**Windows (PowerShell):**
```powershell
az monitor action-group update `
  --name ag-cost-alerts-[yourname] `
  --resource-group rg-cost-dashboard-[yourname] `
  --add-action logicapp la-webhook la-cost-alert-[yourname] `
    /subscriptions/<sub-id>/resourceGroups/rg-cost-dashboard-[yourname]/providers/Microsoft.Logic/workflows/la-cost-alert-[yourname] `
    <logic-app-callback-url>
```

> Replace `<sub-id>` with your subscription ID (`az account show --query id -o tsv`) and `<logic-app-callback-url>` with the URL copied from the designer.

---

## Dashboard Setup

### Step 7 — Build the Cost Workbook in Azure Monitor

1. In the portal, search **Monitor** → click **Workbooks** in the left menu
2. Click **+ New**
3. Click **+ Add** → **Add query**
4. Set **Data source** to **Azure Resource Graph**
5. Paste this query:

```kusto
resourcecontainers
| where type == "microsoft.resources/subscriptions/resourcegroups"
| project resourceGroup, location
```

6. Click **Run Query** to verify results, then click **Done Editing**
7. Click **+ Add** → **Add metric** → select your subscription → choose **Cost Management** as the resource type
8. Click **Save** → name it `Cost Visibility Dashboard` → select your resource group → click **Apply**

The workbook is now saved and accessible from **Monitor → Workbooks** any time you open the portal.

---

## Verification Checklist

Run through these after deployment to confirm everything is wired correctly.

- [ ] Resource group `rg-cost-dashboard-[yourname]` exists in the portal
- [ ] Budget `budget-cost-[yourname]` appears in **Cost Management → Budgets**
- [ ] Action group `ag-cost-alerts-[yourname]` exists in **Monitor → Action groups**
- [ ] Logic App `la-cost-alert-[yourname]` shows a green **Enabled** status
- [ ] Logic App designer shows an HTTP trigger and a **Send an email** action
- [ ] Log Analytics workspace `law-cost-[yourname]` exists and is receiving logs
- [ ] Azure Workbook is saved and visible in **Monitor → Workbooks**

---

## Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| `BudgetStartDateInvalid` | `start_date` must be the first of a current or future month | Update `start_date` in `main.tf` to the first of the current month in RFC3339 format |
| `AuthorizationFailed` on budget | Account may lack Cost Management Contributor role | `az role assignment create --role "Cost Management Contributor" --assignee <your-email> --scope /subscriptions/<sub-id>` |
| Logic App email step asks for sign-in | Office 365 connection requires interactive OAuth | Sign in through the portal designer — this step cannot be automated by Terraform |
| Alert email not received | Budget thresholds require actual spend to cross the limit | From the Action Group in the portal, use **Test** to fire a test notification and verify email delivery |

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
| **Terraform IaC** | Deployed 6 Azure resources from code — no clicking through the portal |
| **Azure Cost Management** | Configured subscription-level budget with tiered alert thresholds |
| **Azure Monitor** | Built Action Groups and wired them to budget notification events |
| **Logic Apps** | Designed a webhook-triggered automation workflow with email output |
| **Log Analytics** | Configured diagnostic settings to ingest subscription activity logs |
| **Azure Workbooks** | Built a live cost dashboard using Resource Graph and Cost Management data |
| **Azure CLI** | Used `az monitor action-group update` to register the Logic App receiver |

---

*Deployed by Glen Page — Cloud Engineer | [linkedin.com/in/glen-page-862730246](https://linkedin.com/in/glen-page-862730246) | [github.com/glenpagesr-dev](https://github.com/glenpagesr-dev)*
