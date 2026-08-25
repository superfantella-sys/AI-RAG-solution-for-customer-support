# Architecture one-pager

## Request path

```text
Browser → Streamlit (Container App web)
        → FastAPI (Container App api)
            → Governance (input)
            → Azure AI Search (hybrid retrieval + rerank)
            → Azure OpenAI gpt-4.1
            → Governance (output)
        → Redis (session) + Cosmos (history)
```

## Resource groups

| RG | Role |
|---|---|
| `rg-sanitasrag-dev-chn` | Dev hosting (ACR, CAE, API, web, logs, App Insights, Key Vault) |
| `rg-sanitasrag-prod-chn` | Prod hosting (same shape) |
| `rg-sanitasrag-shared-chn` | New Foundry / OpenAI deployments (chat + embeddings) |
| `mct_santicuni` | Original RAG data plane (Search, Cosmos, Redis, Document Intelligence, blobs) until moved |

## Lifecycle

LLMOps here is **build → push → deploy → approve**, plus **prompt/app governance**.
