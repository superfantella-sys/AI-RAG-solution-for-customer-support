# Governance — safety gates on the chat path

How the insurance RAG chatbot **blocks unsafe or non-compliant turns** before (and after) the model answers.

This is the public, recruiter-oriented summary. Application source stays in a private repository.

---

## Why governance exists

The assistant answers from **indexed policy documents only**. It must also respect:

- **PII / privacy** expectations (GDPR / Swiss nFADP-oriented)
- **Jailbreak / prompt-injection** attempts (ignore rules, reveal system prompt, bypass filters)
- **Harmful content** categories (via Azure Content Safety)

Governance is part of the live request path — not a separate offline tool.

---

## Where it sits

```text
User message
  → Governance IN  (block or allow)
  → RAG retrieve + gpt-4.1
  → Governance OUT (block or allow answer)
  → Response to UI
```

If **input** is blocked, the LLM is **not** called.  
If **output** is blocked, the generated answer is **not** shown.

Settings (PII, jailbreak, Azure checks, etc.) are **configurable from the UI** and applied by the API.

---

## Controls (layers)

| Layer | What it does | Typical outcome |
|---|---|---|
| **PII detection** | Flags emails, phones, AVS/AHV-style IDs, cards, IBANs, etc. | Strong block rate on PII adversarial cases |
| **Azure Prompt Shields** | Multilingual **user-prompt attack** / jailbreak detection (`text:shieldPrompt`) | Primary control for injection / system-bypass style prompts |
| **Azure Content Safety (`AnalyzeText`)** | Hate / sexual / violence / self-harm severities | Blocks toxic content (not the main jailbreak tool) |
| **Small local pattern backup** | Short high-precision regex (not a multi-language dictionary) | Extra catch for canonical English phrases |

Design choice: **do not** maintain large FR/DE/IT keyword lists for jailbreaks — that does not scale. Prompt Shields is the main multilingual control; regex stays small.

Default product choice: **block on PII** rather than redact (safer for a support bot; can be toggled).

---

## Adversarial evaluation (Prod)

A dedicated **adversarial suite** (PII / jailbreak / system_bypass) runs in **CD-Prod as a soft report** (does not hard-fail the deploy by itself).

Example soft-report shape:

- Cases defended (governance blocked) vs answered (got through)
- Breakdown by attack type

**PII** cases are typically well defended. **Jailbreak / system_bypass** are harder when only English regex is used — that is why Prompt Shields was added as the primary detector. After deploy, the soft report is the place to re-check defense rates.

Golden quality eval (pass rate / empty retrieval) remains a **separate hard gate**.

---

## What recruiters can verify

- Live UI: ask a normal policy question → grounded answer  
- Live UI: paste obvious PII or a jailbreak-style prompt → **blocked** message (see screenshot)  
- Docs: this page + [MONITORING.md](MONITORING.md) for how block rate is observed in Prod  

---

## Related

- [Architecture](Architecture.md) — request path  
- [MONITORING.md](MONITORING.md) — dashboard KPIs including governance block %  
- README screenshots: `docs/screenshots/governance_block.png`
