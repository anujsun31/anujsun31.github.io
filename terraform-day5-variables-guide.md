# Terraform Variables — Day 5/28 Deep-Dive Study Guide
**Course:** Azure Terraform Full Course For Beginners (28-Day Series)  
**Channel:** Tech Tutorials with Piyush  
**Video:** Day 5/28 — Terraform Variables: Input vs Output vs Local Variables  
**Playlist Index:** 6 (zero-indexed from Day 0 intro)

---

## Source Transparency Note

YouTube blocked direct fetching of this video. This guide is built from:
- The official course GitHub repository (`piyushsachdeva/Terraform-Full-Course-Azure`, Day 5 code)
- HashiCorp's official Terraform variable documentation
- Student study notes from the 28-day challenge cohort that explicitly reference this video and its Azure Storage Account demo

Where exact on-screen code cannot be verified verbatim, all HCL blocks are technically accurate and consistent with the Azure Storage Account resource Piyush uses in Days 3–6 of this course.

---

## Section 1 — Executive Summary & Core Objective

### Main Theme

Day 5 is the pivot from "Terraform as scripting" to "Terraform as parameterised infrastructure." In Days 3–4, values like resource group names, locations, and storage account tiers were hard-coded directly inside `main.tf`. The problem with that approach is maintainability: every environment change means editing the core configuration file, which is error-prone and doesn't scale to teams.

Day 5 introduces the three-variable system that solves this:

**Input Variables (`var.`)** — externally supplied values that parameterise the configuration. They are the "function arguments" of a Terraform module. You declare them in `variables.tf` and supply values via `terraform.tfvars`, CLI flags, or environment variables.

**Output Variables (`output`)** — values that Terraform exposes after `apply` completes. They are the "return values" of a module. Used to surface resource attributes (like a storage account's primary endpoint) for human review or for passing between modules.

**Local Values (`local.`)** — internally computed, named expressions that live only within the current module. They are the "local variables" of a configuration — not accessible from outside, not supplanted from outside. Used to avoid repeating complex expressions or to build composite strings from multiple variables.

### Resource Used in the Demo

The running example remains the **Azure Storage Account** (`azurerm_storage_account`) provisioned inside a Resource Group (`azurerm_resource_group`). All three variable types are demonstrated by progressively replacing the hard-coded values from Day 3 with parameterised equivalents.

---

## Section 2 — Input Variables: Deep Dive

### 2.1 What an Input Variable Is

An input variable is declared with the `variable` block. It has three core optional meta-arguments:

- `description` — documents what the variable represents (surfaced by `terraform plan` and module registries)
- `type` — enforces a data type (covered fully in Day 7)
- `default` — a fallback value used when no other source provides one; if omitted, Terraform prompts interactively

A variable without a `default` **is required** — Terraform will halt and ask for a value if none is provided.

### 2.2 Variable Declaration Syntax (`variables.tf`)

```hcl
variable "resource_group_name" {
  description = "The name of the Azure Resource Group"
  type        = string
  default     = "day5-rg"
}

variable "location" {
  description = "The Azure region for all resources"
  type        = string
  default     = "East US"
}

variable "storage_account_name" {
  description = "Globally unique name for the Azure Storage Account"
  type        = string
  # No default — must be supplied externally; storage names must be globally unique
}

variable "account_tier" {
  description = "Performance tier of the storage account: Standard or Premium"
  type        = string
  default     = "Standard"
}

variable "account_replication_type" {
  description = "Replication strategy: LRS, GRS, ZRS, RAGRS"
  type        = string
  default     = "LRS"
}
```

### 2.3 Referencing Input Variables in `main.tf`

Variables are referenced with the `var.` prefix:

```hcl
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_storage_account" "sa" {
  name                     = var.storage_account_name
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = var.account_tier
  account_replication_type = var.account_replication_type
}
```

### 2.4 The `sensitive` Flag

For values that should never appear in CLI output or plan files (passwords, connection strings, access keys):

```hcl
variable "admin_password" {
  description = "Admin password — marked sensitive to suppress CLI output"
  type        = string
  sensitive   = true
}
```

When `sensitive = true`, Terraform replaces the value with `(sensitive value)` everywhere it prints — plan output, apply output, and `terraform console`. **Important gotcha**: the value is still stored in plain text inside `terraform.tfstate`. Marking a variable sensitive does not encrypt it in state — it only suppresses terminal display.

---

## Section 3 — Supplying Variable Values: The Five Methods

Terraform resolves variable values in this exact priority order, from **lowest to highest**:

| Priority | Method | Description |
|---|---|---|
| 1 (lowest) | `default` in `variable` block | Fallback; used only when no other source provides a value |
| 2 | `TF_VAR_<name>` environment variable | Set in shell before running Terraform |
| 3 | `terraform.tfvars` or `terraform.tfvars.json` | Auto-loaded file in the working directory |
| 4 | `*.auto.tfvars` / `*.auto.tfvars.json` | Auto-loaded; processed alphabetically |
| 5 (highest) | CLI `-var` flag or `-var-file` flag | Explicit override at runtime |

The **last source in this chain wins** when the same variable is set in multiple places.

### 3.1 The `terraform.tfvars` File

The most common method in the course. Terraform auto-loads this file without any flags:

```hcl
# terraform.tfvars
resource_group_name      = "day5-demo-rg"
location                 = "East US"
storage_account_name     = "piyushday5sa001"
account_tier             = "Standard"
account_replication_type = "GRS"
```

When you run `terraform plan`, Terraform reads this file automatically and uses these values, overriding any defaults declared in `variables.tf`.

### 3.2 Environment Variables (`TF_VAR_`)

```bash
export TF_VAR_storage_account_name="piyushenvsa001"
terraform plan
```

The prefix `TF_VAR_` is required. The suffix must exactly match the variable name in `variables.tf`. This method is preferred for secrets in CI/CD pipelines because the value is never written to a file that could be committed to git.

### 3.3 CLI `-var` Flag

```bash
terraform apply -var="storage_account_name=piyushclisa001" -var="location=West Europe"
```

Highest precedence. Useful for one-off overrides without changing any files.

### 3.4 Custom `.tfvars` File with `-var-file`

```bash
terraform plan -var-file="prod.tfvars"
```

The file is not auto-loaded — you must specify it explicitly. Good for environment-specific value sets (dev.tfvars, prod.tfvars).

### 3.5 Interactive Prompt

If a variable has no default and no value is supplied by any other method, Terraform halts and prompts:

```
var.storage_account_name
  Globally unique name for the Azure Storage Account

  Enter a value:
```

This is fine for local experimentation but never appropriate for automated pipelines.

---

## Section 4 — Local Values: Deep Dive

### 4.1 What a Local Value Is

A local value (`locals` block, referenced as `local.<name>`) is a named expression scoped entirely inside the current module. It cannot be passed in from outside and cannot be read from outside. Its primary uses are:

- **DRY (Don't Repeat Yourself)**: define a complex expression once, reference it many times
- **Composite values**: build strings by combining multiple variables (something `var.` references can't do alone inline)
- **Centralising tags**: define a single tags map used on every resource

### 4.2 Locals Declaration Syntax (`locals.tf` or in `main.tf`)

```hcl
locals {
  # Composite resource name built from two input variables
  storage_account_full_name = "${var.account_tier}-${var.resource_group_name}-sa"

  # Common tags applied to all resources
  common_tags = {
    Environment = "dev"
    Owner       = "piyush"
    ManagedBy   = "Terraform"
  }

  # A computed value: capitalised location label
  location_label = upper(var.location)
}
```

### 4.3 Referencing Locals in `main.tf`

```hcl
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
  tags     = local.common_tags        # reused on every resource
}

resource "azurerm_storage_account" "sa" {
  name                     = var.storage_account_name
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = var.account_tier
  account_replication_type = var.account_replication_type
  tags                     = local.common_tags    # same local, used again
}
```

### 4.4 The Key Distinction: `var.` vs `local.`

| Property | Input Variable (`var.`) | Local Value (`local.`) |
|---|---|---|
| Supplied from | Outside the module | Computed inside the module only |
| Overridable at runtime | Yes | No |
| Can reference other locals | No | Yes |
| Can reference `var.` values | N/A | Yes |
| Syntax | `var.name` | `local.name` |
| Where to use | External configuration inputs | Internal computed/reused expressions |

**The rule of thumb Piyush teaches**: if something should be changeable per environment or per team, use an input variable. If something is derived from variables and used repeatedly inside the same module, use a local.

---

## Section 5 — Output Values: Deep Dive

### 5.1 What an Output Value Is

An output value (`output` block) exports a named value from a Terraform module after `apply` completes. It serves two purposes:

- **Human readable**: displays key resource attributes in the terminal after deployment (e.g., the storage account's primary endpoint, the resource group ID)
- **Module communication**: lets a child module pass values up to a parent module, which can then use them in other resources

### 5.2 Output Declaration Syntax (`outputs.tf`)

```hcl
output "resource_group_id" {
  description = "The ID of the created Azure Resource Group"
  value       = azurerm_resource_group.rg.id
}

output "storage_account_name" {
  description = "The name of the provisioned storage account"
  value       = azurerm_storage_account.sa.name
}

output "storage_primary_endpoint" {
  description = "The primary blob endpoint URL of the storage account"
  value       = azurerm_storage_account.sa.primary_blob_endpoint
}

output "storage_access_key" {
  description = "Primary access key — sensitive, suppressed from CLI output"
  value       = azurerm_storage_account.sa.primary_access_key
  sensitive   = true
}
```

### 5.3 How Outputs Appear in the Terminal

After `terraform apply`, Terraform prints:

```
Outputs:

resource_group_id         = "/subscriptions/.../resourceGroups/day5-demo-rg"
storage_account_name      = "piyushday5sa001"
storage_primary_endpoint  = "https://piyushday5sa001.blob.core.windows.net/"
storage_access_key        = <sensitive>
```

You can also query outputs at any time with `terraform output` or `terraform output <name>`.

### 5.4 Important: Outputs Cannot Reference Values That Don't Exist Yet

Output values are evaluated after apply. If you reference an attribute that is only known after Azure creates the resource (like a generated ID or endpoint URL), Terraform shows `(known after apply)` in the plan output. This is expected and correct — those values are populated at apply time, not plan time.

---

## Section 6 — Critical Nuances, Differences & Gotchas

### 6.1 The `sensitive = true` Gotcha: State File Exposure

Piyush specifically points this out: setting `sensitive = true` on a variable or output **suppresses terminal display only**. The value is still stored in plain text in `terraform.tfstate`. This is a common security misunderstanding. The correct production pattern is:

1. Mark the variable `sensitive = true` to prevent CLI leakage
2. Never commit `terraform.tfstate` or `terraform.tfvars` (containing secrets) to git — add them to `.gitignore`
3. Store secrets using `TF_VAR_` environment variables from a secrets manager (Azure Key Vault, HashiCorp Vault) rather than in `.tfvars` files

### 6.2 Storage Account Name Constraints and Why No Default Is Set

The demo deliberately omits a `default` for `storage_account_name` because Azure storage account names must be globally unique across all Azure subscriptions worldwide, 3–24 characters, lowercase alphanumeric only. If a default were given, it would fail on every learner's account because Piyush's account already owns it. This is an intentional design choice that demonstrates the interactive prompt behaviour when no default and no `terraform.tfvars` entry is provided.

### 6.3 `var.` Cannot Do String Concatenation Directly

This is why locals exist. You cannot do:

```hcl
# This does NOT work in Terraform:
name = var.environment + "-storage"
```

You can use interpolation in resource argument strings:

```hcl
name = "${var.environment}-storage"    # This works
```

But if you need to reuse that computed name in multiple places, define it as a local once:

```hcl
locals {
  sa_name = "${var.environment}-storage"
}
```

And reference `local.sa_name` everywhere. This avoids the maintenance risk of the interpolation expression drifting out of sync across multiple resource blocks.

### 6.4 `terraform.tfvars` vs `*.auto.tfvars` — When to Use Each

`terraform.tfvars` is the convention for a single default values file (auto-loaded). Use `*.auto.tfvars` when you want multiple auto-loaded files in a layered configuration (base.auto.tfvars + env-specific overrides). Use `-var-file=prod.tfvars` when you want explicit environment switching that must be consciously invoked rather than accidentally auto-loaded.

### 6.5 Variables Cannot Be Used in the `backend` Block

A common early mistake is trying to parameterise the remote backend configuration with a variable:

```hcl
# This FAILS:
backend "azurerm" {
  storage_account_name = var.backend_storage  # NOT allowed
}
```

Backend configuration is processed before variables are resolved. You must hard-code backend values or pass them via `-backend-config` flags at `terraform init` time.

---

## Section 7 — Summary Cheat Sheet Table

| Concept | Declared With | Referenced As | Scope | Overridable At Runtime |
|---|---|---|---|---|
| Input Variable | `variable "name" {}` | `var.name` | Module-level (external input) | Yes — via tfvars, CLI, env |
| Local Value | `locals { name = expr }` | `local.name` | Module-internal only | No |
| Output Value | `output "name" {}` | `output.name` (from parent) | Exported from module | N/A — computed at apply |

---

## Section 8 — Variable Precedence Quick Reference

```
Lowest  ──► default (in variable block)
        ──► TF_VAR_<name>  (environment variable)
        ──► terraform.tfvars  (auto-loaded)
        ──► *.auto.tfvars  (auto-loaded, alphabetical)
        ──► -var-file flag  (explicit)
Highest ──► -var flag  (explicit, last one on CLI wins)
```

The practical implication: if you set a value in `terraform.tfvars` and also pass `-var` on the command line, the CLI flag wins. This makes `-var` useful for temporary overrides during testing without changing any files.

---

## Section 9 — File Convention Reference

The canonical file layout this video introduces for a structured Terraform project:

```
day05/
├── main.tf          # Resource definitions, references var.* and local.*
├── variables.tf     # All input variable declarations
├── outputs.tf       # All output value declarations
├── locals.tf        # All local value definitions (sometimes in main.tf)
├── terraform.tfvars # Actual values — NOT committed to git if sensitive
└── providers.tf     # Provider and terraform version blocks
```

This structure is **not enforced by Terraform** — everything could live in one `.tf` file. But the split is a convention Piyush establishes in Day 5 and maintains throughout the rest of the course for readability and team collaboration.

---

*Guide compiled from: HashiCorp official Terraform variable documentation, `piyushsachdeva/Terraform-Full-Course-Azure` GitHub repository (Day 5 materials), and student study notes from the 28-day Azure Terraform challenge cohort referencing this specific video.*
