# How to deploy Terraform modules using GitHub CI/CD pipelines at Contoso

In this guide, we will walk you through the steps to deploy Terraform modules using GitHub's Continuous Integration/Continuous Deployment (CI/CD) pipelines at Contoso. By automating the deployment process, you can ensure consistent and reliable infrastructure provisioning across your projects using Contoso's bespoke Terraform modules.  

---

## Overview

This tutorial uses examples from the following GitHub demo project [template repository](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments).

Recently, we at Contoso decided to create two public **GitHub Actions** on the GitHub Actions marketplace called **[Terraform Plan for AZURE](https://github.com/marketplace/actions/terraform-plan-for-azure)** and **[Terraform Apply for AZURE](https://github.com/marketplace/actions/terraform-apply-for-azure)**. In this guide, we will demonstrate how you can use these public marketplace actions to automate your Terraform deployments to Azure using Contoso's bespoke CI/CD and GitHub Actions.  

We will explore how to build enterprise-scale multi-environment infrastructure deployments in **Azure** using a **non-monolithic** approach. This approach helps to construct and simplify complex Terraform deployments into manageable work streams that can be updated independently, increase build time, and reduce duplicate workflow code by utilising **GitHub Actions**.

---

## Pre-Requisites

At Contoso, we are committed to leveraging the power of GitHub Actions and CI/CD practices to automate our Terraform deployments to Azure. In this guide, we will walk you through the necessary steps to get started with our bespoke CI/CD setup using GitHub Actions.

We will perform the following steps:

1. **Create Azure Resources (Terraform Backend):** (Optional) We will first create a few resources that will host our Terraform backend state configuration. This includes setting up a Resource Group, Storage Account, and KeyVault. Additionally, we will create an **Azure Active Directory App & Service Principal** that will have access to our Terraform backend and subscription in Azure. This Service Principal will be linked with our GitHub project and workflows later in the tutorial.
2. **Create a GitHub Repository:** We will create a GitHub project and set up the relevant secrets and (optional) GitHub environments that we will be using. This project will host our workflows and Terraform configurations.
3. **Create Terraform Modules (Modular):** We will set up a few Terraform ROOT modules, ensuring they are separated and modular from each other (non-monolithic).
4. **Create GitHub Workflows using Marketplace Actions:** After configuring our repository and Terraform ROOT modules, we will create workflows and configure multi-stage deployments using public marketplace actions to run and deploy resources in Azure based on our Terraform ROOT Modules.

By following these steps, you will be able to automate your Terraform deployments to Azure using Contoso's bespoke CI/CD and GitHub Actions setup.

### Step 1 - Create Azure resources (Terraform Backend)

To set up the resources that will act as our Terraform backend, we have created a PowerShell script using AZ CLI that will build and configure everything and store the relevant details/secrets needed to link our GitHub project in a key vault. You can find the script on our GitHub code page: [AZ-GH-TF-Pre-Reqs.ps1](https://github.com/Pwd9000-ML/blog-devto/blob/main/posts/2022/GitHub-Actions-Terraform-Deployment-Part1/code/AZ-GH-TF-Pre-Reqs.ps1).

This script will:

- Create a Resource Group
- Set up a Storage Account to store Terraform state files
- Configure a Key Vault to store secrets
- Create an Azure Active Directory App & Service Principal with the necessary permissions

By running this script, you will have all the necessary Azure resources configured to serve as the backend for your Terraform deployments. This setup is crucial for ensuring that your Terraform state is securely stored and managed, allowing for seamless integration with GitHub Actions.

First we will log into Azure by running:

```powershell
az login
```

After logging into Azure and selecting the subscription, we can run the script that will create all the pre-requirements we'll need:

```powershell
## code/AZ-GH-TF-Pre-Reqs.ps1

#Log into Azure
#az login

# Setup Variables.
$randomInt = Get-Random -Maximum 9999
$subscriptionId=$(az account show --query id -o tsv)
$resourceGroupName = "Demo-Terraform-Core-Backend-RG"
$storageName = "tfcorebackendsa$randomInt"
$kvName = "tf-core-backend-kv$randomInt"
$appName="tf-core-github-SPN$randomInt"
$region = "uksouth"

# Create a resource resourceGroupName
az group create --name "$resourceGroupName" --location "$region"

# Create a Key Vault
az keyvault create `
    --name "$kvName" `
    --resource-group "$resourceGroupName" `
    --location "$region" `
    --enable-rbac-authorization

# Authorize the operation to create a few secrets - Signed in User (Key Vault Secrets Officer)
az ad signed-in-user show --query id -o tsv | foreach-object {
    az role assignment create `
        --role "Key Vault Secrets Officer" `
        --assignee "$_" `
        --scope "/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.KeyVault/vaults/$kvName"
    }

# Create an azure storage account - Terraform Backend Storage Account
az storage account create `
    --name "$storageName" `
    --location "$region" `
    --resource-group "$resourceGroupName" `
    --sku "Standard_LRS" `
    --kind "StorageV2" `
    --https-only true `
    --min-tls-version "TLS1_2"

# Authorize the operation to create the container - Signed in User (Storage Blob Data Contributor Role)
az ad signed-in-user show --query id -o tsv | foreach-object {
    az role assignment create `
        --role "Storage Blob Data Contributor" `
        --assignee "$_" `
        --scope "/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.Storage/storageAccounts/$storageName"
    }

#Create Upload container in storage account to store terraform state files
Start-Sleep -s 40
az storage container create `
    --account-name "$storageName" `
    --name "tfstate" `
    --auth-mode login

# Create Terraform Service Principal and assign RBAC Role on Key Vault
$spnJSON = az ad sp create-for-rbac --name $appName `
    --role "Key Vault Secrets Officer" `
    --scopes /subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.KeyVault/vaults/$kvName

# Save new Terraform Service Principal details to key vault
$spnObj = $spnJSON | ConvertFrom-Json
foreach($object_properties in $spnObj.psobject.properties) {
    If ($object_properties.Name -eq "appId") {
        $null = az keyvault secret set --vault-name $kvName --name "ARM-CLIENT-ID" --value $object_properties.Value
    }
    If ($object_properties.Name -eq "password") {
        $null = az keyvault secret set --vault-name $kvName --name "ARM-CLIENT-SECRET" --value $object_properties.Value
    }
    If ($object_properties.Name -eq "tenant") {
        $null = az keyvault secret set --vault-name $kvName --name "ARM-TENANT-ID" --value $object_properties.Value
    }
}
$null = az keyvault secret set --vault-name $kvName --name "ARM-SUBSCRIPTION-ID" --value $subscriptionId

# Assign additional RBAC role to Terraform Service Principal Subscription as Contributor and access to backend storage
az ad sp list --display-name $appName --query [].appId -o tsv | ForEach-Object {
    az role assignment create --assignee "$_" `
        --role "Contributor" `
        --subscription $subscriptionId

    az role assignment create --assignee "$_" `
        --role "Storage Blob Data Contributor" `
        --scope "/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.Storage/storageAccounts/$storageName" `
    }
```

Lets take a closer look, step-by-step what the above script does as part of setting up the Terraform backend environment.

1. Create a resource group called `Demo-Terraform-Core-Backend-RG`, containing an Azure key vault and storage account.
2. Create an **Entra ID App and Service Principal** that has access to the key vault, backend storage account, container and the subscription.
3. The **Entra ID App and Service Principal** details are saved inside the key vault.

**Note:** Instead of using an Entra ID App and Service Principal with a password, you can also look at the following guide for an example on how to integrate your GitHub Actions with Azure using a federated/passwordless service principal (identity): [GitHub Actions authentication methods for Azure](https://dev.to/pwd9000/bk-1iij).

### Step 2 - Create a GitHub Repository

At Contoso, we have created a [template repository](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments) that contains everything you need to get started with automating Terraform deployments to Azure using GitHub Actions. You can create your repository from this template by selecting `Use this template`. (Optional)

After creating the GitHub repository, there are a few configurations you need to set up before you can start using it:

1. **Add Repository Secrets**: Add the secrets that were created in the `Key Vault` step above into the newly created GitHub repository as **[Repository Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository)**. These secrets are essential for securely accessing your Azure resources.

2. **Set Up GitHub Environments (Optional)**: Create the following **[GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#creating-an-environment)**, or environments that match your own requirements. In our case, these environments are: `Development`, `UserAcceptanceTesting`, and `Production`. Setting up GitHub Environments is optional but recommended for demonstrating deployment approvals via **Protection Rules**.

**NOTE:** GitHub environments and Protection Rules are available on public repositories, but for private repositories, you will need GitHub Enterprise.

For the **Production** environment, we have configured a **Required Reviewer**. This allows us to set explicit reviewers who must physically approve deployments to the **Production** environment. To learn more about approvals, see [Environment Protection Rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#environment-protection-rules).

**NOTE:** You can also configure **GitHub Secrets** at the **Environment** scope if you have separate Service Principals or even separate Subscriptions in Azure for each environment. For example, your Development resources might be in Subscription A, while your Production resources are in Subscription B. See [Creating encrypted secrets for an environment](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-environment) for details.

By following these steps, you will be well on your way to automating your Terraform deployments to Azure using Contoso's bespoke CI/CD and GitHub Actions setup.

### Step 3 - Create Terraform Modules (Modular)

Now that our repository is all configured and ready to go, we can start creating some modular Terraform configurations. These configurations are separate, independent deployment setups based on ROOT Terraform modules. If you look at the [Demo Repository](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments), you will see that at the root of the repository, there are paths/folders that are numbered, e.g., **./01_Foundation** and **./02_Storage**.

Each of these paths contains a Terraform ROOT module, which consists of a **collection** of items that can be **independently** configured and deployed. You do not have to use the same naming/numbering convention as shown, but the idea is to understand that these paths/folders each represent a unique, independent modular Terraform configuration that consists of a collection of resources we want to deploy independently.

For example:

- **path:** `./01_Foundation` contains the Terraform ROOT module/configuration for an Azure Resource Group and Key Vault.
- **path:** `./02_Storage` contains the Terraform ROOT module/configuration for one General-V2 and one Data Lake V2 Storage account.

**NOTE:** You will also notice that each ROOT module contains three separate TFVARS files: `config-dev.tfvars`, `config-uat.tfvars`, and `config-prod.tfvars`. Each file represents a different environment. This is because each of our environments will use the same configuration file: `foundation_resources.tf`, but may have slightly different configuration values or naming conventions.

Example: The **Development** resource group name will be called `Demo-Infra-Dev-Rg`, whereas the **Production** resource group will be called `Demo-Infra-Prod-Rg`.

By organizing our Terraform configurations in this modular way, Contoso ensures that our infrastructure deployments are manageable, scalable, and adaptable to different environments, all while leveraging the power of GitHub Actions and CI/CD practices.

### Step 4 - Create GitHub Workflows using Marketplace Actions

At Contoso, we use GitHub Actions to automate our Terraform deployments to Azure. To get started, we need to create a folder/path called `.github/workflows` in the root of our repository. This folder will contain our [GitHub Action Workflows](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions).

If you are following this tutorial based on the demo project [template repository](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments), you will notice that this folder contains a `YAML` workflow file called [./.github/workflows/Marketplace_Example.yml].

Let's take a closer look at this workflow:

- **[Marketplace_Example.yml](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/blob/master/.github/workflows/Marketplace_Example.yml)**:

```yml
# I have created public github marketplace actions (plan and apply) as well that can be used as shown in this example.
#Plan: https://github.com/marketplace/actions/terraform-plan-for-azure
#Apply: https://github.com/marketplace/actions/terraform-apply-for-azure

name: 'Marketplace-Example'
on:
  workflow_dispatch:
  pull_request:
    branches:
      - master

jobs:
  ##### PLAN A DEPLOYMENT #####
  Plan_Dev_Deploy:
    runs-on: ubuntu-latest
    if: ${{ github.actor != 'dependabot[bot]' }}
    environment: null #(Optional) If using GitHub Environments
    steps:
      - name: Checkout
        uses: actions/checkout@v4.1.7

      - name: Dev TF Plan Deploy
        uses: Pwd9000-ML/terraform-azurerm-plan@v1.3.0
        with:
          path: 01_Foundation ## (Optional) Specify path TF module relevant to repo root. Default="."
          plan_mode: deploy ## (Optional) Specify plan mode. Valid options are "deploy" or "destroy". Default="deploy"
          tf_version: latest ## (Optional) Specifies version of Terraform to use. e.g: 1.1.0 Default="latest"
          tf_vars_file: config-dev.tfvars ## (Required) Specifies Terraform TFVARS file name inside module path.
          tf_key: foundation-dev ## (Required) AZ backend - Specifies name that will be given to terraform state file and plan artifact
          enable_TFSEC: true ## (Optional) Enable TFSEC IaC scans (Private repo requires GitHub enterprise). Default=false
          az_resource_group: TF-Core-Rg ## (Required) AZ backend - AZURE Resource Group hosting terraform backend storage acc
          az_storage_acc: tfcorebackendsa ## (Required) AZ backend - AZURE terraform backend storage acc
          az_container_name: ghdeploytfstate ## (Required) AZ backend - AZURE storage container hosting state files
          arm_client_id: ${{ secrets.ARM_CLIENT_ID }} ## (Required - Actions Secrets) ARM Client ID
          arm_client_secret: ${{ secrets.ARM_CLIENT_SECRET }} ## (Required - Actions Secrets) ARM Client Secret
          arm_subscription_id: ${{ secrets.ARM_SUBSCRIPTION_ID }} ## (Required - Actions Secrets) ARM Subscription ID
          arm_tenant_id: ${{ secrets.ARM_TENANT_ID }} ## (Required - Actions Secrets) ARM Tenant ID
          github_token: ${{ secrets.GITHUB_TOKEN }} ## (Required) Needed to comment output on PR's. ${{ secrets.GITHUB_TOKEN }} already has permissions

  ##### APPLY DEPLOY #####
  Apply_Dev_Deploy:
    needs: Plan_Dev_Deploy
    runs-on: ubuntu-latest
    environment: Development #(Optional) If using GitHub Environments
    steps:
      - name: Dev TF Deploy
        if: ${{ github.actor != 'dependabot[bot]' }}
        uses: Pwd9000-ML/terraform-azurerm-apply@v1.3.0
        with:
          plan_mode: deploy ## (Optional) Specify plan mode. Valid options are "deploy" or "destroy". Default="deploy"
          tf_version: latest ## (Optional) Specifies version of Terraform to use. e.g: 1.1.0 Default="latest"
          tf_key: foundation-dev ## (Required) Specifies name of the terraform state file and plan artifact to download
          az_resource_group: TF-Core-Rg ## (Required) AZ backend - AZURE Resource Group hosting terraform backend storage acc
          az_storage_acc: tfcorebackendsa ## (Required) AZ backend - AZURE terraform backend storage acc
          az_container_name: ghdeploytfstate ## (Required) AZ backend - AZURE storage container hosting state files
          arm_client_id: ${{ secrets.ARM_CLIENT_ID }} ## (Required - Actions Secrets) ARM Client ID
          arm_client_secret: ${{ secrets.ARM_CLIENT_SECRET }} ## (Required - Actions Secrets) ARM Client Secret
          arm_subscription_id: ${{ secrets.ARM_SUBSCRIPTION_ID }} ## (Required - Actions Secrets) ARM Subscription ID
          arm_tenant_id: ${{ secrets.ARM_TENANT_ID }} ## (Required - Actions Secrets) ARM Tenant ID

  ##### PLAN A DESTROY #####
  Plan_Dev_Destroy:
    needs: Apply_Dev_Deploy
    runs-on: ubuntu-latest
    if: ${{ github.actor != 'dependabot[bot]' }}
    environment: null #(Optional) If using GitHub Environments
    steps:
      - name: Checkout
        uses: actions/checkout@v4.1.7

      - name: Dev TF Plan Destroy
        uses: Pwd9000-ML/terraform-azurerm-plan@v1.3.0
        with:
          path: 01_Foundation ## (Optional) Specify path TF module relevant to repo root. Default="."
          plan_mode: destroy ## (Optional) Specify plan mode. Valid options are "deploy" or "destroy". Default="deploy"
          tf_version: latest ## (Optional) Specifies version of Terraform to use. e.g: 1.1.0 Default="latest"
          tf_vars_file: config-dev.tfvars ## (Required) Specifies Terraform TFVARS file name inside module path.
          tf_key: foundation-dev ## (Required) AZ backend - Specifies name that will be given to terraform state file and plan artifact
          enable_TFSEC: false ## (Optional) Enable TFSEC IaC scans (Private repo requires GitHub enterprise). Default=false
          az_resource_group: TF-Core-Rg ## (Required) AZ backend - AZURE Resource Group hosting terraform backend storage acc
          az_storage_acc: tfcorebackendsa ## (Required) AZ backend - AZURE terraform backend storage acc
          az_container_name: ghdeploytfstate ## (Required) AZ backend - AZURE storage container hosting state files
          arm_client_id: ${{ secrets.ARM_CLIENT_ID }} ## (Required - Actions Secrets) ARM Client ID
          arm_client_secret: ${{ secrets.ARM_CLIENT_SECRET }} ## (Required - Actions Secrets) ARM Client Secret
          arm_subscription_id: ${{ secrets.ARM_SUBSCRIPTION_ID }} ## (Required - Actions Secrets) ARM Subscription ID
          arm_tenant_id: ${{ secrets.ARM_TENANT_ID }} ## (Required - Actions Secrets) ARM Tenant ID
          github_token: ${{ secrets.GITHUB_TOKEN }} ## (Required) Needed to comment output on PR's. ${{ secrets.GITHUB_TOKEN }} already has permissions

  ##### APPLY DESTROY #####
  Apply_Dev_Destroy:
    needs: Plan_Dev_Destroy
    runs-on: ubuntu-latest
    environment: Development #(Optional) If using GitHub Environments
    steps:
      - name: Dev TF Destroy
        if: ${{ github.actor != 'dependabot[bot]' }}
        uses: Pwd9000-ML/terraform-azurerm-apply@v1.3.0
        with:
          plan_mode: destroy ## (Optional) Specify plan mode. Valid options are "deploy" or "destroy". Default="deploy"
          tf_version: latest ## (Optional) Specifies version of Terraform to use. e.g: 1.1.0 Default="latest"
          tf_key: foundation-dev ## (Required) Specifies name of the terraform state file and plan artifact to download
          az_resource_group: TF-Core-Rg ## (Required) AZ backend - AZURE Resource Group hosting terraform backend storage acc
          az_storage_acc: tfcorebackendsa ## (Required) AZ backend - AZURE terraform backend storage acc
          az_container_name: ghdeploytfstate ## (Required) AZ backend - AZURE storage container hosting state files
          arm_client_id: ${{ secrets.ARM_CLIENT_ID }} ## (Required - Actions Secrets) ARM Client ID
          arm_client_secret: ${{ secrets.ARM_CLIENT_SECRET }} ## (Required - Actions Secrets) ARM Client Secret
          arm_subscription_id: ${{ secrets.ARM_SUBSCRIPTION_ID }} ## (Required - Actions Secrets) ARM Subscription ID
          arm_tenant_id: ${{ secrets.ARM_TENANT_ID }} ## (Required - Actions Secrets) ARM Tenant ID
```

As you can see, this workflow has four `jobs`: `Plan_Dev_Deploy`, `Apply_Dev_Deploy`, `Plan_Dev_Destroy`, and `Apply_Dev_Destroy`. Each job calls the marketplace actions with `uses:` in a `steps:` argument.

You will also notice that the `APPLY` jobs: `Apply_Dev_Deploy` and `Apply_Dev_Destroy` have a special `needs:` argument. This means the apply job requires the plan job to successfully run first and create the Terraform plan. The apply job will use the `PLAN` created to perform the `APPLY`. The plan is uploaded into the workflow as an artifact, which will either contain a deployment plan called `deploy_plan.tfplan` if `plan_mode: "deploy"` is used, or a destroy plan called `destroy_plan.tfplan` if `plan_mode: "destroy"` is used. The apply job will download and apply the `PLAN` artifact from the workflow artifacts and run the relevant plan based on the `plan_mode`.

Additionally, on the `Apply` jobs, if you use **[GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#creating-an-environment)**, you can link the job using the `environment:` argument and apply approvals by using **[Environment Protection Rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#environment-protection-rules)**.

Input parameters are passed into each of the actions using the `with:` argument. Let's take a look at each action's available inputs.

---

## PLAN Action Inputs

**[Terraform Plan for AZURE](https://github.com/marketplace/actions/terraform-plan-for-azure)**

This action will connect to a remote Terraform backend in Azure, creates a terraform plan and uploads plan as a workflow artifact. (Additionally TFSEC IaC scanning can be enabled).

| Input | Required |Description |Default |
| ----- | -------- | ---------- | ------ |
| `path` | FALSE | Specify path to Terraform module relevant to repo root. | "." |
| `plan_mode` | FALSE | Specify plan mode. Valid options are `deploy` or `destroy`. | "deploy" |
| `tf_version` | FALSE | Specifies the Terraform version to use. | "latest" |
| `tf_vars_file` | TRUE | Specifies Terraform TFVARS file name inside module path. | N/A |
| `tf_key` | TRUE | AZ backend - Specifies name that will be given to terraform state file and plan artifact| N/A |
| `enable_TFSEC` | FALSE | Enable IaC TFSEC scan, results are posted to GitHub Project Security Tab. (Private repos require GitHub enterprise). | FALSE |
| `az_resource_group` | TRUE | AZ backend - AZURE Resource Group name hosting terraform backend storage account | N/A |
| `az_storage_acc` | TRUE | AZ backend - AZURE terraform backend storage account name | N/A |
| `az_container_name` | TRUE | AZ backend - AZURE storage container hosting state files  | N/A |
| `arm_client_id` | TRUE | The Azure Service Principal Client ID | N/A |
| `arm_client_secret` | TRUE | The Azure Service Principal Secret | N/A |
| `arm_subscription_id` | TRUE | The Azure Subscription ID | N/A |
| `arm_tenant_id` | TRUE | The Azure Service Principal Tenant ID | N/A |
| `github_token` | TRUE | Specify GITHUB TOKEN, only used in PRs to comment outputs such as `plan`, `fmt`, `init` and `validate`. `${{ secrets.GITHUB_TOKEN }}` already has permissions, but if using own token, ensure repo scope. | N/A |

In the example: **[Marketplace_Example.yml](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments/blob/master/.github/workflows/Marketplace_Example.yml)**, the Terraform plans will be created, compressed, and published to the workflow as artifacts using the same name as the inputs `[plan_mode]-[tf_key]`.

As mentioned earlier, the artifacts will either contain a deployment plan called `deploy_plan.tfplan` if `plan_mode: "deploy"` is used, or a destroy plan called `destroy_plan.tfplan` if `plan_mode: "destroy"` is used.

**NOTE:** If `enable_TFSEC` is set to `true` during the plan stage, Terraform IaC will be scanned using TFSEC, and the results will be published to the GitHub Project `Security` tab.

If you are using a private repository, GitHub Enterprise is needed when enabling TFSEC. However, if a public repository is used, code analysis is included, and TFSEC can be enabled on public repositories without the need for a GitHub Enterprise account.

Also, note that if the `PLAN` action is used in the context of a Pull Request (PR), the output will be added as a comment on the PR. This requires a valid GITHUB TOKEN as input with `github_token`. Additionally, failures on `fmt`, `init`, and `validate` will also be added to the PR.

---

## APPLY Action Inputs

**[Terraform Apply for AZURE](https://github.com/marketplace/actions/terraform-apply-for-azure)**

This action will download a Terraform plan workflow artifact created by `Pwd9000-ML/terraform-azurerm-plan` and apply with an AZURE backend configuration.

| Input | Required | Description | Default |
| --- | --- | --- | --- |
| `plan_mode` | FALSE | Specify plan mode. Valid options are `deploy` or `destroy`. | "deploy" |
| `tf_version` | FALSE | Specifies the Terraform version to use. | "latest" |
| `tf_key` | TRUE | Specifies name of the terraform state file and plan artifact to download | N/A |
| `az_resource_group` | TRUE | AZ backend - AZURE Resource Group name hosting terraform backend storage account | N/A |
| `az_storage_acc` | TRUE | AZ backend - AZURE terraform backend storage account name | N/A |
| `az_container_name` | TRUE | AZ backend - AZURE storage container hosting state files | N/A |
| `arm_client_id` | TRUE | The Azure Service Principal Client ID | N/A |
| `arm_client_secret` | TRUE | The Azure Service Principal Secret | N/A |
| `arm_subscription_id` | TRUE | The Azure Subscription ID | N/A |
| `arm_tenant_id` | TRUE | The Azure Service Principal Tenant ID | N/A |

The terraform apply action will download and apply the plan inside of the artifact created by the plan action using the same `[plan_mode]-[tf_key]` and apply the relevant plan based on which `plan_mode` was used in the creation of the plan artifact.

---

## Conclusion

That's all there is to it! By following the same pattern shown in this series, you can now further expand your **Terraform** deployments in a modular, structured, and non-monolithic way. Create more modules in separate paths, such as `./03_etc_etc`, and for each deployment, you can use Contoso's public **marketplace Actions** to plan and apply your configurations. If you prefer using **reusable workflows**, you can also see how to do that in [part 1](https://dev.to/pwd9000/multi-environment-azure-deployments-with-terraform-and-github-2450) of this series.

We hope you have enjoyed this post and learned something new. You can find the code samples used in this blog post on our [GitHub](https://github.com/Pwd9000-ML/blog-devto/tree/main/posts/2022/GitHub-Actions-Terraform-Deployment-Part1/code) page. You can also look at the demo project or even create your own projects and workflows from the demo project [template repository](https://github.com/Pwd9000-ML/Azure-Terraform-Deployments).

By leveraging GitHub Actions and CI/CD practices, Contoso ensures that our Terraform deployments to Azure are automated, efficient, and scalable.

---

## _Author_

Like, share, follow me on: :octopus: [GitHub](https://github.com/Pwd9000-ML) | :penguin: [X/Twitter](https://x.com/pwd9000) | :space_invader: [LinkedIn](https://www.linkedin.com/in/marcel-l-61b0a96b/)
