# Private AKS Cluster

Deploying private infrastructure for Azure Kubernetes Services (AKS). You can execute this locally or using Azure Pipelines.

## Azure Pipelines Setup

* Create AKS Variable group **aks_dev_vars**. Replace variables with your own preferred values, also check for all **<replace-me>** values and update them accordingly. 

* Create ADO Variable group **aks_dev_vars**. Replace variables with your own preferred values, also check for all **<replace-me>** values and update them accordingly. Create AKS Infra Pipeline [from CLI](https://docs.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline-cli). See [here](/infra/private-aks/azure-pipelines.sh) for full script.
```bash
#!/bin/bash

# Adding Azure DevOps Extension
az extension add --name azure-devops

#################################################
################# Variables #####################
#################################################

ORG_URL='https://dev.azure.com/<your-organization-name>'
PROJECT_NAME='<your-project-name>'

# Environment Name: Most common examples: DEV, ACC, PRD, QA (Abbreviate if possible)
environment='dev'

# Azure region / location. Modify as needed.
location='eastus2' 

# Prefix for all your resources. Use alphanumeric characters only. Avoid special characters. Ex. ado001
# Ex. For resource group: <prefix>-<environment>-rg
prefix='aks001'

# AKS Version
aks_version='1.15.7'

# AKS Service Principal Client ID (AppId)
aks_service_principal_client_id='<replace-me>'

# AKS Service Principal Object ID
aks_service_principal_id='<replace-me>'

# AKS Service Principal Client Secret
aks_service_principal_client_secret='<replace-me>'

# Resource Group Name for ADO Self-Hosted Agent for Private Endpoint
pe_rg_name='<replace-me>'

# Virtual Network Name for ADO Self-Hosted Agent for Private Endpoint
pe_vnet_name='<replace-me>'

# Subnet name for ADO Self-Hosted Agent for Private Endpoint
pe_subnet_name='<replace-me>'

# Private Endpoint Request
pe_is_manual_connection=false
pe_request_message=''

# ADO variable group name - if you change this name you will need to change azure-pipelines.yml file.
ado_var_group_name='aks_dev_vars'

#################################################
################### Setup #######################
#################################################

# Make sure your Azure DevOps defaults include the organization and project from the command prompt
az devops configure --defaults organization=$ORG_URL project=$PROJECT_NAME

# Sign in to the Azure CLI
az login

# Create ADO Variable group with non-secret variables
az pipelines variable-group create \
--name $ado_var_group_name \
--authorize true \
--variables \
environment=$environment \
location=$location \
prefix=$prefix \
aks_version=$aks_version \
aks_service_principal_client_id=$aks_service_principal_client_id \
aks_service_principal_id=$aks_service_principal_id \
pe_rg_name=$pe_rg_name \
pe_vnet_name=$pe_vnet_name \
pe_subnet_name=$pe_subnet_name \
pe_is_manual_connection=$pe_is_manual_connection \
pe_request_message=$pe_request_message \
aks_name='$(prefix)-$(environment)-aks' \
resource_group='$(prefix)-$(environment)-rg' \
vnet_name='$(prefix)-$(environment)-vnet' \
acr_name='$(prefix)$(environment)acr' \
acr='$(acr_name).azurecr.io' \
storagekey='PipelineWillGetThisValueRuntime' \
terraformstorageaccount='tf$(prefix)$(environment)sa' \
terraformstoragerg='tf-$(prefix)-$(environment)-rg' 

# Create Variable Secrets
VAR_GROUP_ID=$(az pipelines variable-group list --group-name aks_dev_vars --top 1 --query "[0].id" -o tsv)
az pipelines variable-group variable create \
--group-id $VAR_GROUP_ID \
--secret true \
--name 'aks_service_principal_client_secret' \
--value $aks_service_principal_client_secret

# Create Infra Pipeline
az pipelines create --name 'Private.AKS.Infra.CI.CD' --yaml-path '/infra/private-aks/azure-pipelines.yml' --repository private-aks-app --repository-type tfsgit --branch master

# Create Application Pipeline
az pipelines create --name 'Private.AKS.App.CI.CD' --yaml-path '/app/azure-pipelines.yml' --repository private-aks-app --repository-type tfsgit --branch master

```

## Running with Terraform locally

* Copy and paste file **terraform.tfvars** and name the new file **terraform.auto.tfvar** use this new file to set your local variables values. Terraform will use this file instead for local executions.
* Comment 'backend "azurerm" {}' line inside **versions.tf** (line 9). You can use Azure CLI authentication locally.
* Run the following commands.

```bash
# Set Working Directory
cd /infra/private-aks

# Login into Azure
az login 

# Run Terraform 
terraform init
terraform plan
terraform apply 
```