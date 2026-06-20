# Azure VM Provisioning with Terraform and Azure DevOps

This project provisions Azure infrastructure using Terraform and automates the workflow with an Azure DevOps YAML pipeline.

The pipeline can run Terraform `plan`, `apply`, and `destroy` actions from Azure DevOps. It also stores Terraform state in an Azure Storage Account so the state is not lost when the Azure DevOps build agent is removed after a pipeline run.

## Project Overview

This project creates the following Azure resources:

- Resource Group
- Virtual Network
- Subnet
- Public IP Address
- Network Interface
- Windows Virtual Machine
- Azure Storage backend for Terraform state

## Repository Structure

```text
azure-vm-terraform/
├── azure-pipelines.yml
└── infra/
    ├── main.tf
    ├── outputs.tf
    ├── provider.tf
    └── variables.tf
```

## How the Project Works

The Azure DevOps pipeline performs these steps:

1. Installs Terraform on the build agent.
2. Reads Azure credentials and backend configuration from an Azure DevOps Variable Group.
3. Validates that all required variables are available.
4. Logs in to Azure using a Service Principal.
5. Creates the Terraform state resource group, storage account, and container if they do not already exist.
6. Runs `terraform init` using the Azure Storage backend.
7. Runs `terraform validate`.
8. Runs one of the selected Terraform actions:
   - `plan`
   - `apply`
   - `destroy`

## Azure DevOps to Azure Connection

The pipeline connects Azure DevOps to Azure using a Service Principal.

### Create a Service Principal

Create the Service Principal from the Azure Portal UI:

1. Open the Azure Portal.
2. Search for `Microsoft Entra ID`.
3. Go to `App registrations`.
4. Select `New registration`.
5. Enter a name, for example `terraform-sp`.
6. Keep the supported account type as `Accounts in this organizational directory only`.
7. Select `Register`.

After registration, copy these values from the `Overview` page:

- `Application (client) ID` -> `ARM_CLIENT_ID`
- `Directory (tenant) ID` -> `ARM_TENANT_ID`

Create a client secret:

1. Open the created App Registration.
2. Go to `Certificates & secrets`.
3. Select `New client secret`.
4. Add a description, for example `terraform-secret`.
5. Choose an expiry period.
6. Select `Add`.
7. Copy the secret `Value` immediately.

Use the secret value as:

- `Client secret Value` -> `ARM_CLIENT_SECRET`

Give the Service Principal access to the Azure subscription:

1. Open `Subscriptions` in the Azure Portal.
2. Select the subscription where resources should be created.
3. Go to `Access control (IAM)`.
4. Select `Add` -> `Add role assignment`.
5. Choose the `Contributor` role.
6. Select `User, group, or service principal`.
7. Search for the App Registration name, for example `terraform-sp`.
8. Select it and complete the role assignment.

Copy the subscription ID from the subscription `Overview` page and use it as:

- `Subscription ID` -> `ARM_SUBSCRIPTION_ID`

## Azure DevOps Variable Group

Create one Variable Group in Azure DevOps. Example name:

```text
terraform-azure-creds
```

Add the following variables:

| Variable Name | Description |
| --- | --- |
| `ARM_CLIENT_ID` | Service Principal Application Client ID |
| `ARM_CLIENT_SECRET` | Service Principal Client Secret Value |
| `ARM_SUBSCRIPTION_ID` | Azure Subscription ID |
| `ARM_TENANT_ID` | Azure Tenant ID |
| `TFSTATE_RESOURCE_GROUP` | Resource group for Terraform state storage |
| `TFSTATE_LOCATION` | Azure region for Terraform state storage |
| `TFSTATE_STORAGE_ACCOUNT` | Storage account name for Terraform state |
| `TFSTATE_CONTAINER` | Blob container name for Terraform state |
| `TFSTATE_KEY` | Terraform state file name |

Example values:

```text
TFSTATE_RESOURCE_GROUP = tfstate-rg
TFSTATE_LOCATION = centralindia
TFSTATE_STORAGE_ACCOUNT = tfstatebhargavi001
TFSTATE_CONTAINER = tfstate
TFSTATE_KEY = azure-vm-terraform.tfstate
```

Mark `ARM_CLIENT_SECRET` as a secret variable.

Storage account naming rules:

- Use only lowercase letters and numbers.
- Do not use hyphens or underscores.
- Use 3 to 24 characters.
- The name must be globally unique across Azure.

## Terraform Remote State

Terraform state is stored in Azure Blob Storage.

The pipeline creates the backend storage resources before running `terraform init`.

The state file is stored in this format:

```text
Resource Group:  TFSTATE_RESOURCE_GROUP
Storage Account: TFSTATE_STORAGE_ACCOUNT
Container:       TFSTATE_CONTAINER
State File:      TFSTATE_KEY
```

Using remote state is important because Azure DevOps hosted agents are temporary. Without remote state, Terraform state would be stored only on the build agent and lost after the run.

## Azure DevOps to GitHub Connection

This project can be stored in GitHub and connected to Azure DevOps Pipelines.

### Steps to Connect GitHub Repository to Azure DevOps

1. Push this project to a GitHub repository.
2. Open Azure DevOps.
3. Go to Pipelines.
4. Select New Pipeline.
5. Choose GitHub as the source.
6. Sign in to GitHub if prompted.
7. Select this repository.
8. Choose Existing Azure Pipelines YAML file.
9. Select:

```text
/azure-pipelines.yml
```

10. Save and run the pipeline.

If Azure DevOps asks for permission, authorize Azure Pipelines to access the GitHub repository.

## Pipeline Parameters

The pipeline has these runtime parameters:

| Parameter | Description |
| --- | --- |
| `terraformAction` | Selects `plan`, `apply`, or `destroy` |
| `variableGroupName` | Azure DevOps Variable Group name |

Recommended first run:

```text
terraformAction = plan
variableGroupName = terraform-azure-creds
```

After reviewing the plan, run:

```text
terraformAction = apply
```

To remove the infrastructure, run:

```text
terraformAction = destroy
```

## Pipeline Flow

```text
GitHub Repository
        |
        v
Azure DevOps Pipeline
        |
        v
Azure DevOps Variable Group
        |
        v
Azure Service Principal Authentication
        |
        v
Terraform Init with Azure Storage Backend
        |
        v
Terraform Plan / Apply / Destroy
        |
        v
Azure Infrastructure
```

## Important Notes

- Always run `plan` before `apply`.
- Do not commit secrets to GitHub.
- Store secrets only in Azure DevOps Variable Groups or Azure Key Vault.
- The Service Principal needs enough permissions on the Azure subscription, such as Contributor.
- The Terraform state storage account name must be unique across Azure.

## Security Improvement

The current VM admin password is defined in Terraform code for learning purposes. For real projects, move passwords and sensitive values into secure variables, Azure Key Vault, or secret pipeline variables.

## Technologies Used

- Terraform
- Azure DevOps Pipelines
- Azure Virtual Machines
- Azure Storage Account
- Azure Blob Storage
- Azure Service Principal
- GitHub
