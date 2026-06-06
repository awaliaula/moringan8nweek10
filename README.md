# 🗞️ Nairobi Nexus — Automated Weekly Newsletter Generator

> **Production-grade AI newsletter automation built on n8n**
> Week 10 Final Capstone Project · AI Workflow Engineering · June 2026

---

## 👥 Team

| Role | Name |
|---|---|
| Lead Automation Engineer | Anthony Waliaula |
| Documentation & Slides   | Anne Nyambura |
GitHub: https://github.com/awaliaula/moringan8nweek10

---

## 📌 Project Overview

Nairobi Nexus solves a real editorial pain point: **manually monitoring 5 separate Kenyan publications every week** just to write a newsletter intro.

This n8n workflow fully automates that process end-to-end. Every Friday at 2:00 PM (Africa/Nairobi), the system wakes up, fans out across 5 news sources in parallel, extracts the top headline from each via a dedicated AI Field Reporter agent, merges the structured outputs, and hands everything to **Kamau** — the Chief Editor AI — who synthesises a witty, Nairobi street-smart newsletter in under 200 words. The final draft is logged to Google Sheets and dispatched via Gmail to the marketing team for approval.

**This is not a prototype. It is a production-grade system with three independent fault-tolerance layers, strict JSON enforcement on every agent, and full operational logging for governance and auditability.**

---

## 🏗️ Architecture

```
⏰  Schedule Trigger (Every Friday 2:00 PM EAT)
        │
        ├── 📡 HTTP TechCabal       → 🤖 AI Field Reporter
        ├── 📡 HTTP Business Daily  → 🤖 AI Field Reporter
        ├── 📡 HTTP The Star        → 🤖 AI Field Reporter
        ├── 📡 HTTP Standard Media  → 🤖 AI Field Reporter
        └── 📡 HTTP Kenyan Wall St. → 🤖 AI Field Reporter
                                            │
                                   🔀 Merging Intelligence (Append Mode)
                                            │
                                   ✍️  Chief Editor — Kamau (GPT-4o-mini)
                                     /                    \
                              [Success]                [Error Branch]
                                 │                          │
                        ┌────────┴────────┐       🔁 Backup Editor (GPT-4-turbo)
                        │                 │
                   📊 Google Sheets    📧 Gmail → Marketing Team
```

**39 nodes · 5 news sources · 2 fault layers · 3-tier model fallback strategy**

---

## 📰 News Sources Monitored

| Branch | Publication | Coverage |
|---|---|---|
| 2A | [TechCabal](https://techcabal.com) | African tech & startups |
| 2B | [Business Daily Africa](https://businessdailyafrica.com) | Business & finance |
| 2C | [The Star Kenya](https://the-star.co.ke) | General news & tech |
| 2D | [Standard Media Kenya](https://standardmedia.co.ke) | Tech & economy |
| 2E | [Kenyan Wall Street](https://kenyanwallstreet.com) | Financial & investment |

---

## 🔄 Node-by-Node Workflow Logic

| Step | Node | Role & Logic |
|---|---|---|
| 1 | **Schedule Trigger** | Fires every Friday at 14:00 Africa/Nairobi. Triggers all 5 parallel branches simultaneously. |
| 2A | **HTTP – TechCabal** | Fetches RSS feed. `continueRegularOutput` enabled — source failure does not stop workflow. |
| 2A | **Summarize TechCabal (AI Agent)** | Field Reporter: extracts top tech/business headline from raw RSS. Outputs structured JSON via Output Parser. |
| 2B | **HTTP – Business Daily** | Fetches RSS from businessdailyafrica.com. Fault tolerant. |
| 2B | **Summarize Business Daily (AI Agent)** | Field Reporter: extracts top business/finance headline. Outputs structured JSON. |
| 2C | **HTTP – The Star** | Fetches RSS from the-star.co.ke. Fault tolerant. |
| 2C | **Summarize The Star (AI Agent)** | Field Reporter: extracts top general/tech headline. Outputs structured JSON. |
| 2D | **HTTP – Standard Media** | Fetches RSS from standardmedia.co.ke. Fault tolerant. |
| 2D | **Summarize Standard Media (AI Agent)** | Field Reporter: extracts top tech/economy headline. Outputs structured JSON. |
| 2E | **HTTP – Kenyan Wall Street** | Fetches RSS from kenyanwallstreet.com. Fault tolerant. |
| 2E | **Summarize Kenyan Wall Street (AI Agent)** | Field Reporter: extracts top financial/investment headline. Outputs structured JSON. |
| 3 | **Merging Intelligence (Merge)** | Append mode. Combines all 5 Field Reporter outputs into one list regardless of which sources were online. |
| 4 | **Chief Editor — Kamau (AI Agent)** | Primary model (GPT-4o-mini). `executeOnce: true`. Reads all 5 summaries, filters `status: 'success'` only, writes witty Kenyan newsletter under 200 words with slang. |
| 4F | **Backup Editor AI Agent** | Activated only if Chief Editor fails (Error branch). Uses GPT-4-turbo as fallback model. `executeOnce: true`. |
| 5a | **Log Draft to NewsletterManager (Sheets)** | Appends to Drafts tab: Date, Successful Sources, Draft Body (markdown stripped), Model Used. |
| 5b | **Send Draft to Marketing Team (Gmail)** | Sends styled HTML email with newsletter body, source count, and date to marketing team for approval. |

---

## 🤖 AI Agent Configuration

### Field Reporter Agents (×5) — Prompt Template

Each of the 5 Field Reporter agents uses a source-specific variant of this prompt. The `RAW CONTENT` is populated dynamically from the HTTP node output via n8n expression.

```
You are a Field Reporter AI. Analyze this raw RSS/HTML content from **TechCabal**
(a leading African tech publication) and extract the single most important tech or business headline.

RAW CONTENT:
{{ $json.data ?? $json.body ?? 'No content received - source may be offline' }}

Extract the data following the strict JSON formatting constraints requested by the output parser.
```

### Field Reporter — Structured Output Parser Schema

All 5 Field Reporter agents share one Structured Output Parser enforcing this JSON schema:

```json
{
  "type": "object",
  "properties": {
    "source": {
      "type": "string",
      "description": "The name of the Kenyan media publication source."
    },
    "headline": {
      "type": "string",
      "description": "The single most critical technology, financial, or economical headline extracted from the source text."
    },
    "status": {
      "type": "string",
      "description": "Set to 'success' if content was parsed, or 'offline' if fallback text was supplied."
    }
  },
  "required": ["source", "headline", "status"]
}
```

### Chief Editor — Kamau (Primary Agent) Prompt

The Chief Editor runs with `executeOnce: true`, receiving all 5 merged summaries in one shot. It filters offline sources internally and writes the final newsletter.

```
You are Kamau, the Witty Tech Editor of Nairobi Nexus newsletter.

You have received JSON summaries from 5 Kenyan news sources:
{{ JSON.stringify($input.all().map(item => item.json.output), null, 2) }}

TASK: Write this week's Nairobi Nexus newsletter intro.

STRICT RULES:
1. Use ONLY sources where "status" equals "success". Skip all others silently.
2. Under 200 words total.
3. Weave in Kenyan slang naturally — use at least 3 of: Bazeng, Form ni gani, Poa sana, Manze, Unaona?, Hii ni kubwa, Wueh!, Msee, Mambo.
4. Tone: energetic, witty, Nairobi street-smart but still professional.
5. Format as Markdown.
6. No code blocks, no backticks, no labels, no explanations — output ONLY the newsletter.

OUTPUT STRUCTURE:
**[Bold Newsletter Title]**
[Short 2-sentence intro paragraph with slang]
- **[Headline]** — [1-sentence witty take] *(Source: [Source Name])*
[Sign-off line with slang]

Return ONLY the final newsletter text. Nothing else.
```

### Chief Editor — Structured Output Parser Schema

```json
{
  "type": "object",
  "properties": {
    "newsletter_title": { "type": "string" },
    "newsletter_body": {
      "type": "string",
      "description": "Full Markdown newsletter"
    },
    "sources_used": {
      "type": "array",
      "items": { "type": "string" }
    },
    "word_count": { "type": "integer" }
  },
  "required": ["newsletter_title", "newsletter_body", "sources_used"]
}
```

---

## 🛡️ Fault Tolerance & Error Handling

The workflow implements **three independent layers of fault tolerance**:

### Layer 1 — HTTP Safety
All 5 HTTP Request nodes have `onError: continueRegularOutput`. If any news source is offline, its branch gracefully passes empty content to the Field Reporter, which marks `status: 'offline'`. The remaining branches continue uninterrupted.

### Layer 2 — Model Fallback
The Chief Editor node has `onError: continueErrorOutput`. If the primary GPT-4o-mini model times out or errors, n8n automatically routes execution to the **Backup Editor AI Agent**, which uses GPT-4-turbo to generate the newsletter.

### Layer 3 — Per-Branch Field Reporter Fallbacks
Each Field Reporter has its own fallback LLM via OpenRouter. If the primary OpenAI call fails on any individual branch, that branch's OpenRouter model takes over — without affecting any other branch. Failure is isolated at the source level.

| Setting | Both Chief and Backup Editors |
|---|---|
| `executeOnce: true` | Ensures exactly **one** newsletter is produced per run regardless of how many items flow through the Merge node. |

---

## 🧠 Model Selection Strategy

| Agent | Primary Model | Fallback Model | executeOnce | Role |
|---|---|---|---|---|
| Field Reporters (×5) | GPT-4o-mini (OpenAI) | OpenRouter (per branch) | No | JSON headline extractor |
| Chief Editor | GPT-4o-mini (OpenAI) | GPT-4-turbo (OpenRouter) | Yes | Newsletter writer |
| Backup Editor | GPT-4-turbo (OpenRouter) | — | Yes | Emergency fallback |

### Output Constraints Enforced on Chief Editor

| Constraint | Enforcement |
|---|---|
| Under 200 words | Hard limit in prompt; `word_count` field captured in JSON output |
| Kenyan slang ×3 minimum | At least 3 of: *Bazeng, Manze, Hii ni kubwa, Wueh!, Msee, Poa sana* |
| Markdown format | `newsletter_body` field enforces Markdown structure via output parser schema |
| Success sources only | Chief filters `status === 'success'` — offline sources silently excluded |

---

## 📊 Operational Logging — NewsletterManager Google Sheet

A Google Sheet named **NewsletterManager** is updated on every run with the following columns in the **Drafts** tab:

| Date | Successful Sources | Draft Body | Model Used |
|---|---|---|---|
| `$now.setZone('Africa/Nairobi')` | `$json.output.sources_used.join(',')` | `$json.output.newsletter_body` | Primary / Fallback model name |

This provides full **governance and auditability**: if someone asks *"What happened last Friday?"* — the system can answer.

---

## 📁 Repository Contents

| File | Description |
|---|---|
| `Automated_Weekly_Newsletter_Generator.json` | Complete n8n workflow export — importable directly into any n8n instance |
| `Nairobi_Nexus_Week10_Capstone.pptx` | Week 10 final capstone presentation with architecture diagrams and screenshots |
| `Nairobi_Nexus_Week10_Capstone.pdf` | PDF export of the Week 10 capstone presentation |
| `README.md` | This file |

---

## 🚀 Getting Started

### Prerequisites

- n8n instance (self-hosted or cloud)
- OpenAI API key (GPT-4o-mini access)
- OpenRouter API key (for fallback models)
- Google Sheets credential connected in n8n
- Gmail credential connected in n8n

### Import & Deploy

1. Clone or download this repository.
2. Open your n8n instance and navigate to **Workflows → Import from File**.
3. Select `Automated_Weekly_Newsletter_Generator.json`.
4. Connect your credentials to the respective nodes (OpenAI, OpenRouter, Google Sheets, Gmail).
5. Create a Google Sheet named **NewsletterManager** with a **Drafts** tab containing columns: `Date`, `Successful Sources`, `Draft Body`, `Model Used`.
6. Activate the workflow — it will fire automatically every **Friday at 2:00 PM Africa/Nairobi**.

---

## 📋 Week 8 → Week 10 Upgrade Summary

| Area | Week 8 Prototype | Week 10 Production |
|---|---|---|
| Output parsing | Free-text newsletter output | Strict JSON schema on every agent |
| Field Reporter schema | Basic structure | `source`, `headline`, `status` enforced |
| Chief Editor schema | Article-level schema (mismatch) | `newsletter_title`, `newsletter_body`, `sources_used`, `word_count` |
| Google Sheets mapping | Wrong field key (`$json.output.newsletter`) | Correct key (`$json.output.newsletter_body`) |
| HTTP fault tolerance | `continueRegularOutput` on all HTTP nodes | Same, now with per-branch Field Reporter fallback LLMs |
| Model fallback | Chief Editor → Backup Editor | Per-branch fallback + Chief Editor → Backup Editor (3 layers total) |
| Governance | None | Full logging of date, sources, model, body on every run |

---

## 📸 Workflow Screenshots

Screenshots of the live workflow execution — including the n8n canvas, Merging Intelligence node output, Chief Editor newsletter output, Google Sheets Drafts tab, and Gmail delivery — are embedded in the capstone presentation (`Nairobi_Nexus_Week10_Capstone.pptx`).

---

*Built with ❤️ in Nairobi. Poa sana! 🇰🇪*
