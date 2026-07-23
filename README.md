# Omniagent

**One agent. Every channel. Every job.**

Omniagent is a production-ready AI-agent product for real businesses. It deploys
agents that answer customers and run the back office over **WhatsApp** — understanding
**text, voice notes, images, PDFs and spreadsheets**, answering from the business's
**own knowledge** (RAG), remembering the conversation, and taking **real actions in
real tools** with **human approval on anything that moves money.**

> Live on WhatsApp today; the same agent core is channel-agnostic by design.

---

## What's in this repo

| Path | What it is |
| --- | --- |
| `index.html` | The **Omniagent marketing & sales site** — self-contained, no build step. Open it in a browser or serve the folder. This is what a prospective business sees. |
| `omniagent-engine/` | The **agent engine** — importable [n8n](https://n8n.io) workflows that run the real product (multimodal RAG pipeline + memory + gated tool actions). See its [README](omniagent-engine/README.md). |
| `docs/` | The business in writing: [sales one-pager](docs/SALES-ONEPAGER.md) · [deployment runbook](docs/DEPLOYMENT.md) · [operations manual](docs/OPERATIONS.md) · [sales playbook](docs/SALES-PLAYBOOK.md) · [The System offer spec](docs/THE-SYSTEM.md) · [the $300k design](docs/VALUATION.md). |

## The agent catalog

Every agent shares the **identical multimodal front-end** (WhatsApp trigger → type
routing → voice/image/document handling → unified prompt → agent → reply). Only the
**persona and the tools** change — so a new agent is configuration, not a rebuild.

| Agent | Workflow | What it does |
| --- | --- | --- |
| **Concierge** | `omniagent-engine/workflow.json` | Always-on first responder — grounded answers over every format, clean escalation. |
| **Support Agent** | `omniagent-engine/workflow-customer-support.json` | Front-line support for any business + ticket/escalation tool. |
| **Bookkeeper Agent** | `omniagent-engine/workflow-quickbooks-specialist.json` | Live QuickBooks Online reads + **CONFIRM-gated** invoice/payment writes. |
| **Reservations Agent** | `omniagent-engine/workflow-reservations.json` | Real availability checks + **CONFIRM-gated** booking / reschedule / cancel. |
| **Sales Qualifier** | `omniagent-engine/workflow-sales-qualifier.json` | Greets & qualifies inbound leads, books demos, pushes scored leads to the CRM. |

## The Night Shift — scheduled automations

Three services that start themselves — no inbound message required:

| Service | Workflow | Runs |
| --- | --- | --- |
| **Night Auditor** | `omniagent-engine/workflow-night-auditor.json` | Daily 06:45 — reconciles bookings/finance/threads, morning brief to the owner's WhatsApp. |
| **The Chaser** | `omniagent-engine/workflow-chaser.json` | Daily 09:00 — escalating follow-ups (day 3/7/14/21) on silent quotes & invoices; Monday recovered-revenue report. |
| **Reputation Loop** | `omniagent-engine/workflow-reputation-loop.json` | Event + 1 day — asks "how was it?"; the Support Agent routes happy replies to the review link and intercepts unhappy ones into a ticket. |

The three specialists and both new agents are **generated** from the base workflow —
edit `omniagent-engine/workflow.json`, run `node build-variants.mjs`, and every agent
inherits the change.

## Run the site locally

No build step — it's a single self-contained file (fonts from Google Fonts, nothing else).

```bash
python3 -m http.server 4600   # then open http://localhost:4600
```

`index.html` is what GitHub Pages serves.

## Why businesses buy it

- **Grounded, not guessing** — answers are retrieved from the customer's own documents
  and cited inline; if it's not in the knowledge base, the agent says so and escalates.
- **Human-gated actions** — anything irreversible is held behind a typed `CONFIRM`
  protocol baked into the agent's system prompt.
- **Their data, their tenancy** — runs on the customer's own n8n, OpenAI key and MongoDB
  Atlas cluster. No middleman, no training on their conversations.
- **Live in a weekend** — one importable workflow, ~20 minutes of credential wiring.

See [`docs/SALES-ONEPAGER.md`](docs/SALES-ONEPAGER.md) for the pitch and
[`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) for the go-live runbook.
