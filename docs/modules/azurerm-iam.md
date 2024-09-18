# Contoso Role Assignment (IAM) Module

## Table of Contents

1. [Overview](#overview)
2. [Version History](#version-history)
3. [Deployed Resources](#deployed-resources)
4. [Pre-requisites](#pre-requisites)
5. [Contoso Naming Convention](#contoso-naming-convention)
6. [Examples](#examples)
7. [Input arguments and outputs](#input-arguments-and-outputs)

## Overview

Azure Role Assignment grants users, groups, or applications access to Azure resources. It involves associating a security principal (user, group, or application) with a specific role definition and a scope (a resource group, subscription, or resource). This determines what actions the security principal can perform on the specified resources.

This module will:

- Grant a specific role over a scope to a Service Principal, Managed Identity, User, or Group.

[Learn more about Role-Based Access Control at Microsoft Learn](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal/?wt.mc_id=DT-MVP-5004771)

## Version History

| **Version** | **Date** | **Description** |
|:------------|:---------|:----------------|
| **v1.0.4**  | 25/08/2024 | Initial release with basic IAM functionalities. |
| **v1.0.2**  | 15/07/2024 | Added support for Managed Identities. |
| **v1.0.1**  | 05/06/2024 | Improved error handling and logging. |
| **v1.0.0**  | 25/05/2024 | Updated documentation and examples. |

## Deployed Resources

The following resources will be deployed when using Contoso's Role Assignment (IAM) Module:

- azurerm_role_assignment

## Pre-requisites

Make sure to update the Terraform versions and providers according to your specific needs.

- **Minimum Terraform version:** >= 1.8.0
- **AzureRM provider version:** ~> 4.2.0

Before deploying Contoso's IAM module, ensure the following Azure resources are in place:

- **User, Group, Service Principal, or Managed Identity (required)**: The security principal to which the role will be assigned.
- **Target Resource ID (required)**: The ID of the resource to which the role will be assigned.

## Contoso Naming Convention

There is no specific naming convention for this module because the module is not creating any resources. It is only assigning roles to existing resources in Azure using the Azure RBAC/IAM.

## Examples

### module.tf

Module usage example:

```hcl
module "role-assignment" {
  source              = "git::ssh://git@ssh.dev.azure.com/v3/Contoso/Contoso-Modules/azurerm_role_assignments?ref=v1.0.4"
  azure_iam_config    = var.azure_iam_config
}
```

## module.tf.tfvars

Example input values for the Role Assignment module:

```hcl
// Required inputs:
azure_iam_config = [
    {
      description          = "Example - Azure IAM permission on Subscription"
      scope                = "/subscriptions/00000000-0000-0000-0000-000000000000"
      role_definition_name = "Contributor"
      principal_id         = "00000000-0000-0000-0000-000000000000"
    },
    {
      description          = "Example - Azure IAM permission on Resource Group"
      scope                = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myGroup"
      role_definition_name = "Contributor"
      principal_id         = "00000000-0000-0000-0000-000000000000"
    },
    {
      description          = "Example - Azure IAM permission on Resource (VM)"
      scope                = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myGroup/providers/Microsoft.Compute/virtualMachines/myVM"
      role_definition_name = "Contributor"
      principal_id         = "00000000-0000-0000-0000-000000000000"
    },
    {
      description          = "Example - Azure IAM permission on Management Group"
      scope                = "/providers/Microsoft.Management/managementGroups/myMG"
      role_definition_name = "Contributor"
      principal_id         = "00000000-0000-0000-0000-000000000000"
    }
]

// Optional inputs:
// None
```

### variables.tf

Example input values for the IAM module variables:

```hcl
// Required Inputs

variable "azure_iam_config" {
  description = "Azure IAM role assignment (permissions) configuration."
  type = list(object({
    description          = string
    scope                = string
    role_definition_name = string
    principal_id         = string
  }))
}

// Optional inputs - None
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
```

## Input arguments and outputs

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| `azure_iam_config` | Azure IAM role assignment (permissions) configuration. | `list(object({ description = string, scope = string, role_definition_name = string, principal_id = string }))` | [<br>  {<br>    description = "Example - Azure IAM permission on Subscription",<br>    principal_id = "00000000-0000-0000-0000-000000000000",<br>    role_definition_name = "Contributor",<br>    scope = "/subscriptions/00000000-0000-0000-0000-000000000000"<br>  },<br>  {<br>    description = "Example - Azure IAM permission on Resource Group",<br>    principal_id = "00000000-0000-0000-0000-000000000000",<br>    role_definition_name = "Contributor",<br>    scope = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myGroup"<br>  },<br>  {<br>    description = "Example - Azure IAM permission on Resource",<br>    principal_id = "00000000-0000-0000-0000-000000000000",<br>    role_definition_name = "Contributor",<br>    scope = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myGroup/providers/Microsoft.Compute/virtualMachines/myVM"<br>  },<br>  {<br>    description = "Example - Azure IAM permission on Management Group",<br>    principal_id = "00000000-0000-0000-0000-000000000000",<br>    role_definition_name = "Contributor",<br>    scope = "/providers/Microsoft.Management/managementGroups/myMG"<br>  }<br>] | no |

### Outputs

| Name | Description |
|------|-------------|
| `role_assignment_ids` | The Role Assignment IDs. |