# Azure VM Deployment

Deploy a Windows Server virtual machine and its supporting Azure network resources with an ARM template and GitHub Actions. Configure the deployment in `parameters.json`; when Actions is enabled, changes to that file on `main` start the deployment workflow.

## Repository files

| File | Purpose |
| --- | --- |
| `template.json` | ARM template for the VM, OS disk, NIC, public IP, NSG, and nested VNet deployment. |
| `parameters.json` | Deployment values, including VM sizing, names, region, and network settings. |
| `.github/workflows/deploy.yml` | GitHub Actions workflow that logs into Azure and runs the ARM deployment. |

## Requirements and one-time setup

- An Azure subscription with quota and availability for the selected VM size and resources.
- A GitHub Actions service principal with permission to deploy resources in the target resource group.
- The following repository secrets, configured under **Settings → Secrets and variables → Actions**:

| Secret | Description |
| --- | --- |
| `AZURE_CREDENTIALS` | Service-principal credential JSON in the format expected by [`azure/login`](https://github.com/Azure/login). |
| `AZURE_SUBSCRIPTION_ID` | Subscription ID used for the deployment; it must match the service principal's subscription. |
| `VM_ADMIN_PASSWORD` | Strong password for the Windows administrator account. It is passed to the template at deployment time and is not stored in `parameters.json`. |

Keep credentials and passwords in GitHub Secrets, never in the template, parameters file, logs, or commits. The workflow uses `RG5` as its deployment resource group; ensure it exists and the service principal can deploy there.

## Deploying

1. Make the desired changes to `parameters.json`.
2. Commit and push to `main`.
3. If GitHub Actions is disabled for the repository, enable it under the repository's **Actions** settings. Disabled Actions will not run a deployment; a push made while disabled is not automatically deployed when Actions is later re-enabled.
4. Open **Actions**, select **Deploy ARM Template final**, and inspect the run and its deployment step.

The workflow runs on pushes to `main` that change `template.json`, `parameters.json`, or `.github/workflows/deploy.yml`. It does not run for README-only changes. Its Azure deployment name is `github-actions-deployment`, and it uses ARM incremental mode. Incremental mode may update resources with matching names, but it does not delete resources omitted from a deployment. Changing resource names can leave the prior resources in place and create additional ones.

## Configuration reference

Values in `parameters.json` use the shape `"parameterName": { "value": ... }`. Keep parameter names and JSON types compatible with `template.json`. The current file's values are examples for the deployed configuration; change names and settings deliberately to avoid collisions or unintended changes to existing resources.

### VM and operating system

| Parameter | Purpose |
| --- | --- |
| `virtualMachineName` | Azure VM resource name. |
| `virtualMachineComputerName` | Windows computer name. |
| `virtualMachineSize` | Azure VM SKU. Check regional availability, quota, and feature compatibility. |
| `adminUsername` | Windows administrator username. |
| `adminPassword` | Secure template parameter; the workflow supplies it from `VM_ADMIN_PASSWORD`. Leave it out of source control. |
| `osDiskType` | Managed OS disk storage SKU. |
| `osDiskDeleteOption` | Whether the OS disk is deleted with the VM. |
| `diskControllerType` | OS disk controller setting; confirm support for the chosen VM size. |
| `securityType` | VM security type. |
| `patchMode`, `enablePeriodicAssessment`, `enableHotpatching`, `rebootSetting` | Windows update and patch assessment behavior. |
| `hibernationEnabled` | Whether VM hibernation is enabled. |

The image is currently fixed in `template.json` to Microsoft Windows Server 2025 Datacenter Azure Edition, using the latest image version. Changing `parameters.json` alone does not select a different operating system image.

### VM networking

| Parameter | Purpose |
| --- | --- |
| `location` | Region for the VM, NIC, public IP, and NSG. |
| `virtualNetworkName`, `subnetName` | VNet and subnet identifiers used by the NIC configuration. |
| `networkInterfaceName` | NIC resource name. |
| `enableAcceleratedNetworking` | Whether accelerated networking is enabled; the VM size must support it. |
| `networkSecurityGroupName` | NSG resource name attached to the NIC. |
| `networkSecurityGroupRules` | Array of custom NSG security rules. The committed value is empty. |
| `publicIpAddressName` | Public IP resource name. |
| `publicIpAddressType`, `publicIpAddressSku` | Public IP allocation method and SKU. |
| `publicIpAddressId` | Declared parameter, but the current template builds the public IP reference from `publicIpAddressName`; changing this value alone does not change the referenced resource. |
| `pipDeleteOption`, `nicDeleteOption` | Delete behavior for the public IP and NIC when the VM is deleted. |

The NIC is configured with a public IP. The template creates an NSG, but an empty `networkSecurityGroupRules` array adds no custom rules. Review the effective NSG rules and your access requirements before deployment; do not expose management ports broadly to the internet.

### Virtual network deployment

| Parameter | Purpose |
| --- | --- |
| `vnetLocation` | Region for the VNet and nested network resources. |
| `vnetVirtualNetworkName` | VNet name created or updated by the nested deployment. |
| `vnetProperties` | VNet properties, including address space and subnet definitions. |
| `vnetResourceGroupName` | Resource group targeted by the nested VNet deployment. |
| `vnetDeploymentName` | Name of the nested VNet deployment. |
| `vnetExtendedLocation`, `vnetTagsByResource` | Optional extended location and resource tags. |
| `vnetNatGatewaysWithNewPublicIpAddress`, `vnetNatGatewaysWithoutNewPublicIpAddress` | Optional NAT Gateway configurations. |
| `vnetNatGatewayPublicIpAddressesNewNames` | Names for public IPs created for NAT Gateways. |
| `vnetNetworkSecurityGroupsNew` | Optional NSGs created as part of the VNet deployment. |
| `virtualMachineRG` | Declared parameter, but the workflow's `resourceGroupName: RG5` determines the main deployment scope. |

Make sure the subnet configured in `vnetProperties` exists and its address range is contained by the VNet address space. Keep the VNet and subnet names consistent with the NIC settings.

### Fixed deployment references

Some deployment values are currently fixed outside `parameters.json`:

- The GitHub Actions workflow deploys the main template to resource group `RG5`.
- `template.json` contains a subnet resource ID with a fixed subscription ID, resource group (`RG5`), and VNet name (`vnet-australiaeast-1`).

Therefore, changing `vnetResourceGroupName`, `virtualNetworkName`, or related parameter values alone may not redirect the NIC to a different network. To use a different subscription, resource group, or VNet, update the corresponding workflow and template references as well, then verify the resource IDs before deploying.

## Troubleshooting

Open the workflow run in **Actions** and inspect the **Deploy ARM Template** step for the Azure error details.

| Symptom | Check |
| --- | --- |
| No workflow run | Actions must be enabled; the push must be to `main` and change one of the watched paths. |
| Azure login fails | Check `AZURE_CREDENTIALS` JSON and service-principal status. |
| Authorization or resource-group error | Confirm `AZURE_SUBSCRIPTION_ID`, access to `RG5`, and the service principal's deployment permissions. |
| Network or NIC deployment fails | Check VNet/subnet names, address ranges, fixed subnet resource ID, region, and resource groups. |
| VM deployment fails | Check password requirements, VM SKU availability/quota, resource naming, accelerated networking, and disk controller compatibility. |

## Costs and cleanup

Azure charges may apply for the VM, disks, public IP, and other provisioned resources. Review current Azure pricing and your subscription budget. Stopping the VM may not stop all charges.

This repository does not include a cleanup workflow. Review the target resource group and delete only resources you no longer need using the Azure portal or Azure CLI; avoid removing resources used by other workloads.
