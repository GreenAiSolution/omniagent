# Omniagent — Deployment Runbook

How to take a customer from "interested" to a live agent. Hosted model: **you run one
shared engine; each customer just gets a WhatsApp number.**

## Architecture at a glance

```
Customer's WhatsApp number ─▶ Omniagent (hosted, one n8n) ─▶ OpenAI (chat/voice/vision/embeddings)
        (yours, per tenant)          │                              │
                                      │                        MongoDB Atlas
                                      │                  (shared cluster, isolated per tenant)
                                      └─▶ Tools: booking / QuickBooks / ticket / CRM webhooks
                                                  (money-moving actions gated by a typed CONFIRM)
```

Everything runs on **your infrastructure** — your n8n, your OpenAI account, your
MongoDB cluster. The customer never sees any of it and never creates an account
anywhere; they get a working WhatsApp number. Their knowledge base and conversation
history are isolated to them alone inside your shared systems.

## Prerequisites (you provide, once — not per customer)

1. **n8n** instance — Cloud or self-hosted, v1.60+ (the AI Agent v2 and MongoDB Atlas
   Vector Store nodes need a recent build). One instance serves every customer.
2. **OpenAI** API key, on your billing.
3. **MongoDB Atlas** cluster, on your billing, with Vector Search enabled — one cluster,
   one `knowledge_base` collection, documents tagged per tenant (see Tenant isolation
   below).
4. A **Meta / WhatsApp Cloud API** business account you control, capable of provisioning
   a new phone number per customer (a WhatsApp Business Platform account supports many
   numbers under one business).

## Tenant isolation — the piece that makes this actually multi-tenant

**Status: not yet built.** The current `omniagent-engine/workflow-*.json` files were
built for one-tenant-per-instance (the BYO-infra model). Running many customers on one
shared n8n instance needs one real addition before the first paying customer goes live
on it:

- A **tenant resolver** step early in each workflow: given the inbound WhatsApp
  `phone_number_id`, look up which customer that number belongs to (a small config
  table — a Google Sheet or Airtable base is enough to start) and pull their business
  name, system-prompt overrides, and a `tenantId`.
- Every knowledge-base write (ingestion) and read (`MongoDB Vector Search`) tagged and
  filtered by that `tenantId` in `metadata`, so one customer's documents never surface
  in another's answers.
- Every tool webhook (booking, QuickBooks, ticket, CRM) parameterized per tenant instead
  of hardcoded per workflow file.

Until this ships, treat each early customer as its own workflow copy in the same n8n
instance (duplicate the workflow, hardcode their `tenantId` in the Mongo filter) — safe,
just not the scalable end state. Build the real resolver once you're past a handful of
customers.

## Go-live steps (per customer, once tenant isolation exists)

1. **Provision the number.** Add a new phone number to your WhatsApp Business Platform
   account for this customer.
2. **Register the tenant.** Add a row to the tenant config with their `phone_number_id`,
   business name, and any system-prompt overrides.
3. **Seed their knowledge base.** Run `ingest-knowledge-base.json` against their PDFs /
   docs / price lists / policies, tagged with their `tenantId`.
4. **Wire their tools** (Growth+ agents) — point the relevant tool node's webhook at
   their actual booking system / QuickBooks realm / ticket system / CRM, keyed by
   `tenantId` so it only ever touches their systems.
5. **Smoke test on their number.** A text question (expect a cited answer), a voice
   note, a photo, a follow-up, and — for tool agents — a write request (expect a
   `CONFIRM` prompt before anything happens).

## Safety checklist before handing over

- [ ] Knowledge base seeded; a known question returns the right answer **with a citation**.
- [ ] An out-of-scope question is **declined / escalated**, not hallucinated.
- [ ] Every write tool prompts for `CONFIRM` and only acts after the user types it.
- [ ] Vector search results checked against another tenant's documents — confirm zero
      cross-tenant leakage before this customer goes live.
- [ ] Cost guardrails set (model tier, `contextWindowLength`, `topK`) if needed.

## Regenerating / adding agents

Personas and tools live in `omniagent-engine/build-variants.mjs`. Edit the base
`workflow.json` for pipeline changes, or add a new persona block, then:

```bash
cd omniagent-engine && node build-variants.mjs
```

Every specialist regenerates from the shared base — the multimodal front-end stays
identical across the whole catalog.
