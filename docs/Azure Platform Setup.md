# Azure Platform Setup — insurance RAG LLMOps (Dev + Prod)

Uses Git Bash.

## 1) Naming convention

| Resource | Pattern | Dev example |
|---|---|---|
| Resource group | `rg-insurancerag-{env}-chn` | `rg-insurancerag-dev-chn` |
| Log Analytics | `log-insurancerag-{env}-chn` | `log-insurancerag-dev-chn` |
| Application Insights | `appi-insurancerag-{env}-chn` | `appi-insurancerag-dev-chn` |
| Key Vault | `kv-srag-{env}-chn8167` | `kv-srag-dev-chn8167` |
| Storage | `stsrag{env}chn8167` | `stsragdevchn8167` |
| ACR | `acrsrag{env}chn8167` | `acrsragdevchn8167` |
| Container Apps Environment | `cae-insurancerag-{env}-chn` | `cae-insurancerag-dev-chn` |
| API app | `ca-insurancerag-api-{env}-chn` | `ca-insurancerag-api-dev-chn` |
| Web app | `ca-insurancerag-web-{env}-chn` | `ca-insurancerag-web-dev-chn` |
| Action Group | Application Insights Smart Detection |

**Application Insights Smart Detection** is a default Action Group Azure adds the first time Application Insights is created in a **subscription**. It emails the subscription admins about things like failed requests or unusual exceptions.

Azure creates one per subscription (location: Global), not one per environment.

- `{env}` = `dev` or `prod`
- `{region}` = `chn` (Switzerland North)
- `8167` is a uniqueness suffix (from your subscription prefix already used in storage names)

Insurance RAG services (OpenAI, AI Search, Cosmos, Redis, Document Intelligence) stay shared.

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
az group create --name rg-insurancerag-dev-chn --location switzerlandnorth

az monitor log-analytics workspace create \
  --resource-group rg-insurancerag-dev-chn \
  --workspace-name log-insurancerag-dev-chn \
  --location switzerlandnorth \
  --sku PerGB2018

az monitor app-insights component create \
  --app appi-insurancerag-dev-chn \
  --resource-group rg-insurancerag-dev-chn \
  --location switzerlandnorth \
  --workspace "$(az monitor log-analytics workspace show -g rg-insurancerag-dev-chn -n log-insurancerag-dev-chn --query id -o tsv)"

az keyvault create \
  --name kv-srag-dev-chn8167 \
  --resource-group rg-insurancerag-dev-chn \
  --location switzerlandnorth \
  --enable-rbac-authorization true

az storage account create \
  --name stsragdevchn8167 \
  --resource-group rg-insurancerag-dev-chn \
  --location switzerlandnorth \
  --sku Standard_LRS --kind StorageV2

az acr create \
  --name acrsragdevchn8167 \
  --resource-group rg-insurancerag-dev-chn \
  --location switzerlandnorth \
  --sku Basic
```
