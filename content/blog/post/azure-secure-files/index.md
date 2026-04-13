---
title: "How to actually use Azure Secure Files with Terraform"
date: 2024-11-03T17:25:41+02:00
draft: false
tags: [azure, security, devops]
---

I could not find a real guide to use Azure secure files securely in combination with Terraform. All guides either used a PAT (Privileged Access Token) or in worst cases, commiting secrets to code.

# The problem:

We are trying to deploy Azure Devops Agent with Terraform to bootstrap our Ansible deployments. Kind of a chicken-egg issue is that how do you deploy the Azure Devops Agent runner to a linux machine without doing manual commands?

Most of the guides found are using PAT added to the code and repo (not very secure as they are printed in plaintext to extension logs!):
https://jamescook.dev/azuredevops-linux-agent-install-using-terraform or manually install & configure the agent after deploying the infra using terraform: https://opstree.com/blog/2022/08/30/how-to-setup-an-agent-on-azure-devops/

# Requirements:

- Secure, so no credentials or access tokens in code. Even if we are using private repos, we do not want to make exceptions for this!
- Automated, no manual steps other than starting the pipeline on deployments.
- Must run in private VNET:s as build machines will not have any inbound SSH possibilities (we use Azure Bastion).
- All code commited to git repo so that someone can edit the code, generate new secure file and deploy agents.

# Known issues:

- Previous Agent must be removed before building the agent, as Azure has no way to replace Agent with same name.

# Solution:

In the end, we went with a combination of:
- Azure Secure Files: [Link to MS Docs](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/secure-files?view=azure-devops)
- Terraform [Link to MS Docs for Terraform AzAPI](https://learn.microsoft.com/en-us/azure/developer/terraform/overview)
- Azure VM Extension [Link to Terraform docs for azurestack_virtual_machine_extension](https://registry.terraform.io/providers/hashicorp/azurestack/latest/docs/resources/virtual_machine_extension)

Quickly checking through the documentation, there are no instructions on how to replace a file in build directory using secure files (disclaimer: I'n not a coder :) ). Also no documentation is found on Azure VM Extensions and how to dynamically build the extensions so the following guide is very much a combination of hacks to make it work.

### Bitbucket repo for all code:

https://bitbucket.org/sami_nieminen/terraform-azure-secure-files

## Part 0: Folder setup

```
Repository                                                       
├─►terraform                                 
│  - setup_agent_script.sh                   
│  - main.tf                                 
│  - agent-vm.tf                             
│  - agent-vm.prod.tfvars                    
│  - providers.tf                            
│  - variables.tf                            
└┬►deploy                                    
 │ - devops-infrastructure-prod-terraform.yml
 └──►templates                               
     - terraform-template.yml 
```

## Part 1: Create a service principal

You must first link your Devops Organization to Azure Entra ID.

Then follow instructions from MS Docs to [Create an application service principal](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops#create-an-application-service-principal).

Then add the SPN as a "Project Administrator" to your devops project, [instructions from MS](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/service-principal-agent-registration?view=azure-devops).

Then also add the SPN as "Contributor" permissions to your Azure Resource group where you want to deploy the VM to.
If you dont want to precreate the 

TLDR: Create new Enterprise App, create new secret, add secret to your secrets manager to use later. Then add correct permissions to the SPN for your Devops project & organization.

## Part 2: Azure + Terraform link

Mostly following this [excellent guide from CarbonLogiQ](https://www.carbonlogiq.io/post/azure-devops-pipeline-terraform-deployment-tutorial).

To link Azure add file ```terraform/providers.tf```:
```
terraform {
  required_version = ">= 1.9"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~>3.0"
    }
    azapi = {
      source  = "azure/azapi"
      version = "~>1.5"
    }
  }
  backend "azurerm" {
    resource_group_name  = "devops-Infrastructure"
    storage_account_name = "tfstatedemostg"
    container_name       = "tfstate"
    key                  = "devops-Infrastructure.prod.tfstate"
  }
}

provider "azurerm" {
  features {}
  skip_provider_registration = "true"
}
```

You can use the same SPN you created previously.

1. Create a new resource group, devops-Infrastructure.
2. Create a new storage account, tfstatedemostg (accept all default configuration settings).
3. Create a tfstate blob container (stores Terraform state for deployments).

    These variable names are of special significance to Terraform. When set as environment variables within the ADO build agent, Terraform will automatically attempt to authenticate against Azure using their values.

4. Pipelines -> Environments -> Create new environment.
5. Name it how ever you want.
6. Add Approvals and Checks -> add yourself or groups as approved approver :)
7. Continue to pipeline setup.

## Part 3: Azure Pipeline setup.

Pipeline setup is quite simple, add the following code to repo and import the pipeline.

### 3.1: Pipeline variables:

We need to set up few variables before we can use the code:
1. Go to Pipelines -> Library -> Variable Groups
2. Create new variable group and name it "Terraform_SPN"
3. Add in details of your target deployment subscription.
  1. ARM_CLIENT_ID: SPN ID 
  2. ARM_CLIENT_SECRET: SPN Secret
  3. ARM_SUBSCRIPTION_ID: Your Azure Subscription
  4. ARM_TENANT_ID: 

### 3.2: Pipeline code:

Here we just set the general settings for pipeline.

Code to file ```deploy/devops-infrastructure-prod-terraform.yml```:
```
name: Terraform - deploy agent VM to prod

trigger: none

variables:
  - group: Terraform_SPN
  - name: rootFolder
    value: 'terraform/'
  - name: tfvarsFile
    value: 'agent-vm.prod.tfvars'
  - name: adoEnvironment
    value: 'prod'
  
stages:
- template: templates/terraform-template.yml
  parameters:
    rootFolder: $(rootFolder)
    tfvarsFile: $(tfvarsFile)
    adoEnvironment: $(adoEnvironment)
```

### 3.3: Pipeline template code:

Important steps for the usage of secure files are "DownloadSecureFile@1" and "CopyFiles@2".
What these do, is first tell the pipeline to download the secure file named "setup_agent_script.sh" to a variable "setupagentscript". Then we copy the file (in folder $(Agent.TempDirectory)) over the repository included "template" of that file, so we can later convert it to base64 and add it as extension.

Code to file ```deploy/templates/terraform-template.yml```:

```
parameters:
- name: rootFolder
  type: string
- name: tfvarsFile
  type: string
- name: adoEnvironment
  type: string

stages:
- stage: 'Terraform_Plan'
  displayName: 'Terraform Plan'
  jobs:
  - job: 'Terraform_Plan'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: DownloadSecureFile@1
      name: setupagentscript
      displayName: 'Download bash script securely'
      inputs:
        secureFile: 'setup_agent_script.sh'
    - task: CopyFiles@2
      displayName: "Import secure file to build root"
      inputs:
        SourceFolder: '$(Agent.TempDirectory)'
        Contents: 'setup_agent_script.sh'
        targetFolder: "$(System.DefaultWorkingDirectory)"
    - script: |
        echo "Running Terraform init..."
        terraform init
        echo "Running Terraform plan..."
        terraform plan -var-file ${{ parameters.tfvarsFile }}
      displayName: 'Terraform plan'
      workingDirectory: ${{ parameters.rootFolder }}
      env:
        ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET) # this needs to be explicitly set as it's a sensitive value

- stage: 'Terraform_Apply'
  displayName: 'Terraform Apply'
  dependsOn:
  - 'Terraform_Plan'
  condition: succeeded()
  jobs:
  - deployment: 'Terraform_Apply'
    pool:
      vmImage: 'ubuntu-latest'
    environment: ${{ parameters.adoEnvironment }} # using an ADO environment allows us to add a manual approval check
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
          - task: DownloadSecureFile@1
            name: setupagentscript
            displayName: 'Download bash script securely'
            inputs:
              secureFile: 'setup_agent_script.sh'
          - task: CopyFiles@2
            displayName: "Import secure file to build root"
            inputs:
              SourceFolder: '$(Agent.TempDirectory)'
              Contents: 'setup_agent_script.sh'
              targetFolder: "$(System.DefaultWorkingDirectory)"
          - script: |
              echo "Running Terraform init..."
              terraform init
              echo "Running Terraform apply..."
              terraform apply -var-file ${{ parameters.tfvarsFile }} -auto-approve
            displayName: 'Terraform apply'
            workingDirectory: ${{ parameters.rootFolder }}
            env:
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
```

Import pipeline:
1. Go to Project -> Pipelines -> Add Pipeline
2. Select Azure Repos GIT
3. Select "Existing Axure Pipelines YAML file"
4. Select your branch -> deploy/devops-infrastructure-prod-terraform.yml

## Part 4: Azure resource setup

To make Terraform agent deployments automated, you will need to setup a RSA4096 SSH key which will be used to setup the VM.
Then we need to import the RG + SSH key for Terraform. Otherwise Terraform would complain that it cannot overwrite existing resources.

### 4.1: Adding SSH key for the VM deployment

1. Go to Azure -> your RG -> Create
2. Search for SSH key
3. Import your own or create it via the guided setup.
4. Name it "az-agent-service" or something that indicates it is used for the VM.

### 4.2: Import resources to Terraform

This is done so that terraform knows about our SSH key and the Resource Group we are deploying into.
- Replace id = with correct URL for your RG
- Replace name = with your RG name

Code to file ```terraform/main.tf```:
```
# Create resource group if not imported
resource "azurerm_resource_group" "rg" {
  name     = "devops-Infrastructure"
  location = var.location
  tags = local.tags
}

import {
  id = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx/resourceGroups/devops-Infrastructure"
  to = azurerm_resource_group.rg
}
```

### 4.3 TF Environment setup & settings

This controls your environment tags, sets the VM name and what region we want to deploy.

File ```terraform/agent-vm.prod.tfvars```:
```
# Variables for env01
project              = "devops-infrastructure"
environment          = "prod"
location             = "northeurope"
vm-name              = "agent01"
vm-root-username     = "your-root-user"
```

This file controls our variables and sets the correct default Agent Extension filename.

File ```terraform/variables.tf```:
```
# Variable definitions
variable "project" {
  type        = string
  description = "Project name i.e. tfdemo"
}

variable "environment" {
  type        = string
  description = "Environment name i.e. env01, env02 etc."
}

variable "location" {
  type        = string
  description = "Location of the Azure resources i.e. uksouth "
}

variable "vm-name" {
  type        = string
  description = "Virtual machine name."
}

variable "vm-root-username" {
  type        = string
  description = "Virtual machine root username"
}

variable "scfile" {
  type = string
  description = "Secure file location"
  default = "../setup_agent_script.sh"
}

# Local variables
locals {
  tags = {
    Project     = var.project
    Environment = var.environment
  }
}
```

## Part 5: Azure VM Extension

This will be the code which is executed when the VM is provisioned.
It will install the agent and connect is using a [Service Principal credential](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/service-principal-agent-registration?view=azure-devops).

To install the Azure VSTS Agent completely automated, use the following script and replace:
- url: this must be your Azure Devops URL with the Devops organization included!
- ClientId: your Azure devops agent service credential (Enterprise App) Client ID.
- TenantID: your Azure subscription ID.
- ClientSecret: secret (credential) for the Devops agent service Enterprise App.
- pool: the Devops agent pool you want to add the agent to.
- projectName: name of your Devops project.
- your-root-user: replace with your precreated Agent linux service account, must have root permissions.

The script saved as ```setup-agent-script.sh```
```
#!/bin/sh

# Creates directory & download ADO agent install files

sudo mkdir -p /myagent
sudo wget -qN https://vstsagentpackage.azureedge.net/agent/3.246.0/vsts-agent-linux-x64-3.246.0.tar.gz -O /myagent/vsts-agent-linux-x64-3.246.0.tar.gz
sudo tar --overwrite -zxf /myagent/vsts-agent-linux-x64-3.246.0.tar.gz -C /myagent

# Unattended install

sudo su - your-root-user -c "
/myagent/config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "https://dev.azure.com/yourdevopsproject/" \
  --auth SP \
  --ClientId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx" \
  --tenantId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx" \
  --ClientSecret "stored_in_secret_files" \
  --pool "Agent01" \
  --projectName "yourprojectname" \
  --replace \
  --acceptTeeEula & wait $!"

#Configure as a service
cd /myagent/
sudo /myagent/svc.sh install your-root-user

#Start svc
sudo /myagent/svc.sh start
```

**!IMPORTANT!**
Do not commit this code with the details included as it contains the secret. Follow the next steps to actually use it in secure files.

## Part 6: Adding secure files.

Now that we have template code to run on the VM, we can add the juicy credentials to the code *locally* then upload the file (with credentials included) to Azure Devops Secure files.

1. Go to Project -> Pipelines -> Library -> Secure files.
2. Upload the files as "setup_agent_script.sh".
3. Add your pipeline to the "Pipeline permissions".

## Part 7: Combine everything!

Here is the main terraform code to set up one VM, in private VNET. After setting up the VM, we then add the VM extension from secure files which handles the Devops Agent installation.

Mind that this code sets up public IP for testing purposes and allows SSH from All, so make sure you are using this only for testing.

File ```terraform/agent-vm.tf```:
```
data "azurerm_ssh_public_key" "sshkey" {
  name                = "az-agent-service"
  resource_group_name = azurerm_resource_group.rg.name
}

# Define virtual network
resource "azurerm_virtual_network" "vnet" {
  name                = "${var.project}-${var.environment}-vnet"
  address_space       = ["10.100.6.0/23"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Define Subnet
resource "azurerm_subnet" "internal" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.100.6.0/24"]
}

resource "azurerm_public_ip" "public_ip" {
  name                = "${var.vm-name}-${var.environment}-ip"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  allocation_method   = "Dynamic"
}

resource "azurerm_network_interface" "nic" {
  name                = "${var.vm-name}-${var.environment}-nic"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.public_ip.id
  }
}

resource "azurerm_network_security_group" "nsg" {
  name                = "ssh_nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "allow_ssh_sg"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  #security_rule  {
  #  name                       = "allow_publicIP"
  #  priority                   = 103
  #  direction                  = "Inbound"
  #  access                     = "Allow"
  #  protocol                   = "Tcp"
  #  source_port_range          = "*"
  #  destination_port_range     = "80"
  #  source_address_prefix      = "*"
  #  destination_address_prefix = "*"
  #}
}

resource "azurerm_network_interface_security_group_association" "association" {
  network_interface_id      = azurerm_network_interface.nic.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                            = "${var.vm-name}-${var.environment}-vm"
  resource_group_name             = azurerm_resource_group.rg.name
  location                        = azurerm_resource_group.rg.location
  size                            = "Standard_D2s_v3"
  admin_username                  = "${var.vm-root-username}"
  disable_password_authentication = true
  network_interface_ids = [
    azurerm_network_interface.nic.id,
  ]
  patch_assessment_mode           = "AutomaticByPlatform"
  patch_mode                      = "AutomaticByPlatform"
  provision_vm_agent              = true
  reboot_setting                  = "IfRequired"
  secure_boot_enabled             = true

  admin_ssh_key {
    username   = "${var.vm-root-username}"
    public_key = data.azurerm_ssh_public_key.sshkey.public_key
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }
}

resource "azurerm_virtual_machine_extension" "vmext" {
  name                   = "update-vm"
  publisher              = "Microsoft.Azure.Extensions"
  type                   = "CustomScript"
  type_handler_version   = "2.1"
  virtual_machine_id     = azurerm_linux_virtual_machine.vm.id

  protected_settings = <<PROT
    {
        "script": "${base64encode(file(var.scfile))}"
    }
    PROT
}
```

## Results:

Your Code should look something like this:
{{< img src="code_overview.png" alt="Repo overview of files." >}}

Now run the pipeline:
1. Go to Pipelines -> Pipeline
2. Select your imported pipeline -> Run pipeline

It will first run "Terraform plan" part of the pipeline, which checks what is already set up, what needs to be set up and in the end you will have a clear output of what happens when you run "Terraform Apply".
{{< img src="terraform_plan.png" alt="Results of terraform Plan" >}}

Now when you have plan ready, you will need to approve the results before the Apply part will be ran. This makes it easy to Dev using the pipeline, before sending it to prod ;)
{{< img src="terraform_apply.png" alt="Sending it to prod using Terraform apply." >}}

After pressing apply, waiting for terraform to finish, you should see something like this in your Azure RG:
{{< img src="azure_rg.png" alt="Overview of Azure Resource Group." >}}

Also you should have a working agent in your agent pool!
Go to Project Settings -> Pipelines -> Agent pools and check that the agent is registered OK.
{{< img src="agent_pool.png" alt="Overview of agent pool & agent." >}}