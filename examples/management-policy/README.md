<!-- BEGIN_TF_DOCS -->
# Default example

This deploys the module in its simplest form.

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.7.0, < 5.0.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.5.0, < 4.0.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
  resource_provider_registrations = "none"
  storage_use_azuread             = true
}
locals {
  test_regions = ["eastus", "eastus2", "westus2", "westus3"]
}
resource "random_integer" "region_index" {
  max = length(local.test_regions) - 1
  min = 0
}

# This allow use to randomize the name of resources
resource "random_string" "this" {
  length  = 6
  special = false
  upper   = false
}
# This ensures we have unique CAF compliant names for resources.
module "naming" {
  source  = "Azure/naming/azurerm"
  version = "0.4.0"
}

# This is required for resource modules
resource "azurerm_resource_group" "this" {
  location = local.test_regions[random_integer.region_index.result]
  name     = module.naming.resource_group.name_unique
}

resource "azurerm_virtual_network" "vnet" {
  location            = azurerm_resource_group.this.location
  name                = module.naming.virtual_network.name_unique
  resource_group_name = azurerm_resource_group.this.name
  address_space       = ["192.168.0.0/16"]
}

resource "azurerm_subnet" "private" {
  address_prefixes     = ["192.168.0.0/24"]
  name                 = module.naming.subnet.name_unique
  resource_group_name  = azurerm_resource_group.this.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  service_endpoints    = ["Microsoft.Storage"]
}

resource "azurerm_network_security_group" "nsg" {
  location            = azurerm_resource_group.this.location
  name                = module.naming.network_security_group.name_unique
  resource_group_name = azurerm_resource_group.this.name
}

resource "azurerm_subnet_network_security_group_association" "private" {
  network_security_group_id = azurerm_network_security_group.nsg.id
  subnet_id                 = azurerm_subnet.private.id
}

resource "azurerm_network_security_rule" "no_internet" {
  access                      = "Deny"
  direction                   = "Outbound"
  name                        = module.naming.network_security_rule.name_unique
  network_security_group_name = azurerm_network_security_group.nsg.name
  priority                    = 100
  protocol                    = "*"
  resource_group_name         = azurerm_resource_group.this.name
  destination_address_prefix  = "Internet"
  destination_port_range      = "*"
  source_address_prefix       = azurerm_subnet.private.address_prefixes[0]
  source_port_range           = "*"
}

module "public_ip" {
  source  = "lonegunmanb/public-ip/lonegunmanb"
  version = "0.1.0"
  count   = var.bypass_ip_cidr == null ? 1 : 0
}
# We need this to get the object_id of the current user
data "azurerm_client_config" "current" {}

resource "azurerm_user_assigned_identity" "example_identity" {
  location            = azurerm_resource_group.this.location
  name                = module.naming.user_assigned_identity.name_unique
  resource_group_name = azurerm_resource_group.this.name
}
# We use the role definition data source to get the id of the Contributor role
data "azurerm_role_definition" "example" {
  name = "Contributor"
}

module "this" {
  source = "../.."

  location                 = azurerm_resource_group.this.location
  name                     = module.naming.storage_account.name_unique
  resource_group_name      = azurerm_resource_group.this.name
  account_kind             = "StorageV2"
  account_replication_type = "GRS"
  account_tier             = "Standard"
  blob_properties = {
    versioning_enabled       = true
    last_access_time_enabled = true

  }
  containers = {
    blob_container0 = {
      name = "blob-container-${random_string.this.result}-0"
    }
    blob_container1 = {
      name = "blob-container-${random_string.this.result}-1"

    }

  }
  https_traffic_only_enabled = true
  managed_identities = {
    system_assigned = true
    user_assigned_resource_ids = [
      azurerm_user_assigned_identity.example_identity.id
    ]
  }
  min_tls_version = "TLS1_2"
  network_rules = {
    bypass                     = ["AzureServices"]
    default_action             = "Deny"
    ip_rules                   = [try(module.public_ip[0].public_ip, var.bypass_ip_cidr)]
    virtual_network_subnet_ids = toset([azurerm_subnet.private.id])
  }
  public_network_access_enabled = true
  queues = {
    queue0 = {
      name = "queue-${random_string.this.result}-0"

    }
    queue1 = {
      name = "queue-${random_string.this.result}-1"

      metadata = {
        key1 = "value1"
        key2 = "value2"
      }
    }
  }
  role_assignments = {
    role_assignment_1 = {
      role_definition_id_or_name       = data.azurerm_role_definition.example.name
      principal_id                     = coalesce(var.msi_id, data.azurerm_client_config.current.object_id)
      skip_service_principal_aad_check = false
    },
    role_assignment_2 = {
      role_definition_id_or_name       = "Owner"
      principal_id                     = data.azurerm_client_config.current.object_id
      skip_service_principal_aad_check = false
    },

  }
  shared_access_key_enabled = true
  shares = {
    share0 = {
      name  = "share-${random_string.this.result}-0"
      quota = 10
      signed_identifiers = [
        {
          id = "1"
          access_policy = {
            expiry_time = "2025-01-01T00:00:00Z"
            permission  = "r"
            start_time  = "2024-01-01T00:00:00Z"
          }
        }
      ]
    }
    share1 = {
      name        = "share-${random_string.this.result}-1"
      quota       = 10
      access_tier = "Hot"
      metadata = {
        key1 = "value1"
        key2 = "value2"
      }
    }
  }
  storage_management_policy_rule = {
    rule = {
      enabled = true
      name    = "rule1"
      actions = {
        base_blob = {
          #         delete_after_days_since_creation_greater_than = 30
          #         delete_after_days_since_modification_greater_than = 30
          #         tier_to_archive_after_days_since_last_access_time_greater_than = 30
          #         tier_to_archive_after_days_since_modification_greater_than = 30
          #         tier_to_cold_after_days_since_last_access_time_greater_than = 30
          #         tier_to_cold_after_days_since_modification_greater_than = 30
          #         tier_to_cool_after_days_since_creation_greater_than = 30
          #       tier_to_cool_after_days_since_modification_greater_than = 30
          auto_tier_to_hot_from_cool_enabled = true
          #         delete_after_days_since_creation_greater_than = 30
          delete_after_days_since_last_access_time_greater_than = 30
          #         delete_after_days_since_modification_greater_than = 30
          tier_to_archive_after_days_since_creation_greater_than = 30
          #         tier_to_archive_after_days_since_last_access_time_greater_than = 30
          tier_to_archive_after_days_since_last_tier_change_greater_than = 30
          #         tier_to_archive_after_days_since_modification_greater_than = 30
          tier_to_cold_after_days_since_creation_greater_than = 30
          #         tier_to_cold_after_days_since_last_access_time_greater_than = 30
          #         tier_to_cold_after_days_since_modification_greater_than = 30
          #         tier_to_cool_after_days_since_creation_greater_than = 30
          tier_to_cool_after_days_since_last_access_time_greater_than = 30
          #       tier_to_cool_after_days_since_modification_greater_than = 30
        }
        #         snapshot = {
        #           change_tier_to_archive_after_days_since_creation               = 30
        #           change_tier_to_cool_after_days_since_creation                  = 30
        #           delete_after_days_since_creation_greater_than                  = 30
        #           tier_to_archive_after_days_since_last_tier_change_greater_than = 30
        #           tier_to_cold_after_days_since_creation_greater_than            = 30
        #         }
        #         version = {
        #           change_tier_to_archive_after_days_since_creation               = 30
        #           change_tier_to_cool_after_days_since_creation                  = 30
        #           delete_after_days_since_creation                               = 30
        #           tier_to_archive_after_days_since_last_tier_change_greater_than = 30
        #           tier_to_cold_after_days_since_creation_greater_than            = 30
        #         }
      }
      filters = {
        blob_types   = ["blockBlob"]
        prefix_match = ["test"]
        match_blob_index_tag = [
          {
            name      = "tag1"
            operation = "=="
            value     = "value1"
          }
        ]
      }

    }
  }
  tables = {
    table0 = {
      name = "table${random_string.this.result}0"
      signed_identifiers = [
        {
          id = "1"
          access_policy = {
            expiry_time = "2025-01-01T00:00:00Z"
            permission  = "r"
            start_time  = "2024-01-01T00:00:00Z"
          }
        }
      ]
    }
    table1 = {
      name = "table${random_string.this.result}1"

      signed_identifiers = [
        {
          id = "1"
          access_policy = {
            expiry_time = "2025-01-01T00:00:00Z"
            permission  = "r"
            start_time  = "2024-01-01T00:00:00Z"
          }
        }
      ]
    }
  }
  tags = {
    env   = "Dev"
    owner = "John Doe"
    dept  = "IT"
  }
}
```

<!-- markdownlint-disable MD033 -->
## Requirements

The following requirements are needed by this module:

- <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) (>= 1.7.0)

- <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) (>= 3.7.0, < 5.0.0)

- <a name="requirement_random"></a> [random](#requirement\_random) (>= 3.5.0, < 4.0.0)

## Resources

The following resources are used by this module:

- [azurerm_network_security_group.nsg](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/network_security_group) (resource)
- [azurerm_network_security_rule.no_internet](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/network_security_rule) (resource)
- [azurerm_resource_group.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group) (resource)
- [azurerm_subnet.private](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet) (resource)
- [azurerm_subnet_network_security_group_association.private](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet_network_security_group_association) (resource)
- [azurerm_user_assigned_identity.example_identity](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/user_assigned_identity) (resource)
- [azurerm_virtual_network.vnet](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_network) (resource)
- [random_integer.region_index](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/integer) (resource)
- [random_string.this](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string) (resource)
- [azurerm_client_config.current](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/client_config) (data source)
- [azurerm_role_definition.example](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/role_definition) (data source)

<!-- markdownlint-disable MD013 -->
## Required Inputs

No required inputs.

## Optional Inputs

The following input variables are optional (have default values):

### <a name="input_bypass_ip_cidr"></a> [bypass\_ip\_cidr](#input\_bypass\_ip\_cidr)

Description: value to bypass the IP CIDR on firewall rules

Type: `string`

Default: `null`

### <a name="input_msi_id"></a> [msi\_id](#input\_msi\_id)

Description: If you're running this example by authentication with identity, please set identity object id here.

Type: `string`

Default: `null`

## Outputs

The following outputs are exported:

### <a name="output_containers"></a> [containers](#output\_containers)

Description: value of containers

### <a name="output_queue"></a> [queue](#output\_queue)

Description: value of queues

### <a name="output_resource"></a> [resource](#output\_resource)

Description: value of storage\_account

### <a name="output_shares"></a> [shares](#output\_shares)

Description: value of shares

### <a name="output_tables"></a> [tables](#output\_tables)

Description: value of tables

## Modules

The following Modules are called:

### <a name="module_naming"></a> [naming](#module\_naming)

Source: Azure/naming/azurerm

Version: 0.4.0

### <a name="module_public_ip"></a> [public\_ip](#module\_public\_ip)

Source: lonegunmanb/public-ip/lonegunmanb

Version: 0.1.0

### <a name="module_this"></a> [this](#module\_this)

Source: ../..

Version:

<!-- markdownlint-disable-next-line MD041 -->
## Data Collection

The software may collect information about you and your use of the software and send it to Microsoft. Microsoft may use this information to provide services and improve our products and services. You may turn off the telemetry as described in the repository. There are also some features in the software that may enable you and Microsoft to collect data from users of your applications. If you use these features, you must comply with applicable law, including providing appropriate notices to users of your applications together with a copy of Microsoft’s privacy statement. Our privacy statement is located at <https://go.microsoft.com/fwlink/?LinkID=824704>. You can learn more about data collection and use in the help documentation and our privacy statement. Your use of the software operates as your consent to these practices.
<!-- END_TF_DOCS -->