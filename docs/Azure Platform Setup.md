# Azure Platform Setup — Sanitas RAG LLMOps (Dev + Prod)

Run the CLI from Azure Cloud Shell or Git Bash. Scripts live in `infra/

## 1) Naming convention

| Resource | Pattern | Dev example |
|---|---|---|
| Resource group | `rg-sanitasrag-{env}-chn` | `rg-sanitasrag-dev-chn` |
| Log Analytics | `log-sanitasrag-{env}-chn` | `log-sanitasrag-dev-chn` |
| Application Insights | `appi-sanitasrag-{env}-chn` | `appi-sanitasrag-dev-chn` |
| Key Vault | `kv-srag-{env}-chn8167` | `kv-srag-dev-chn8167` |
| Storage | `stsrag{env}chn8167` | `stsragdevchn8167` |
| ACR | `acrsrag{env}chn8167` | `acrsragdevchn8167` |
| Container Apps Environment | `cae-sanitasrag-{env}-chn` | `cae-sanitasrag-dev-chn` |
| API app | `ca-sanitasrag-api-{env}-chn` | `ca-sanitasrag-api-dev-chn` |
| Web app | `ca-sanitasrag-web-{env}-chn` | `ca-sanitasrag-web-dev-chn` |
| Action Group | Application Insights Smart Detection |

**Application Insights Smart Detection** is a default Action Group Azure adds the first time Application Insights is created in a **subscription**. It emails the subscription admins about things like failed requests or unusual exceptions.

Azure creates one per subscription (location: Global), not one per environment.

- `{env}` = `dev` or `prod`
- `{region}` = `chn` (Switzerland North)
- `8167` is a uniqueness suffix (from your subscription prefix already used in storage names)

Existing Sanitas RAG services (OpenAI, AI Search, Cosmos, Redis, Document Intelligence) can stay shared at first. Isolate them later if Dev must never touch Prod data.

## 2) Login and subscription

```bash
az login
az account show --query "{name:name,id:id,tenant:tenantId}" -o table
az account set --subscription ""
```

## 3) Create both environments

From the repository root:

```bash
bash infra/setup-platform.sh dev
bash infra/setup-platform.sh prod
```

Equivalent explicit commands for **dev** (repeat with `prod` / `prod` names):

```bash
az group create --name rg-sanitasrag-dev-chn --location switzerlandnorth

az monitor log-analytics workspace create \
  --resource-group rg-sanitasrag-dev-chn \
  --workspace-name log-sanitasrag-dev-chn \
  --location switzerlandnorth \
  --sku PerGB2018

az monitor app-insights component create \
  --app appi-sanitasrag-dev-chn \
  --resource-group rg-sanitasrag-dev-chn \
  --location switzerlandnorth \
  --workspace "$(az monitor log-analytics workspace show -g rg-sanitasrag-dev-chn -n log-sanitasrag-dev-chn --query id -o tsv)"

az keyvault create \
  --name kv-srag-dev-chn8167 \
  --resource-group rg-sanitasrag-dev-chn \
  --location switzerlandnorth \
  --enable-rbac-authorization true

az storage account create \
  --name stsragdevchn8167 \
  --resource-group rg-sanitasrag-dev-chn \
  --location switzerlandnorth \
  --sku Standard_LRS --kind StorageV2

az acr create \
  --name acrsragdevchn8167 \
  --resource-group rg-sanitasrag-dev-chn \
  --location switzerlandnorth \
  --sku Basic
```