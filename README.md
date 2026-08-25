# Insurace RAG — LLMOps

Multilingual **customer-support RAG** for **Basic and complementary health insurance** (EN / FR / DE / IT). Advisors and customers ask coverage and genral questions; the system answers only from indexed policy documents (Knowledge base).

Application source is **private**. This public repo is the product + LLMOps story.

---

## Project summary

| | |
|---|---|
| **What** | RAG chatbot: Streamlit UI + FastAPI, hybrid search, grounded `gpt-4.1` answers |
| **For whom** | Swiss-style insurance support (current vs new policies, 4 languages) |
| **Ops** | Isolated **Dev** and **Prod** on Azure Container Apps, shared AI/data plane, GitHub promotion |

**Live demo**

| | UI | API |
|---|---|---|
| **Dev** | [Web](https://ca-sanitasrag-web-dev-chn.braveisland-7adede75.switzerlandnorth.azurecontainerapps.io/) | [Swagger](https://ca-sanitasrag-api-dev-chn.braveisland-7adede75.switzerlandnorth.azurecontainerapps.io/docs) |
| **Prod** | [Web](https://ca-sanitasrag-web-prod-chn.greenground-3327a232.switzerlandnorth.azurecontainerapps.io/) | [Swagger](https://ca-sanitasrag-api-prod-chn.greenground-3327a232.switzerlandnorth.azurecontainerapps.io/docs) |

---

## Tech stack

- **App:** Streamlit, FastAPI, LangGraph orchestrator  
- **RAG:** Azure AI Search (hybrid + rerank), Azure Document Intelligence, Blob Storage  
- **LLM:** Azure AI Foundry / OpenAI (`gpt-4.1`, `text-embedding-3-small`)  
- **State:** Azure Redis (sessions), Cosmos DB (profiles + history)  
- **Safety:** Azure Content Safety, PII / jailbreak gates (GDPR / Swiss nFADP-oriented)  
- **Run:** Docker, Azure Container Registry, Azure Container Apps  
- **Observe:** Log Analytics + Application Insights (per environment)  
- **Deliver:** GitHub Actions, OIDC to Azure (no long-lived deploy passwords)

---

## Architecture

```text
Browser → Streamlit (web Container App)
       → FastAPI (api Container App)
            → Governance IN  → Search → gpt-4.1 → Governance OUT
       → Redis + Cosmos

GitHub (private) --OIDC--> Dev RG | Prod RG
Both apps call shared Foundry + Search + Cosmos + Redis
```

More detail: [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Screenshots / demo

Added images under `docs/screenshots/` and embed them here (recruiters scan these first):

1. Chat: a real policy answer in markdown  
2. Governance: a blocked jailbreak or PII query  
3. Swagger `/docs`  
4. Azure portal: Dev RG vs Prod RG  
5. Live Flow: short **GIF** of login → question → answer  

---

## My contributions

Solo build (product + platform), including:

- End-to-end RAG for Swiss insurance PDFs (ingest → index → retrieve → generate)  
- Multilingual UI and policy-type flow (current vs new)  
- Governance/guardrails on the live chat path, configurable from the UI  
- LLMOps: two Azure environments, CAF naming, secrets on Container Apps  
- Shared vs isolated resource split (cost vs blast radius)  
- CI tests + CD design (`dev` → Dev, `main` → Prod with approval)

---

## Results / impact

What is in place and working:

- **Dev and Prod** both serve the API (`/docs`) and the UI  
- Answers grounded in `policy_index`, not a generic chatbot  
- Unsafe/PII turns can be **blocked** without calling the LLM  
- A bad Dev image cannot overwrite Prod (separate ACR + apps)  
- Secrets are not in git; they live as Container App `secretref`s  

---

## Challenges solved

- **TLS 1.0/1.1 retirement** on new storage accounts. Now only **TLS 1.2**  
- **Same Docker layout** as the original `Production.*` imports
- **Secrets vs env vars:** secrets do nothing until mapped with `secretref:`
---

## Trade-offs

| Choice | Trade-off |
|---|---|
| Share OpenAI / Search / Cosmos across Dev and Prod | Lower cost, same answers; Dev can touch Prod-like data |
| Isolate hosting only (two RGs, two ACRs) | Safer deploys; more resources to operate |
| Container Apps| Fits RAG serving; |
| Two environments, not three (no staging) | Simpler; no extra pre-prod ring |
| Block on PII instead of redact | Safer default; legitimate emails/AHV in questions get blocked unless toggled off |
| Public repo without Python | Safer for credentials and IP;

---

## What I would improve next

1. **Monitoring:** dashboards for latency, tokens, cost, governance block rate (App Insights is already per env)  
2. **CI/CD live:** OIDC + GitHub Environments on the **private** repo 
3. **Data isolation:** optional Dev index / Cosmos so Prod data is never used in experiments  
4. **Private networking** when this is real Prod (VNet, private endpoints)  
5. **Eval in the pipeline:** DeepEval faithfulness/relevancy as a deploy gate  
6. **Key Vault references** instead of Container App secrets only  

---

## What is not in this repository

Python, Dockerfiles used to ship the app, `.env`, and secret scripts. Available in a private repo for business interview.
