# FitApp — Azure Infrastructure (Terraform, Modular)

Infrastructure-as-Code for a containerized **FastAPI app** on Azure.

## Components
- **Azure Container App** + Container Apps Environment  
- **Azure Container Registry (ACR)**  
- **PostgreSQL Flexible Server**  
- **User-Assigned Managed Identity (UAMI)**  
- **Log Analytics**  
- **Resource Group**  

The app receives its database connection via the secret `DB_URL` and listens on `PORT` (default **8030**).

---

## 📑 Contents
- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Modules](#modules)
- [Input Variables](#input-variables)
- [Getting Started](#getting-started-any-azure-subscription)
- [Common Terraform Commands](#common-terraform-commands)
- [Best Practices Implemented](#best-practices-implemented)
- [Reusing in Other Environments](#reusing-in-other-environments)
- [Advanced Next Steps](#advanced-next-steps)

---

## Overview
- Container image is stored in **ACR**.  
- App runs in **Azure Container Apps** (public ingress to a target port).  
- **PostgreSQL Flexible Server** provides the managed DB; a psycopg-style `db_url` is output.  
- **UAMI** pulls images from ACR (AcrPull role; no admin creds).  
- **Log Analytics** captures Container App diagnostics and logs.  

---

## Repository Structure
```bash
terraform/
├─ versions.tf                 # provider constraints (azurerm, random)
├─ variables.tf                # root vars (prefix, env, image, sizing, pg, ingress)
├─ main.tf                     # wires modules + grants AcrPull to UAMI
├─ outputs.tf                  # exported values (URLs, ACR server, etc.)
├─ backend.tf                  # optional: Azure Blob remote state + locking
├─ terraform.tfvars.example    # sample values for quick start
└─ modules/
   ├─ resource_group/          # Resource Group
   ├─ log_analytics/           # Log Analytics Workspace
   ├─ acr/                     # Azure Container Registry
   ├─ managed_identity/        # User-Assigned Managed Identity
   ├─ container_apps_env/      # Container Apps Environment (CAE)
   ├─ postgres/                # PostgreSQL Flexible Server + DB + db_url output
   └─ container_app/           # Container App (ingress, scaling, secrets, env)
```

---

## Modules

- **resource_group** – creates the RG for all resources.  
- **log_analytics** – workspace used by Container Apps for logs/metrics.  
- **acr** – private container registry (admin disabled; UAMI pulls).  
- **managed_identity** – UAMI; root config grants AcrPull on ACR.  
- **container_apps_env** – Azure Container Apps Environment (diagnostics boundary).  
- **postgres** – PostgreSQL Flexible Server + database; outputs ready-to-use `db_url`.  
- **container_app** – deploys the API container, sets ingress, injects secrets/env, routes traffic.  

## Input Variables

See `variables.tf` and `terraform.tfvars.example`.

| Name                 | Type    | Default             | Description |
|----------------------|---------|---------------------|-------------|
| prefix               | string  | n/a                 | Name prefix (e.g., `fitapp`) |
| environment          | string  | n/a                 | Environment (e.g., `dev`, `stg`, `prod`) |
| location             | string  | eastus              | Azure region |
| container_image_name | string  | n/a                 | ACR repo name |
| container_image_tag  | string  | latest              | Image tag to deploy |
| container_cpu        | number  | 0.5                 | vCPU per replica |
| container_memory     | string  | 1.0Gi               | Memory per replica |
| min_replicas         | number  | 1                   | Minimum replicas |
| max_replicas         | number  | 1                   | Maximum replicas |
| app_port             | number  | 8030                | Ingress target port (must match app’s PORT) |
| pg_version           | string  | 16                  | PostgreSQL major version |
| pg_sku_name          | string  | B_Standard_B1ms     | PostgreSQL SKU |
| pg_storage_mb        | number  | 32768               | PostgreSQL storage (MB) |
| pg_admin_login       | string  | fitappadmin         | PostgreSQL admin username |
| pg_admin_password    | string? | null (auto-random)  | Admin password (sensitive) |
| pg_public_access     | bool    | true                | Enable public access (disable for private networking) |
| tags                 | map     | {}                  | Common tags |

## Getting Started (any Azure subscription)

### 0) Prerequisites
- Terraform **1.5+**  
- Azure CLI (`az login`)  
- Docker  

---

### 1) Authenticate & Select Subscription
```bash
az login
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"

### 2) (Recommended) Remote State with Locking

```bash
RG=tfstate-rg
LOC=eastus
SA=stfitapptfstate$RANDOM$RANDOM

az group create -n $RG -l $LOC
az storage account create -g $RG -n $SA -l $LOC --sku Standard_LRS --kind StorageV2
az storage container create --name tfstate --account-name $SA

Configure backend.tf:

```bash
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "<yourStorageName>"
    container_name       = "tfstate"
    key                  = "containerapp/dev/terraform.tfstate"
    use_azuread_auth     = true
  }
}
```


Initialize:

```bash
terraform init -reconfigure
```

3) Configure inputs

```bash
cp terraform.tfvars.example terraform.tfvars
# edit: prefix, environment, location, image name/tag, sizing, pg settings
```

4) Plan and apply
```bash
terraform validate
terraform plan -out plan.bin
terraform apply plan.bin
```

6) Build and push the image to ACR
```bash   
ACR=$(terraform output -raw acr_login_server)
IMG=<your-image-repo>     # e.g., fitapp-api
TAG=<your-tag>            # e.g., v1 or latest

az acr login --name ${ACR%%.*}
docker build -t $ACR/$IMG:$TAG .
docker push $ACR/$IMG:$TAG
```

We have created the Github actions pipeline to deploy the application to Azure container Apps. 
Refer to this URL - https://github.com/venkateshdevops0808/H-N-Product-Catalog.git

If you changed the tag, update container_image_tag and:

```bash   
terraform apply -auto-approve
```


7) Common Terraform Commands:

```bash 
terraform fmt -recursive
terraform validate
terraform init
terraform plan -out plan.bin
terraform show -json plan.bin > plan.json
terraform apply plan.bin
terraform destroy
```


State helpers:

```bash 
terraform state list
terraform state show <addr>
terraform state pull > terraform.tfstate   # read-only snapshot (do not commit)
```


Recommended .gitignore:


```bash 
.terraform/
*.tfstate
*.tfstate.*
plan.bin
terraform.tfvars
backend.hcl
```

## Best Practices Implemented

- Modular, single-responsibility modules with typed inputs/outputs.

- Managed Identity for ACR pulls (AcrPull; no admin creds).

- Secrets hygiene: DB password marked sensitive; app consumes DB_URL via secret.

- Remote state and locking with Azure Blob.

- Provider pinning via versions.tf and committed .terraform.lock.hcl.

- Consistent naming/tagging using prefix + environment and local.tags.

- Idempotent, predictable resource names; safe re-apply.

## Reusing in Other Environments

- Change prefix and environment (e.g., fitapp-stg, fitapp-prod).

- Use a different backend key per env (e.g., containerapp/stg/terraform.tfstate).

- Switch subscription with az account set and re-initialize (terraform init -reconfigure).

- Push the app image to the new ACR and set container_image_tag.

## Advanced Next Steps

- Private networking: VNet + delegated subnets; attach CAE; disable PG public access.
- AKS : If we have multiple microservices, we can create K8s cluster to manage complete services. As it is for a demo purpose, I choose Azure container apps
- Autoscaling (KEDA): scale on HTTP concurrency, CPU/memory, or custom metrics.
- Blue/Green & canary: multiple revisions with traffic splits.
- Policy & guardrails: Azure Policy, tflint, checkov.




