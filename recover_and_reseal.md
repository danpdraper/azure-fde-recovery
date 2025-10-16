# Recover and Re-seal

This guide describes the steps that Azure customers should follow when a package upgrade on their Ubuntu virtual machine followed by a reboot leads to the following message in the serial console:

```
Please enter the recovery key for disk ...:
```

This guide only applies to virtual machines employing [confidential disk encryption](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption-overview).

## 1. Stop the virtual machine and obtain the recovery key

The customer should start by [installing the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) and [authenticating with Azure](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli-interactively) on a host other than their virtual machine. They should then use the Azure CLI to obtain a SAS URI for their virtual machine's guest state file, replacing `$resource_group_name` and `$virtual_machine_name` in the following with the appropriate values:

```
$ az vm deallocate --resource-group $resource_group_name --name $virtual_machine_name
$ os_disk_name=$(az vm show --resource-group $resource_group_name --name $virtual_machine_name --query 'storageProfile.osDisk.name' --output tsv)
$ virtual_machine_guest_state_sas_uri=$(az disk grant-access --resource-group $resource_group_name --name $os_disk_name --access-level Read --duration-in-seconds 3600 --secure-vm-guest-state-sas --query securityDataAccessSas)
```

The next steps depend on whether the customer's disk is encrypted with a customer-managed key or a platform-managed key.

### Customer-managed key

Instruct the customer to install [Azure PowerShell](https://learn.microsoft.com/en-us/powershell/azure/install-azure-powershell). They should then copy the contents of the [get\_uki\_recovery\_key\_cmk script](https://github.com/Azure-Samples/Azure-Confidential-Computing-VMs/blob/main/CVMIssueMitigations/nullboot_upgrade_issue_111924/get_uki_recovery_key_cmk.ps1) into a local file and use that script to obtain the recovery key:

```
$ pwsh /path/to/local/get_uki_recovery_key_cmk.ps1 -vmgs_sas_uri $virtual_machine_guest_state_sas_uri
```

### Platform-managed key

Instruct the customer to install [PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell). They should then query the virtual machine guest state SAS URI headers:

```
$ pwsh -Command '$response = Invoke-WebRequest -Uri '$virtual_machine_guest_state_sas_uri' -Method Head; $response.Headers | Export-Clixml headers.xml'
```

Ask the customer for the headers.xml file, use the headers to obtain the recovery key, and provide the recovery key to the customer.

## 2. Revoke access to the disk and start the virtual machine

With the recovery key in hand, the customer should revoke access to the disk (required to start the virtual machine) and start the virtual machine:

```
$ az disk revoke-access --resource-group $resource_group_name --disk-name $os_disk_name
$ az vm start --resource-group $resource_group_name --name $virtual_machine_name
```

Through the serial console, the customer should see the recovery key prompt. After providing the recovery key, initialization should proceed and the customer should eventually be prompted for guest user credentials.

## 3. Install a new version of the problematic package

Through correspondence with the customer and Canonical support, you should have identified the package version whose installation led to the recovery key prompt. If Canonical support concludes that a new version of the package needs to be published to avoid recovery key prompts on future reboots, wait for publication of the new version to occur, and then instruct the customer to install the new version using the commands provided by Canonical support.

## 4. Update the TPM PCRs and re-seal

Proceed with this step regardless of whether a new package version was installed under the previous step. The following ensures that the values in the TPM PCRs reflect the present state of the guest software and that the disk encryption key has been sealed to the TPM against those measurements, thereby preventing recovery key prompts on future reboots. Instruct the customer to clone the [fix-azure-fde repository](https://github.com/chrisccoulson/fix-azure-fde), build and run the executable, and reboot:

```
$ git clone git@github.com:chrisccoulson/fix-azure-fde.git
$ cd fix-azure-fde/cmd/fix-azure-fde
$ sudo snap install --classic go
$ go build
$ sudo ./fix-azure-fde
$ sudo reboot
```

If the `fix-azure-fde` script produced the intended effect, the reboot should proceed through to guest user authentication without the customer's intervention.
