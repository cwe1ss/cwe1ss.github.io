---
title: Deploy from GitHub to Azure without any secrets using managed identities
date: 2022-10-26T20:00:00+02:00
categories:
- Azure
toc:
  enable: true
---

I've been building [a microservices template](https://github.com/cwe1ss/msa-template) as part of my master's thesis. It's using GitHub for code hosting and Microsoft Azure for hosting the resources. One key requirement of my template is to use [Managed identities for Azure](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/) everywhere and not use any secrets when connection to dependent resources.

Managed identities are a great feature and very easy to use for built-in workloads like VMs, Azure Container Apps, App Services. However, until recently, managed identities could not be used for non-native workloads like GitHub Actions. We had to use an Azure AD app registration instead and store its credentials (including a `CLIENT_SECRET`) as GitHub secrets. With the introduction of [workload identity federation](https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation) for app registrations, it was then possible to [configure a trust relationship between GitHub and Azure](https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp#configure-a-federated-identity-credential-on-an-app) that allows the GitHub Actions to authenticate to Azure without the need for providing a `CLIENT_SECRET`. This would have already solved my requirement for not needing any secrets, but the problem is, that creating an Azure AD app registration requires elevated permissions and therefore often can't easily be done by regular developers. 

The good news is that **Microsoft now also supports federated credentials for user-assigned managed identities**! Managed identities only require regular Azure RBAC rights and can be created via Bicep templates and are therefore much easier to integrate into Infrastructure as Code-processes and CI/CD systems.

[Uday Hegde](https://github.com/udayxhegde) has written a good blog post explaining federated credentials for managed identities: https://blog.identitydigest.com/azuread-federate-mi/

The remainder of this blog post will focus on how federated credentials for managed identities are used in my microservices template. Have a look at the project's [README.md](https://github.com/cwe1ss/msa-template/blob/main/README.md) for more details.

## Overview

For my microservices template, the entire process for creating the Azure resources and connecting GitHub with Azure is automated via the [init-platform.ps1](https://github.com/cwe1ss/msa-template/blob/main/infrastructure/init-platform.ps1) script. The script must be executed *manually* once, since there's the "chicken and egg"-problem of already needing the managed identity to deploy Azure resources via a GitHub workflow.

The script will execute the following steps (among other things that are out of scope for this post):
* It will create a resource group in Azure that will host the managed identity.
* It will create a user-assigned managed identity in this resource group.
* The managed identity will be given `Contributor` & `UserAccessAdministrator` rights on the Azure subscription.
  * (So that the GitHub workflows of my services can create Azure resources and assign rights to newly created managed identities)
* The managed identity will be given additional AAD permissions.
  * (My GitHub workflows need to be able to query AAD groups)
* A GitHub environment called `platform`  will be created via the GitHub CLI. 
  * This will be used to protect further deployments with required reviewers or other protection rules.
* Federated credentials will be added to the managed identity for the `main`-branch and for the `platform`-environment.
  * There *must* be a federated credential for each branch and environment that we want to deploy from.
* The necessary resource IDs (tenant id, subscription id, managed identity client id) will be created as GitHub secrets

## Deploying the managed identity and its federated credentials to Azure

The template creates all Azure resources via [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep)-templates stored in the [infrastructure](https://github.com/cwe1ss/msa-template/tree/main/infrastructure) directory. The managed identity for GitHub is part of the [platform-resources](https://github.com/cwe1ss/msa-template/tree/main/infrastructure/platform) that are shared by all environments of my microservice template (e.g., development, production).

To create the managed identity for GitHub, the following Bicep-template is used:

```bicep
resource githubIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2022-01-31-preview' = {
  name: githubIdentityName
  location: location
  tags: tags
}
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/platform/github-identity-resources.bicep#L37))*

Creating the federated credentials via Bicep is more complex. Since I need federated credentials for the `main`-branch, the shared `platform`-environment and each actual application environment (`development`, `production`), I'm creating a list variable that holds the `name` and `subject` for each credential based on my global config-file:

```bicep
var config = loadJsonContent('./../config.json')

// All credentials must be in one list as concurrent writes to /federatedIdentityCredentials are not allowed.
var ghBranchCredentials = [{
  name: 'github-branch-${githubDefaultBranchName}'
  subject: 'repo:${githubRepoNameWithOwner}:ref:refs/heads/${githubDefaultBranchName}'
}]
var ghPlatformCredentials = [{
  name: 'github-env-platform'
  subject: 'repo:${githubRepoNameWithOwner}:environment:platform'
}]
var ghEnvironmentCredentials = [for item in items(config.environments): {
  name: 'github-env-${item.key}'
  subject: 'repo:${githubRepoNameWithOwner}:environment:${item.key}'
}]
var githubCredentials = concat(ghBranchCredentials, ghPlatformCredentials, ghEnvironmentCredentials)
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/platform/github-identity-resources.bicep#L16-L31))*

The credential resources are then created via a Bicep-loop. It's important that `batchSize(1)` is used because concurrent writes are not supported and will result in a deployment error.

```bicep
@batchSize(1)
resource federatedCredentials 'Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials@2022-01-31-preview' = [for item in githubCredentials: {
  name: item.name
  parent: githubIdentity
  properties: {
    audiences: [
      'api://AzureADTokenExchange'
    ]
    issuer: 'https://token.actions.githubusercontent.com'
    subject: item.subject
  }
}]
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/platform/github-identity-resources.bicep#L43-L58))*

The fields `audiences`, `issuer` and `subject` are set according to [the requirements by GitHub](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure).

## Assigning RBAC-roles to the managed identity

In order for my GitHub-worflows to be able to deploy resources to Azure, the managed identity must have the appropriate RBAC permissions. For my template, I'm assigning the `Contributor`-role and `UserAccessAdministrator`-role to the identity. The `UserAccessAdministrator`-role is necessary to allow my GitHub workflows to create other managed identities and to assign RBAC-roles to them.

To assign a RBAC-role, we must know its internal ID. For built-in roles, this ID can be found here: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles

Here's the code to reference the `Contributor`-role:
```bicep
resource contributorRoleDefinition 'Microsoft.Authorization/roleDefinitions@2022-04-01' existing = {
  scope: subscription()
  name: 'b24988ac-6180-42a0-ab88-20f7382dd24c'
}
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/platform/github-identity.bicep#L16-L20))*

With the reference to the role definition, we can now create the actual role assignment for the managed identity:

```bicep
resource githubIdentityContributor 'Microsoft.Authorization/roleAssignments@2020-04-01-preview' = {
  name: guid(subscription().id, 'github', 'Contributor')
  properties: {
    roleDefinitionId: contributorRoleDefinition.id
    principalId: githubIdentity.outputs.githubIdentityPrincipalId
    principalType: 'ServicePrincipal'
  }
}
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/platform/github-identity.bicep#L52-L59))*

**NOTE:** Azure does not automatically delete RBAC-role assignments when the managed identity is deleted. You must *manually* delete them or future re-deployments will fail with a conflict.

## Assigning AAD permissions to the managed identity

Some of my GitHub workflows need to be able to query the Azure AD graph for details about an Azure AD group and to do so, the managed identity must have the proper AAD permissions.

Unfortunately, AAD resources and permissions can NOT be created via Bicep templates, so we need to either use the [AzureAD](https://learn.microsoft.com/en-us/powershell/module/azuread)-module (which is planned for deprecation), its successor-module [Microsoft.Graph](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview), or use the Graph REST API.

Since I'm already using the [Az](https://learn.microsoft.com/en-us/powershell/azure/install-az-ps)-module to deploy my Bicep-templates, I didn't want to use another module and potentially deal with separate sign-in methods and tokens, so I decided to just call the Graph API directly:

```ps
# A list of all AAD permissions that should be granted for the managed identity
# https://learn.microsoft.com/en-us/graph/permissions-reference
$githubIdentityMsGraphPermissions = @(
    "Group.Read.All"
)

# This is a well-known ID for the MS Graph service principal
$msGraphSp = Get-AzAdServicePrincipal -ApplicationId "00000003-0000-0000-c000-000000000000"
# We're using the Az-module to aquire an access token for the Graph API
$graphAccessToken = Get-AzAccessToken -ResourceUrl "https://graph.microsoft.com/"

# https://learn.microsoft.com/en-us/graph/api/resources/approleassignment
$apiUrl = "https://graph.microsoft.com/v1.0/servicePrincipals/$($githubIdentity.Id)/appRoleAssignments"

# We're only creating assignments that don't yet exist
$existingAssignments = Invoke-RestMethod -Uri $apiUrl -Method Get -Headers @{ Authorization = "Bearer $($graphAccessToken.Token)" }

foreach ($permissionName in $githubIdentityMsGraphPermissions) {
    #$permissionName = "Group.Read.All"
    $appRoleId = ($msGraphSp.AppRole | Where-Object { $_.Value -eq $permissionName } | Select-Object).Id

    $exists = $existingAssignments.value | Where-Object { $_.appRoleId -eq $appRoleId }
    if ($exists) {
        Write-Success "Permission '$permissionName' already exists"
    } else {
        $body = @{
            appRoleId = $appRoleId
            resourceId = $msGraphSp.Id
            principalId = $githubIdentity.Id
        }
        Invoke-RestMethod -Uri $apiUrl -Method Post -ContentType "application/json" `
            -Headers @{ Authorization = "Bearer $($graphAccessToken.Token)" } `
            -Body $($body | convertto-json) | Out-Null

        Write-Success "Permission '$permissionName' created"
    }
}
````
*([Original source](https://github.com/cwe1ss/msa-template/blob/main/infrastructure/init-platform.ps1#L204-L229))*

## Creating a GitHub environment for the `platform`

My template uses [GitHub environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) to protect deployments to Azure.

As all previous steps, this could be done manually via the UI, but I prefer to have automation and therefore create the environment via the script.

The GitHub CLI does not yet have support for environments, so we need to use `gh api` to call the GitHub REST API. The script also uses a custom [Exec](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/_includes/helpers.ps1#L12)-function (copied from [psake](https://github.com/psake/psake/blob/master/src/public/Exec.ps1)) to fail the PowerShell script if the CLI-invocation fails.

To create a GitHub environment, I'm using the following code:

```ps
$body = @{
  reviewers = @(
    @{ type = "User"; id = $ghUser.id }
  )
} | ConvertTo-Json -Compress

$ghEnv = Exec { $body | gh api "/repos/$($ghRepo.nameWithOwner)/environments/$environment" -X PUT -H "Accept: application/vnd.github+json" --input - } | ConvertFrom-Json
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/init-platform.ps1#L354-L360))*

## Creating the resource IDs as secrets in GitHub

While there are no passwords to authenticate GitHub with Azure, we still need to tell GitHub about the managed identity and its target tenant & subscription. We therefore need to store some IDs as GitHub secrets. These IDs will then be used by the GitHub workflows when running deployments to Azure.

```ps
Exec { gh secret set "AZURE_CLIENT_ID" -b $githubIdentity.AppId }
Exec { gh secret set "AZURE_SUBSCRIPTION_ID" -b $((Get-AzContext).Subscription.Id) }
Exec { gh secret set "AZURE_TENANT_ID" -b $((Get-AzContext).Subscription.TenantId) }
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/init-platform.ps1#L372-L374))*

## Using the managed identity in a GitHub workflow

With the preceding steps, the managed identity has been created, federated credentials have been assigned and GitHub has references to the required IDs as GitHub secrets. We're therefore finally ready to do any further deployments via GitHub workflows.

My microservice template includes [multiple GitHub workflows](https://github.com/cwe1ss/msa-template/tree/main/.github/workflows). There's a workflow for each service, for shared environment resources, and for the shared platform resources.

We'll look at the [platform.yml](https://github.com/cwe1ss/msa-template/blob/main/.github/workflows/platform.yml) workflow as an example for the following steps.

In order for GitHub-workflows to work with federated credentials, we [must add permissions for the token](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure#updating-your-github-actions-workflow):

```yml
permissions:
  id-token: write
  contents: read
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/.github/workflows/platform.yml#L6-L8))*

We can then use `azure/login` to authenticate with Azure:
```yml
    - uses: azure/login@v1
      with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          # This allows us to use Azure PowerShell in addition to Azure CLI
          enable-AzPSSession: true
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/.github/workflows/platform.yml#L20-L25))*

That's it. Further calls via the `Az`-module should then be able to run successfully:

```ps
New-AzSubscriptionDeployment `
    -Location $config.location `
    -Name ("platform-" + (Get-Date).ToString("yyyyMMddHHmmss")) `
    -TemplateFile .\platform\main.bicep `
    -TemplateParameterObject @{
        deployGitHubIdentity = $false
    } `
    -Verbose | Out-Null
```
*([Original source](https://github.com/cwe1ss/msa-template/blob/650565fb5be038ae737676ac3f87291763b082bd/infrastructure/deploy-platform.ps1#L16-L23))*

Feel free to start a discussion or create an issue in [cwe1ss/msa-template](https://github.com/cwe1ss/msa-template) if you have any feedback.