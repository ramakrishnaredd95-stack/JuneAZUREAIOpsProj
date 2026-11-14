# Azure DevOps + AIOps deployment runbook

This runbook records the repeatable process for validating, deploying, operating,
destroying, and rebuilding the Telugu Pickles Azure platform.

## 1. Architecture and deployment order

The active platform is Azure-only:

1. Local tests validate the React frontend, Node.js services, Python AIOps API,
   Terraform, and Kubernetes manifests.
2. Bootstrap creates Azure Blob Terraform state and three passwordless GitHub
   OIDC identities.
3. Foundation Terraform creates the VNet, ACR, AKS, monitoring workspace, and
   Azure RBAC assignments.
4. Platform Terraform installs ingress-nginx, Argo CD, Argo Rollouts,
   Prometheus, Grafana, application secrets, and Azure OpenAI AIOps resources.
5. Azure App CI builds eight images, pushes commit-SHA tags to ACR, and commits
   those tags to the Azure GitOps overlays.
6. Argo CD watches `gitops/overlays/azure` and reconciles AKS from Git.

Do not deploy the platform before the foundation. Kubernetes and Helm providers
cannot connect until AKS exists and Azure RBAC has propagated.

## 2. Prerequisites

- Azure CLI authenticated to the intended subscription
- Terraform, kubectl, kubelogin, Git, Node.js, Python, and Docker Desktop
- Azure permissions to create resource groups and role assignments
- GitHub repository access

Verify Azure context:

```powershell
az login
az account list --output table
az account set --subscription "<subscription name or ID>"
az account show --output table
```

## 3. Local code validation

### Terraform

```powershell
cd projects\AzureInfrastructure
terraform init -backend=false
terraform fmt -check -recursive
terraform validate

cd bootstrap
terraform init
terraform fmt -check -recursive
terraform validate
```

### Application tests

```powershell
cd projects\telugu-pickles\services
npm ci --no-audit --no-fund
npm test

cd ..\frontend
npm ci --no-audit --no-fund
npm run build

cd ..\..\aiops-assistant
python -m pip install -r requirements-dev.txt
python -m unittest discover -s tests -v
```

### Kubernetes and Compose rendering

```powershell
kubectl kustomize gitops/overlays/azure > $null
kubectl kustomize gitops/overlays/azure-no-aiops > $null
docker compose -f projects/telugu-pickles/docker-compose.yml config --quiet
```

Docker image builds require the Docker Desktop Linux engine to be running.

## 4. Bootstrap Azure state and GitHub OIDC

Bootstrap state is local and must be preserved privately. It manages the remote
state account and GitHub identities.

```powershell
cd projects\AzureInfrastructure\bootstrap
terraform init
$subscriptionId = az account show --query id -o tsv

& 'C:\Terraform\terraform.exe' `
  'plan' `
  "-var=subscription_id=$subscriptionId" `
  '-out=bootstrap.tfplan'

& 'C:\Terraform\terraform.exe' 'apply' '.\bootstrap.tfplan'
terraform output
```

Bootstrap creates:

- `telugu-pickles-tfstate` resource group
- versioned Azure Blob state with soft deletion
- a `CanNotDelete` storage-account lock
- `telugu-pickles-azure` application resource group
- separate Terraform, image-CI, and release managed identities
- GitHub OIDC credentials with no stored Azure client secret

Back up `bootstrap/terraform.tfstate` in encrypted private storage. Never commit
state, plan, variable, kubeconfig, environment, or credential files.

## 5. Configure GitHub Actions

Read authoritative values from bootstrap:

```powershell
cd projects\AzureInfrastructure\bootstrap
terraform output
```

Repository secrets:

| GitHub secret | Terraform output |
|---|---|
| `AZURE_TERRAFORM_CLIENT_ID` | `azure_terraform_client_id` |
| `AZURE_CI_CLIENT_ID` | `azure_ci_client_id` |
| `AZURE_RELEASE_CLIENT_ID` | `azure_release_client_id` |
| `AZURE_TENANT_ID` | `azure_tenant_id` |
| `AZURE_SUBSCRIPTION_ID` | `azure_subscription_id` |

Repository variables:

| GitHub variable | Value |
|---|---|
| `AZURE_TFSTATE_RESOURCE_GROUP` | `state_resource_group` output |
| `AZURE_TFSTATE_STORAGE_ACCOUNT` | `state_storage_account` output |
| `AZURE_TFSTATE_CONTAINER` | `state_container` output |
| `AZURE_RESOURCE_GROUP` | `app_resource_group` output |
| `AZURE_AKS_CLUSTER` | `telugu-pickles-aks` |
| `AZURE_LOCATION` | `centralindia` |
| `AZURE_AKS_ADMIN_PRINCIPAL_IDS` | JSON list containing `operator_object_id` |

Create a protected GitHub environment called `production`, preferably with a
required reviewer. Enable repository workflow read/write permission so Azure App
CI can commit immutable image tags into the GitOps overlays.

## 6. Initialize the root remote backend

Use explicit quoted arguments in PowerShell. This avoids Windows native-argument
parsing errors such as `Too many command line arguments`.

```powershell
cd projects\AzureInfrastructure

$stateRg = terraform -chdir=bootstrap output -raw state_resource_group
$stateAccount = terraform -chdir=bootstrap output -raw state_storage_account
$stateContainer = terraform -chdir=bootstrap output -raw state_container

& 'C:\Terraform\terraform.exe' `
  'init' `
  '-reconfigure' `
  "-backend-config=resource_group_name=$stateRg" `
  "-backend-config=storage_account_name=$stateAccount" `
  "-backend-config=container_name=$stateContainer" `
  '-backend-config=key=telugu-pickles/azure.tfstate' `
  '-backend-config=use_azuread_auth=true'
```

If `terraform init` asks interactively for `container_name`, cancel with
Ctrl+C and rerun the complete backend-config command.

## 7. Configure local deployment values

Copy `terraform.tfvars.example` to the ignored `terraform.tfvars` and set the
subscription and operator values. Recommended quota-safe AKS values are:

```hcl
node_vm_size   = "Standard_D2s_v3"
node_count     = 2
node_min_count = 2
node_max_count = 3
```

Get the current operator object ID:

```powershell
az ad signed-in-user show --query id -o tsv
```

Set it as a JSON/HCL list:

```hcl
aks_admin_principal_ids = ["<operator object ID>"]
```

## 8. Foundation plan and apply

Use the staged foundation plan only for initial creation or recovery:

```powershell
& 'C:\Terraform\terraform.exe' `
  'plan' `
  '-out=foundation.tfplan' `
  '-target=module.network' `
  '-target=module.registry' `
  '-target=module.aks' `
  '-target=azurerm_role_assignment.acr_pull' `
  '-target=azurerm_role_assignment.terraform_cluster_user' `
  '-target=azurerm_role_assignment.terraform_rbac_admin' `
  '-target=azurerm_role_assignment.release_cluster_user' `
  '-target=azurerm_role_assignment.release_rbac_writer' `
  '-target=azurerm_role_assignment.operator_cluster_user' `
  '-target=azurerm_role_assignment.operator_rbac_admin'

& 'C:\Terraform\terraform.exe' 'apply' '.\foundation.tfplan'
```

The target warning is expected for this one staged bootstrap. Do not use targeted
plans for normal updates.

## 9. Verify AKS and Azure RBAC

```powershell
az aks get-credentials `
  --resource-group telugu-pickles-azure `
  --name telugu-pickles-aks `
  --overwrite-existing

kubelogin convert-kubeconfig -l azurecli
kubectl auth can-i list nodes
kubectl get nodes
```

Expected results are `yes` and two `Ready` nodes. Azure RBAC may take several
minutes to propagate.

## 10. Platform plan and apply

Refresh Helm repository indexes when planning locally:

```powershell
helm repo update
```

Then create a normal, non-targeted plan:

```powershell
& 'C:\Terraform\terraform.exe' `
  'plan' `
  '-out=platform.tfplan' `
  '-input=false' `
  '-lock-timeout=10m'

& 'C:\Terraform\terraform.exe' 'apply' '.\platform.tfplan'
```

This creates billable resources, including AKS workloads, an Azure Load Balancer,
Azure disks, Log Analytics, and Azure OpenAI when AIOps is enabled.

Verify:

```powershell
kubectl get pods -A
kubectl get applications -n argocd
kubectl get svc -n ingress-nginx
terraform output
```

## 11. Feature branch and GitHub Actions flow

Use this sequence:

1. Commit Azure changes to `feature/azureAIOpsProj`.
2. Push the feature branch.
3. Open a pull request into `main`.
4. Pull-request workflows run validation, security scans, tests, and builds that
   do not require production Azure mutation.
5. Merge only when checks pass and GitHub secrets/variables are current.
6. A `main` push runs Azure Infrastructure and Azure App CI.
7. On the first deployment, wait for Azure Infrastructure to finish before
   rerunning Azure App CI if ACR or AKS was not ready during its first attempt.
8. Argo CD then syncs ACR commit-SHA images from Git into AKS.

Do not deploy Azure from an unreviewed feature branch. Production OIDC is bound
to the protected GitHub environment, while image CI is bound to `main`.

## 12. Release operations

Use the **Azure Release Control** workflow for frontend blue-green or catalog
canary status, promotion, full promotion, abort, retry, and undo.

```powershell
kubectl -n telugu-pickles get rollouts.argoproj.io
kubectl -n argocd get application telugu-pickles
```

## 13. Safe platform destroy

Preferred method:

1. Open **Actions -> Azure Infrastructure -> Run workflow**.
2. Select `destroy`.
3. Enter `destroy` in the confirmation field.
4. The workflow runs `scripts/pre-destroy-azure.sh` before Terraform destroy so
   Kubernetes load balancers and persistent resources can be cleaned up.
5. Confirm the application and AKS-managed resource groups are gone.

Local method from Git Bash/WSL:

```bash
./scripts/pre-destroy-azure.sh telugu-pickles-azure telugu-pickles-aks
terraform -chdir=projects/AzureInfrastructure destroy
```

Do not delete the state resource group first. The root state is required for an
orderly destroy.

## 14. Full state reset and rebuild

Use this only after the application platform is destroyed or when intentionally
starting from an empty Azure environment.

Back up remote root state:

```powershell
az storage blob download `
  --account-name "<state account>" `
  --container-name tfstate `
  --name telugu-pickles/azure.tfstate `
  --auth-mode login `
  --file projects/AzureInfrastructure/pre-delete.azure.tfstate `
  --overwrite
```

Resolve and remove the exact storage-account lock:

```powershell
$lockId = az lock list `
  --resource-group telugu-pickles-tfstate `
  --query "[?name=='protect-terraform-state'].id | [0]" `
  -o tsv

az lock delete --ids $lockId
```

Delete the state resource group only after explicit confirmation:

```powershell
az group delete --name telugu-pickles-tfstate --yes
az group exists --name telugu-pickles-tfstate
```

Expected final result: `false`.

When rebuilding from retained local bootstrap state, force a new random suffix
so Azure does not reuse a recently deleted storage-account name:

```powershell
cd projects\AzureInfrastructure\bootstrap
$subscriptionId = az account show --query id -o tsv

& 'C:\Terraform\terraform.exe' `
  'plan' `
  "-var=subscription_id=$subscriptionId" `
  '-replace=random_string.suffix' `
  '-out=bootstrap-fresh.tfplan'

& 'C:\Terraform\terraform.exe' 'apply' '.\bootstrap-fresh.tfplan'
```

After rebuilding, update all generated GitHub client IDs and state variables,
then rerun root `terraform init -reconfigure`.

## 15. Troubleshooting record

### `InsufficientVCPUQuota`

The subscription had zero DSv5-family quota in Central India. Use
`Standard_D2s_v3`, two initial/minimum nodes, and maximum three nodes, or request
a quota increase.

### `Too many command line arguments`

PowerShell native argument expansion failed when Terraform wrote a saved plan.
Invoke the executable with individually quoted arguments as shown above.

### `kubectl ... Forbidden`

Authentication succeeded, but the signed-in operator lacked AKS data-plane
RBAC. Add the Entra object ID to `aks_admin_principal_ids`, plan/apply the two
operator role assignments, and wait for propagation.

### Helm `no cached repo found`

The global Helm repository file referenced a missing index. Run:

```powershell
helm repo update
```

### Azure resources missing while Terraform state still lists them

This is external-deletion drift. Check the Azure activity log, recover/recreate
bootstrap prerequisites first, and never reuse a saved plan created before the
deletion.

### Storage account refresh returns key-authentication forbidden

The hardened account disables shared-key access. Bootstrap creates the container
and roles first, then disables shared-key access through AzAPI. Preserve this
ordering and use Entra/Azure AD authentication for the remote backend.

## 16. Security rules

- Never commit `.tfstate`, `.tfplan`, `.tfvars`, `.env`, kubeconfig, credentials,
  or generated secrets.
- Use GitHub OIDC and Azure workload identity; do not create long-lived client
  secrets or registry passwords.
- Keep the `production` environment protected.
- Use immutable commit-SHA image tags.
- Keep AIOps Kubernetes permissions read-only and namespace-scoped.
- Review every plan for unexpected deletes before apply.
- Delete the protected state account only as the final, explicitly confirmed
  teardown step.
