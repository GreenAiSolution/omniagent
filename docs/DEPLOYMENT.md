# Omniagent — Deployment Runbook

How to take a customer from "interested" to a live agent. Budget ~20 minutes of
credential wiring per agent; the pipeline itself is already built.

## Architecture at a glance

```
WhatsApp number ─▶ n8n (Omniagent engine) ─▶ OpenAI (chat/voice/vision/embeddings)
                        │                        │
                        │                  MongoDB Atlas (your knowledge, vector search)
                        │
                        └─▶ Tools: booking / QuickBooks / ticket / CRM webhooks
                                    (money-moving actions gated by a typed CONFIRM)
```

Everything runs in **the customer's own tenancy** — their n8n, their OpenAI key,
their MongoDB cluster. Omniagent is the workflow + the knowledge, not a middleman.

## Prerequisites (the customer provides)

1. **n8n** instance — Cloud or self-hosted, v1.60+ (the AI Agent v2 and MongoDB Atlas
   Vector Store nodes need a recent build).
2. **OpenAI** API key.
3. **MongoDB Atlas** cluster (M0 free tier works) with Vector Search enabled.
4. **Meta / WhatsApp Cloud API** app with a Business number.

## Go-live steps

1. **Import the agent.** In n8n → *Workflows → Import from File*, pick the workflow for
   the agent you're deploying from [`omniagent-engine/`](../omniagent-engine):
   - `workflow.json` — Concierge (base)
   - `workflow-customer-support.json` — Support
   - `workflow-quickbooks-specialist.json` — Bookkeeper
   - `workflow-reservations.json` — Reservations
   - `workflow-sales-qualifier.json` — Sales Qualifier
   Also import `ingest-knowledge-base.json` (used to load the knowledge base).

2. **Create credentials** (once): OpenAI, MongoDB Atlas, WhatsApp API, WhatsApp Trigger,
   and a generic **Header Auth** named e.g. `WhatsApp Media Bearer`
   (`Authorization = Bearer <permanent token>`) for the media-download nodes. Assign
   them to any node showing a red credential badge.

3. **Set up Vector Search.** Create a database (e.g. `omniagent`) and a `knowledge_base`
   collection, then a Vector Search index named `vector_index`
   (`numDimensions: 3072` for `text-embedding-3-large`). Full JSON is in the engine README.

4. **Seed the knowledge base.** Open `ingest-knowledge-base.json`, open the Upload Form
   trigger, and upload the customer's PDFs / docs / price lists / policies. Each file is
   chunked, embedded and stored with a `source` for citations.

5. **Wire the agent's tools** (Growth+ agents):
   - **Bookkeeper** — QuickBooks OAuth2 credential + replace `YOUR_REALM_ID` in the tool URLs.
   - **Reservations** — point `YOUR_AVAILABILITY_WEBHOOK_URL` and `YOUR_BOOKING_WEBHOOK_URL`
     at the booking system (or an n8n webhook that reads/writes it).
   - **Support** — set `YOUR_TICKET_WEBHOOK_URL` to Zendesk/Freshdesk/HubSpot/Slack/Sheets.
   - **Sales Qualifier** — set `YOUR_CRM_WEBHOOK_URL` and `YOUR_SCHEDULING_WEBHOOK_URL`.
   - Replace **`YOUR BUSINESS`** in the agent's system message with the company name.

6. **Connect WhatsApp.** Copy the Production webhook URL from the WhatsApp Trigger into
   Meta → WhatsApp → Configuration (Callback URL + Verify Token), subscribe to `messages`,
   add a test recipient, then **Activate** the workflow.

7. **Smoke test.** Message the number: a text question (expect a cited answer), a voice
   note, a photo, a PDF, a follow-up ("and the second one?"), and — for tool agents — a
   write request (expect a `CONFIRM` prompt before anything happens).

## Safety checklist before handing over

- [ ] Knowledge base seeded; a known question returns the right answer **with a citation**.
- [ ] An out-of-scope question is **declined / escalated**, not hallucinated.
- [ ] Every write tool prompts for `CONFIRM` and only acts after the user types it.
- [ ] To disable actions entirely, delete the tool nodes — reads still work.
- [ ] Cost guardrails set (model tier, `contextWindowLength`, `topK`) if needed.

## Regenerating / adding agents

Personas and tools live in `omniagent-engine/build-variants.mjs`. Edit the base
`workflow.json` for pipeline changes, or add a new persona block, then:

```bash
cd omniagent-engine && node build-variants.mjs
```

Every specialist regenerates from the shared base — the multimodal front-end stays
identical across the whole catalog.
