# Contoso Application Gateway Module

## Table of Contents

1. [Overview](#overview)
2. [Version History](#version-history)
3. [Deployed Resources](#deployed-resources)
4. [Pre-requisites](#pre-requisites)
5. [Contoso Naming Convention](#contoso-naming-convention)
6. [Examples](#examples)
7. [Input arguments and outputs](#input-arguments-and-outputs)

## Overview

Azure Application Gateway is a web traffic load balancer that enables you to manage traffic to your web applications. It offers features such as SSL termination, URL-based routing, Web Application Firewall (WAF) capabilities, and support for multiple site hosting. By using this module, you can configure the Application Gateway with various settings, including backend pools, HTTP listeners, routing rules, and probes, allowing you to build a scalable and secure infrastructure for your web applications in the Azure cloud.

[Learn more about Application Gateway at Microsoft Learn](https://learn.microsoft.com/en-us/azure/application-gateway/?wt.mc_id=DT-MVP-5004771)

## Version History

| **Version** | **Date** | **Description** |
|:------------|:---------|:----------------|
| **v1.3.0**  | 23/08/2024 | General availability of the module with support for the latest version of the AzureRM provider. |

## Deployed Resources

----------------------------;

The following resources will be deployed when using Contoso's Application Gateway module:

- azurerm_user_assigned_identity
- azurerm_application_gateway
- azurerm_network_interface (optional - based on variable `create_backend_nic`)
- azurerm_network_interface_application_gateway_backend_address_pool_association (optional - based on variable `create_backend_nic`)
- azurerm_public_ip (optional - based on variable `appgw_public`)
- azurerm_key_vault_access_policy (optional - based on variable `key_vault_rbac`)
- azurerm_role_assignment (optional - based on variable `key_vault_rbac`)

## Pre-requisites

----------------------------;

Make sure to update the Terraform versions and providers according to your specific needs.

- **Minimum Terraform version:** >= 1.8.0
- **TLS provider version:** = 4.0.5
- **AzureRM provider version:** ~> 4.0.1

Before deploying Contoso's Application Gateway module, ensure the following Azure resources are in place:

- **Resource Group (required)**: Essential for organizing and managing related Azure resources.
- **Virtual Network (required)**: Provides network isolation and security for the Application Gateway instance.
- **Delegated Subnet (required)**: A subnet within the virtual network to host the Application Gateway instance. Recommended size is /24.
- **KeyVault (required)**: Securely stores and manages sensitive information such as SSL certificates.
- **Certificate Stored in KeyVault (optional)**: Required if custom domain names or SSL certificates are needed. Store these in KeyVault based on deployment requirements.

## Contoso Naming Convention

----------------------------;

Contoso's naming convention for the Application Gateway module follows a structured format to ensure consistency and clarity across resources. The naming convention derived from the following variables `environment` and `project`:

**Construct:** `"conto-${var.environment}-${var.project}-appgw"`  
**Example Naming:** `conto-dev-prj-appgw`  

## Examples

----------------------------;

### module.tf

Module usage example:

```hcl
module "appgwV2" {
  source                              = "git::ssh://git@ssh.dev.azure.com/v3/Contoso/Contoso-Modules/azurerm_application_gateway?ref=v1.3.0"
  

}
```

### module.tf.tfvars

Example input values for the Application Gateway module variables:

```hcl
project                             = "prj"
environment                         = "dev"
location                            = "westeurope"
resource_group_name                 = "conto-np-appl-ssp-test-rg"
virtual_network_name                = "vnet-ssp-nonprod-conto-vnet"
key_vault_name                      = "kv-ssp-0-nonprod-conto"
key_vault_secret_or_certificate_id  = "https://kv-ssp-0-nonprod-conto.vault.azure.net/secrets/test/39d478f59bf14e45ab36a630a7a87db9"
virtual_network_resource_group_name = "conto-np-appl-ssp-test-rg"
vint_subnet_name                    = "appgw-subnet"

frontend_ip_private = "10.0.3.10"
frontend_port       = ["443"]

key_vault_rbac = true
appgw_public   = true

backend_address_pools = [
  {
    name  = "test-beap"
    ip    = ["10.0.4.4"]
    fqdns = []
  }
]
backend_http_settings = [
  {
    cookie_based_affinity           = "Disabled"
    name                            = "test-be-htst"
    path                            = ""
    port                            = "80"
    protocol                        = "Http"
    request_timeout                 = "20"
    host_name                       = "test.corp"
    probe_name                      = "test-probe"
    connection_draining_enabled     = false
    connection_draining_timeout_sec = "30"
    pick_hostname                   = false
  }
]


http_listeners = [
  {
    name                 = "test-httplstn"
    frontend_port_number = "443"
    protocol             = "Https"
    ssl_certificate_name = "test"
    host_name            = "test.com"
    require_sni          = false
  }
]

probes = [
  {
    name                  = "test-probe"
    interval              = "2"
    timeout               = "5"
    protocol              = "Http"
    path                  = "/"
    unhealthy_threshold   = "2"
    match_status_code     = ["200"]
    pick_hostname_backend = false
    host                  = "test.corp"
  }
]

request_routing_rules = [
  {
    name                        = "test-rqrt"
    priority                    = 1
    http_listener_name          = "test-httplstn"
    backend_address_pool_name   = "test-beap"
    backend_http_settings_name  = "test-be-htst"
    rule_type                   = "Basic"
    url_path_map_name           = ""
    redirect_configuration_name = ""
  }
]

waf_enabled  = true
waf_mode     = "Prevention"
sku_name     = "WAF_v2"
sku_tier     = "WAF_v2"
```

### variables.tf

Example variable definitions for the Application Gateway module:

```hcl// Required Inputs

variable "backend_address_pools" {
  type = list(object({
    name  = string
    fqdns = list(string)
    ip    = list(string)
  }))
 description = "List of backend address pools"
}

variable "backend_http_settings" {
  type = list(object({
    cookie_based_affinity           = string
    name                            = string
    path                            = string
    port                            = number
    protocol                        = string
    request_timeout                 = number
    host_name                       = string
    probe_name                      = string
    connection_draining_enabled     = bool
    connection_draining_timeout_sec = number
    pick_hostname                   = bool
    trusted_root_certificate_names  = optional(list(string))
  }))
  description = "List of backend HTTP settings"
}

variable "capacity" {
  type        = object({ min = number, max = number })
  description = "The Capacity of the SKU to use for this Application Gateway - which must be between 1 and 10"
}

variable "frontend_port" {
  type        = list(string)
  description = "The frontend port used by the listener."
}

variable "http_listeners" {
  type = list(object({
    name                 = string
    host_name            = optional(string)
    require_sni          = bool
    ssl_certificate_name = optional(string)
    protocol             = string
    frontend_port_number = string
  }))
  description = "List of HTTP listeners"
}

variable "key_vault_name" {
  type        = string
  description = "The name of the key vault who store the certificate."
}

variable "private_ip" {
  type        = string
  description = "Not required only if frontend_config_private_ip_address_allocation is set to Dynamic. The Private IP Address to use for the Application Gateway."
}

variable "request_routing_rules" {
  type = list(object({
    name                        = string
    priority                    = number
    http_listener_name          = string
    backend_address_pool_name   = string
    backend_http_settings_name  = string
    url_path_map_name           = string
    rule_type                   = string
    redirect_configuration_name = string
  }))
  description = "List of request routing rules"
}

variable "resource_group_name" {
  type        = string
  description = "The name of the resource group in which to create the Product."
}

variable "subnet_name" {
  type        = string
  description = "The name of the Subnet within which the WAF will be connected. Needs to be different from the name of the subnet in the subnet_id."
}

variable "virtual_network_resource_group_name" {
  type        = string
  description = "The name of the resource group that contains the virtual network."
}

variable "vnet_name" {
  type        = string
  description = "The name of the Vnet within which the WAF will be connected."
}

variable "waf_enabled" {
  type        = bool
  description = "Is the Web Application Firewall enabled? Allowed values: true or false."
}

// Optional Inputs

variable "appgw_public" {
  type        = bool
  description = "Indicate if the Application gateway should have private or public exposure"
  default     = false
}

variable "backend_pools_associate_nics" {
  type        = map(list(string))
  description = "When you want to create a backend pool pointing directly to VM nics. Expects the backend pool with the `backend_name` property to already be created. The name of the map element should be the name of the backend."
  default     = {}
}

variable "certificates_load_type" {
  type        = string
  description = "Upload method used for SSL and Trusted Root Certificates. Accepted values are Local or Keyvault."
  default     = "Keyvault"
}

variable "certificates_path" {
  type        = string
  description = "Only if ssl_certificates is set (not the ssl_certificates_keyvault) and certificates_load_type is set to Local. Path where are located the certificates (both authentication_certificates and ssl certificates). Required if ssl_certificates is set."
  default     = ""
}

variable "cipher_suites" {
  type        = list(any)
  description = "A List of accepted cipher suites. Accepted values are: TLS_RSA_WITH_AES_128_GCM_SHA256, TLS_RSA_WITH_AES_256_GCM_SHA384, TLS_DHE_RSA_WITH_AES_128_GCM_SHA256, TLS_DHE_RSA_WITH_AES_256_GCM_SHA384, TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256, TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384, TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256, TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
  default     = ["TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256", "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]
}

variable "create_backend_nic" {
  type        = bool
  description = "Enable creation of a Network Interface Card for the Backend Address Pool Association. Accepted values are true or false"
  default     = false
}

variable "environment" {
  type        = string
  description = "The environment. e.g. dev, qa, uat, prod"
  default     = "dev"
}

variable "firewall_policy_id" {
  type        = string
  description = "The ID of the Web Application Firewall Policy."
  default     = null
}

variable "key_vault_rbac" {
  type        = bool
  description = "extension tags"
  default     = true
}

variable "location" {
  type        = string
  description = "Specifies the supported Azure location where the resource exists. Changing this forces a new product to be created."
  default     = "westeurope"
}

variable "network_interfaces" {
  type        = list(string)
  description = "Required if there are backend_address_pools and create_backend_nic is set to true. A list of names of network Interfaces that should be created. IE: nic01, nic02, ... in a format of a list of strings. It Needs to be the same number of network_interfaces as the number of backend_address_pools there are configured for the app gateway, and the same number of IPs in the private_ip_address_nic_association"
  default     = []
}

variable "private_ip_address_allocation" {
  type        = string
  description = "Required if there are backend_address_pools and create_backend_nic is set to true. The allocation method used for the Private IP Address in the NIC Association with the Backend Address Pool (only if necessary). Possible values are Dynamic and Static. If Static then private_ip_address_nic_association (list of Static IPs) needs to be configured"
  default     = "Static"
}

variable "probes" {
  type = list(object({
    host                  = string
    interval              = number
    name                  = string
    protocol              = string
    path                  = string
    timeout               = number
    unhealthy_threshold   = number
    match_status_code     = list(string)
    pick_hostname_backend = bool
  }))
  default = []
}

variable "project" {
  type        = string
  description = "The name of the project. e.g. prj"
  default     = "prj"
}

variable "rewrite_rule_sets" {
  type = list(object({
    name = string
    rewrite_rules = list(object({
      name          = string
      rule_sequence = string
      conditions = list(object({
        variable    = string
        pattern     = string
        ignore_case = bool
        negate      = bool
      }))
      request_header_configurations = list(object({
        header_name  = string
        header_value = string
      }))
      response_header_configurations = list(object({
        header_name  = string
        header_value = string
      }))
      urls = list(object({
        path         = string
        query_string = string
        reroute      = bool
      }))
    }))
  }))
  description = "List of rewrite rule sets."
  default     = []
}

variable "private_ip_addresses_nic_association" {
  type        = list(string)
  description = "Required if there are backend_address_pools and create_backend_nic is set to true. A list of static private IP addresses which should be used to associate NIC within the backend address pool. As many IP addresses as network_interfaces and backend_address_pools there are. IMPORTANT: the IP addresses need to be located in the subnet_id, and need to be different from the IPs configured within the app gateway (backend_address_pools)."
  default     = []
}

variable "redirect_configurations" {
  type = list(object({
    name                 = string
    redirect_type        = string
    target_url           = string
    include_path         = bool
    include_query_string = bool
  }))
  description = "A list of redirections."
  default     = []
}

variable "sku_name" {
  type        = string
  description = "The Name of the SKU to use for this Application Gateway. Accepted values are Standard_v2 and WAF_v2. Please refrain from using the WAF_v2 mode (use only if necessary), instead use the Santander Imperva WAF."
  default     = "Standard_v2"
}

variable "sku_tier" {
  type        = string
  description = "The Tier of the SKU to use for this Application Gateway. Accepted values are Standard_v2 and WAF_v2. Please refrain from using the WAF_v2 mode (use only if necessary), instead use the Santander Imperva WAF."
  default     = "Standard_v2"
}

variable "ssl_certificates_keyvault" {
  type = list(object({
    name                               = string
    key_vault_secret_or_certificate_id = string
  }))
  description = "Only if ssl_certificates is not set and certificates_load_type is set to Keyvault. A list of ssl certificates with .pfx or .pem extension located in the keyvault as certificates (important: extract the secret id url from the **certificate**. Take note that it needs to be a Certificate, not a secret. This certificate in the keyvault has a Secret ID). It consists of a ==> name:The Name of the SSL certificate that is unique within this Application Gateway; key_vault_secret_or_certificate_id: Secret ID of the Keyvault Certificate that has the .pfx or .pem file."
  default     = []
}

variable "subnet_ids_nic_association" {
  type        = list(string)
  description = "Required only if create_backend_nic is set to true. IDs of the subnets used for NIC association to the backend."
  default     = []
}

variable "trusted_root_certificates" {
  type = list(
    object({
      name        = string
      certificate = string
    })
  )
  description = "List of objects taking the trusted root certificates in .crt extension for backend http settings"
  default     = []
}

variable "url_path_map" {
  type = list(object({
    name = string
    path_rule = list(
      object({
        name                        = string
        paths                       = list(string)
        backend_http_settings_name  = string
        backend_address_pool_name   = string
        redirect_configuration_name = string
      })
    )
    redirect_setting = list(
      object({
        name                        = string
        paths                       = list(string)
        backend_http_settings_name  = string
        backend_address_pool_name   = string
        redirect_configuration_name = string
      })
    )
    default_backend_address_pool_name   = string
    backend_settings_name               = string
    default_redirect_configuration_name = string
  }))
  description = "Maps directly to Application gateways's url_path_map."
  default     = []
}

variable "waf_disabled_rule_group" {
  type = list(
    object({
      rule_group_name = string
      rules           = list(string)
    })
  )
  description = "List of rules that will be removed of the waf"
  default     = []
}

variable "waf_file_upload_limit_mb" {
  type        = string
  default     = "100"
  description = "Required if waf_enabled is true. The File Upload Limit in MB. Accepted values are in the range 1MB to 500MB. Defaults to 100MB."
}

variable "waf_max_request_body_size_kb" {
  type        = string
  default     = "128"
  description = "Required if waf_enabled is true. The Maximum Request Body Size in KB. Accepted values are in the range 1KB to 128KB. Defaults to 128KB."
}

variable "waf_mode" {
  type        = string
  description = "The WAF mode, Prevention of Detection for test purposes."
  default     = "Prevention"
}

variable "waf_rule_version" {
  type        = string
  description = "Required if waf_enabled is true. The Version of the Rule Set used for this Web Application Firewall. Possible values are 2.2.9, 3.0, 3.1 and 3.2"
  default     = "3.2"
}

variable "zones" {
  description = "A collection of availability zones to spread the Application Gateway over."
  type        = list(string)
  default     = null
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
```

## Input arguments and outputs

----------------------------;

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| `appgw_public` | Indicate if the Application gateway should have private or public exposure | `bool` | `false` | no |
| `backend_address_pools` | List of backend address pools | <pre>list(object({<br>    name  = string<br>    fqdns = list(string)<br>    ip    = list(string)<br>  }))</pre> | n/a | yes |
| `backend_http_settings` | List of backend HTTP settings | <pre>list(object({<br>    cookie_based_affinity           = string<br>    name                            = string<br>    path                            = string<br>    port                            = number<br>    protocol                        = string<br>    request_timeout                 = number<br>    host_name                       = string<br>    probe_name                      = string<br>    connection_draining_enabled     = bool<br>    connection_draining_timeout_sec = number<br>    pick_hostname                   = bool<br>    trusted_root_certificate_names  = optional(list(string))<br>  }))</pre> | n/a | yes |
| `backend_pools_associate_nics` | When you want to create a backend pool pointing directly to VM nics. Expects the backend pool with the `backend_name` property to already be created. The name of the map element should be the name of the backend. | `map(list(string))` | `{}` | no |
| `capacity` | The Capacity of the SKU to use for this Application Gateway - which must be between 1 and 10 | `object({ min = number, max = number })` | n/a | yes |
| `certificates_load_type` | Upload method used for SSL and Trusted Root Certificates. Accepted values are Local or Keyvault. | `string` | `"Keyvault"` | no |
| `certificates_path` | Only if ssl\_certificates is set (not the ssl\_certificates\_keyvault) and certificates\_load\_type is set to Local. Path where are located the certificates (both authentication\_certificates and ssl certificates). Required if ssl\_certificates is set. | `string` | `""` | no |
| `cipher_suites` | A List of accepted cipher suites. Accepted values are: TLS\_RSA\_WITH\_AES\_128\_GCM\_SHA256, TLS\_RSA\_WITH\_AES\_256\_GCM\_SHA384, TLS\_DHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256, TLS\_DHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384, TLS\_ECDHE\_ECDSA\_WITH\_AES\_128\_GCM\_SHA256, TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_GCM\_SHA384, TLS\_ECDHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256, TLS\_ECDHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384 | `list(any)` | <pre>[<br>  "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",<br>  "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"<br>]</pre> | no |
| `create_backend_nic` | Enable creation of a Network Interface Card for the Backend Address Pool Association. Accepted values are true or false | `bool` | `false` | no |
| `environment` | The environment. e.g. dev, qa, uat, prod | `string` | `"dev"` | no |
| `firewall_policy_id` | The ID of the Web Application Firewall Policy. | `string` | `null` | no |
| `frontend_port` | The frontend port used by the listener. | `list(string)` | n/a | yes |
| `http_listeners` | List of HTTP listeners | <pre>list(object({<br>    name                 = string<br>    host_name            = optional(string)<br>    require_sni          = bool<br>    ssl_certificate_name = optional(string)<br>    protocol             = string<br>    frontend_port_number = string<br>  }))</pre> | n/a | yes |
| `key_vault_name` | The name of the key vault who store the certificate. | `string` | n/a | yes |
| `key_vault_rbac` | extension tags | `bool` | `true` | no |
| `location` | Specifies the supported Azure location where the resource exists. Changing this forces a new product to be created. | `string` | `"westeurope"` | no |
| `network_interfaces` | Required if there are backend\_address\_pools and create\_backend\_nic is set to true. A list of names of network Interfaces that should be created. IE: nic01, nic02, ... in a format of a list of strings. It Needs to be the same number of network\_interfaces as the number of backend\_address\_pools there are configured for the app gateway, and the same number of IPs in the private\_ip\_address\_nic\_association | `list(string)` | `[]` | no |
| `private_ip` | Not required only if frontend\_config\_private\_ip\_address\_allocation is set to Dynamic. The Private IP Address to use for the Application Gateway. | `string` | n/a | yes |
| `private_ip_address_allocation` | Required if there are backend\_address\_pools and create\_backend\_nic is set to true. The allocation method used for the Private IP Address in the NIC Association with the Backend Address Pool (only if necessary). Possible values are Dynamic and Static. If Static then private\_ip\_address\_nic\_association (list of Static IPs) needs to be configured | `string` | `"Static"` | no |
| `private_ip_addresses_nic_association` | Required if there are backend\_address\_pools and create\_backend\_nic is set to true. A list of static private IP addresses which should be used to associate NIC within the backend address pool. As many IP addresses as network\_interfaces and backend\_address\_pools there are. IMPORTANT: the IP addresses needs to be located in the subnet\_id, and needs to be different from the IPs configured within the app gateway (backend\_address\_pools) | `list(string)` | `[]` | no |
| `probes` | List of probes | <pre>list(object({<br>    host                  = string<br>    interval              = number<br>    name                  = string<br>    protocol              = string<br>    path                  = string<br>    timeout               = number<br>    unhealthy_threshold   = number<br>    match_status_code     = list(string)<br>    pick_hostname_backend = bool<br>  }))</pre> | `[]` | no |
| `project` | The name of the project. e.g. prj | `string` | `"prj"` | no |
| `redirect_configurations` | A list of redirections. | <pre>list(object({<br>    name                 = string<br>    redirect_type        = string<br>    target_url           = string<br>    include_path         = bool<br>    include_query_string = bool<br>  }))</pre> | `[]` | no |
| `request_routing_rules` | List of request routing rules | <pre>list(object({<br>    name                        = string<br>    priority                    = number<br>    http_listener_name          = string<br>    backend_address_pool_name   = string<br>    backend_http_settings_name  = string<br>    url_path_map_name           = string<br>    rule_type                   = string<br>    redirect_configuration_name = string<br>  }))</pre> | n/a | yes |
| `resource_group_name` | The name of the resource group in which to create the Product. | `string` | n/a | yes |
| `rewrite_rule_sets` | List of rewrite rule sets | <pre>list(object({<br>    name = string<br>    rewrite_rules = list(object({<br>      name          = string<br>      rule_sequence = string<br>      conditions = list(object({<br>        variable    = string<br>        pattern     = string<br>        ignore_case = bool<br>        negate      = bool<br>      }))<br>      request_header_configurations = list(object({<br>        header_name  = string<br>        header_value = string<br>      }))<br>      response_header_configurations = list(object({<br>        header_name  = string<br>        header_value = string<br>      }))<br>      urls = list(object({<br>        path         = string<br>        query_string = string<br>        reroute      = bool<br>      }))<br>    }))<br>  }))</pre> | `[]` | no |
| `sku_name` | The Name of the SKU to use for this Application Gateway. Accepted values are Standard\_v2 and WAF\_v2. Please refrain from using the WAF\_v2 mode (use only if necessary), instead use the Santander Imperva WAF. | `string` | `"Standard_v2"` | no |
| `sku_tier` | The Tier of the SKU to use for this Application Gateway. Accepted values are Standard\_v2 and WAF\_v2. Please refrain from using the WAF\_v2 mode (use only if necessary), instead use the Santander Imperva WAF. | `string` | `"Standard_v2"` | no |
| `ssl_certificates_keyvault` | Only if ssl\_certificates is not set and certificates\_load\_type is set to Keyvault. A list of ssl certificates with .pfx or .pem extension located in the keyvault as certificates (important: extract the secret id url from the **certificate**. Take note that it needs to be a Certificate, not a secret. This certificate in the keyvault has a Secret ID). It consists of a ==> name:The Name of the SSL certificate that is unique within this Application Gateway; key\_vault\_secret\_or\_certificate\_id: Secret ID of the Keyvault Certificate that has the .pfx or .pem file. | <pre>list(object({<br>    name                               = string<br>    key_vault_secret_or_certificate_id = string<br>  }))</pre> | `[]` | no |
| `subnet_ids_nic_association` | Required only if create\_backend\_nic is set to true. IDs of the subnets used for NIC association to the backend. | `list(string)` | `[]` | no |
| `subnet_name` | The name of the Subnet within which the WAF will be connected. Needs to be different from the name of the subnet in the subnet\_id. | `string` | n/a | yes |
| `trusted_root_certificates` | List of objects taking the trusted root certificates in .crt extension for backend http settings | <pre>list(<br>    object({<br>      name        = string<br>      certificate = string<br>    })<br>  )</pre> | `[]` | no |
| `url_path_map` | Maps directly to Application gateways's url\_path\_map. | <pre>list(object({<br>    name = string<br>    path_rule = list(<br>      object({<br>        name                        = string<br>        paths                       = list(string)<br>        backend_http_settings_name  = string<br>        backend_address_pool_name   = string<br>        redirect_configuration_name = string<br>      })<br>    )<br>    redirect_setting = list(<br>      object({<br>        name                        = string<br>        paths                       = list(string)<br>        backend_http_settings_name  = string<br>        backend_address_pool_name   = string<br>        redirect_configuration_name = string<br>      })<br>    )<br>    default_backend_address_pool_name   = string<br>    backend_settings_name               = string<br>    default_redirect_configuration_name = string<br>  }))</pre> | `[]` | no |
| `virtual_network_resource_group_name` | The name of the resource group that contains the virtual network. | `string` | n/a | yes |
| `vnet_name` | The name of the Vnet within which the WAF will be connected. | `string` | n/a | yes |
| `waf_disabled_rule_group` | List of rules that will be removed of the waf | <pre>list(<br>    object({<br>      rule_group_name = string<br>      rules           = list(string)<br>    })<br>  )</pre> | `[]` | no |
| `waf_enabled` | Is the Web Application Firewall be enabled? Allowed values: true or false. | `bool` | n/a | yes |
| `waf_file_upload_limit_mb` | Required if waf\_enabled is true. The File Upload Limit in MB. Accepted values are in the range 1MB to 500MB. Defaults to 100MB. | `string` | `"100"` | no |
| `waf_max_request_body_size_kb` | Required if waf\_enabled is true. The Maximum Request Body Size in KB. Accepted values are in the range 1KB to 128KB. Defaults to 128KB. | `string` | `"128"` | no |
| `waf_mode` | The WAF mode, Prevention of Detection for test purposes. | `string` | `"Prevention"` | no |
| `waf_rule_version` | Required if waf\_enabled is true. The Version of the Rule Set used for this Web Application Firewall. Possible values are 2.2.9, 3.0, 3.1 and 3.2 | `string` | `"3.2"` | no |
| `zones` | A collection of availability zones to spread the Application Gateway over. | `list(string)` | `null` | no |

### Outputs

| Name | Description |
|------|-------------|
| `application_gateway_id` | The ID of the Application Gateway. |
