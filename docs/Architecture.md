# Architecture one-pager

## Request path

```text
Browser → Streamlit (Container App web)
        → FastAPI (Container App api)
            → Governance IN   (PII, Prompt Shields, Content Safety)
            → Azure AI Search (hybrid retrieval + rerank)
            → Azure OpenAI gpt-4.1
            → Governance OUT
        → Redis (session) + Cosmos (history)
```

Unsafe or non-compliant **input** can stop the request before the LLM.  
Blocked **output** is not shown to the user.

Details: [GOVERNANCE.md](GOVERNANCE.md).

## Resource groups

| RG | Role |
|---|---|
| `rg-insurancerag-dev-chn` | Dev hosting (ACR, CAE, API, web, logs, App Insights, Key Vault) |
| `rg-insurancerag-prod-chn` | Prod hosting (same shape) |
| `rg-insurancerag-shared-chn` | Foundry / OpenAI deployments (chat + embeddings) |
| Shared / legacy data plane | Search, Cosmos, Redis, Document Intelligence, blobs (shared or migrated as needed) |

Each env has its own **Log Analytics + Application Insights**. Live KPIs and dig-down live in the Prod workbook — see [MONITORING.md](MONITORING.md).

## Lifecycle

LLMOps here is **build → push → deploy → observe → evaluate**:

- `dev` branch → Dev Container Apps  
- `main` branch → Prod (with approval)  
- Prod golden eval = hard gate; adversarial suite = soft report  
- App Insights workbook = day-2 operations (Health / RAG / Governance / Cost)
