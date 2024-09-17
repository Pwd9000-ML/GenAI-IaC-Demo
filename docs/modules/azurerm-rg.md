| **Build Status** | **Latest Version** | **Date** |
|:-----------------|:-------------------|:---------|
| [![Build Status](https://dev.azure.com/Axpo-AXSO/TIM-INFRA-MODULES/_apis/build/status%2FProd_Branch_Testing%2Fazurerm_resource_group?repoName=azurerm_resource_group&branchName=main)](https://dev.azure.com/Axpo-AXSO/TIM-INFRA-MODULES/_build/latest?definitionId=2321&repoName=azurerm_resource_group&branchName=main) | **v1.4.8** | 17/09/2024 |  

To see more available Axso services please see **[Production Ready Services](https://dev.azure.com/Axpo-AXSO/TIM-INFRA-MODULES/_wiki/wikis/Axso%20Terraform%20Self%20Service/3912/PRODUCTION.SERVICES)** or **[Production Ready Blueprints](https://dev.azure.com/Axpo-AXSO/TIM-INFRA-MODULES/_wiki/wikis/Axso%20Terraform%20Self%20Service/3911/PRODUCTION.BLUEPRINTS)**  

# AzureRM Resource Group Deployment Module

This module deploys a Resource Group in Azure.
By default a `Reader` and `Contributor` AAD group will be created and assigned to the Resource Group.
The requester of the Azure Resource Group must also specify the AAD group owner.

## Axso Naming convention example

The naming convention is derived from the following variables `subscription`, `project_name` and `environment`:  

**Construct:** `"axso-${var.subscription}-appl-${var.project_name}-${var.environment}-rg"`  
**NonProd:** `axso-np-appl-etools-dev-rg`  
**Prod:** `axso-p-appl-etools-prod-rg`

Additionally two AAD groups will also be created and assigned to the Resource Group:

**Construct:** `"cl-axso-az-appl-${var.subscription}-${var.project_name}-${var.environment}-reader"`  
**NonProd:** `cl-axso-az-appl-np-etools-dev-reader`  
**Prod:** `cl-axso-az-appl-p-etools-prod-reader`

**Construct:** `"cl-axso-az-appl-${var.subscription}-${var.project_name}-${var.environment}-contributor"`  
**NonProd:** `cl-axso-az-appl-np-etools-dev-contributor`  
**Prod:** `cl-axso-az-appl-p-etools-prod-contributor`

## Usage

```hcl
module "axso_resource_group" {
  source       = "git::ssh://git@ssh.dev.azure.com/v3/Axpo-AXSO/TIM-INFRA-MODULES/azurerm_resource_group?ref=v1.4.8"
  subscription = "np"
  project_name = "mds"
  environment  = "dev"
  owners       = ["d.t.user.name@axpo.com"]
  location     = "westeurope"
}
```

The above example will create an **Azure Resource Group** for the project **mds** in the **nonprod** subscription: `"axso-nonprod-appl-mds-dev-rg"`.  
Corresponding **AAD groups** will be created and assigned to the Resource Group: `"cl-axso-az-appl-nonprod-mds-dev-Reader""` and `"cl-axso-az-appl-nonprod-mds-dev-Contributor"`.  
The **owner** is also set and will be able to add and remove members who need `Read` or `Contribute` access to the Resource Group.  

## Resources Created

- AAD / EntraID Groups (Reader, Contributor)
- Resource Group.

<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.8.0 |
| <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) | >= 4.0.1 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_azuread"></a> [azuread](#provider\_azuread) | n/a |
| <a name="provider_azurerm"></a> [azurerm](#provider\_azurerm) | >= 4.0.1 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [azuread_group.aad_group_contributor](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/group) | resource |
| [azuread_group.aad_group_reader](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/group) | resource |
| [azurerm_resource_group.resource_group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group) | resource |
| [azurerm_role_assignment.contributor](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/role_assignment) | resource |
| [azurerm_role_assignment.reader](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/role_assignment) | resource |
| [azuread_user.owners](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/data-sources/user) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_create_default_groups"></a> [create\_default\_groups](#input\_create\_default\_groups) | Create default AAD groups to associate with the Resource Group e.g. Reader, Contributor | `bool` | `true` | no |
| <a name="input_environment"></a> [environment](#input\_environment) | The environment. e.g. dev, qa, uat, prod | `string` | `"dev"` | no |
| <a name="input_location"></a> [location](#input\_location) | The location/region where the resource group will be created. | `string` | `"westeurope"` | no |
| <a name="input_mail_enabled"></a> [mail\_enabled](#input\_mail\_enabled) | Is the AAD group mail enabled? | `bool` | `false` | no |
| <a name="input_owners"></a> [owners](#input\_owners) | The owner/s of the AAD groups to associate with the Resource Group. Value must be a valid UUID | `list(string)` | `[]` | no |
| <a name="input_project_name"></a> [project\_name](#input\_project\_name) | The name of the project. e.g. MDS | `string` | `"project"` | no |
| <a name="input_security_enabled"></a> [security\_enabled](#input\_security\_enabled) | Is the AAD group security enabled? | `bool` | `true` | no |
| <a name="input_subscription"></a> [subscription](#input\_subscription) | The subscription type e.g. 'p' or 'np' | `string` | `"np"` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_owner_uuids"></a> [owner\_uuids](#output\_owner\_uuids) | List of AAD UUIDs for the specified users |
| <a name="output_resource_group_id"></a> [resource\_group\_id](#output\_resource\_group\_id) | value of the resource group id |
| <a name="output_resource_group_name"></a> [resource\_group\_name](#output\_resource\_group\_name) | value of the resource group name |
<!-- END_TF_DOCS -->
