**Simple VM Image Pipeline**

This repository contains a Packer + Azure DevOps pipeline for building a Linux VM image (Ubuntu) and publishing it as a managed image in an Azure resource group.

**Overview**
- **What it does:** Builds an Azure VM image using Packer (azure-arm builder), runs provisioning steps, and outputs a `manifest.json` with the created image artifact.
- **Key files:** `image.pkr.hcl`, `azure-pipelines.yml`.

**Prerequisites**
- An Azure subscription with permission to create resources and role-assign a service principal.
- Azure CLI installed locally when testing or set up in pipeline tasks.
- Packer installed on the agent (see Install section below).

**Packer plugin and HCL**
- This template requires the Azure Packer plugin: `github.com/hashicorp/azure` (example in `image.pkr.hcl`).
- Useful links:
	- Packer HCL docs: https://www.packer.io/docs/hcl_templates
	- Packer plugin docs: https://www.packer.io/docs/plugins/overview

**Installation / Agent setup**
- Install Packer on a build agent in a separate task before running `packer build`:

```bash
# Example install step (Linux agent)
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y packer
```

**Microsoft-hosted vs Self-hosted agents (parallel jobs)**
- Microsoft-hosted agents: Azure DevOps provides limited free build capacity depending on project type (private vs public). For larger or guaranteed parallelism, use self-hosted agents.
- Self-hosted agents: install your own runner(s) to get unlimited control over parallel jobs and concurrency; recommended when you need more parallelism than the provided free tier.
- See Azure DevOps hosted agent docs for current free-tier details: https://learn.microsoft.com/azure/devops/pipelines/agents/hosted

**Service principal / Service connection (Contributor role)**
Create an Azure AD app registration (service principal) and grant it the `Contributor` role on the subscription or resource group the pipeline should manage. This service principal is then used to create an Azure Resource Manager service connection in Azure DevOps.

Portal steps (Azure):
- Azure Portal → Azure Active Directory → App registrations → New registration → record the `Application (client) ID` and `Directory (tenant) ID`.
- Azure Portal → Subscriptions (or Resource Group) → Access control (IAM) → Add role assignment → choose **Contributor** → Assign to the app registration (service principal).

Azure DevOps service connection (Project settings):
- Project settings → Service connections → New service connection → **Azure Resource Manager** → select **App registration** (service principal) → enter Subscription and Resource Group → set scope and save.
- When selecting the role within the service connection, use **Contributor** to allow Packer to create and delete temporary resources.

CLI role assignment example:

```bash
# Assign Contributor role to the service principal on a resource group
az role assignment create --assignee <appId> --role "Contributor" --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>
```

**Pipeline variables**
- `client_id` and `tenant_id` are declared in `image.pkr.hcl` as input variables.
	- `client_id`: the Application (client) ID of the service principal (from the App registration / service connection).
	- `tenant_id`: the Directory (tenant) ID for the Azure AD tenant.
- In your pipeline you pass them to Packer using `-var "client_id=$CLIENT_ID" -var "tenant_id=$TENANT_ID"`.
- To find values:
	- Azure Portal: App registrations → your app → Application (client) ID and Directory (tenant) ID.
	- Azure CLI: `az account show --query tenantId -o tsv` (tenant), and `az ad app show --id <appId>` to inspect the app.

**Why `use_azure_cli_auth = true`**
- The template uses `use_azure_cli_auth = true` to rely on Azure CLI authentication (or pipeline service connection) instead of requiring a `client_secret`. This avoids creating and maintaining a secret vault entry and speeds up pipeline setup.

**Notes and recommended configuration**
- Use Bash tasks instead of PowerShell on Linux agents to avoid: "Unable to locate executable file: 'powershell'" when the agent has no PowerShell.
- Add `public_ip_sku = "Standard"` in the Packer source to avoid IPv4 Basic SKU public IP limits: prevents the error about Basic SKU public IP limit reached.
- Ensure the image SKU contains `-gen2` where required (for Gen2 images) to avoid image SKU errors.
- Install Packer in a dedicated pipeline task before `packer build`.
- Packer creates a temporary resource group to stage VM and image creation; after the managed image is created, Packer moves it to the configured resource group and deletes the temporary group.

**Explanation of `image.pkr.hcl` (high level)**
- `packer { required_plugins { azure = { source = "github.com/hashicorp/azure" version = "2.3.3" } } }` — declares the Azure plugin dependency used by the HCL template.
- `variable "client_id"` and `variable "tenant_id"` — variables passed at `packer build` time.
- `source "azure-arm" "test-image" { ... }` — configures the azure-arm builder (subscription, tenant, publisher/offer/sku, vm_size, location, managed image details).
- `build { sources = ["source.azure-arm.test-image"] ... }` — defines the build pipeline: provisioners run commands on the temporary VM and the post-processor produces a `manifest.json`.
- `provisioner "shell"` — runs shell commands to provision the image (installing packages, writing files, etc.).
- `post-processor "manifest"` — creates `manifest.json` containing the artifact_id (used later to extract the created image id).

**Explanation of `azure-pipelines.yml` cron schedule**
- `schedules:` → `- cron: "0 19 * * 1"` — standard cron syntax: minute hour day-of-month month day-of-week.
	- `0 19 * * 1` means: run at minute 0, hour 19 (19:00), every day-of-month, every month, only on day-of-week 1 (Monday). Timezone is the pipeline host time/UTC depending on Azure DevOps settings — validate timezone in pipeline settings when scheduling.

**How the pipeline extracts the created image ID**
- After `packer build` the `manifest.json` contains `artifact_id` like `Managed Image:<image-id>`.
- The pipeline uses `jq` to parse `manifest.json` and extract image id, then sets it as an output variable for downstream jobs.

**Common errors & fixes**
- Error about PowerShell: use Bash tasks for Linux agents (set `scriptType: 'bash'` in pipeline tasks).
- `IPv4BasicSkuPublicIpCountLimitReached`: set `public_ip_sku = "Standard"` in `image.pkr.hcl` to avoid Basic SKU limits.
- Image SKU errors: ensure the SKU includes `-gen2` when targeting Gen2 images.

**Example commands referenced in pipeline**
- Install plugin and initialize:
	- `packer plugins install github.com/hashicorp/azure`
	- `packer init .`
- Run packer build (example):
	- `packer build -var "client_id=$CLIENT_ID" -var "tenant_id=$TENANT_ID" *.pkr.hcl`

**Next steps / contributor guidance**
- Confirm the `client_id` and `tenant_id` values in your pipeline variables and service connection.
- Ensure the service principal has `Contributor` on the target subscription or resource group.
- If you want dedicated concurrency, add self-hosted agents to your Azure DevOps organization.


