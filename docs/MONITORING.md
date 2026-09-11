# Monitoring — live ops & eval gates

How this insurance RAG is **observed in Prod** and how quality/safety are checked in CD.

Public, recruiter-oriented summary (no secrets, no cutover runbooks).

---

## Two layers

| Layer | Tool | Question it answers |
|---|---|---|
| **Live ops** | Application Insights + workbook **LLMOps Metrics** (per env) | Is the service healthy *right now*? Trend / degradation? |
| **Eval / promotion** | MLflow on Azure ML + GitHub Actions | Did this build pass golden / adversarial checks? |

Dev and Prod each have their own Log Analytics + App Insights. Hosting is isolated; the AI/data plane can be shared.

---

## Dashboard design (IT admin view)

The Prod workbook is built so an operator can answer three questions quickly:

1. **Is something wrong?** Glance KPIs + trends vs best-practice targets  
2. **Where?** Areas: **Health · RAG · Governance · Cost** — each area = section with a **row of related metrics**  
3. **Which call?** Dig tables with `operation_Id` → full pipeline for that chat  

### Best-practice targets (Prod starting values)

| KPI | Target |
|---|---|
| Chat success % | ≥ 95% |
| p95 chat duration | ≤ 20 s |
| Empty retrieval % | ≤ 15% |
| Governance block % | Own section (context-dependent; adversarial may be higher by design) |
| Exceptions | Near zero |

### Denominator rule

Summary rates use the **same chat population** (one row per `POST /chat`). Coverage checks compare HTTP chats vs stamped KPI chats vs workflow spans.

---

## What each area shows

| Area | Examples |
|---|---|
| **Glance** | Chats, success %, p95, empty %, exceptions, short trends |
| **Health** | Failed `/chat` and exceptions over time |
| **RAG** | Answer mix, empty by language, language/policy, retrieval/stage latency |
| **Governance** | Block %, stage, trend; adversarial tags / reactions (when eval tags present) |
| **Cost** | LLM / embeddings / Search CHF estimates, CHF per chat, breakdown |
| **Dig** | Blocked · empty · slow · failed → pipeline for selected `operation_Id` |

---

## CD quality gates (short)

| Gate | Role |
|---|---|
| **Prod golden** | Hard fail if pass rate drops below threshold or vs champion |
| **Empty retrieval** | Configurable max empty rate on golden |
| **Adversarial** | Soft report (defense rate by attack type) — see [GOVERNANCE.md](GOVERNANCE.md) |

---

## Related

- [GOVERNANCE.md](GOVERNANCE.md) — safety gates  
- [Architecture.md](Architecture.md) — request path  
- Live Prod UI / API — links on the [README](../README.md)
