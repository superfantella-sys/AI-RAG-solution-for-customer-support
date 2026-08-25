# Azure runtime for Sanitas RAG — Container Apps

The original notebook provisioned **Azure Machine Learning** workspaces, CPU clusters, sklearn/torch environments, train/validate components, champion/challenger model aliases, and AML online/batch endpoints.

That lifecycle does **not** fit this RAG:

| LLMOps RAG (this repo) |
|---|
| Azure Container Apps (already live) |
| Retrieval + generation + DeepEval (later) |
| Version Docker images of API + UI |
| `ca-sanitasrag-api-{env}-chn` + `ca-sanitasrag-web-{env}-chn` |
| Image tag = git SHA; Prod promoted from `dev` |

Use two Container Apps environments (dev/prod).

## 1) Create the apps

After `bash infra/setup-platform.sh dev` (and prod):

```bash
bash infra/setup-containerapps.sh dev
bash infra/setup-containerapps.sh prod
```

This builds `sanitasrag-api` and `sanitasrag-web` in ACR, then creates:

- `ca-sanitasrag-api-dev-chn` (FastAPI, port 8000)
- `ca-sanitasrag-web-dev-chn` (Streamlit, port 8501, `BACKEND_URL` pointing at the API FQDN)

Same names with `prod` for production.

## 2) Wire secrets (do not commit values)

Copy keys from `.env` into Container App secrets. Example for the **dev API**:

```bash
RG=rg-sanitasrag-dev-chn
API=ca-sanitasrag-api-dev-chn

az containerapp secret set -g $RG -n $API --secrets \
  azure-openai-key="<AZURE_OPENAI_API_KEY>" \
  azure-search-key="<AZURE_SEARCH_API_KEY>" \
  cosmos-key="<COSMOS_KEY>" \
  redis-password="<REDIS_PASSWORD>" \
  content-safety-key="<AZURE_CONTENT_SAFETY_KEY>"

az containerapp update -g $RG -n $API --set-env-vars \
  AZURE_OPENAI_API_KEY=secretref:azure-openai-key \
  AZURE_SEARCH_API_KEY=secretref:azure-search-key \
  COSMOS_KEY=secretref:cosmos-key \
  REDIS_PASSWORD=secretref:redis-password \
  AZURE_CONTENT_SAFETY_KEY=secretref:content-safety-key \
  AZURE_OPENAI_ENDPOINT="https://insuranceragembedding.openai.azure.com/" \
  AZURE_SEARCH_ENDPOINT="https://sanitasindex.search.windows.net" \
  AZURE_SEARCH_INDEX=policy_index \
  COSMOS_ENDPOINT="https://historicalchat.documents.azure.com:443/" \
  COSMOS_DATABASE_NAME=insurance_db

## 3) Manual deploy (same path GitHub Actions uses)

```bash
bash infra/deploy-containerapps.sh dev local-test
bash infra/deploy-containerapps.sh prod local-test
```

GitHub CD tags images with `$GITHUB_SHA` instead of `local-test.
Quality gates for LLMOps are governance tests in CI and (next) monitoring