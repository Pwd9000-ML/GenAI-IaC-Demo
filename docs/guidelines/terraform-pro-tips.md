# Terrafrom Pro Tips at Contoso

Welcome to Contoso's Terraform Pro Tips! In this guide, you will learn about best practices and guidelines for using Terraform at Contoso. These tips will help you write more efficient and maintainable Terraform configurations, and ensure that your infrastructure is secure and reliable.

## Table of Contents

- [Module 1: Terraform Pro Tips - Complex Variable Types](#module-1-terraform-pro-tips---complex-variable-types)
- [Module 2: Terraform Pro Tips - Creating dynamic variables using locals](#module-2-terraform-pro-tips---creating-dynamic-variables-using-locals)
- [Module 3: Terraform Pro Tips - Filter results using 'for' loops](#module-3-terraform-pro-tips---filter-results-using-for-loops)
- [Module 4: Terraform Pro Tips - Selective configuration with 'lookup()'](#module-4-terraform-pro-tips---selective-configuration-with-lookup)
- [Module 5: Terraform Pro Tips - Sensitive Output](#module-5-terraform-pro-tips---sensitive-output)
- [Module 6: Terraform Pro Tips - Fun with Functions](#module-6-terraform-pro-tips---fun-with-functions)
- [Module 7: Terraform Pro Tips - Understanding the Lifecycle Block](#module-7-terraform-pro-tips---understanding-the-lifecycle-block)
- [Module 8: Terraform Pro Tips - Variable Validation](#module-8-terraform-pro-tips---variable-validation)
- [Module 9: Terraform Pro Tips - Understanding Dynamic Blocks](#module-9-terraform-pro-tips---understanding-dynamic-blocks)
- [Module 10: Terraform Pro Tips - Understanding Implicit and Explicit Dependencies](#module-10-terraform-pro-tips---understanding-implicit-and-explicit-dependencies)
- [Module 11: Terraform Pro Tips - Understanding Count and For_Each Loops](#module-11-terraform-pro-tips---understanding-count-and-for_each-loops)

---

# Module 1: Terraform Pro Tips - Complex Variable Types

## Terraform Variables

When creating a terraform configuration, you have to configure and declare [Input Variables](https://www.terraform.io/docs/language/values/variables.html). Input variables serve as parameters for a Terraform module and resources, allowing aspects of the module to be customized without altering the module's own source code, and allowing modules to be shared between different configurations.

The Terraform language uses the following types for its values:

- `string`: a sequence of Unicode characters representing some text, like "hello".
- `number`: a numeric value. The number type can represent both whole numbers like 15 and fractional values like 6.283185.
- `bool`: a boolean value, either true or false. bool values can be used in conditional logic.
- `list` (or `tuple`): a sequence of values, like `["one", "two"]`. Elements in a list or tuple are identified by consecutive whole numbers, starting with zero.
- `map` (or `object`): a group of values identified by named labels, like `{name = "Mabel", age = 52}`.

Strings, numbers, and bools are sometimes called _primitive_ types. Lists/tuples and maps/objects are sometimes called _complex_ types, _structural_ types, or _collection_ types.

## Using Primitive Variable Types

In the following example we create a basic Azure Resource Group and we declare each resource argument with it's own separate variable using _Primitive_ types:

```hcl
#main.tf
resource "azurerm_resource_group" "demo_rg" {
  count    = var.create_rg ? 1 : 0
  name     = var.name
  location = var.location
}
```

Each variable is declared separately:

```hcl
#variables.tf
variable "create_rg" {
    type = bool
    default = false
}

variable "name" {
    type = string
    default = "Default-RG-Name"
}

variable "location" {
    type = string
    default = "uksouth"
}
```

As you can see from the above example each resource argument is declared using a _primitive_ variable type.

## Using Complex Variable Types

In the following example we create an Azure Resource Group and two storage accounts, but instead of declaring each variable individually using _primitive_ types we will use **Collections** using _complex_ types. We will create our Resource Group by using a single complex variable called `rg_config` and we will create our storage account/s using a single complex variable list of objects called `storage_config`.

As you can see from the following variable declaration, we are only declaring each resources values using a _complex_ variable type of **Object** (Resource Group config) and **List Object** (List of Storage Account configs):

```hcl
#// code/variables.tf#L1-L20
#Resource Group Config - Object
variable "rg_config" {
  type = object({
    create_rg = bool
    name      = string
    location  = string
  })
}

#Storage Account Config - List of Objects (Each object represents a storage config)
variable "storage_config" {
  type = list(object({
    name                      = string
    account_kind              = string
    account_tier              = string
    account_replication_type  = string
    access_tier               = string
    enable_https_traffic_only = bool
    min_tls_version           = string
    is_hns_enabled            = bool
  }))
}
```

**NOTE:** Because we are using variable objects we can just reference and lookup each key of the relevant object passed in to obtain the corresponding configuration value e.g. `var.config.key`:

### Example using COUNT

```hcl
#// code/resources.tf#L6-L32
resource "azurerm_resource_group" "demo_rg" {
  count    = var.rg_config.create_rg ? 1 : 0
  name     = var.rg_config.name
  location = var.rg_config.location
  tags     = { Purpose = "Demo-RG", Automation = "true" }
}

## COUNT Example ##
resource "azurerm_storage_account" "sas" {
  count = length(var.storage_config)

  #Implicit dependency from previous resource
  resource_group_name = azurerm_resource_group.demo_rg[0].name
  location            = azurerm_resource_group.demo_rg[0].location

  #values from variable storage_config objects
  name                      = var.storage_config[count.index].name
  account_kind              = var.storage_config[count.index].account_kind
  account_tier              = var.storage_config[count.index].account_tier
  account_replication_type  = var.storage_config[count.index].account_replication_type
  access_tier               = var.storage_config[count.index].access_tier
  enable_https_traffic_only = var.storage_config[count.index].enable_https_traffic_only
  min_tls_version           = var.storage_config[count.index].min_tls_version
  is_hns_enabled            = var.storage_config[count.index].is_hns_enabled

  #Apply tags
  tags = { Purpose = "Demo-sa-${count.index + 1}", Automation = "true" }
}
```

### Example using FOR_EACH

```hcl
resource "azurerm_resource_group" "demo_rg" {
  count    = var.rg_config.create_rg ? 1 : 0
  name     = var.rg_config.name
  location = var.rg_config.location
  tags     = { Purpose = "Demo-RG", Automation = "true" }
}

## FOR_EACH Example ##
resource "azurerm_storage_account" "sas" {
  for_each = { for each in var.storage_config : each.name => each )

  #Implicit dependency from previous resource
  resource_group_name = azurerm_resource_group.demo_rg[0].name
  location            = azurerm_resource_group.demo_rg[0].location

  #values from variable storage_config objects
  name                      = each.value.name
  account_kind              = each.value.account_kind
  account_tier              = each.value.account_tier
  account_replication_type  = each.value.account_replication_type
  access_tier               = each.value.access_tier
  enable_https_traffic_only = each.value.enable_https_traffic_only
  min_tls_version           = each.value.min_tls_version
  is_hns_enabled            = each.value.is_hns_enabled

  #Apply tags
  tags = { Purpose = "Demo-sa-${count.index + 1}", Automation = "true" }
}
```

Because we are now using a **list of objects** as the variable for storage accounts, each storage account we want to create can be configured on our **TFVARS** file as an object inside its own block, and so we can simply add additional object blocks into our **TFVARS** to build `one` or `many` storage accounts, each with different configs:

```hcl
#// code/common.auto.tfvars.tf#L1-L30
#Resource Group Config - Object Values
rg_config = {
  create_rg = true
  name      = "Demo-Terraform-RG"
  location  = "uksouth"
}

#Storage Account Configs - List of Objects Values
storage_config = [
  #Storage Account 1 (Object1): StorageV2
  {
    name                      = "pwd9000v2sa001"
    account_kind              = "StorageV2"
    account_tier              = "Standard"
    account_replication_type  = "LRS"
    min_tls_version           = "TLS1_2"
    enable_https_traffic_only = true
    access_tier               = "Cool"
    is_hns_enabled            = false
  },
  #Storage Account 2 (object2): Azure Data Lake Storage V2 (ADLS2)
  {
    name                      = "pwd9000adls2sa001"
    account_kind              = "BlockBlobStorage"
    account_tier              = "Premium"
    account_replication_type  = "ZRS"
    min_tls_version           = "TLS1_2"
    enable_https_traffic_only = false
    access_tier               = "Hot"
    is_hns_enabled            = true
  }
]
```

As you can see from the last example, using complex variable types and making our configurations more object oriented can offer much greater flexibility and granularity in terraform deployments.

You can also find the code samples used in this blog post on my [GitHub](https://github.com/Pwd9000-ML/blog-devto/tree/main/posts/2021/DevOps-Terraform-Complex-Vars/code) page. :heart:

---

# Module 2: Terraform Pro Tips - Creating dynamic variables using locals

## Overview

This tutorial uses examples from the following GitHub project: [Azure Terraform Deployments](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments).

In todays tutorial we will look at an interesting use case example whereby we will be creating a dynamic Terraform variable using [locals](https://www.terraform.io/language/values/locals) and a [for loop](https://www.terraform.io/language/expressions/for).

Let's take a moment to talk about the use case before going into the code. We will use Terraform to build the following:

- Resource Group
- Virtual network
- App Service Plan
- App Insights
- VNET integrated App Service
- Azure Container Registry (ACR)

Some Azure PaaS services (such an ACR) has networking features called **Firewalls and Virtual networks** which gives us the ability to configure allowed public network access where we can define **Firewall IP whitelist** rules or allow only **selected networks** access, in order to limit network connectivity to the PaaS service.

By default an ACR is public and accepts connections over the internet from hosts on any network. So we will as part of the Terraform configuration restrict all network access to the ACR and use the **Firewall IP whitelist** to only allow the outbound IPs of our VNET integrated **App service**. In addition we will also provide a list that contains **custom IP ranges** we can set which will represent the on premises public IPs of our company to also be included on the **Firewall IP whitelist** of the ACR.

**IMPORTANT:** ACR `network_rule_set_set` can only be specified for a **Premium** Sku.

Since we are building all of this with IaC using Terraform the question is how can we allow all the **possible outbound IPs** of our VNET integrated **App Service** to be whitelisted on the **ACR** if the outbound IPs of the App Service will not be known to us until the App Service is deployed?

This is where I will demonstrate how we can achieve this using **Dynamic Variables** to dynamically create the IP whitelist we can use using **locals**.

## App Service (VNET integrated)

In the following demo [configuration](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/04_App_Acr). Let's take a closer look at the App service configuration and VNET integration:

**NOTE:** All the code samples used in this tutorial are updated to use the the latest version of the **AzureRM provider 3.0**.

### App Service resource ([appservices.tf](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/blob/master/04_App_Acr/appservices.tf))

```hcl
## appservices.tf ##
resource "azurerm_linux_web_app" "APPSVC" {
  name                = var.appsvc_name
  location            = azurerm_resource_group.RG.location
  resource_group_name = azurerm_resource_group.RG.name
  service_plan_id     = azurerm_service_plan.ASP.id
  https_only          = true

  identity {
    type = "SystemAssigned"
  }

  site_config {
    container_registry_use_managed_identity = true
    ftps_state                              = "FtpsOnly"
    application_stack {
      docker_image     = "${var.acr_name}.azurecr.io/${var.appsvc_name}"
      docker_image_tag = "latest"
    }
    vnet_route_all_enabled = var.vnet_route_all_enabled
  }

  app_settings = lookup(local.app_settings, "linux_app_settings", null)
}
```

### VNET integration resource ([appservices.tf](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/blob/master/04_App_Acr/appservices.tf))

**NOTE:** Outbound IPs will only become available once the VNET integration resource has been created.

```hcl
## appservice.tf ##
resource "azurerm_app_service_virtual_network_swift_connection" "azure_vnet_connection" {
  count          = var.vnet_integ_required == true ? 1 : 0
  app_service_id = azurerm_linux_web_app.APPSVC.id
  subnet_id      = azurerm_subnet.SUBNETS["App-Service-Integration-Subnet"].id
}
```

### Azure Container Registry (ACR) resource ([acr.tf](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/blob/master/04_App_Acr/acr.tf))

```hcl
## acr.tf ##
resource "azurerm_container_registry" "ACR" {
  name                = var.acr_name
  location            = azurerm_resource_group.RG.location
  resource_group_name = azurerm_resource_group.RG.name
  sku                 = var.acr_sku
  admin_enabled       = var.acr_admin_enabled

  dynamic "identity" {
    for_each = var.acr_requires_identity == true ? [1] : []
    content {
      type = "SystemAssigned"
    }
  }

  dynamic "georeplications" {
    for_each = var.acr_sku == "Premium" ? var.acr_georeplications_configuration : []
    content {
      location                = georeplications.value.location
      zone_redundancy_enabled = georeplications.value.zone_redundancy_enabled
    }
  }

  network_rule_set = [
    {
      default_action = var.acr_network_rule_set_default_action
      ip_rule = [for each in local.acr_ip_rules :
        {
          action   = each["action"]
          ip_range = each["ip_range"]
        }
      ]
      virtual_network = [for each in local.acr_virtual_network_subnets :
        {
          action    = each["action"]
          subnet_id = each["subnet_id"]
        }
      ]
    }
  ]

}
```

As you can see from the above resource block that is building out the ACR notice the attribute called `network_rule_set`:

```hcl
## acr.tf ##
## Need Premium SKU to use the following config ##
network_rule_set = [
  {
    default_action = var.acr_network_rule_set_default_action
    ip_rule = [for each in local.acr_ip_rules :
      {
        action   = each["action"]
        ip_range = each["ip_range"]
      }
    ]
    virtual_network = [for each in local.acr_virtual_network_subnets :
      {
        action    = each["action"]
        subnet_id = each["subnet_id"]
      }
    ]
  }
]
```

You will note that in `network_rule_set` we are using `for` loops on **local** values called `local.acr_ip_rules` and `local.acr_virtual_network_subnets`. We are also using a variable to declare the default action `default_action = var.acr_network_rule_set_default_action` which is set to `"Deny"`.

Lets take a look at the **local.tf** file in more detail:

### Locals ([local.tf](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/blob/master/04_App_Acr/local.tf))

Notice the locals variables called `allowed_ips`, `local.acr_ip_rules` and `local.acr_virtual_network_subnets`:

```hcl
## ACR Firewall rules ##
#Get all possible outbound IPs from VNET integrated App services and combine with allowed On Prem IP ranges from var.acr_custom_fw_rules
allowed_ips = distinct(flatten(concat(azurerm_linux_web_app.APPSVC.possible_outbound_ip_address_list, var.acr_custom_fw_rules)))

acr_ip_rules = [for i in local.allowed_ips :
  {
    action   = "Allow"
    ip_range = i
  }
]

acr_virtual_network_subnets = [
  {
    action    = "Allow"
    subnet_id = azurerm_subnet.SUBNETS["App-Service-Integration-Subnet"].id
  }
]
```

Let's take a closer look at `allowed_ips` first:

```hcl
## local.tf ##
allowed_ips = distinct(flatten(concat(azurerm_linux_web_app.APPSVC.possible_outbound_ip_address_list, var.acr_custom_fw_rules)))
```

This locals variable uses a few Terraform functions and I will explain each function separately.

The first function is called [concat()](https://www.terraform.io/language/functions/concat). The **concat function** will combine two or more lists into a single list. As you can see from the values in the brackets, we are taking the output from the **App service (APPSVC)** we created earlier, called **possible_outbound_ip_address_list** and combining it with a variable (list) called **var.acr_custom_fw_rules**.

```hcl
concat(azurerm_linux_web_app.APPSVC.possible_outbound_ip_address_list, var.acr_custom_fw_rules)
```

Here is the variable we can expand on manually if needed:

```hcl
## variables.tf ##
variable "acr_custom_fw_rules" {
  type        = list(string)
  description = "Specifies a list of custom IPs or CIDR ranges to whitelist on the ACR."
  default     = null
}

## config-dev.tfvars ##
acr_custom_fw_rules = ["183.44.33.0/24", "8.8.8.8"]
```

So the end result of our function: `concat(azurerm_linux_web_app.APPSVC.possible_outbound_ip_address_list, var.acr_custom_fw_rules)` will give us one list of our custom IPs and IP ranges, combined with the list of possible outbound IPs from the **App service** we are building.

The next function is called [flatten()](https://www.terraform.io/language/functions/flatten). This function will just flatten any nested lists we combined using concat, into a single flat list:

```hcl
flatten(concat(azurerm_linux_web_app.APPSVC.possible_outbound_ip_address_list, var.acr_custom_fw_rules))
```

The last function is called [distinct()](https://www.terraform.io/language/functions/distinct). This function will just remove any duplicate IPs or ranges. (The `distinct()` function is handy if we are building more than one app service and want to combine all the IPs of all the App services, and remove any the duplicate IPs from our final list.)

```hcl
distinct(flatten(concat(azurerm_linux_web_app.APPSVC.possible_outbound_ip_address_list, var.acr_custom_fw_rules)))
```

Let's take a closer look at `acr_ip_rules` next:

```hcl
## local.tf##
acr_ip_rules = [for i in local.allowed_ips :
  {
    action   = "Allow"
    ip_range = i
  }
]
```

If you remember the attribute `network_rule_set` we created on our **acr.tf** resource config earlier, we need to specify a nested attribute called `ip_rule`and the default action is set to `"Deny"`, but if you see the `ip_rule` value, we are using a **for loop** to construct a dynamic rule set based on the value of our `local.acr_ip_rules`:

```hcl
## acr.tf ##
ip_rule = [for each in local.acr_ip_rules :
  {
    action   = each["action"]
    ip_range = each["ip_range"]
  }
]
```

This loop will **dynamically** create an **"Allow"** entry on our ACR firewall for each outbound IP of our **App service**, as well as the custom IPs/ranges we added via our custom variable called **var.acr_custom_fw_rules**.

As you can see from this tutorial, we can secure our public ACR using **Firewall rules** that are dynamically created by only allowing our **App services** and **On premise IPs/ranges** to connect into our ACR.

You can also find the code samples used in this blog post on my [GitHub](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/04_App_Acr) page. :heart:

---

# Module 3: Terraform Pro Tips - Filter results using 'for' loops

## Overview

In todays tutorial we will take a look at a fairly common question I often get from the community and it is around how to filter results in Terraform or even if it is possible. We will also look at a real world usage example so that we can see how and when we would use filters in Terraform.

**Filtering** in Terraform can be achieved using [for loop](https://www.terraform.io/language/expressions/for) expressions. Though `for` loop constructs in terraform performs looping, it can also be used for manipulating data structures such as the following to name a few:

- **Transform:** Changing the data structure.
- **Filter:** Filter only on desired items in combination with an `if` expression.
- **Group:** Group elements together in a new `list` by key.

## Filtering results

Let's take a look at the following example variable where we have a list of applications:

```hcl
variable "apps" {
  type = list(object({
    app_name            = string
    app_kind            = string
    app_require_feature = bool
  }))
  default = [
    {
      app_name            = "App1"
      app_kind            = "Linux"
      app_require_feature = false
    },
    {
      app_name            = "App2"
      app_kind            = "Linux"
      app_require_feature = false
    },
    {
      app_name            = "App3"
      app_kind            = "Windows"
      app_require_feature = true
    },
    {
      app_name            = "App4"
      app_kind            = "Windows"
      app_require_feature = false
    }
  ]
}
```

Say you want to filter only on `app_require_feature = true` you could write a `for` loop with an `if` expression like in the following local variable:

```hcl
locals {
  apps_that_require_feature = toset([for each in var.apps : each.app_name if each.app_require_feature == true])
}

output "result" {
  value = local.apps_that_require_feature
}
```

This will return a set of `app_names` that have the objects key `"app_require_feature"` set to `true`

```sh
$ terraform apply
Outputs:

result = ["App3"]
```

So let's say you want to filter the same variable but this time you want to only see the apps that are `windows`, you could write a `for` loop with an `if` expression like in the following local variable:

```hcl
locals {
  windows_apps = toset([for each in var.apps : each.app_name if each.app_kind == "windows"])
}

output "result2" {
  value = local.windows_apps
}
```

This will return a set of `app_names` that have the objects key `"app_kind"` set to `"windows"`

```sh
$ terraform apply
Outputs:

result2 = ["App3", "App4"]
```

## Real world example

Let's take a real world usage case where we would need such a `for` construct to filter and only configure something based on certain criteria.

Say we have a variable with four `storage accounts` we want to create, but we only want to configure `private endpoints` on certain storage accounts. We could create an extra object `key` item called `requires_private_endpoint` like in the following example:

```hcl
## variables ##

variable "storage_config" {
  type = list(object({
    name                      = string
    account_kind              = string
    account_tier              = string
    account_replication_type  = string
    access_tier               = string
    enable_https_traffic_only = bool
    is_hns_enabled            = bool
    requires_private_endpoint = bool
  }))
  default = [
    #V2 Storage (hot) without private endpoint
    {
      name                      = "pwd9000v2sa001"
      account_kind              = "StorageV2"
      account_tier              = "Standard"
      account_replication_type  = "LRS"
      enable_https_traffic_only = true
      access_tier               = "Hot"
      is_hns_enabled            = false
      requires_private_endpoint = false
    },
    #V2 Storage (cool) without private endpoint
    {
      name                      = "pwd9000v2sa002"
      account_kind              = "StorageV2"
      account_tier              = "Standard"
      account_replication_type  = "LRS"
      enable_https_traffic_only = true
      access_tier               = "Cool"
      is_hns_enabled            = false
      requires_private_endpoint = false
    },
    #ADLS2 Storage with private endpoint enabled
    {
      name                      = "pwd9000adls2sa001"
      account_kind              = "BlockBlobStorage"
      account_tier              = "Premium"
      account_replication_type  = "ZRS"
      enable_https_traffic_only = false
      access_tier               = "Hot"
      is_hns_enabled            = true
      requires_private_endpoint = true
    },
    #ADLS2 Storage without private endpoint
    {
      name                      = "pwd9000adls2sa002"
      account_kind              = "BlockBlobStorage"
      account_tier              = "Premium"
      account_replication_type  = "ZRS"
      enable_https_traffic_only = false
      access_tier               = "Hot"
      is_hns_enabled            = true
      requires_private_endpoint = false
    }
  ]
}
```

We can then create all four storage accounts with the following resource config:

```hcl
## storage resources ##

resource "azurerm_resource_group" "RG" {
  name     = "example-resources"
  location = "uksouth"
}

resource "azurerm_storage_account" "SAS" {
  for_each = { for n in var.storage_config : n.name => n }

  #Implicit dependency from previous resource
  resource_group_name = azurerm_resource_group.RG.name
  location            = azurerm_resource_group.RG.location

  #values from variable storage_config objects
  name                      = each.value.name
  account_kind              = each.value.account_kind
  account_tier              = each.value.account_tier
  account_replication_type  = each.value.account_replication_type
  access_tier               = each.value.access_tier
  enable_https_traffic_only = each.value.enable_https_traffic_only
  is_hns_enabled            = each.value.is_hns_enabled
}
```

In the following resource block we can now configure private endpoints, but we will only do so for storage accounts that have an object `"key"` of `"requires_private_endpoint"` set to `"true"` like in the following resource config:

```hcl
## private endpoint resources ##

resource "azurerm_private_endpoint" "SASPE" {
  for_each            = toset([for pe in var.storage_config : pe.name if pe.requires_private_endpoint == true])
  name                = "${each.value}-pe"
  location            = azurerm_resource_group.RG.location
  resource_group_name = azurerm_resource_group.RG.name
  subnet_id           = data.azurerm_subnet.data_subnet.id

  private_service_connection {
    name                           = "${each.value}-pe-sc"
    private_connection_resource_id = azurerm_storage_account.SAS[each.value].id
    is_manual_connection           = false
    subresource_names              = ["dfs"]
  }
}
```

If you take a closer look at the `for_each` in the `azurerm_private_endpoint` resource we are using the filter there as follow:

`for_each = toset([for pe in var.storage_config : pe.name if pe.requires_private_endpoint == true])`

This `for` loop will filter and return a set of storage account names that we can use to loop the resource creation of the private endpoints for the selected storage accounts. The storage account name values will be represented by `each.value` that matches the filter: `requires_private_endpoint == true`.

So in the example above, all four storage accounts will be created.  
But only one storage account was configured to have private endpoints enabled, namely storage account: `pwd9000adls2sa001`

---

# Module 4: Terraform Pro Tips - Selective configuration with 'lookup()'

## Overview

This tutorial uses examples from the following GitHub project: [Azure Terraform Deployments](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments).

In todays tutorial we will take a look at an interesting Terraform function called [lookup()](https://www.terraform.io/language/functions/lookup).

The `lookup()` function can be used to lookup a particular value inside of a `map`, given its `key` and if the given key does not exist, the given `default` value is returned instead:

```hcl
lookup(map, key, default)
```

### Example

```sh
$ lookup({a="hello", b="world"}, "a", "what?")
"hello"

$ lookup({a="hello", b="world"}, "b", "what?")
"world"

$ lookup({a="hello", b="world"}, "c", "what?")
"what?"
```

So how can this be useful in Infrastructure as Code (IaC)?

It allows us to be more creative and granular with Terraform configurations by allowing us to create multiple configurations for different scenarios and be able to select what scenario or configuration we want to deploy. Let's take a look at a real world example of this.

## Real world example

The example code used in the following section can also be found here: [05_lookup_demo](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/05_lookup_demo).

Say for example we have to create Azure cloud resources for multiple sites of our organisation. In the following example we will use **Site A** in _UK South_ and **Site B** in _UK West_ as two separate sites for our Org.

We start off by creating a list of sites in a variable for **siteA** and **siteB**:

```hcl
## variables.tf ##

variable "site_names" {
  type        = list(string)
  default     = ["siteA", "siteB"]
  description = "Provide a list of all Contoso site names - Will be mapped to local var 'site_configs'"
}
```

Next we create a `locals` variable called `site_configs`, a `map` configuration containing child `maps` for each of the sites we want to set certain criteria against:

```hcl
## local.tf ##

locals {
  site_configs = {
    siteA = {
      resource_group_name = "Demo-Inf-SiteA-RG"
      location            = "UKSouth"
      allowed_ips         = ["8.8.8.8", "8.8.8.9"]
    },
    siteB = {
      resource_group_name = "Demo-Inf-SiteB-RG"
      location            = "UKWest"
      allowed_ips         = ["7.7.7.7", "7.7.7.8"]
    }
  }
}
```

So for our first set of resources we will deploy azure resource groups for each of our sites:

```hcl
## storage_resources.tf ##

resource "azurerm_resource_group" "RGS" {
  for_each = toset(var.site_names)
  name     = lookup(local.site_configs[each.value], "resource_group_name", null)
  location = lookup(local.site_configs[each.value], "location", null)
}
```

Notice that we are using a `for_each` loop using the list we created earlier with our site names, **siteA** and **siteB**. The `lookup()` function is then used to lookup the corresponding `key` for each site config inside of our `site_configs` locals variable map, that corresponds to **siteA** and **siteB**.

As you can see each Azure resource group was created for each site in the locations we defined in our `local` variable for _UK South_ and _UK West_

Next we will create a few storage accounts for each of our sites. We have a variable called `storage_config` which is a list of objects where each object represents a storage account configuration. But notice that one of the keys of each storage config/object has a `key` called `site_name`.

```hcl
## config-dev.tfvars ##

storage_config = [
  #V2 Storage - SiteA
  {
    name                      = "pwd9000v2sitea"
    account_kind              = "StorageV2"
    account_tier              = "Standard"
    account_replication_type  = "LRS"
    enable_https_traffic_only = true
    access_tier               = "Hot"
    is_hns_enabled            = false
    site_name                 = "siteA"
  },
  #ADLS2 Storage - SiteA
  {
    name                      = "pwd9000dfssitea"
    account_kind              = "BlockBlobStorage"
    account_tier              = "Premium"
    account_replication_type  = "ZRS"
    enable_https_traffic_only = true
    access_tier               = "Hot"
    is_hns_enabled            = true
    site_name                 = "siteA"
  },
  #V2 Storage - SiteB
  {
    name                      = "pwd9000v2siteb"
    account_kind              = "StorageV2"
    account_tier              = "Standard"
    account_replication_type  = "LRS"
    enable_https_traffic_only = false
    access_tier               = "Hot"
    is_hns_enabled            = false
    site_name                 = "siteB"
  }
]
```

This `site_name` corresponds with the local variable maps `key` of each of the `site_configs` maps:

```hcl
## local.tf ##

locals {
  site_configs = {
    siteA = {
      resource_group_name = "Demo-Inf-SiteA-RG"
      location            = "UKSouth"
      allowed_ips         = ["8.8.8.8", "8.8.8.9"]
    },
    siteB = {
      resource_group_name = "Demo-Inf-SiteB-RG"
      location            = "UKWest"
      allowed_ips         = ["7.7.7.7", "7.7.7.8"]
    }
  }
}
```

Notice that when we are building out the storage accounts for each of the sites we can now lookup the `network_rules` to apply to each of our storage accounts that corresponds to the allowed IPs for that site using the `lookup()` function `ip_rules = lookup(local.site_configs[each.value.site_name], "allowed_ips", null)` as shown below:

```hcl
resource "azurerm_storage_account" "SAS" {
  for_each = { for n in var.storage_config : n.name => n }

  #Implicit dependency from previous resource
  resource_group_name = azurerm_resource_group.RGS[each.value.site_name].name
  location            = azurerm_resource_group.RGS[each.value.site_name].location

  #values from variable storage_config objects
  name                      = "${lower(each.value.name)}${random_integer.sa_num.result}"
  account_kind              = each.value.account_kind
  account_tier              = each.value.account_tier
  account_replication_type  = each.value.account_replication_type
  access_tier               = each.value.access_tier
  enable_https_traffic_only = each.value.enable_https_traffic_only
  is_hns_enabled            = each.value.is_hns_enabled

  #Lookup allowed ips
  network_rules {
    default_action = "Deny"
    ip_rules       = lookup(local.site_configs[each.value.site_name], "allowed_ips", null)
  }
}

resource "random_integer" "sa_num" {
  min = 0001
  max = 9999
}
```

As you can see **Site A** storage accounts are set with allowed IPs of `allowed_ips = ["8.8.8.8", "8.8.8.9"]`.

And **Site B** storage accounts are set with allowed IPs of `allowed_ips = ["7.7.7.7", "7.7.7.8"]`

As you can see the Terraform `lookup()` function can be quite useful in cases where we have multiple sites or different configs and having the ability match and correlate different configurations for different scenarios.

Code samples used in this tutorial can also be found here: [05_lookup_demo](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/05_lookup_demo).

---

# Module 5: Terraform Pro Tips - Sensitive Output

## Overview

This tutorial uses examples from the following GitHub project: [Azure Terraform Deployments](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments).

When creating terraform configurations, especially when using CI/CD tooling such as Azure DevOps or GitHub it is very easy to overlook what exactly is being output as part of a Terraform configuration plan, especially if the configuration contains sensitive data. This could lead to sensitive data and settings to be leaked.

In todays tutorial we will look at examples on how we can protect and hide sensitive data in terraform output using masking.

## Sensitive Variable Type

In the following demo [configuration](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/04_App_Acr) we will use Terraform to create the following resources in Azure:

- Resource Group
- App Service Plan
- App Insights
- App Service

**NOTE:** All the code samples used in this tutorial are updated to use the the latest version of the **AzureRM provider 3.0**.

Let's take a closer look at the App service configuration:

```hcl
## appservices.tf ##
resource "azurerm_linux_web_app" "APPSVC" {
  name                = var.appsvc_name
  location            = azurerm_resource_group.RG.location
  resource_group_name = azurerm_resource_group.RG.name
  service_plan_id     = azurerm_service_plan.ASP.id
  https_only          = true

  identity {
    type = "SystemAssigned"
  }

  site_config {
    container_registry_use_managed_identity = true
    ftps_state                              = "FtpsOnly"
    application_stack {
      docker_image     = "${var.acr_name}.azurecr.io/${var.appsvc_name}"
      docker_image_tag = "latest"
    }
    vnet_route_all_enabled = var.vnet_route_all_enabled
  }

  app_settings = var.appsvc_settings
}
```

Notice the setting called `app_settings = var.appsvc_settings`. The variable for this setting is defined as a map:

```hcl
## variables.tf ##
variable "appsvc_settings" {
  type        = map(any)
  description = "Specifies the app service settings to be created."
  default     = null
}
```

If we pass the following variable into the terraform config:

```hcl
appsvc_settings = {
  APPINSIGHTS_INSTRUMENTATIONKEY = "!!sensitive_Key!!"
  sensitive_key1                 = "P@ssw0rd01"
  sensitive_key2                 = "P@ssw0rd02"
}
```

Notice that when the terraform plan is being run, terraform will actually output the variable into the terraform plan and log to the CI/CD tooling as output.

This can be an issue because the data will be leaked to anyone who has access to the CI/CD logs/output. Especially dangerous if the repository or project is public.

So what we can do to mask the setting from output is to mark the variable as [sensitive](https://www.terraform.io/language/values/variables#suppressing-values-in-cli-output). We can do that by adding `sensitive = true` to the variable:

```hcl
## variables.tf ##
variable "appsvc_settings" {
  type        = map(any)
  description = "Specifies the app service settings to be created."
  default     = null
  sensitive   = true
}
```

Notice now that when the terraform plan is being run, terraform will mask the output of the variable
## Sensitive Output Type

Similarly to variables, outputs can also be marked as [sensitive](https://www.terraform.io/language/values/outputs#sensitive-suppressing-values-in-cli-output). For example say we want to create a sensitive output we can mark the `output` as `sensitive = true` as shown the the below example:

```hcl
## appservices.tf ##
resource "azurerm_application_insights" "INSIGHTS" {
  name                = var.app_insights_name
  location            = azurerm_resource_group.RG.location
  resource_group_name = azurerm_resource_group.RG.name
  application_type    = "web"
  workspace_id        = var.workspace_id != null ? var.workspace_id : null
}

output "insights_key" {
    value = azurerm_application_insights.INSIGHTS.instrumentation_key
    sensitive = true
}
```

## Sensitive Function

Another way to mark output as sensitive is by using the `sensitive()` [function](https://www.terraform.io/language/functions/sensitive). in the demo [configuration](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/04_App_Acr) let's change the way we send `app_settings` to the app service configuration by creating a dynamic `locals` config instead of a variable:

```hcl
## local.tf ##
locals {
  #Appsvc Settings
  app_settings = {
    default_settings = {
      APPINSIGHTS_INSTRUMENTATIONKEY = "${azurerm_application_insights.INSIGHTS.instrumentation_key}"
      DOKCER_REGISTRY_SERVER_URL     = "${var.acr_name}.azurecr.io"
    },
    linux_app_settings = {
      APPINSIGHTS_INSTRUMENTATIONKEY = "${azurerm_application_insights.INSIGHTS.instrumentation_key}"
      DOKCER_REGISTRY_SERVER_URL     = "${var.acr_name}.azurecr.io"
      WEBSITE_PULL_IMAGE_OVER_VNET   = "true"
      LINUX_SENSITIVE_VALUE          = "!!sensitive_value!!"
    }
  }

}
```

As you can see the locals configuration has two configurations, one called `default_settings` and another called `linux_app_settings`, we can send the relevant config by using the terraform `lookup()` function as shown below `app_settings = lookup(local.app_settings, "linux_app_settings", null)`:

```hcl
## appservices.tf ##
resource "azurerm_linux_web_app" "APPSVC" {
  name                = var.appsvc_name
  location            = azurerm_resource_group.RG.location
  resource_group_name = azurerm_resource_group.RG.name
  service_plan_id     = azurerm_service_plan.ASP.id
  https_only          = true

  identity {
    type = "SystemAssigned"
  }

  site_config {
    container_registry_use_managed_identity = true
    ftps_state                              = "FtpsOnly"
    application_stack {
      docker_image     = "${var.acr_name}.azurecr.io/${var.appsvc_name}"
      docker_image_tag = "latest"
    }
    vnet_route_all_enabled = var.vnet_route_all_enabled
  }

  app_settings = lookup(local.app_settings, "linux_app_settings", null)
}
```

Since our app insights instrumentation key output is already marked as a sensitive output, it is all good and well for that value to be hidden from the output.

But what about if we want the entire `app_settings` config block to be hidden?  
This is where the `sensitive()` function comes in, as you can see by just wrapping the relevant locals variables in the `sensitive()` function will instruct terraform to hide the entire block from output:

```hcl
## local.tf ##
locals {
  #Appsvc Settings
  app_settings = {
    default_settings = sensitive({
      APPINSIGHTS_INSTRUMENTATIONKEY = "${azurerm_application_insights.INSIGHTS.instrumentation_key}"
      DOKCER_REGISTRY_SERVER_URL     = "${var.acr_name}.azurecr.io"
    }),
    linux_app_settings = sensitive({
      APPINSIGHTS_INSTRUMENTATIONKEY = "${azurerm_application_insights.INSIGHTS.instrumentation_key}"
      DOKCER_REGISTRY_SERVER_URL     = "${var.acr_name}.azurecr.io"
      WEBSITE_PULL_IMAGE_OVER_VNET   = "true"
      LINUX_SENSITIVE_VALUE          = "!!sensitive_value!!"
    })
  }

}
```

You can also find the code samples used in this blog post on my [GitHub](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/tree/master/04_App_Acr) page. :heart:

---

# Module 6: Terraform Pro Tips - Fun with Functions

## Overview

In todays tutorial we will take a look at [Terraform functions](https://developer.hashicorp.com/terraform/language/functions) and how we can use them in a few real world examples, and boy are there many functions to get creative and have fun with.

But what are they?  
When writing **Infrastructure as Code**, you may come across certain complexities or maybe you want to improve or simplify your code by using **Terraform functions**. You can even use functions to guardrail and safeguard your code from platform related limitations. (For example character limitations or case sensitivity when building certain resources in a cloud provider like **Azure**).

Functions are expressions to transform and combine values in order to manipulate these values to be used in other ways. Functions can also be nested within each other.

Most **Terraform functions** follow a common syntax, for example:

```hcl
<FUNCTION NAME>(<ARGUMENT 1>, <ARGUMENT 2>)
```

## Example

**NOTE:** You can use `terraform console` in a command prompt to run any of the function examples shown later, or to test your own function logic.

Say for example you want to provision an **Azure storage account** using **Terraform**. As you may know, storage account names in **Azure** have certain [name rules and character limitations](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules/?wt.mc_id=DT-MVP-5004771).  
The **length** of a storage account name must be between **3-24** characters, and can only be **lowercase letters and numbers**.

Take this example of provisioning a storage account in **Azure**:

```hcl
variable "storage_account_name" {
  type        = string
  description = "Specifies Storage account name"
  default     = "MySuperCoolStorageAccountName9000"
}

resource "azurerm_storage_account" "example" {
  name                     = var.storage_account_name
  resource_group_name      = "MyRgName9000"
  location                 = "uksouth"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

As you can see from the above example the storage account name provided by the **default** value in the variable **'storage_account_name'** is: `'MySuperCoolStorageAccountName9000'`

Because of the [provider limitations](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules/?wt.mc_id=DT-MVP-5004771), if the **default** value is used in the deployment, this resource creation would fail.

So how can we safeguard that the default value, or any value that is provided will always work?  
You guessed it, we can use **Terraform functions**.

So we will use two functions namely, **['substr'](https://developer.hashicorp.com/terraform/language/functions/substr)** and **['lower'](https://developer.hashicorp.com/terraform/language/functions/substr)**:

Lets look at each function:

- `substr` extracts a substring from a given string by offset and (maximum) length. **_Usage:_** `substr(string, offset, length)`
- `lower` converts all cased letters in the given string to lowercase. **_Usage:_** `lower(string)`

So lets test this using terraform console:

```hcl
$ substr("MySuperCoolStorageAccountName9000", 0, 24)
"MySuperCoolStorageAccoun"
```

The result **'MySuperCoolStorageAccoun'** has now been truncated to only **24 characters**, but this would still fail because there are still **uppercase characters** present. Let's nest this inside the **'lower'** function:

```hcl
$ lower(substr("MySuperCoolStorageAccountName9000", 0, 24))
"mysupercoolstorageaccoun"
```

This is much better! The storage account can now be provisioned by simply amending our original terraform code as follow:

```hcl
variable "storage_account_name" {
  type        = string
  description = "Specifies Storage account name"
  default     = "MySuperCoolStorageAccountName9000"
}

resource "azurerm_storage_account" "example" {
  name                     = lower(substr(var.storage_account_name, 0, 24))
  resource_group_name      = "MyRgName9000"
  location                 = "uksouth"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

But what if we want to improve this even more, by making the value always work, but always be **unique** as well?  
Maybe we can shorten the name a bit more using **'subsr'** and then add a unique random string?  
As you thought, yes we can!!!

Let's look at another special function called **['uuid'](https://developer.hashicorp.com/terraform/language/functions/uuid)**:

- `uuid` generates a unique identifier string. **_Usage:_** `uuid()`

So lets test this using terraform console again:

```hcl
$ uuid()
"908b8d83-f33e-aa2e-7318-8232077dfe10"
```

First, let's shorten our storage account name down to **18 characters**:

```hcl
$ lower(substr("MySuperCoolStorageAccountName9000", 0, 18))
"mysupercoolstorage"
```

Now that our storage account name is only **18 characters** long, we are left with **6 characters** to play around with, that we can generate a random identifier string that would act as a **suffix**, using the **'uuid'** function with the **'substr'** to get the following result:

```hcl
$ substr(uuid(), 0, 6)
"e807be"
```

You may be wondering?... How can we combine the function: `'lower(substr("MySuperCoolStorageAccountName9000", 0, 18))'` with the function that creates the unique suffix: `'substr(uuid(), 0, 6)'`?

You guessed it, there is a function we can use!!!

Let's look at the function called **['join'](https://developer.hashicorp.com/terraform/language/functions/join)**:

- `join` produces a string by concatenating together all elements of a given **list** of strings with the given delimiter. **_Usage:_** `join(separator, list)`

So as a basic example join can combine two strings in the following way:

```hcl
$ join("", ["StringA", "StringB"])
"StringAStringB"
```

Let's apply **'join'** to our storage account name function and unique suffix function:

```hcl
$ join("", [lower(substr("MySuperCoolStorageAccountName9000", 0, 18)), substr(uuid(), 0, 6)])
"mysupercoolstoraged29fc5"
```

Voila! We now have a randomly generated storage account name, that will always be unique and not be limited to character and case limitations for our storage account/s.

Let's run this function a few times to see the results:

```hcl
$ join("", [lower(substr("MySuperCoolStorageAccountName9000", 0, 18)), substr(uuid(), 0, 6)])
"mysupercoolstoraged29fc5"

$ join("", [lower(substr("MySuperCoolStorageAccountName9000", 0, 18)), substr(uuid(), 0, 6)])
"mysupercoolstorage3b39e2"

$ join("", [lower(substr("MySuperCoolStorageAccountName9000", 0, 18)), substr(uuid(), 0, 6)])
"mysupercoolstoragefe9d25"

$ join("", [lower(substr("MySuperCoolStorageAccountName9000", 0, 18)), substr(uuid(), 0, 6)])
"mysupercoolstorage716b99"
```

Lastly, let's apply this to our original code:

```hcl
variable "storage_account_name" {
  type        = string
  description = "Specifies Storage account name"
  default     = "MySuperCoolStorageAccountName9000"
}

resource "azurerm_storage_account" "example" {
  name                     = join("", [lower(substr(var.storage_account_name, 0, 18)), substr(uuid(), 0, 6)])
  resource_group_name      = "MyRgName9000"
  location                 = "uksouth"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

## Bonus example

If you are familiar with provisioning resources in the cloud on **Azure** you'll know that each resource has a resource ID.  
Here is a fun little function that I have used in the past to get the last element of any resource ID (usually the name of the resource), without fail:

```hcl
#Basic Example
$ element(split("/", "/x/y/z"), length(split("/", "/x/y/z"))-1)
"z"

#Resource Group name based of resource ID
$ element(split("/", "/subscriptions/829efd7e-aa80-4c0d-9c1c-7aa2557f8e07/resourceGroups/MSDO-Lab-ADO"), length(split("/", "/subscriptions/829efd7e-aa80-4c0d-9c1c-7aa2557f8e07/resourceGroups/MSDO-Lab-ADO"))-1)
"MSDO-Lab-ADO"

#VNET name based of resource ID
$ element(split("/", "/subscriptions/829efd7e-aa80-4c0d-9c1c-7aa2557f8e07/resourceGroups/Pwd9000-EB-Network/providers/Microsoft.Network/virtualNetworks/UKS-EB-VNET"), length(split("/", "/subscriptions/829efd7e-aa80-4c0d-9c1c-7aa2557f8e07/resourceGroups/Pwd9000-EB-Network/providers/Microsoft.Network/virtualNetworks/UKS-EB-VNET"))-1)
"UKS-EB-VNET"

#Key Vault name based on Resource ID
$ element(split("/", "/subscriptions/829efd7e-aa80-4c0d-9c1c-7aa2557f8e07/resourceGroups/Pwd9000-EB-Core/providers/Microsoft.KeyVault/vaults/pwd9000-core-kv"), length(split("/", "/subscriptions/829efd7e-aa80-4c0d-9c1c-7aa2557f8e07/resourceGroups/Pwd9000-EB-Core/providers/Microsoft.KeyVault/vaults/pwd9000-core-kv"))-1)
"pwd9000-core-kv"
```

Take a closer look at the functions in use above and how they are combined and nested together.

- [`element`](https://developer.hashicorp.com/terraform/language/functions/element) retrieves a single element from a list. **_Usage:_** `element(list, index)`
- [`split`](https://developer.hashicorp.com/terraform/language/functions/split) produces a list by dividing a given string at all occurrences of a given separator. **_Usage:_** `split(separator, string)`
- [`length`](https://developer.hashicorp.com/terraform/language/functions/length) determines the length of a given list, map, or string. **_Usage:_** `length(["a", "b"])`

**NOTES on 'element':**

```hcl
$ element(["a", "b", "c"], 1)
"b"
```

If the given index is greater than the length of the list then the index is "wrapped around" by taking the index modulo the length of the list:

```hcl
$ element(["a", "b", "c"], 3)
"a"
```

To get the last element from the list use [**'length'**](https://developer.hashicorp.com/terraform/language/functions/length) to find the size of the list (minus 1 as the list is zero-based) and then pick the last element:

```hcl
$ element(["a", "b", "c"], length(["a", "b", "c"])-1)
"c"
```

## Conclusion

There are so many more cool **Terraform functions** out there to make your code even better and more robust!  
Go check out the [official documentation](https://developer.hashicorp.com/terraform/language/functions) for more details.

---

# Module 7: Terraform Pro Tips - Understanding the Lifecycle Block

## Overview

Terraform offers a range of capabilities to handle infrastructure changes in an elegant and controlled manner. One such capability is the **lifecycle** configuration block.

The [lifecycle block](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle) provides several meta-arguments to manage how Terraform creates, updates, checks and deletes resources. In this post, we will dive into Terraform's **lifecycle** block and demonstrate its usage with a few examples with **Microsoft Azure resources**.

## Understanding the Lifecycle Block

**Note**: This post was written using **_Terraform (v1.4.x)_**

The **lifecycle** block in Terraform provides control over how a resource is managed. It's a configuration block that is nested within a [resource block](https://developer.hashicorp.com/terraform/language/resources/behavior) and supports four **meta-arguments**:

```hcl
resource "provider_resource" "block" {
  // ... some resource configuration ...

  lifecycle {
    argument = value
  }
}
```

The **lifecycle** block also has an option to configure **custom condition checks** using [preconditions and postconditions](https://developer.hashicorp.com/terraform/language/v1.4.x/expressions/custom-conditions#preconditions-and-postconditions):

```hcl
resource "provider_resource" "block" {
  // ... some resource configuration ...

  lifecycle {
    precondition {
      condition = expression
      error_message = "error(string)"
    }

    postcondition {
      condition = expression
      error_message = "error(string)"
    }
  }
}
```

We will take a closer look at **preconditions and postconditions** a bit later, but let's first look at a few examples using **meta-arguments**:

- create_before_destroy
- prevent_destroy
- ignore_changes
- replace_triggered_by

### 1. Create Before Destroy

**Argument Type**: _Boolean_

In some scenarios, destroying a resource before creating a new one can lead to downtime. To circumvent this, we can set `create_before_destroy` to `true`. This can be particularly useful when working with **Azure Virtual Machines** or **App Services**, where you'd want to minimize downtime.

Here is an example with an Azure Virtual Machine:

```hcl
resource "azurerm_virtual_machine" "example" {
  // ... other configuration ...

  lifecycle {
    create_before_destroy = true
  }
}
```

In this scenario, when an update is required that can't be performed in place, Terraform will first create the replacement VM and then destroy the old one, reducing potential downtime.

### 2. Prevent Destroy

**Argument Type**: _Boolean_

Setting `prevent_destroy` to `true` is a protective measure to prevent **accidental deletion** of **critical resources**. If you attempt to destroy such a resource, Terraform will return an error and stop the operation. This can be useful when working with **Azure SQL Databases**, **Storage Accounts**, or any resource that holds important data.

Here is an example with an Azure SQL Database:

```hcl
resource "azurerm_sql_database" "example" {
  // ... other configuration ...

  lifecycle {
    prevent_destroy = true
  }
}
```

With this configuration, Terraform will prevent the SQL database from being accidentally destroyed.

### 3. Ignore Changes

**Argument Type**: _list of attribute names_

The `ignore_changes` argument is useful when you want to manage certain resource attributes outside of Terraform, or when you want to avoid spurious diffs.

Here is an example with an Azure App Service:

```hcl
resource "azurerm_app_service" "example" {
  // ... other configuration ...

  lifecycle {
    ignore_changes = [
      app_settings,  // Ignore changes to app_settings attribute
    ]
  }
}
```

Say a different team manages an **App Services** `app_settings` for example, you may be provisioning that **App Service**, but the configuration is left up to someone else, or maybe even a different automation all together is taking care of the `app_settings` configuration, and you do not want Terraform to revert, interfere or potentially remove those settings.

In this case, any changes to the `app_settings` of the **App Service** will be ignored by Terraform.

**Tip**: You can also use a special value `all` that will ignore all settings once a resource is provisioned.

```hcl
resource "azurerm_app_service" "example" {
  // ... other configuration ...

  lifecycle {
    ignore_changes = all
  }
}
```

This will provision the resource any any subsequent configuration outside of Terraform will be ignored by Terraform.

### 4. Replace Triggered By

**Argument Type**: _list of resource or attribute references_

The `replace_triggered_by` argument allows you to replace a resource when another resource changes. You can only reference **managed resources** in `replace_triggered_by` expressions. Supply a list of expressions referencing managed resources, instances, or instance attributes.

```hcl
resource "azurerm_sql_database" "example" {
  // ... other configuration ...
}

resource "azurerm_app_service" "example" {
  // ... other configuration ...

  lifecycle {
    replace_triggered_by = [
      azurerm_sql_database.example.id, //Replace `azurerm_app_service` each time `azurerm_sql_database` id changes
    ]
  }
}
```

`replace_triggered_by` allows only resource addresses because the decision is based on the planned actions for all of the given resources, meaning that variables, data sources and modules are not supported.

Plain values such as **local values** or **input variables** do not have planned actions of their own, but you can treat them with a resource-like lifecycle by using them with the **terraform_data** resource type.

## Custom Condition Checks

You can add `precondition` and `postcondition` blocks with a lifecycle block to specify assumptions and guarantees about how resources and data sources operate. The following examples creates a precondition that checks whether the AMI is properly configured.

```hcl
data "azurerm_mssql_server" "example" {
  // ... other configuration ...
}

resource "azurerm_mssql_database" "test" {
  // ... other configuration ...

  lifecycle {
    precondition {
      condition     = data.azurerm_mssql_server.example.version == "12.0"
      error_message = "MSSQL server version incorrect (Needs to be version 12.0)."
    }

    postcondition {
      condition     = self.transparent_data_encryption_enabled == true
      error_message = "The Database must have TDE enabled."
    }
  }
}
```

**NOTE:** The [self object](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions#self-object) above in the `postcondition` block refers to attributes of the instance under evaluation (e.g. the MSSQL database).

You can implement a validation check as either a `postcondition` of the resource producing the data, or as a `precondition` of a resource or output value using the data. To decide which is most appropriate, consider whether the check is representing an **assumption** or a **guarantee**.

In our example above:

- **Assumption:** Validate using `preconditions` that the database is being created, is on a **MSSQL server** that is version `12.0`.
- **Guarantee:** Validating using `postcondition` that the **MSSQL database** being created **(SELF)**, has `transparent_data_encryption_enabled` set to `true`.

## Conclusion

Terraform's **lifecycle block** provides a powerful way to control and manage your resources. Whether it's preventing accidental destruction of critical resources, managing zero-downtime updates, or ignoring changes to certain attributes, the lifecycle block offers you the flexibility you need. As always, be sure to test these configurations in a non-production environment before rolling out to production to ensure they work as expected. Happy Terraforming!

---

# Module 8: Terraform Pro Tips - Variable Validation

## Overview

Today we will discuss an interesting feature of Terraform by taking a closer look at **[variable validation rules](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules)** inside **terraform variables**.

```hcl
variable "fruit" {
  type        = string
  description = "What fruit to pick?"
  default     = "apple"

  validation {
    condition     = can(regex("^(lemon|apple|mango|banana|cherry)$", var.fruit))
    error_message = "Invalid fruit selected, only allowed fruits are: 'lemon', 'apple', 'mango', 'banana', 'cherry'. Default 'apple'"
  }
}
```

The above example **validates** the `fruit` variable against a set of predefined fruits. An input that matches the regex pattern passes the validation. If any invalid fruit is passed, the error message quickly informs the users about the allowed values.

**Terraform's** consistent updates have always aimed to make **infrastructure-as-code (IaC)** development more efficient, secure, and error-free. One of these updates, the addition of a **validation capability** for **input variables**, has significantly improved this IaC tool's robustness.

This blog post will demystify **Terraform validation**, discussing its utility and providing practical examples using the Azure platform.

## Understanding Terraform Variable Validation

Terraform's variable validation helps ensure the values assigned to variables meet specific criteria defined in advance. Introduced in version 0.13, this feature allows you to provide custom `validation` rules for your input variables. Validation is essentially added as a validation block within the variable declaration, and it uses a condition to determine whether a value is acceptable.

There are two key components in a variable validation block - the `condition` that expresses the criteria a value must meet, and an error `message` to alert the user if the value doesn't pass the check. The condition is a boolean expression. When the condition is false, Terraform produces an error displaying the custom message.

### The Importance of Terraform Variable Validation

Terraform variable validation brings four key advantages to IaC developers:

1. **Greater Consistency:** With variable validation, you ensure that acceptable input values adhere to specific standards. This fosters uniformity across each infrastructure deployment.

2. **Error Reduction:** By flagging erroneous or unacceptable variable inputs, the tool drastically reduces errors that might arise during the 'terraform apply' phase.

3. **Enhanced Security:** By validating inputs adequately, developers can avoid potential security vulnerabilities resulting from insecure or dangerous variable input.

4. **Improved Developer Experience:** Validation improves user experience by quickly catching errors and guiding users towards acceptable values, accelerating the development process.

## Variable Validation In Action: Azure Platform Examples

Let's dive into some practical use cases of variable validation with Azure infrastructure deployments.

For instance, when creating a virtual machine, if a user wants to choose their known operating system, we define a terraform variable and validate whether the input falls within our accepted values:

```hcl
variable "os_type" {
  description = "Operating System to use for the VM"

  validation {
    condition     = contains(["Windows", "Linux"], var.os_type)
    error_message = "The os_type must be either 'Windows' or 'Linux'."
  }
}
```

In the example above, we utilise the `contains` function to validate whether the `os_type` falls within `[Windows, Linux]`. If a user enters an OS type out of this list, Terraform will display the error message, preventing potential confusion or failure in deployments.

Similarly, you might want to implement a naming convention for a **resource group** in Azure. For example, the name must always start with a **'rg-'** prefix. Terraform validation can assist here:

```hcl
variable "resource_group_name" {
  description = "Name of the resource group"

  validation {
    condition     = can(regex("^rg-", var.resource_group_name))
    error_message = "The resource group name must start with 'rg-'."
  }
}
```

Our `condition` uses a regular expression to test whether the given resource group name starts with **'rg-'**. If not, Terraform prompts the user with the specified `error_message`.

Let's take a look at one more example. The following example demonstrates another use case for Terraform's variable validation, when working with private DNS records and validating the record type. This code sample ensures that only valid DNS record types are specified when deploying resources:

```hcl
variable "private_dns_record_type" {
  type        = string
  description = "value of the private dns record type, only allowed options are 'A', 'AAAA', 'CNAME', 'MX', 'PTR', 'SRV', 'TXT'"
  default     = "A"

  validation {
    condition     = can(regex("^(A|AAAA|CNAME|MX|PTR|SRV|TXT)$", var.private_dns_record_type))
    error_message = "Invalid value for private_dns_record_type, only allowed options are: 'A', 'AAAA', 'CNAME', 'MX', 'PTR', 'SRV', 'TXT'"
  }
}
```

The above `variable` block is a stricter example where it validates the `private_dns_record_type` against a set of predefined DNS record types. An input that matches the regex pattern passes the validation. If any invalid DNS record type is passed, the error message quickly informs the users about the allowed values.

This variable validation can then be used in the resource block as follows to create a corresponding private DNS record for that type for example:

```hcl
#A record example
resource "azurerm_private_dns_a_record" "private_dns_a_record" {
  count               = var.private_dns_record_type == "A" ? 1 : 0
  name                = lower(var.private_dns_record_name)
  resource_group_name = var.resource_group_name
  ttl                 = var.private_dns_record_ttl
  zone_name           = var.private_dns_zone_name
  records             = var.private_dns_record_value
  tags                = var.tags
}
```

or

```hcl
#CNAME record example
resource "azurerm_private_dns_cname_record" "private_dns_cname_record" {
  count               = var.private_dns_record_type == "CNAME" ? 1 : 0
  name                = lower(var.private_dns_record_name)
  resource_group_name = var.resource_group_name
  ttl                 = var.private_dns_record_ttl
  zone_name           = var.private_dns_zone_name
  record              = var.private_dns_record_value
  tags                = var.tags
}
```

The resource blocks above creates an Azure Private DNS **'A'** or **'CNAME'** record, but only if the `private_dns_record_type` variable is **"A"** or **"CNAME"** for example. If the DNS record type doesn't fit that criteria of the validation, count equals 0, and Terraform won't create this record type. This approach ensures the correct DNS record gets created based on the initially validated variable and avoids unnecessary or inaccurate resources.

By integrating variable validation in your Terraform configurations like this, you enhance the **accuracy and predictability** of your deployments. With this advanced feature, you'll be able to manage resources more effectively on platforms like Azure.

## Conclusion

In summary, **variable validation** enhances Terraform's robustness as an IaC tool by **reducing errors**, **improving security**, and fostering a **consistent**, developer-friendly experience. Our examples on the Azure platform demonstrate how you can avoid complications by ensuring variable inputs align with your requirements during the infrastructure setup process. As you continue to build with Terraform, consider this feature to help bolster your code's reliability and your infrastructure's integrity.

---

# Module 9: Terraform Pro Tips - Understanding Dynamic Blocks

## Overview

In the complex world of infrastructure management, simplicity and reusability are key to maintaining sanity. This is where Terraform shines, offering a suite of advanced syntax and features to streamline the way you define and deploy resources in the cloud.

Today, we focus on **dynamic blocks** - a powerful feature that introduces greater flexibility and dynamism into your Terraform configurations. I'll walk you through various scenarios and show you how dynamic blocks can make a substantial difference in managing Azure resources and we will also look at a few real world uses cases and scenarios with a few examples using the **AzureRM provider**.

## Understanding Dynamic Blocks

Dynamic blocks let you generate nested block configurations within resources or data structures dynamically. They are particularly useful when the configuration of a resource involves repeated nested blocks whose number and content may vary based on input variables or external data.

In Terraform, a dynamic block consists of two parts: the `dynamic` keyword followed by the name of the nested block, and a `content` block that defines the structure of the dynamic block. Inside this `content` block, you reference iterator objects to assign values:

```hcl
resource "provider_resource" "example" {

  argument = "value"
  # ... other arguments ...

  dynamic "argument_block_name" {
    for_each = var.collection # or expression
    content {
      # Block content
    }

  }
}
```

## Scenario 1: Azure Network Security Group with Variable Rules

Imagine you need to create a network security group in Azure with a varying number of security rules that can change over time. Instead of hardcoding each rule, you can use a `dynamic block` to generate these rules from a variable:

```hcl
variable "security_rules" {
  description = "A list of security rules"
  type = list(object({
    name                     = string
    priority                 = number
    direction                = string
    access                   = string
    protocol                 = string
    source_port_range        = string
    destination_port_range   = string
    source_address_prefix    = string
    destination_address_prefix = string
  }))
  default = [
    {
      name                     = "allow-ssh"
      priority                 = 100
      direction                = "Inbound"
      access                   = "Allow"
      protocol                 = "Tcp"
      source_port_range        = "*"
      destination_port_range   = "22"
      source_address_prefix    = "*"
      destination_address_prefix = "VirtualNetwork"
    },
    // ... more rules ...
  ]
}

resource "azurerm_network_security_group" "example" {
  name                = "example-nsg"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  dynamic "security_rule" {
    for_each = var.security_rules

    content {
      name                       = security_rule.value.name
      priority                   = security_rule.value.priority
      direction                  = security_rule.value.direction
      access                     = security_rule.value.access
      protocol                   = security_rule.value.protocol
      source_port_range          = security_rule.value.source_port_range
      destination_port_range     = security_rule.value.destination_port_range
      source_address_prefix      = security_rule.value.source_address_prefix
      destination_address_prefix = security_rule.value.destination_address_prefix
    }
  }
}
```

In this scenario, dynamic blocks iterate over the `var.security_rules` list object, creating security rules based on its content. This dynamic approach keeps your code DRY (Don't Repeat Yourself) by avoiding repetitive block definitions.

## Scenario 2: Tagging Azure Resources Dynamically

Tagging resources is critical for cost tracking, compliance, and management. However, not every resource may share the same set of tags. Using dynamic blocks can conditionally add tags based on the context.

```hcl
variable "common_tags" {
  type = map(string)
  default = {
    Environment = "Development"
    Owner       = "Infrastructure Team"
  }
}

variable "extra_tags" {
  type = map(string)
  default = {
    Project = "Phoenix"
    Tier    = "Backend"
  }
}

resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West Europe"

  dynamic "tag" {
    for_each = merge(var.common_tags, var.extra_tags)

    content {
      key   = tag.key
      value = tag.value
    }
  }
}
```

In this example, the resource group is tagged with a **merged** set of common and extra tags. Using dynamic blocks, you can easily combine these tags and apply them flexibly without having to declare each tag separately, simplifying the management of resource metadata. As you can see in the example, the `for_each` expression uses a `merge` function that combines the two maps into a single map.

## Scenario 3: Conditional DNS Zone Groups with Private Endpoint

Let's take a look at a few more advanced scenarios using conditions and expressions. In this example, we will create a private DNS zone group with a private endpoint.

```hcl
variable "private_dns_zone_group" {
  type = list(object({
    enabled              = bool
    name                 = string
    private_dns_zone_ids = list(string)
  }))
  default = [
    {
      enabled              = true
      name                 = "privatelink.vaultcore.azure.net"
      private_dns_zone_ids = [<DNS ZONE ID>]
    }
  ]
  description = "List of private dns zone groups to associate with the private endpoint."
}

resource "azurerm_private_endpoint" "private_endpoint" {

  # ... other arguments ...

  dynamic "private_dns_zone_group" {
    for_each = [for each in var.private_dns_zone_group :
      {
        name                 = each.name
        private_dns_zone_ids = each.private_dns_zone_ids
        enabled              = each.enabled
      } if each.enabled == true
    ]
    content {
      name                 = private_dns_zone_group.value.name
      private_dns_zone_ids = private_dns_zone_group.value.private_dns_zone_ids
    }
  }
}
```

In this example, the `for_each` argument is used to iterate over the `var.private_dns_zone_group` list. For each item in the list, it creates a new map with `name`, `private_dns_zone_ids`, and `enabled` keys if `enabled` is `true`.

The `content` block then uses these values to create a new `private_dns_zone_group` block for each item in the `for_each` list. The `private_dns_zone_group.value.name` and `private_dns_zone_group.value.private_dns_zone_ids` expressions refer to the current item in the `for_each` list.

This dynamic block allows us to create a flexible number of `private_dns_zone_group` blocks based on the input variable, which can be incredibly useful when dealing with complex infrastructure setups.

## Scenario 4: Conditional Azure Virtual Network Subnets

Suppose you are managing an Azure Virtual Network that needs to support multiple subnets. Each subnet has specific requirements and might only be necessary under certain conditions — driven by environment types, features toggling, or specific compliance needs.

Here's how you can use dynamic blocks with a condition to selectively create subnets:

```hcl
variable "subnets" {
  description = "A map of subnets with their properties and a creation condition"
  type = map(object({
    address_prefixes = list(string)
    create_subnet    = bool
  }))
  default = {
    subnet1 = {
      address_prefixes = ["10.0.1.0/24"]
      create_subnet    = true
    },
    subnet2 = {
      address_prefixes = ["10.0.2.0/24"]
      create_subnet    = false // This can be driven by your specific conditions
    }
    // ...other subnets...
  }
}

resource "azurerm_virtual_network" "example" {
  name                = "example-network"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  dynamic "subnet" {
    # We are using the for each to iterate only over subnets that should be created.
    for_each = {
      for s_name, s_details in var.subnets : s_name => s_details
      if s_details.create_subnet
    }

    content {
      name           = subnet.key
      address_prefix = subnet.value.address_prefixes[0]
      // It’s common for the first item of the address_prefixes to be used,
      // or integrate further logic to handle multiple prefixes.
    }
  }
}
```

In this example, the `for_each` expression has been augmented with a conditional. The iteration now only includes subnet configurations where the `create_subnet` attribute is set to `true`. As a result, despite the `var.subnets` variable containing multiple definitions, only those explicitly marked for creation are acted upon - `subnet1` in this case, while `subnet2` is ignored.

Using this pattern, you can fine-tune your Terraform configurations to respond dynamically not just to the contents of variables, but also to the logical conditions your infrastructure setup may require.

## Conclusion

**Dynamic blocks** enhanced with **conditional logic** are among Terraform's most potent features for crafting maintainable and adaptable **Infrastructure as Code** in **Azure**. Leveraging the power of dynamic blocks with conditions gives you the ability to construct intricate IaC configurations that are both powerful and elegant. By carefully combining these advanced **Terraform** features, your Azure templates will become more modular, less error-prone, and far easier to extend as your Azure landscapes evolve.

---

# Module 10: Terraform Pro Tips - Understanding Implicit and Explicit Dependencies

## Overview

When working with Terraform, it is important to understand the difference between **implicit** and **explicit** dependencies. This is important as it can help you to understand how Terraform creates the **dependency graph** and how it determines the order in which resources are created.

## What Are Dependencies in Terraform?

Dependencies in Terraform dictate the order in which resources are created, updated, or destroyed. Terraform automatically determines dependencies between your resources, ensuring that they are managed in the correct sequence.

## Implicit Dependencies

Implicit dependencies are automatically discovered by Terraform by analysing resource attributes. When one resource refers to another using interpolation syntax, Terraform recognises this as a dependency.

In other words, implicit dependencies in Terraform are created when one resource property references another resource's property or output. Terraform uses these references to automatically determine the order of resource creation.

Consider this example involving an Azure virtual network and a subnet:

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "East US"
}

resource "azurerm_virtual_network" "example_vnet" {
  name                = "example-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}

resource "azurerm_subnet" "example_subnet" {
  name                 = "example-subnet"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example_vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

In this example, the `azurerm_subnet` resource has an implicit dependency on the `azurerm_virtual_network` resource. This is because the `virtual_network_name` property of the `azurerm_subnet` resource references the `name` property of the `azurerm_virtual_network` resource. Terraform automatically recognises this and creates the dependency.

## Explicit Dependencies

Sometimes, however, the relationship between resources is not captured by direct references. In these instances, you can use the `depends_on` attribute to create an explicit dependency.

Explicit dependencies should only be defined when Terraform can't automatically infer the required order for resource creation, or when specific provisioning steps are necessary before or after a resource is deployed.

Let's illustrate an explicit dependency in a scenario where an Azure App Service depends on certain configuration settings that are applied via an Azure CLI script after the creation of an Azure Key Vault.

Here's a simple Terraform configuration demonstrating this relationship:

```hcl
resource "azurerm_resource_group" "example_rg" {
  name     = "example-resources"
  location = "West Europe"
}

resource "azurerm_key_vault" "example_kv" {
  name                        = "exampleKeyVault"
  location                    = azurerm_resource_group.example_rg.location
  resource_group_name         = azurerm_resource_group.example_rg.name
  tenant_id                   = "00000000-0000-0000-0000-000000000000"
  sku_name                    = "standard"

  soft_delete_enabled         = true
  purge_protection_enabled    = false
}

resource "null_resource" "example_kv_settings" {
  # Dummy example of an Azure CLI script command that sets configuration in the Key Vault
  provisioner "local-exec" {
    command = "echo Configuring Key Vault Settings"
  }

  # Explicitly state that the Key Vault settings should be applied after the Key Vault is created
  depends_on = [ azurerm_key_vault.example_kv ]
}

resource "azurerm_app_service" "example_app_service" {
  name                = "example-appservice"
  location            = azurerm_resource_group.example_rg.location
  resource_group_name = azurerm_resource_group.example_rg.name
  app_service_plan_id = azurerm_app_service_plan.example_asp.id

  # Explicitly state the dependency on the Key Vault settings to ensure these are set before creating the App Service
  depends_on = [ null_resource.example_kv_settings ]
}

resource "azurerm_app_service_plan" "example_asp" {
  name                = "example-asp"
  location            = azurerm_resource_group.example_rg.location
  resource_group_name = azurerm_resource_group.example_rg.name

  sku {
    tier = "Standard"
    size = "S1"
  }
}
```

In the above example:

1. `azurerm_resource_group.example_rg` creates a resource group.
2. `azurerm_key_vault.example_kv` creates the Azure Key Vault.
3. `null_resource.example_kv_settings` represents a hypothetical Azure CLI script that configures settings in the Key Vault (replaced with an echo command for simplicity). The `depends_on` ensures the Key Vault is in place before the configuration script runs.
4. `azurerm_app_service_plan.example_asp` sets up the required App Service Plan.
5. `azurerm_app_service.example_app_service` creates the App Service with a `depends_on` pointing to `null_resource.example_kv_settings`. This explicit dependency ensures that the App Service is only provisioned after the Key Vault settings have been applied by the script.

By using `depends_on`, we establish an explicit dependency chain: Resource Group -> Key Vault -> Key Vault Settings -> App Service. This ensures the resources are provisioned in the correct order, even though the dependencies aren't apparent from the resource attributes alone.

## When to Use Implicit vs. Explicit Dependencies

Implicit dependencies should be your first go-to in Terraform since they are automatically detected, and Terraform handles the ordering for you. However, there are cases when Terraform cannot discern the right order, or you have custom steps in your provisioning process which can warrant the use of explicit dependencies with the `depends_on` attribute.

To minimise potential issues:

- Rely mostly on implicit dependencies through resource attribute references.
- Only use explicit dependencies when necessary, and keep them to a minimum to avoid tightly coupled architecture.
- Always document why an explicit dependency is required to help other developers understand the rationale behind it.

## Conclusion

Grasping the concept of implicit and explicit dependencies and applying that knowledge to Azure resources with Terraform will lead to smoother deployments and a more robust infrastructure.

---

# Module 11: Terraform Pro Tips - Understanding Count and For_Each Loops

## Overview

When working with Terraform, you may need to create multiple instances of the same resource. This is where **count** and **for_each** loops come in. These loops allow you to create multiple resources with the same configuration, but with different values. This guide will explain how to use **count** and **for_each** loops in Terraform.

## Count in Terraform

The `count` parameter in Terraform allows you to create a specified number of identical resources. It is an integral part of a resource block that defines how many instances of a particular resource should be created.

Here's an example of how to use `count` in Terraform:

```hcl
resource "azurerm_resource_group" "example" {
  count    = 3
  name     = "resourceGroup-${count.index}"
  location = "East US"
  tags = {
    iteration = "Resource Group number ${count.index}"
  }
}
```

In the example above, we create three identical resource groups in the Azure region "East US" with differing names using the `count` parameter.

## Pros:

- **Simple to use:** The `count` parameter is straightforward for creating multiple instances of a resource.
- **Suitable for homogeneous resources:** When all the resources you're creating are identical except for an identifier, `count` is likely a good fit.

## Cons:

- **Lacks key-based identification:** `count` doesn’t include a way to address a resource with a unique key directly; you have to rely on an index.
- **Immutable:** If you remove an item from the middle of the `count` list, Terraform marks all subsequent resources for recreation which can be disruptive in certain scenarios. For example: Let's say you have a Terraform configuration that manages a fleet of virtual machines in Azure using the `count` parameter. Assume that you initially set the `count` parameter to 5, which provisioned five VMs:

```hcl
resource "azurerm_virtual_machine" "vm" {
  count               = 5
  name                = "vm-${count.index}"
  location            = "East US"
  resource_group_name = azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.nic[count.index].id]
  # ... (other configuration details)
}
```

In the above example. Say After some time, you decide that you no longer need the second VM (**"vm-1"**, since **"count.index"** is zero-based). To remove this VM, you might change the `count` to `4` and adjust your resource names or indexes, which might intuitively seem like the correct approach.

The problem arises here: Terraform determines the creation and destruction of resources based on their index. If you simply remove or comment out the definition for **"vm-1"**, Terraform won't know that you specifically want to destroy **"vm-1"**. It would interpret that every VM from index 1 and onward (vm-1, vm-2, vm-3, and vm-4) should be destroyed and recreated because their indices have changed.

This could have several disruptive consequences:

- **Downtime:** Recreating VMs would lead to downtime for the services running on them, which may be unacceptable in a production environment.
- **Data Loss:** If there's local data on the VMs that you haven't backed up, it would be lost when the VMs are destroyed and recreated.
- **IP Changes:** If the VMs are assigned dynamic public IPs, these IPs would change and could cause connectivity issues.
- **Costs:** Destroying and recreating resources might incur unnecessary costs in terms of the compute hours consumed.

To avoid such issues with `count`, you'd want to use `create_before_destroy` [lifecycle rules](https://dev.to/pwd9000/terraform-understanding-the-lifecycle-block-4f6e) or consider whether `for_each` is a better choice for such a scenario because it provides a way to uniquely identify resources without relying on sequence. With `for_each`, each VM would be managed individually, and you could remove a single map entry that corresponds to the unwanted VM, leading to the destruction of only that particular VM without impacting the others.

## For_Each in Terraform

The `for_each` loop in Terraform, used within the `for_each` argument, iterates over a map or a set of strings, allowing you to create resources that correspond to the given elements.

Here's an example of how to use `for_each` in Terraform:

```hcl
resource "azurerm_resource_group" "example" {
  for_each  = toset(["rg-prod", "rg-dev", "rg-test"])
  name      = each.value
  location  = "East US"
  tags = {
    Name = each.value
  }
}

# Alternatively, using a map
resource "azurerm_storage_account" "example" {
  for_each  = {
    prod = "eastus2"
    dev  = "westus"
    test = "centralus"
  }
  name                     = "storage${each.key}"
  resource_group_name      = azurerm_resource_group.example[each.key].name
  location                 = each.value
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```

In the first example using a **set** of strings, we create resource groups with specific names: **"rg-prod"**, **"rg-dev"**, and **"rg-test"**.  
In the second example using a **map**, we create storage accounts in different locations and with associations to corresponding resource groups.

## Pros:

- **Detailed declaration:** `for_each` provides greater control when creating resources that require specific attributes or configurations.
- **Key-based identification:** Resources created with `for_each` can be directly identified and accessed by their keys, making modifications more manageable.
- **Non-destructive updates:** If you remove an item from the map or set, only that specific resource will be affected.

## Cons:

- **Complexity:** `for_each` is more complex to use than `count` and requires more planning.
- **Requires a set or map:** You must provide a set or map of items to iterate over, which might not be necessary or straightforward for all situations.

## When to Use Count vs. For_each

Both constructs are powerful, but they shine in different situations. Here's a quick reference to determine which to use:

### Use Count when:

- You need to create a fixed number of similar resources.
- Resource differences can be represented by an index.

### Use For_each when:

- You're dealing with a collection of items that have unique identifiers.
- Your resources are not perfectly identical and require individual configurations.
- You plan to make future modifications that should not affect all resources.

## Conclusion

Choosing between `count` and `for_each` largely depends on the scenario at hand. The `count` parameter is excellent for simplicity and when you're dealing with homogenous resources. Meanwhile, `for_each` is perfect for a more controlled resource declaration, offering flexibility and precision especially beneficial in complex infrastructures.

---

### _Author_

Marcel Lupo - Follow me on: [GitHub](https://github.com/Pwd9000-ML) | [X/Twitter](https://x.com/pwd9000) | [LinkedIn](https://www.linkedin.com/in/marcel-l-61b0a96b/)