# Deploy Azure VM using Terraform and GitHub Actions (CI/CD)

This guide provides a clear, step-by-step approach to deploying an Azure Virtual Machine (VM) using Terraform and automating the process with GitHub Actions.

---

##  Prerequisites

* Azure account with required permissions
* Azure CLI installed and configured
* GitHub account and a new repository created
* Terraform installed (v1.6.6 or compatible)
* SSH key pair generated

---

##  Azure Setup Steps

1. **Create a Resource Group for Backend Storage**:

```bash
az group create --name tfstate-rg --location eastus
```

2. **Create Storage Account** (Name must be globally unique, lowercase, 3-24 chars):

```bash
az storage account create --name prathyusha2025tfstate --resource-group tfstate-rg --location eastus --sku Standard_LRS --encryption-services blob
```

3. **Create Storage Container**:

```bash
az storage container create --name tfstate --account-name prathyusha2025tfstate
```

4. **Create a Service Principal**:

```bash
az ad sp create-for-rbac --name terraform-sp --role="Contributor" --scopes="/subscriptions/<your-subscription-id>" --sdk-auth
```

> Save the output JSON. It includes:
>
> * clientId
> * clientSecret
> * subscriptionId
> * tenantId

5. **Add GitHub Secrets**:
   In your GitHub repository, go to `Settings > Secrets and variables > Actions` and add the following:

* `ARM_CLIENT_ID`
* `ARM_CLIENT_SECRET`
* `ARM_SUBSCRIPTION_ID`
* `ARM_TENANT_ID`
* `SSH_PUBLIC_KEY` (single-line format)

---

##  Terraform Files Overview

```
azure-vm/
├── .github/
│   └── workflows/
│       └── azurevm.yml       # GitHub Actions workflow
├── backend.tf                  # Backend config for remote state
├── resource_group.tf          # Azure resource group
├── network.tf                 # Virtual network
├── subnet.tf                  # Subnet
├── public_ip.tf               # Public IP for VM
├── network_interface.tf       # NIC
├── vm.tf                      # Virtual Machine config
├── variables.tf               # Input variable definitions
├── terraform.tfvars           # Variable values
└── outputs.tf                 # Output variables
```

---

## GitHub Actions Workflow: `.github/workflows/azurevm.yml`

```yaml
name: 'Terraform Deploy Azure VM'

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  TF_VERSION: '1.6.6'
  ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
  ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
  ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
  SSH_PUBLIC_KEY: ${{ secrets.SSH_PUBLIC_KEY }}

jobs:
  deploy:
    name: 'Terraform Apply'
    runs-on: ubuntu-latest

    steps:
      - name: 'Checkout Repository'
        uses: actions/checkout@v4

      - name: 'Set up Terraform'
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: 'Inject SSH Key into override tfvars'
        run: |
          echo "admin_ssh_public_key = \"${{ env.SSH_PUBLIC_KEY }}\"" > override.tfvars

      - name: 'Terraform Init'
        run: terraform init

      - name: 'Terraform Validate'
        run: terraform validate

      - name: 'Terraform Plan'
        run: terraform plan -var-file="override.tfvars" -out=tfplan

      - name: 'Terraform Apply'
        run: terraform apply -auto-approve tfplan
```

---

##  Notes

* Ensure `admin_ssh_public_key` is in one-line format.
* To avoid errors like "resource already exists", destroy manually created Azure resources or import them to state.
* You can trigger deployments manually using the `workflow_dispatch` trigger.

---

##  Outcome

Once the pipeline is successful, a VM will be created with your SSH public key, and output values (like public IP) will be shown.

>  **You’ve automated Azure VM deployment using Terraform and GitHub Actions!**
