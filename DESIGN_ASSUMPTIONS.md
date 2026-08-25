# Sanitas RAG — LLMOps design assumptions

This note explains **what I created, what I reused, and why**. It is the design record for the Dev/Prod Azure setup (`infra/setup-platform.sh` and related scripts).
---

## 1. Global decisions

### Two environments

Dev and Prod only. 

| GitHub branch | Azure environment | Resource group |
|---|---|---|
| `dev` | Dev | `rg-sanitasrag-dev-chn` |
| `main` | Prod | `rg-sanitasrag-prod-chn` |

**Why:** Promotion is `dev` → `main`. A third Azure ML “workspace” or staging RG would add cost and confusion without a third runtime for this chatbot.

### Container Apps, not Azure Machine Learning

The RAG is FastAPI + Streamlit in Docker.

LMOps for this app is: build images, push to ACR, deploy Container Apps, gate Prod with a GitHub approval.

### Two layers of Azure resources

| Layer | What | Isolated per env? |
|---|---|---|
| **Hosting (new)** | Resource groups, ACR, Container Apps, Key Vault, logs, App Insights | Yes — one set in Dev, one in Prod |
| **RAG data/AI (existing)** | OpenAI, AI Search, Cosmos, Redis, Document Intelligence, policy blob storage | **Shared for now** — both apps call the same services |

**Why share the AI layer:** Those services already hold the index, embeddings, chat history, and documents. Duplicating them would mean two indexes, two Cosmos accounts, and two OpenAI deployments. That is a later isolation step, not required to start Dev/Prod deploys.

### Naming

Pattern: `{type}-sanitasrag-{env}-chn` (CAF-style). Storage and ACR cannot use hyphens, so they become `stsrag{env}chn8167` and `acrsrag{env}chn8167`. `8167` is a uniqueness suffix (same idea as `stmctsanticuni8167ef`). `chn` = Switzerland North, matching the existing RAG.

### Git Bash

Scripts export `MSYS_NO_PATHCONV=1` so ARM ids (`/subscriptions/...`) are not rewritten to `C:/Program Files/Git/...`.

---

## 2. New resources (created by `setup-platform.sh`)

### Resource group — `rg-sanitasrag-{env}-chn`

**Assumption:** one RG per environment.

**Why:** Blast radius, RBAC, and billing stay separate. Deleting Dev cannot wipe Prod. GitHub OIDC is granted Contributor on each RG, not on the whole subscription.

### Log Analytics — `log-sanitasrag-{env}-chn`

**Assumption:** each environment has its own workspace (SKU `PerGB2018`).

**Why:** Container Apps Environment and App Insights need a workspace. Separate workspaces keep Dev noise out of Prod queries and match the later monitoring step.

### Application Insights — `appi-sanitasrag-{env}-chn`

**Assumption:** workspace-based App Insights, linked to that env’s Log Analytics.

**Why:** The backend already emits traces (`app_insights_init_sanitas.py`). A dedicated component per env lets you compare Dev vs Prod latency, failures, and (later) governance block rate. 

### Application Insights Smart Detection (Action group)

**Assumption:** Azure creates this automatically; we do not script it.

**Why:** First App Insights in a subscription gets a **global** action group that emails admins about failed requests / anomalies. It appears in **Dev** because Dev was created first. Prod uses the same group. It is not an environment resource and must not be duplicated.

### Key Vault — `kv-srag-{env}-chn8167`

**Assumption:** RBAC authorization (not legacy access policies). Secrets live here later, not in git.

**Why:** `.env` has been choosed because I am not in real Prod hardening. The vault exists so GitHub/Container Apps can move to Key Vault references without creating a new resource later.

I have **not** copied OpenAI/Search/Cosmos keys into the vault yet. That is a follow-up (`az keyvault secret set` / Container App `secretref`).

### Storage account — `stsrag{env}chn8167`

**Assumption:** `StorageV2`, `Standard_LRS`, **TLS 1.2 minimum**, public blob access off.

**Why:** LLMOps hosting needs a general-purpose account (eval artifacts, ACR/build sidecars, future monitoring exports). This is **not** a replacement for `stmctsanticuni8167ef` (policy PDFs / Document Intelligence). TLS 1.2 is required because Azure retired TLS 1.0/1.1;. LRS is enough for Switzerland North Dev/Prod no redudancy.

### Azure Container Registry — `acrsrag{env}chn8167`

**Assumption:** **one ACR per environment**, SKU **Basic**, admin user **off**.

**Why:** Dev and Prod images must not overwrite each other. Basic is enough for two images (`sanitasrag-api`, `sanitasrag-web`). Admin credentials stay off; Container Apps pull with a managed identity (`AcrPull`), GitHub pushes with OIDC (`AcrPush`). 

### Container Apps Environment — `cae-sanitasrag-{env}-chn`

**Assumption:** one CAE per RG, logs sent to that env’s Log Analytics.

**Why:** A CAE is the cluster that hosts the apps (same role as `sanitas-rag-env`). Separate CAEs mean Dev scale/restarts do not hit Prod. Logs stay in the env workspace.

---

## 3. Runtime apps (created by `setup-containerapps.sh`)

### API Container App — `ca-sanitasrag-api-{env}-chn`

**Assumption:** FastAPI on port 8000, external ingress, system-assigned identity, 1 vCPU / 2 Gi, scale 0–2.

**Why:** This is the new `rag-backend`. External ingress is simpler for Streamlit (`BACKEND_URL=https://<api-fqdn>`) and for `/docs` smoke tests in CD. Scale-to-zero on Dev limits cost. Identity is required to pull from ACR without admin passwords.

### Web Container App — `ca-sanitasrag-web-{env}-chn`

**Assumption:** Streamlit on port 8501, `BACKEND_URL` pointing at the API FQDN of the **same** environment.

**Why:** This is the new `rag-frontend`. Dev UI must talk to Dev API,  not to Prod.

### What these apps still consume

| Others resource | Role in the RAG |
|---|---|
| `insuranceragembedding` | Chat + embeddings |
| `sanitasindex` | Policy retrieval index |
| `historicalchat` | User profiles + conversation history |
| `insuranceredis` | Session state |
| `documentsanitas` | PDF ingestion (when you re-index) |
| `stmctsanticuni8167ef` | Source documents |

**Assumption:** Dev and Prod Container Apps may share these until I decide Prod data must never be readable from Dev.

---

## 4. CI/CD identity (created by `setup-github-oidc.sh`)

### Entra app — `sp-sanitasrag-gha-chn`

**Assumption:** **one** app registration for GitHub Actions, federated credentials for:

- GitHub Environment `dev` / `prod`
- branches `dev` / `main`

No client secret.

**Why:** OIDC is the current GitHub–Azure pattern. A password in GitHub Secrets would be a long-lived key. One app, two federated subjects, keeps Dev and Prod deploys in the same identity but scoped by GitHub Environment approval on **prod**.

Role assignments: **Contributor** on each LLMOps RG, **AcrPush** on each ACR.

---

## 5. Explicitly not created (and why)

| Not created | Why |
|---|---|
| VNet + private endpoints | I are not in locked-down Prod yet; add when you harden |
| New App Insights on the old RAG | Leave `pretrainmodel7702744053` on the live app |
| Key Vault secrets populated | I still use `.env` locally;
| Smart Detection action group in Prod | Subscription-wide, created with first App Insights |

---