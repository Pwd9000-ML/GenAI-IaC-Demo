# Contoso Key Vault Module

## Table of Contents

1. [Overview](#overview)
2. [Version History](#version-history)
3. [Deployed Resources](#deployed-resources)
4. [Pre-requisites](#pre-requisites)
5. [Contoso Naming Convention](#contoso-naming-convention)
6. [Examples](#examples)
7. [Input arguments and outputs](#input-arguments-and-outputs)

## Overview

This module deploys a Key Vault in Azure with a private endpoint. It gives you the option to add Entra ID groups as admins (Key Vault Administrator) and readers (Key Vault Secrets User, Key Vault Reader) of the Key Vault. You can add a secret expiry notification to receive an e-mail alert one month before the secret expires.

[Learn more about Azure Key Vault at Microsoft Learn](https://learn.microsoft.com/en-us/azure/key-vault/?wt.mc_id=DT-MVP-5004771)

## Version History

| **Version** | **Date** | **Description** |
|:------------|:---------|:----------------|
| **v2.3.1**  | 24/08/2024 | Removed unnecessary outputs. |
| **v2.3.0**  | 10/08/2024 | Added support for secret expiry notifications. |
| **v2.2.0**  | 01/08/2024 | Improved private endpoint configuration. |
| **v2.1.0**  | 15/07/2024 | Updated dependencies and fixed deprecation warnings. |
| **v2.0.0**  | 01/07/2024 | Major release with new features and improvements. |

## Deployed Resources

The following resources will be deployed when using the Key Vault module:

- azurerm_key_vault
- azurerm_private_endpoint
- azurerm_monitor_action_group
- azurerm_monitor_diagnostic_setting (optional - if `log_analytics_workspace_name` is set)
- azurerm_monitor_scheduled_query_rules_alert_v2 (optional - if `expire_notification` is set to `true`)
- azurerm_role_assignment 

## Pre-requisites

Make sure to update the Terraform versions and providers according to your specific needs.

- **Minimum Terraform version:** >= 1.8.0
- **AzureRM provider version:** ~> 4.2.0
- **AzureAD provider version:** ~> 2.53.1

Before deploying the Key Vault module, ensure the following Azure resources are in place:

- **Resource Group (required)**: The resource group that contains the Key Vault.
- **Virtual Network (required)**: The virtual network where the private endpoint will be created.
- **Log Analytics Workspace (optional)** : Required if you want to send diagnostic logs to Log Analytics.

## Contoso Naming convention

Contoso's naming convention for the key vault module follows a structured format to ensure consistency and clarity across resources. The naming convention is derived from the following variables `environment` and `project` in addition to the there is a `kv_number` variable to differentiate between the key vaults.

**Construct:** `"conto-${var.environment}-${var.project}-kv-${var.kv_number}"`
**Example Naming:** `conto-dev-prj-kv-001`

Additionally two Entra ID groups will be created and assigned to the Key Vault:

**Construct:** `"conto-entra-${var.environment}-${var.project}-Admin"`
**Example Naming:** `conto-entra-dev-prj-Admin`
**Role:** `Key Vault Administrator`

**Construct:** `"conto-entra-${var.environment}-${var.project}-Reader"`
**Example Naming:** `conto-entra-dev-prj-Reader`
**Role:** `Key Vault Reader, Key Vault Secrets User`

## Examples

### module.tf

Module usage example:

```hcl
module "key_vault" {
  source                        = "git::ssh://git@ssh.dev.azure.com/v3/Contoso/Contoso-Modules/azurerm_api_management?ref=v2.3.1"

  // Required inputs:
  network_resource_group_name   = var.network_resource_group_name
  pe_subnet_name                = var.pe_subnet_name
  resource_group_name           = var.resource_group_name
  virtual_network_name          = var.virtual_network_name

  // Optional inputs:
  email_receiver                = var.email_receiver
  environment                   = var.environment
  expire_notification           = var.expire_notification
  kv_number                     = var.kv_number
  log_analytics_workspace_name  = var.log_analytics_workspace_name
  project                       = var.project

  //See the "## Input arguments and outputs" section for all available options or "variables.tf"
  // If not specified, the default values on "variables.tf" will be applied
}
```

### module.tf.tfvars

Example input values for the Key Vault module:

```hcl
// Required inputs:
network_resource_group_name = "conto-dev-paperclips-network-rg"
pe_subnet_name              = "kv-subnet"
resource_group_name         = "conto-dev-paperclips-rg"
virtual_network_name        = "conto-dev-paperclips-vnet"

// Optional inputs:
email_receiver               = ["test.user@contoso.com", "8d73ujd73.contoso.onmicrosoft.com@teams.ms"] #teams channel
environment                  = "dev"
expire_notification          = true
kv_number                    = "002"
log_analytics_workspace_name = "conto-dev-paperclips-law"
project                      = "paperclips"

// See the "## Input arguments and outputs" section for all available options or "variables.tf"
// If not specified, the default values on "variables.tf" will be applied
```

### variables.tf

Example variable definitions for the Key Vault module:

```hcl
// Required Inputs

variable "network_resource_group_name" {
  description = "The existing core network resource group name, to get details of the VNET to enable private endpoint."
  type        = string
}

variable "pe_subnet_name" {
  description = "The subnet name, used in data source to get subnet ID, to enable the private endpoint."
  type        = string
}

variable "resource_group_name" {
  description = "The name of the resource group that contains the Key Vault."
  type        = string
}

variable "virtual_network_name" {
  description = "Virtual network name for the environment to enable private endpoint."
  type        = string
}

// Optional Inputs

variable "admin_groups" {
  description = "Name of the groups that can do all operations on all keys, secrets and certificates."
  type        = list(string)
  default     = []
}

variable "email_receiver" {
  description = "List of email receivers of secret expire notification."
  type        = list(string)
  default     = []
}

variable "enabled_for_deployment" {
  description = "Whether Azure Virtual Machines are permitted to retrieve certificates stored as secrets from the Key Vault."
  type        = bool
  default     = false
}

variable "enabled_for_disk_encryption" {
  description = "Whether Azure Disk Encryption is permitted to retrieve secrets from the vault and unwrap keys."
  type        = bool
  default     = false
}

variable "enabled_for_template_deployment" {
  description = "Whether Azure Resource Manager is permitted to retrieve secrets from the Key Vault."
  type        = bool
  default     = false
}

variable "environment" {
  description = "The environment. e.g. dev, qa, uat, prod."
  type        = string
  default     = "dev"
}

variable "expire_notification" {
  description = "Send a notification before the secret expires."
  type        = bool
  default     = true
}

variable "kv_number" {
  description = "The use case of the keyvault, to be used in the name. e.g. 001, or 002."
  type        = string
  default     = "001"
}

variable "location" {
  description = "The default location where the core network will be created."
  type        = string
  default     = "westeurope"
}

variable "log_analytics_workspace_name" {
  description = "The name of the Log Analytics workspace to send diagnostic logs to."
  type        = string
  default     = null
}

variable "network_acls" {
  description = "Object with attributes: `bypass`, `default_action`, `ip_rules`, `virtual_network_subnet_ids`. Set to `null` to disable. See https://www.terraform.io/docs/providers/azurerm/r/key_vault.html#bypass for more information."
  type = object({
    bypass                     = optional(string, "None")
    default_action             = optional(string, "Deny")
    ip_rules                   = optional(list(string))
    virtual_network_subnet_ids = optional(list(string))
  })
  default = {}
}

variable "project" {
  description = "The name of the project. e.g. prj."
  type        = string
  default     = "prj"
}

variable "public_network_access_enabled" {
  description = "Whether the Key Vault is available from public network."
  type        = bool
  default     = false
}

variable "reader_groups" {
  description = "Name of the groups that can read all keys, secrets, and certificates."
  type        = list(string)
  default     = []
}

variable "sku_name" {
  description = "The Name of the SKU used for this Key Vault. Possible values are 'standard' and 'premium'."
  type        = string
  default     = "standard"
}

variable "soft_delete_retention_days" {
  description = "The number of days that items should be retained for once soft-deleted. This value can be between `7` and `90` days."
  type        = number
  default     = 7
}

variable "webhook_receiver" {
  description = "List of webhook receivers for secret expire notification."
  type = list(object({
    name        = string
    service_uri = string
  }))
  default = []
}
```

### main.tf

```hcl
terraform {
  backend "azurerm" {}
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
  }
  resource_provider_registrations = "none"
}

provider "azuread" {}
```

## Input arguments and outputs

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| `admin_groups` | Name of the groups that can do all operations on all keys, secrets and certificates. | `list(string)` | `[]` | no |
| `email_receiver` | List of email receivers of secret expire notification. | `list(string)` | `[]` | no |
| `enabled_for_deployment` | Whether Azure Virtual Machines are permitted to retrieve certificates stored as secrets from the Key Vault. | `bool` | `false` | no |
| `enabled_for_disk_encryption` | Whether Azure Disk Encryption is permitted to retrieve secrets from the vault and unwrap keys. | `bool` | `false` | no |
| `enabled_for_template_deployment` | Whether Azure Resource Manager is permitted to retrieve secrets from the Key Vault. | `bool` | `false` | no |
| `environment` | The environment. e.g. dev, qa, uat, prod | `string` | `"dev"` | no |
| `expire_notification` | Send a notification before the secret expires | `bool` | `true` | no |
| `kv_number` | The use case of the keyvault, to be used in the name. e.g. 001, or 002 | `string` | `"001"` | no |
| `location` | The default location where the core network will be created | `string` | `"westeurope"` | no |
| `log_analytics_workspace_name` | The name of the Log Analytics workspace to send diagnostic logs to. | `string` | `null` | no |
| `network_acls` | Object with attributes: `bypass`, `default_action`, `ip_rules`, `virtual_network_subnet_ids`. Set to `null` to disable. See https://www.terraform.io/docs/providers/azurerm/r/key_vault.html#bypass for more information. | `object({ bypass = optional(string, "None"), default_action = optional(string, "Deny"), ip_rules = optional(list(string)), virtual_network_subnet_ids = optional(list(string)) })` | `{}` | no |
| `network_resource_group_name` | The existing core network resource group name, to get details of the VNET to enable private endpoint. | `string` | n/a | yes |
| `pe_subnet_name` | The subnet name, used in data source to get subnet ID, to enable the private endpoint. | `string` | n/a | yes |
| `project` | The name of the project. e.g. prj | `string` | `"prj"` | no |
| `public_network_access_enabled` | Whether the Key Vault is available from public network. | `bool` | `false` | no |
| `reader_groups` | Name of the groups that can read all keys, secrets, and certificates. | `list(string)` | `[]` | no |
| `resource_group_name` | The name of the resource group that contains the Key Vault | `string` | n/a | yes |
| `sku_name` | The Name of the SKU used for this Key Vault. Possible values are "standard" and "premium". | `string` | `"standard"` | no |
| `soft_delete_retention_days` | The number of days that items should be retained for once soft-deleted. This value can be between `7` and `90` days. | `number` | `7` | no |
| `virtual_network_name` | Virtual network name for the environment to enable private endpoint. | `string` | n/a | yes |
| `webhook_receiver` | List of webhook receivers for secret expire notification. | `list(object({ name = string, service_uri = string }))` | `[]` | no |

### Outputs

| Name | Description |
|------|-------------|
| `key_vault_id` | ID of the Key Vault. |
| `key_vault_name` | Name of the Key Vault. |
| `key_vault_uri` | URI of the Key Vault. |
