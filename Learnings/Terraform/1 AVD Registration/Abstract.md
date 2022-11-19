
# Ignore Changes with a use case and an AVD lesson.
## Azure Virtual Desktop Terraform stale registration

This is to document an issue I encountered and resolved while using the AzureRM terraform provider implementation for Azure Virtual Desktop. 

I should start by clarifying that everything works as intended by the tf provider. When deploying our AVD solution, Azure Virtual Machine creation is followed by vm extensions, the final extension (once the virtual machine is fully deployed) applies a DSC configuration which registers the vm to the host pool durably. However, this registration process fails somewhere deeper. It appears that when a vm registers to the hostpool with the same metadata, the registration does not update the hostpool in the portal. Those VMs remain in an unavailable state as from when they were destroyed. Because the registration happens on the DSC apply inside the server, destroying the server does not remove the server's registration from the hostpool and prevents the replacement server from registering. 





## Background Details on use-case

Here Terraform identifies an image from the gallery, based on values I'm feeding into the variables through Octopus Deploy.
This image is created and deployed to the gallery by a Packer Pipeline that runs after every patch Tuesday with the option for manual runs if an important need arose.
This image is given a value and saved to the terraform state for the data resource.
If subsequent runs identify that the image id has changed, it will update this resource.

```
data "azurerm_shared_image" "app" {
  gallery_name        = var.image_gallery_name
  name                = var.app_image_name
  resource_group_name = var.app_image_resource_group_name
}
```

When creating new images, we want those new images to replace the images on our existing AVD hosts.
This would require a replacement, but if we do it during out maintenance period, we should be fine.

```
source_image_id = data.azurerm_shared_image.app.id
```

However, if you are running a massive builder deployment that ingests variables, and this is just a small piece of the overall deployment, you may find that other routine operations require a redeployment outside of maintenance hours.

My use-case is deploying all manner of configurations to ensure full feature coverage. Including, managing users. 
So when we get a name change request, it is as simple as changing a csv that has the names of the users and running a new tf deploy.
This could prevent normal operations until the next maintenance window, but we could change this.


## Deployment

Here is the relevant deployment resources to see what I am dealing with

```terraform
resource "azurerm_virtual_desktop_host_pool" "pooledbreadthfirst" {
  location            = var.primary_region
  resource_group_name = var.rg1.name
  # personal_desktop_assignment_type = "Automatic"
  name                     = "${var.client_name}-avdhp"
  friendly_name            = "pooledbreadthfirst"
  validate_environment     = false
  start_vm_on_connect      = false
  custom_rdp_properties    = "audiocapturemode:i:0;audiomode:i:2;drivestoredirect:s:*;encode redirected video capture:i:0;camerastoredirect:s:;devicestoredirect:s:;redirectclipboard:i:1;redirectcomports:i:0;redirectlocation:i:0;redirectprinters:i:1;redirectsmartcards:i:0;usbdevicestoredirect:s:"
  description              = "Acceptance Test: A pooled host pool - pooledbreadthfirst"
  type                     = "Pooled"
  maximum_sessions_allowed = 16
  load_balancer_type       = "BreadthFirst"

  lifecycle {
    ignore_changes = [
      load_balancer_type
    ]
  }

  tags = merge(local.default_tags, { type = "hostpool" })
}

resource "azurerm_virtual_desktop_host_pool_registration_info" "pooledbreadthfirst" {
  hostpool_id     = azurerm_virtual_desktop_host_pool.pooledbreadthfirst.id
  expiration_date = time_rotating.avd_token.rotation_rfc3339

  depends_on = [
    azurerm_virtual_desktop_host_pool.pooledbreadthfirst
  ]
}

data "azurerm_shared_image" "app" {
  gallery_name        = var.image_gallery_name
  name                = var.app_image_name
  resource_group_name = var.app_image_resource_group_name
}

resource "random_string" "avd_vm" {
  count  = var.rdsh_count
  
  length = 8
  special = false
  upper = false
  
  keepers = {
    # Generate a new pet name each time we update the setup_host script
    source_content = "${var.setup_host_template}"
  }
}

resource "azurerm_network_interface" "avd_vm_nic" {
  count               = length(random_string.avd_vm)
  name                = "${var.client_name}-${random_string.avd_vm[count.index].id}-nic"
  resource_group_name = var.resource_group.name
  location            = var.primary_region

  ip_configuration {
    name                          = "${var.client_name}-${random_string.avd_vm[count.index].id}_config"
    subnet_id                     = var.subnet.id
    private_ip_address_allocation = "dynamic"
  }

  tags = local.default_tags
}

resource "azurerm_windows_virtual_machine" "avd_vm" {
  count                 = length(azurerm_network_interface.avd_vm_nic)
  name                  = "${var.client_name}-${random_string.avd_vm[count.index].id}"
  resource_group_name   = var.resource_group.name
  location              = var.primary_region
  size                  = var.avd_vm_size
  network_interface_ids = ["${azurerm_network_interface.avd_vm_nic.*.id[count.index]}"]
  provision_vm_agent    = true
  admin_username        = var.admin_username
  admin_password        = var.admin_password
  timezone              = var.timezone

  os_disk {
    name                      = random_string.avd_vm[count.index].id
    caching                   = "ReadWrite"
    storage_account_type      = "Premium_LRS"
    write_accelerator_enabled = false
  }
  source_image_id = data.azurerm_shared_image.app.id

  identity {
    type = "SystemAssigned"
  }

  depends_on = [
    random_string.avd_vm
  ]

  tags = merge(local.default_tags, local.app_tags, { count = "${count.index + 1}" }, {disaster_recovery = "false" })

  lifecycle {
    ignore_changes = [
      boot_diagnostics,
      identity
    ]
  }
}

# Extensions


# This extension should activate last, as it registers the vm to the hostpool
resource "azurerm_virtual_machine_extension" "last_host_extension_hp_registration" {
  count                      = var.rdsh_count
  name                       = "${var.client_name}-${random_string.avd_vm[count.index].id}-avd_dsc"
  virtual_machine_id         = azurerm_windows_virtual_machine.avd_vm.*.id[count.index]
  publisher                  = "Microsoft.Powershell"
  type                       = "DSC"
  type_handler_version       = "2.73"
  auto_upgrade_minor_version = true
  # automatic_upgrade_enabled = true

  settings = <<-SETTINGS
    {
      "modulesUrl": "https://wvdportalstorageblob.blob.core.windows.net/galleryartifacts/Configuration_3-10-2021.zip",
      "configurationFunction": "Configuration.ps1\\AddSessionHost",
      "properties": {
        "HostPoolName":"${azurerm_virtual_desktop_host_pool.pooledbreadthfirst.name}"
      }
    }
SETTINGS

  protected_settings = <<PROTECTED_SETTINGS
  {
    "properties": {
      "registrationInfoToken": "${azurerm_virtual_desktop_host_pool_registration_info.pooledbreadthfirst.token}"
    }
  }
PROTECTED_SETTINGS

  lifecycle {
    ignore_changes = [settings, protected_settings]
  }

  depends_on = [
    azurerm_virtual_machine_extension.sixth-vminsights_agent
  ]
}

resource "azurerm_virtual_machine_extension" "first-domain_join_extension" {
  count                      = var.rdsh_count
  name                       = "${var.client_name}-avd-${random_string.avd_vm[count.index].id}-domainJoin"
  virtual_machine_id         = azurerm_windows_virtual_machine.avd_vm.*.id[count.index]
  publisher                  = "Microsoft.Compute"
  type                       = "JsonADDomainExtension"
  type_handler_version       = "1.3"
  auto_upgrade_minor_version = true
  # automatic_upgrade_enabled = true

  settings = <<SETTINGS
    {
      "Name": "${var.domain_name}",
      "OUPath": "${var.ou_path}",
      "User": "${var.domain_user_upn}@${var.domain_name}",
      "Restart": "true",
      "Options": "3"
    }
SETTINGS

  protected_settings = <<PROTECTED_SETTINGS
    {
      "Password": "${var.admin_password}"
    }
PROTECTED_SETTINGS

  lifecycle {
    ignore_changes = [settings, protected_settings]
  }
}

# All extensions after domain join are placed after a wait time
# This prevents the reboot from domain join from affecting the extensions afterward.
resource "time_sleep" "domain_join_reboot" {
  count = length(azurerm_windows_virtual_machine.avd_vm)
  create_duration = "240s"
  destroy_duration = "5s"

  triggers = {
    name = "${random_string.avd_vm[count.index].id}"
    virtual_machine_id = azurerm_windows_virtual_machine.avd_vm[count.index].id
  }

  depends_on = [
    azurerm_virtual_machine_extension.first-domain_join_extension,
    random_string.avd_vm
  ] 
}


```

## Identified assumption
AzureRM tf provider doesn't remove the hosts from the hostpool when the host vms are destroyed (and makes sense since they're not tf resource linked directly). 

## What's happening:
The hosts register using a DSC extension which configures the vm to register, creating a link outside of the terraform deployment. The hostpools just think the vms have shutdown and are coming back when they're destroyed. What we're seeing is terraform replacing the hosts on the source image id info. 

## Why it's configured that way 
These VM source image (source_image_id) are set further up the pipeline by packer which is manufacturing new golden images for our hosts (I hope the benefits are self-evident). When the source_image changes after a successful run we'll want to replace the vm image. 

## Ohh the big benefit is
Automating an item on your SOC/Security compliance checklist for this item 👍

## Short term mitigation: 
Issue: ignore_changes = source_image_id

## Medium term fix: 
automated patch job
If host pool host status == unavailable | remove from pool
Reboot vms

## Expected changes:
Professional Services will need to manually destroy their managed AVD host VMs each month to get the latest image after a new one is released to the image gallery to maintain compliance on being updated. 