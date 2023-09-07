# ALZ Bicep configuration guide

This guide is for creating a new Azure Landing Zones (ALZ) - Bicep environment using GitHub actions and a hub-and-spoke network topology instead of a virtual WAN.  
Guide focuses on adjusting parameters to reduce costs by excluding certain components and lowering the SKU of others.
The upstream release version used is v0.15.0.

## 1) Prerequisites
* A global admin account for the tenant.
* **Three subscriptions** to be used for ALZ platform subscriptions: identity, connectivity, and management.
* Optionally, you will need a subscription for placing under a management group and testing the landing zone configurations.
* Ensure that the `microsoft.insights` resource provider is enabled for the platform subscriptions before deployment.
* To work with ALZ, you'll need to install the PowerShell module using the following command: `Install-Module -Name ALZ`.
* ALZ PowerShell module requires:
  * Supported minimum PowerShell version
  * Az PowerShell Module
  * Git
  * Bicep (PowerShell module)

## 2) Create a new Azure Landing Zone Environment
Begin by creating a new folder for your ALZ environment, which you will initialize as a Git repository.  
Modify the prompt answers to match your environment.

```powershell
Test-ALZRequirement // If there is some missing requirements, install them

PS C:\Users\john.doe\repos\project1\alz-bicep> New-ALZEnvironment -o .\
Getting ready to create a new ALZ environment with you...
Copying ALZ-Bicep module to .\upstream-releases
ALZ-Bicep source directory:
The prefix that will be added to all resources created by this deployment. (e.g. 'alz')                                 
Prefix (default: alz): testalz                                                                                         

Deployment location. (e.g. 'uksouth')                                                                                   
Location : northeurope

The Type of environment that will be created. (e.g. 'live', 'canary')                                                   
Environment (default: live):                                                                                            

The identifier of the Identity Subscription. (e.g '00000000-0000-0000-0000-000000000000')                               
IdentitySubscriptionId : 11111111-1111-1111-1111-111111111111                                                           

The identifier of the Connectivity Subscription. (e.g '00000000-0000-0000-0000-000000000000')                           
ConnectivitySubscriptionId : 22222222-2222-2222-2222-222222222222

The identifier of the Management Subscription. (e.g 00000000-0000-0000-0000-000000000000)                               
ManagementSubscriptionId : 33333333-3333-3333-3333-333333333333

The email address of the contact for security issues. (e.g. security@contactme.com)
SecurityContact : john.doe@example.com

Initialize the directory .\ as a git repository? (y/n): y

C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\alzDefaultPolicyAssignments.parameters.all.json
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\customPolicyDefinitions.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\customRoleDefinitions.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\hubNetworking.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\logging.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\managementGroups.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\mgDiagSettingsAll.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\resourceGroupConnectivity.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\resourceGroupLoggingAndSentinel.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\subPlacementAll.parameters.all.json  
C:\Users\john.doe\repos\project1\alz-bicep\config\custom-parameters\vwanConnectivity.parameters.all.json  

```
## 3) Set up version control and make initial commit
Create repository to Github and then run the following commands to push the code to the repository:

```shell
# Match the remote URL with the repository you created on GitHub.
git remote add origin https://github.com/yourgithuborg/alz-bicep.git
# Adds all changes in the working directory to the staging area.
git add .
# Records a snapshot of your repository's staging area.
git commit -m "Initial commit"
# Updates the remote branch with the local commit(s)
git push -u origin main
```

## 4) Customize defaults parameters to minimize costs

Delete the `github\workflows\alz-bicep-4b.yml` file, as we are not using the virtual WAN.  
Note that Microsoft Defender for Cloud plans are enabled by default, and we are not disabling them here.  
If you want to disable Microsoft telemetry tracking, you can do it in VS Code by finding and replacing every instance (Ctrl+Shift+H) of the parTelemetryOptOut parameter set to false with true:  
```powershell
//from
"parTelemetryOptOut": {
      "value": false
//to
"parTelemetryOptOut": {
      "value": true
```
Modify parameter files to minimize costs. Keep in mind that all Microsoft Defender for Cloud plans are deployed by default and are not modified here:

```bash
//config\custom-parameters\alzDefaultPolicyAssignments.parameters.all.json
"parLogAnalyticsWorkspaceLogRetentionInDays": {
  "value": "30"
},

"parDdosProtectionPlanId": {
  "value": ""
},

//config\custom-parameters\hubNetworking.parameters.all.json
"parDdosEnabled": {
  "value": false
},

"parAzFirewallTier": {
  "value": "Basic"
},
// Clear value if you want to disable ExpressRoute Gateway provisioning
"parExpressRouteGatewayConfig": {
  "value": {}
},
// Clear value if you want to disable VPN Gateway provisioning
"parVpnGatewayConfig": {
  "value": {}
},

//config\custom-parameters\logging.parameters.all.json
"parLogAnalyticsWorkspaceLogRetentionInDays": {
  "value": 30
},
```

## 5) Setup authentication

### 5.1 Create new service principal and grant access to root management group
Please ensure you are logged in as a global admin user with the elevated User Access Administrator role (not as a guest user) to be able to grant access to the root scope.  
Run the following command e.g. in the Azure Cloud Shell:

```shell
$spnName = "sp-test-alzbicep"
$newSPJson = az ad sp create-for-rbac -n $spnName --role Owner --scopes '/'
$newSP = $newSPJson | ConvertFrom-Json
az ad app update --id $newSP.appId --sign-in-audience=AzureADMyOrg
# Print the SP information and credentials
$newSPJson
```

### 5.2 Add federated credentials

Create federated credentials from the Azure portal using this [link](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#add-federated-credentials).


### 5.3 Add app secrets to repo 
Add secrets to the repository environment variables using the values from your newly created service principal. In your GitHub repository settings, navigate to Secrets and Variables > Actions > New repository secret:

AZURE_CLIENT_ID  
AZURE_SUBSCRIPTION_ID  
AZURE_TENANT_ID  

## 6) Run GitHub Actions

Start workflows manually in this order:

1. ALZ-Bicep-1 Workflow
2. ALZ-Bicep-2 Workflow
3. ALZ-Bicep-3 Workflow
4. ALZ-Bicep-4a Workflow // as we use the hub and spoke instead of WWAN

## ALZ-bicep-1

When I used version v0.15.0, my action stopped due to an error. In the Bicep module, there is a configuration file named `bicepconfig.json` that contains rules defining the requirements for Bicep code. An error message indicated:

```shell
Error use-recent-api-versions: Use more recent API version for 'Microsoft.Automation/automationAccounts'. '2021-06-22' is 751 days old, should be no more than 730 days old, or the most recent. Acceptable versions: 2022-08-08
```

The recommended approach to address these issues is to move modules to the custom-modules folder and update the deployment script and action trigger paths as described [here](https://github.com/Azure/ALZ-Bicep/wiki/Accelerator#incorporating-modified-alz-modules).  
As a quick fix, I updated the Automation Account API version within the module it was already using. Here's an example from the file `upstream-releases\v0.15.0\infra-as-code\bicep\modules\logging\logging.bicep`:

```bicep
// updated api version from 2021-06-22 to 2022-08-08
resource resAutomationAccount 'Microsoft.Automation/automationAccounts@2022-08-08' = {
```
I encountered a second problem where every subscription generated an error message:
```shell
Status Message: The subscription
     | 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' is not registered to use
     | microsoft.insights. (Code:Conflict) 
```
I resolved this issue by going to the Azure portal and enabling the resource provider on each subscription.

## ALZ-Bicep-1 Summary
- ALZ Management Group structure  
- Logging Resource Group 
- Log analytics, automation account and Sentinel
- Custom policy and policy initiative definitions (for example, Deploy Microsoft Defender for Cloud configuration and Enforce policies in the Sandbox Landing Zone initiatives)
Custom role definitions (note that I only saw Network management (NetOps) and Security operations (SecOps) roles, but there should be more)
- Diagnostic settings for management groups

## ALZ-Bicep-2

Role assignments deployment is commented out. This uses `roleAssignmentManagementGroupMany.servicePrincipal.parameters.all.json` that can be used to grant access to a service principal or group to multiple management groups.  
This is not needed because we have the service principal with owner rights on the root level and also in every management group it created.

The second task here is to deploy the policy assignments.

## ALZ-Bicep-2 Summary
- Role assignments deployment (commented out by default)
- Built-in and custom policy assignments deployment to different management group scopes

## ALZ-Bicep-3

This one does just the subscription placement under management groups. Platform subscription IDs are automatically placed in the correct parameters in `subPlacementAll.parameters.all.json` parameter file when you run the `New-ALZEnvironment` command.
ALZ has many policy assignments enabled by default, so it would be wise to evaluate the policies before placing production subscriptions under the management groups.
Subscription placement under an online management group can be done like this:

```json
//config\custom-parameters\subPlacementAll.parameters.all.json
"value": [
  "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
],
"metadata": {
  "description": "Optionally, you can add a comment here, for example, to define the subscription names"
}
```

The subscription vending module can also be utilized to place subscriptions under management groups and configure additional settings such as tags, VNet, and role assignments. The Bicep subscription vending module is available for consumption from the public Bicep module registry  

## ALZ-Bicep-3 Summary

Subscription placement for platform subscriptions. Optionally, it can also manage subscription placement under management groups for other subscriptions.

## ALZ-Bicep-4a

This one is the hub and spoke deployment.
I have set the `parDdosEnabled` value to `false`, which means that the DDOS protection plan will not be deployed even if the `parDdosPlanName` parameter is specified.
When I changed the VPN SKU to `Basic` to minimize costs, I received an error message stating that the specified SKU is not valid for the gateway. To resolve this issue, I changed the SKU back to `VpnGw1`, and the deployment was successful.
After successfully deploying the configuration, I attempted to redeploy it but encountered a failure with the following error message: "Status Message: Put on Firewall Policy alz-azfwpolicy-northeurope Failed with 1 faulted referenced firewalls (Code:FirewallPolicyUpdateFailed)"
I chose not to attempt to resolve the issue, and I deleted the firewall from the Azure portal after observing the deployed Azure resources to minimize costs. Alternatively, I could have used PowerShell commands to stop the firewall.

## ALZ-Bicep-4a Summary
- Connectivity Resource Group
- Hub VNet
- Azure Bastion + bastion subnet NSG
- Private link DNS Zones + virtual network link to HUB VNet
- Azure firewall with public IP and management public IP
- Azure firewall policy
- VPN Gateway and ExpressRoute Gateway are available options, but we have explicitly disabled them in the parameters.
- Route table

## Closing words
Now you have a working ALZ environment with a hub and spoke network topology. Feel free to explore it from portal and make adjustments to parameter files. Place subscription under management group to explore policy assignments using the subscription placement module or Bicep vending module.
