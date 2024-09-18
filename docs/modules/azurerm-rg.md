# Contoso Resource Group Module

## Table of Contents

1. [Overview](#overview)
2. [Version History](#version-history)
3. [Deployed Resources](#deployed-resources)
4. [Pre-requisites](#pre-requisites)
5. [Contoso Naming convention](#contoso-naming-convention)
6. [Examples](#examples)
7. [Input arguments and outputs](#input-arguments-and-outputs)

## Overview

This module deploys a Resource Group in Azure. Optionally the module can create two EntraID groups (Reader, Contributor) and assign them to the Resource Group. The supplied UUID owner/s will be added to the Entra ID groups. These owner/s can add and remove members who need `Read` or `Contribute` access to the Resource Group.

[Learn more about Azure Resource Groups at Microsoft Learn](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal/?wt.mc_id=DT-MVP-5004771l)

## Version History

| **Version** | **Date** | **Description** |
|:------------|:---------|:----------------|
| **v2.0.0**  | 17/09/2024 | Fixed issue with resource group deletion. |
| **v1.2.0**  | 01/09/2024 | Minor performance improvements. |
| **v1.1.2**  | 15/08/2024 | Updated dependencies and fixed deprecation warnings. |
| **v1.1.1**  | 30/07/2024 | Correct TFLINT warnings. |
| **v1.1.0**  | 15/07/2024 | Improved validation for input parameters. |
| **v1.0.0**  | 01/07/2024 | Initial release with basic RBAC functionalities. |

## Deployed Resources

The following resources will be deployed when using Contoso's Resource Group Module:

- azurerm_resource_group
- azuread_group (optional - Reader, Contributor groups)
- azurerm_role_assignment (optional - Reader, Contributor roles)

## Pre-requisites

Make sure to update the Terraform versions and providers according to your specific needs.

- **Minimum Terraform version:** ~> 1.8.0
- **AzureRM provider version:** ~> 4.2.0
- **AzureAD provider version:** ~> 2.47.0

Before deploying Contoso's Resource Group module, ensure the following Azure resources are in place:

- **Azure Subscription**: The subscription where the Resource Group will be created.

## Contoso Naming convention

Contoso's naming convention for the resource group follows a structured format to ensure consistency and clarity across resources. The naming convention is derived from the following variables `environment` and `project`:

**Construct:** `"conto-${var.environment}-${var.project}-rg"`  
**Example Naming:** `conto-dev-prj-rg`  

Additionally two EntraID groups will be created and assigned to the Resource Group:

**Construct:** `"conto-entra-${var.environment}-${var.project}-reader"`  
**Example Naming:** `conto-entra-dev-prj-reader`  

**Construct:** `"conto-entra-${var.environment}-${var.project}-contributor"`  
**Example Naming:** `conto-entra-dev-prj-contributor`  

## Examples

### module.tf

Module usage example:

```hcl
module "resource_group" {
  source       = "git::ssh://git@ssh.dev.azure.com/v3/Contoso/Contoso-Modules/azurerm_api_management?ref=v2.0.0"

  // Required arguments:
  project               = var.project

  // Optional inputs:
  create_default_groups = var.create_default_groups
  environment           = var.environment
  owners                = var.owners
   
  //See the "## Input arguments and outputs" section for all available options or "variables.tf"
  // If not specified, the default values on "variables.tf" will be applied
}
```

### module.tf.tfvars

Example input values for the resource group module:

```hcl
// Required inputs:
project               = "paperclips"

// Optional inputs:
create_default_groups = true
environment           = "uat"
owners                = ["00000000-0000-0000-0000-000000000000", "11111111-1111-1111-1111-111111111111"]

// See the "## Input arguments and outputs" section for all available options or "variables.tf"
// If not specified, the default values on "variables.tf" will be applied
```

### variables.tf

Example variable definitions for the Resource Group module:

```hcl
// Required Inputs

variable "project" {
  description = "The name of the project. e.g. prj"
  type        = string
}

// Optional Inputs

variable "create_default_groups" {
  description = "Create default Entra ID groups to associate with the Resource Group (Reader, Contributor)"
  type        = bool
  default     = false
}

variable "environment" {
  description = "The environment. e.g. dev, qa, uat, prod"
  type        = string
  default     = "dev"
}

variable "location" {
  description = "The location/region where the resource group will be created."
  type        = string
  default     = "westeurope"
}

variable "mail_enabled" {
  description = "Are the Entra ID groups mail enabled?"
  type        = bool
  default     = false
}

variable "owners" {
  description = "The owner/s of the Entra ID groups to associate with the Resource Group. Value must be a valid UUID"
  type        = list(string)
  default     = []
}

variable "security_enabled" {
  description = "Are the Entra ID groups security enabled?"
  type        = bool
  default     = true
}
```

### main.tf

```hcl
terraform {
  backend "azurerm" {}
}

provider "azurerm" {
  features {}
  resource_provider_registrations = "none"
}

provider "azuread" {}
```

## Input arguments and outputs

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| `create_default_groups` | Create default AAD groups to associate with the Resource Group e.g. Reader, Contributor | `bool` | `false` | no |
| `environment` | The environment. e.g. dev, qa, uat, prod | `string` | `"dev"` | no |
| `location` | The location/region where the resource group will be created. | `string` | `"westeurope"` | no |
| `mail_enabled` | Are the Entra ID groups mail enabled? | `bool` | `false` | no |
| `owners` | The owner/s of the Entra ID groups to associate with the Resource Group. Value must be a valid UUID | `list(string)` | `[]` | no |
| `project` | The name of the project. e.g. prj | `string` | n/a | yes |
| `security_enabled` | Are the Entra ID groups security enabled? | `bool` | `true` | no |

### Outputs

| Name | Description |
|------|-------------|
| `owner_uuids` | List of AAD UUIDs for the specified users/owners |
| `resource_group_id` | Value of the resource group id |
| `resource_group_name` | Value of the resource group name |
