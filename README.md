# Azure VM Deployment

Deploy a Windows Server virtual machine and its supporting Azure network resources with an ARM template and GitHub Actions. Configure the deployment in `parameters.json`; when Actions is enabled, changes to that file on `main` start the deployment workflow.

## Repository files

| File | Purpose |
| --- | --- |
| `template.json` | ARM template for the VM, OS disk, NIC, public IP, NSG, and nested VNet deployment. |
| `parameters.json` | Deployment values, including VM sizing, names, region, and network settings. |
| `.github/workflows/deploy.yml` | GitHub Actions workflow that logs into Azure and runs the ARM deployment. |

## Authentication (read this first)

### Authentication method

GitHub Actions authenticates to Azure as a **Microsoft Entra ID (Azure AD) service principal using a client secret**. The credentials are stored as one JSON secret, `AZURE_CREDENTIALS`, and consumed by the `azure/login@v1` action through its `creds` input. No Azure password or credential is stored in the repository.

| Item | Value used by this repository |
| --- | --- |
| Auth type | Service principal with client secret (not OIDC/federated credentials) |
| Workflow action | `azure/login@v1` with `creds: ${{ secrets.AZURE_CREDENTIALS }}` |
| Recommended Azure RBAC | `Contributor`, scoped to resource group `RG5` |
| Secret storage | GitHub repository secrets (**Settings → Secrets and variables → Actions**) |

### 1. Create the service principal

Run in Azure Cloud Shell or a signed-in Azure CLI. Replace `<subscription-id>`; resource group `RG5` must already exist.

```bash
az ad sp create-for-rbac \
  --name "github-azure-vm-deployment" \
  --role contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/RG5 \
  --sdk-auth
```

The output looks like this (placeholders shown):

```json
{
  "clientId": "<app-client-id>",
  "clientSecret": "<client-secret>",
  "subscriptionId": "<subscription-id>",
  "tenantId": "<tenant-id>",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

The entire JSON is the value of `AZURE_CREDENTIALS`. The client secret is shown only once. `--sdk-auth` is deprecated in newer Azure CLI versions; if unavailable, omit it and build a JSON object with the keys `clientId`, `clientSecret`, `subscriptionId`, and `tenantId`.

`Contributor` at resource-group scope lets the workflow create and update the VM, NIC, public IP, NSG, and VNet in `RG5`, but not manage access (RBAC) or touch other resource groups.

### 2. Add the GitHub secrets

Add all three under **Settings → Secrets and variables → Actions → New repository secret**, or with the GitHub CLI:

```bash
gh secret set AZURE_CREDENTIALS --repo Isuruvh/azure-vm-deployment < sp-credentials.json
gh secret set AZURE_SUBSCRIPTION_ID --repo Isuruvh/azure-vm-deployment --body "<subscription-id>"
gh secret set VM_ADMIN_PASSWORD --repo Isuruvh/azure-vm-deployment
```

Delete `sp-credentials.json` once the secret is saved. Running `gh secret set VM_ADMIN_PASSWORD` without `--body` prompts for the value, keeping it out of shell history.

| Secret | Contents | Used by |
| --- | --- | --- |
| `AZURE_CREDENTIALS` | Full service-principal JSON | `azure/login@v1` (`creds`) |
| `AZURE_SUBSCRIPTION_ID` | Subscription GUID; must match `subscriptionId` in `AZURE_CREDENTIALS` | `azure/arm-deploy@v1` (`subscriptionId`) |
| `VM_ADMIN_PASSWORD` | Windows administrator password meeting Azure complexity rules | `azure/arm-deploy@v1` (`parameters` override for `adminPassword`) |

### 3. How the workflow uses them

From `.github/workflows/deploy.yml`:

```yaml
- name: Azure Login
  uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: Deploy ARM Template
  uses: azure/arm-deploy@v1
  with:
    subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    resourceGroupName: RG5
    template: ./template.json
    parameters: ./parameters.json adminPassword=${{ secrets.VM_ADMIN_PASSWORD }}
    deploymentName: github-actions-deployment
```

- **Login:** `azure/login@v1` signs in as the service principal; later steps run with its permissions.
- **Subscription:** `AZURE_SUBSCRIPTION_ID` selects the target subscription.
- **Admin password:** `parameters.json` sets `adminPassword` to `null` as a placeholder. The `adminPassword=...` override on the `parameters` line injects `VM_ADMIN_PASSWORD` at deploy time, so the real password never appears in the repository. The template declares it as a `SecureString`.
- **Masking:** GitHub masks secret values in logs, and does not pass secrets to workflows triggered from forked pull requests.

### Rotating or revoking credentials

- **Rotate:** run `az ad sp credential reset --id <clientId>`, then update `AZURE_CREDENTIALS` with the new `clientSecret`.
- **Revoke:** run `az ad sp delete --id <clientId>` and remove the three GitHub secrets.
- Client secrets expire. An expired secret makes the **Azure Login** step fail; rotate it as above.

### Optional: OIDC (no stored secret)

This repository does not use it, but Azure supports passwordless GitHub-to-Azure authentication with federated credentials. It requires `permissions: id-token: write` in the workflow and `client-id`, `tenant-id`, and `subscription-id` inputs on `azure/login` instead of `creds`.

## Requirements

- An Azure subscription with quota and availability for the selected VM size and resources.
- An existing resource group `RG5`, the workflow's deployment target.
- The service principal and the three GitHub secrets described above.

Keep credentials and passwords in GitHub Secrets, never in the template, parameters file, logs, or commits.

## Deploying

1. Make the desired changes to `parameters.json`.
2. Commit and push to `main`.
3. If GitHub Actions is disabled for the repository, enable it under the repository's **Actions** settings. Disabled Actions will not run a deployment; a push made while disabled is not automatically deployed when Actions is later re-enabled.
4. Open **Actions**, select **Deploy ARM Template final**, and inspect the run and its deployment step.

The workflow runs on pushes to `main` that change `template.json`, `parameters.json`, or `.github/workflows/deploy.yml`. It does not run for README-only changes. Its Azure deployment name is `github-actions-deployment`, and it uses ARM incremental mode. Incremental mode may update resources with matching names, but it does not delete resources omitted from a deployment. Changing resource names can leave the prior resources in place and create additional ones.

### Successful deployment example

The [latest successful run](https://github.com/Isuruvh/azure-vm-deployment/actions/runs/42112078174) completed both the Azure login and ARM deployment steps. At that run's commit (`9a25d92`), the example configuration deployed VM `FS1` (`Standard_D2ads_v6`) in `australiaeast`, using VNet `vnet-australiaeast-1`, subnet `default`, and resource group `RG5`. The workflow used `azure/login@v1` and `azure/arm-deploy@v1`, passing `VM_ADMIN_PASSWORD` as the secure `adminPassword` parameter.

That successful run predates the path-filter correction: its workflow watched `parameter.json` (singular). The current workflow on `main` watches `parameters.json` (plural), matching the file passed to the ARM deployment step. Thus the run demonstrates a successful deployment configuration, while the corrected workflow path is the one to use for future parameter-only pushes.

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
