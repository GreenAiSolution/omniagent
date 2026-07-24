# Omniagent Engine — WhatsApp AI Agents (RAG + Memory + Gated Actions)

The engine behind [Omniagent](../README.md): importable **n8n** workflows that turn a
WhatsApp Business number into a smart, context-aware **agent**. It understands **text,
voice notes, images, PDFs and spreadsheets**, answers with **RAG over MongoDB Atlas
Vector Search**, remembers the conversation, takes **real actions in real tools** (with
money-moving writes gated behind a typed `CONFIRM`), and replies back on WhatsApp —
end to end.

Every agent in the catalog shares one identical multimodal front-end; only the persona
and tools change, so a new agent is configuration, not a rebuild.

> Stack: **n8n + OpenAI + WhatsApp Cloud API + MongoDB Atlas**.

```
WhatsApp Trigger ─▶ Route Types ─┬─ Text ───────────────────────────────▶ Map text prompt ─────────┐
                                 │                                                                  │
                                 ├─ Voice ─▶ Get URL ▶ Download ▶ Transcribe ▶ Map voice prompt ────┤
                                 │                                                                  │
                                 ├─ Image ─▶ Get URL ▶ Download ▶ Analyze (vision) ▶ Map image ─────┤
                                 │                                                                  ▼
                                 └─ Doc ──▶ Get URL ▶ Download ▶ Route ext ▶ Extract ▶ Map doc ──▶ Knowledge Base Agent ─▶ Send Response
                                                          (PDF / XLS / XLSX)                         │  ▲        ▲
                                                                                    OpenAI Chat Model ┘  │        │
                                                                                         Simple Memory ──┘        │
                                                                          MongoDB Vector Search (tool) ──────────┘
                                                                                     └─ Embeddings OpenAI
```

## Files

| File | What it is |
| --- | --- |
| `workflow.json` | **Concierge** — the base agent (general multimodal assistant). Import this into n8n. |
| `workflow-quickbooks-specialist.json` | **Bookkeeper Agent** — same pipeline + live QuickBooks Online query/report tools + gated invoice/payment writes. |
| `workflow-customer-support.json` | **Support Agent** for any business — RAG + escalation/ticket tool. |
| `workflow-reservations.json` | **Reservations Agent** — availability check + gated booking / reschedule / cancel tools. |
| `workflow-sales-qualifier.json` | **Sales Qualifier** — RAG + lead capture (CRM) + meeting-booking tools. |
| `workflow-night-auditor.json` | **Night Auditor** — cron 06:45: pulls the day's bookings/finance/thread reports, composes a grounded one-page morning brief, sends it to the owner's WhatsApp. |
| `workflow-chaser.json` | **The Chaser** — cron 09:00: escalating follow-ups (day 3/7/14/21) on silent quotes & invoices, stops on reply, logs every nudge; Monday 08:00 recovered-revenue report. |
| `workflow-reputation-loop.json` | **Reputation Loop** — webhook on job completion + 1-day wait: personal "how was it?" ask. Replies are handled by the Support Agent's feedback protocol (review link for happy, `reputation-intercept` ticket for unhappy). |
| `ingest-knowledge-base.json` | Companion workflow to load PDFs/docs into the vector store (RAG is empty until you run this). |
| `build-variants.mjs` | Regenerates the four specialized agents from `workflow.json` (persona + tools only differ). |
| `workflow-voice-agent.json` | **Voice Agent** *(in build)* — answers phone calls on the same grounded/gated core, spoken instead of typed. |
| `workflow-rcs-channel.json` | **RCS Channel** *(in build)* — the Concierge core over RCS Business Messaging instead of WhatsApp. |
| `workflow-checkout.json` | **Concierge Checkout** *(in build)* — takes payment via a Stripe Payment Link, gated by the same CONFIRM protocol as a booking. |

Night-shift setup notes: replace the `YOUR_*` webhook URLs with endpoints that return
your data (an n8n webhook querying your DB is fine), set `YOUR_OWNER_WHATSAPP_NUMBER`,
`YOUR_PHONE_NUMBER_ID` and `YOUR_REVIEW_LINK`, and remember outbound messages beyond
WhatsApp's 24-hour window must use approved templates — register one per service.

All five agents share the **identical multimodal front-end** (WhatsApp trigger →
type routing → voice/image/document handling → unified prompt → agent → reply). Only the
**agent's system prompt and tools** change. Edit `workflow.json` and run `node build-variants.mjs`
to propagate pipeline changes to every specialist.

## "Huge context on anything"

- **Chat model:** `gpt-4.1` (≈1M-token context) so the agent can reason over long documents and long histories.
- **Memory window:** `contextWindowLength: 100` turns of conversation per user.
- **Retrieval:** `topK: 8` chunks from MongoDB Atlas, embedded with `text-embedding-3-large`.
- Any modality (voice→transcript, image→vision analysis, doc→extracted text) is normalized into a single `chatInput` field, so the same big-context agent handles all of them.

Change the model in **OpenAI Chat Model**, **OpenAI Analyze Image**, and **Embeddings OpenAI** nodes if you prefer a different one.

---

## Setup (about 20 minutes)

### 1. Prerequisites
- An **n8n** instance (Cloud or self-hosted, v1.60+ recommended — the AI Agent v2 and MongoDB Atlas Vector Store nodes need a recent build).
- An **OpenAI** API key.
- A **MongoDB Atlas** cluster (M0 free tier works) with Vector Search enabled.
- A **Meta / WhatsApp Cloud API** app (WhatsApp Business Platform).

### 2. Create the credentials in n8n
Under **Credentials → New**, create:
1. **OpenAI** — your API key. Used by the chat model, transcription, vision, and embeddings.
2. **MongoDB Atlas** (or the generic MongoDB credential) — connection string to your cluster.
3. **WhatsApp API** — the Cloud API access token + business account. Used by the trigger and the media/send nodes.
4. **WhatsApp Trigger** (OAuth/app secret) — for verifying incoming webhooks.
5. **Header Auth** (generic) — name it e.g. `WhatsApp Media Bearer`, with header
   `Authorization` = `Bearer YOUR_WHATSAPP_PERMANENT_TOKEN`. The three **Download** HTTP
   nodes use this to fetch the media binary from Meta's CDN.

### 3. Prepare MongoDB Atlas Vector Search
1. Create a database (e.g. `omniagent`) and a collection named **`knowledge_base`**.
2. In **Atlas → Search → Create Search Index → JSON editor**, create a **Vector Search** index named **`vector_index`** on that collection:
   ```json
   {
     "fields": [
       { "type": "vector", "path": "embedding", "numDimensions": 3072, "similarity": "cosine" },
       { "type": "filter", "path": "metadata.source" }
     ]
   }
   ```
   > `numDimensions: 3072` matches `text-embedding-3-large`. If you switch to
   > `text-embedding-3-small`, use `1536`.

### 4. Import the workflows
- In n8n: **Workflows → Import from File** → select `workflow.json`, then again for `ingest-knowledge-base.json`.
- Open each node that shows a red credential badge and pick the credential you created in step 2. (Imported templates never carry secrets — you always assign these once.)
- In **MongoDB Vector Search** / **MongoDB Vector Store (Insert)**, confirm the database is selected and the collection is `knowledge_base` and index is `vector_index`.

### 5. Seed the knowledge base
- Open **`ingest-knowledge-base.json`**, open the **Upload Form** trigger, click **Test / Open form URL**, and upload your PDFs/docs — or wire it to Google Drive / a URL list for bulk loads.
- Each file is chunked (1200 chars, 200 overlap), embedded, and stored with a `source` in metadata for citations.

### 6. Connect WhatsApp
1. In **WhatsApp Trigger**, copy the webhook URL n8n gives you (Production URL once the workflow is **Active**).
2. In the Meta App dashboard → **WhatsApp → Configuration**, set that URL as the **Callback URL**, add your **Verify Token**, and subscribe to the **`messages`** field.
3. Add your test phone number as a recipient (or move the app to production).
4. **Activate** the `workflow.json` workflow.

### 7. Test
Message your WhatsApp number:
- Plain text question → grounded answer with `[source: …]` citations.
- A voice note → transcribed, then answered.
- A photo (with or without caption) → described and answered.
- A PDF / XLSX → extracted and answered.
- Ask a follow-up ("and what about the second point?") → memory keeps context.

---

## How it maps to the blueprint

| Blueprint node | This workflow |
| --- | --- |
| WhatsApp Trigger | `WhatsApp Trigger` (subscribes to `messages`) |
| Route Types (Rules) | `Route Types` switch → Text / Voice / Image / Document |
| Gets … URL (mediaUrlGet) | `Get WhatsApp Voice/Image/Document URL` |
| Download Voicemail/Image/Document | HTTP `Download …` nodes (Header-Auth bearer) |
| OpenAI Transcribe | `OpenAI Transcribe Recording` (Whisper) |
| OpenAI Analyze Image | `OpenAI Analyze Image` (gpt-4.1 vision) |
| Map file extensions / Route Document Types | `Map file extensions` + `Route Document Types` |
| Extract from PDF / XLS / XLSX | three `Extract from …` nodes |
| Map … prompt / Map JSON | the `Map … prompt` Set nodes → unified `chatInput` |
| Knowledge Base Agent (Chat / Memory / Tool) | `Knowledge Base Agent` + `OpenAI Chat Model` + `Simple Memory` + `MongoDB Vector Search` |
| Embeddings OpenAI | `Embeddings OpenAI` |
| Send Response | `Send Response` (WhatsApp text) |

---

## The specialist agents

All four import and set up **exactly like the base workflow** (steps 1–7 above) — same
WhatsApp, OpenAI, MongoDB and Header-Auth credentials. Replace **`YOUR BUSINESS`** in the
agent's system message with your company name.

### Bookkeeper Agent — `workflow-quickbooks-specialist.json`
A live bookkeeping assistant. On top of RAG it gets two QuickBooks Online tools the agent
calls on demand:

- **`QuickBooks_Query`** — runs read-only QuickBooks SQL (invoices, customers, bills,
  vendors, payments, accounts). e.g. *"Which invoices are overdue?"* →
  `SELECT * FROM Invoice WHERE Balance > '0' ORDERBY DueDate`.
- **`QuickBooks_Report`** — pulls ProfitAndLoss, BalanceSheet, AgedReceivables,
  AgedPayables, CustomerBalance for a date range.

**Write actions (gated):**
- **`QuickBooks_Create_Invoice`** and **`QuickBooks_Record_Payment`** let the bot write to
  QuickBooks — but only behind a **confirmation protocol** baked into the system prompt: the
  agent must first summarize the exact change (customer, amount, line item, due date) and
  ask the user to reply **`CONFIRM`**. It calls the write tool only after seeing that
  `CONFIRM` in the conversation memory, and passes the user's word into a required
  `userConfirmation` field. One `CONFIRM` = one write; missing details are asked for first.
  To disable writes entirely, delete those two nodes — reads still work.

**Extra setup:**
1. In n8n create a **QuickBooks Online OAuth2** credential (`quickBooksOAuth2Api`) — client
   id/secret from your Intuit developer app, and authorize it.
2. In both QuickBooks tool nodes, replace **`YOUR_REALM_ID`** in the URL with your
   QuickBooks **Company (realm) ID** (Intuit → your app → the connected company).
3. For the **production** Intuit environment the host is `quickbooks.api.intuit.com`
   (already set); for the **sandbox** use `sandbox-quickbooks.api.intuit.com`.
4. **Write tools are included** (`QuickBooks_Create_Invoice`, `QuickBooks_Record_Payment`)
   and gated by the CONFIRM protocol in the system prompt. They use the same OAuth2
   credential and realm id. Test in the Intuit **sandbox** first. Remove both nodes if you
   want a strictly read-only bot.

### Support Agent — `workflow-customer-support.json`
A business-agnostic front-line agent. RAG-grounded, cites sources, and escalates cleanly:

- **`create_support_ticket`** — the agent calls this to log/escalate an issue (summary,
  customer number, category, priority) when the KB can't resolve it or a human is needed.

**Extra setup:**
1. Load your FAQs, policies, product info and hours via `ingest-knowledge-base.json`.
2. In **Create Support Ticket**, replace **`YOUR_TICKET_WEBHOOK_URL`** with your endpoint —
   a Zendesk/Freshdesk/HubSpot webhook, a Slack incoming webhook, an n8n webhook that opens
   a ticket, or a Google Sheet append. Adjust the JSON body to match your system's fields.
3. If you don't want escalation yet, just delete that node — the agent still answers from RAG.

### Reservations Agent — `workflow-reservations.json`
A bookings concierge for restaurants, salons, clinics and tours. It checks real
availability before promising a slot, then books, reschedules or cancels — each
write **gated by the CONFIRM protocol** (summarize the exact booking, act only after
the user types `CONFIRM`).

- **`check_availability`** — GET open slots for a date / party size / service.
- **`create_booking` / `reschedule_booking` / `cancel_booking`** — gated writes that
  POST to your booking system with a required `userConfirmation` field.

**Extra setup:**
1. Point **`YOUR_AVAILABILITY_WEBHOOK_URL`** at an endpoint that returns open times
   (your reservation system's API, or an n8n webhook that queries it).
2. Point **`YOUR_BOOKING_WEBHOOK_URL`** at an endpoint that creates/updates bookings;
   it receives an `action` field (`create` | `reschedule` | `cancel`) plus the details.
3. Remove the write nodes for an availability-only bot.

### Sales Qualifier — `workflow-sales-qualifier.json`
An inbound SDR. It answers product/pricing questions from RAG, qualifies the lead
naturally (need, budget, timeline, role), then logs it to the CRM and/or books a demo.

- **`capture_lead`** — POST a scored lead (`cold`/`warm`/`hot`) + qualification summary to your CRM.
- **`book_meeting`** — POST a requested time to your scheduling endpoint.

**Extra setup:**
1. Load your product info, pricing and FAQs via `ingest-knowledge-base.json`.
2. Set **`YOUR_CRM_WEBHOOK_URL`** (HubSpot/Salesforce/Pipedrive webhook, or a Sheet append)
   and **`YOUR_SCHEDULING_WEBHOOK_URL`** (Calendly/Cal.com/Google Calendar via n8n).
3. Delete either tool node to run capture-only or booking-only.

---

## What's next — in build

Three workflows extending the catalog beyond WhatsApp text/voice-note/image/document.
They're structurally complete and importable — same node/connection conventions as the
five agents above — but need channel-specific accounts wired in before they're a
customer-facing product, so they ship here as **in build**, not on the public price list.

### Voice Agent — `workflow-voice-agent.json`
Answers phone calls placed to the business's number. n8n has no native telephony
trigger, so this uses the standard pattern for it: a **Webhook** node receiving Twilio
Voice's incoming-call webhook, a `<Gather input="speech">` TwiML response that lets
Twilio do the speech-to-text itself, the same core agent (retrieval, memory, grounded
answers) reasoning over the transcript, and a second TwiML response that speaks the
reply back and re-opens the mic for the next turn. Anything irreversible stays behind
the same CONFIRM pattern as every other agent — spoken instead of typed ("say the word
confirm").

**Extra setup:** a Twilio phone number with Voice webhooks pointed at this workflow's
`/voice-incoming` n8n webhook URL (both the initial call and the `action` callback on
`<Gather>` hit the same endpoint).

### RCS Channel — `workflow-rcs-channel.json`
The identical Concierge core — retrieval, memory, grounded answers — over RCS Business
Messaging instead of WhatsApp. n8n has no native RCS node, so inbound messages arrive
via **Webhook** from an RCS-capable BSP (Infobip or Twilio both support it) and replies
go out via an **HTTP Request** node against that BSP's send-message REST API.

**Extra setup:** an RCS agent registered with your BSP of choice, its webhook pointed at
`/rcs-inbound`, and `YOUR_RCS_BSP_HOST` / `YOUR_BSP_API_KEY` filled in on the **Send RCS
Reply** node.

### Concierge Checkout — `workflow-checkout.json`
Extends the agent with a real payment capability, gated exactly like the Bookkeeper's
QuickBooks writes: the agent summarizes the charge, waits for a typed `CONFIRM`, and
only then calls **`create_payment_link`** (n8n's native Stripe node) to generate a
Stripe Payment Link and send it over WhatsApp. A second flow in the same file — a
**Stripe Trigger** on `checkout.session.completed` — notifies both the customer and the
business owner the moment it's paid.

**Extra setup:** a Stripe account connected via n8n's Stripe credential, and
`YOUR_PHONE_NUMBER_ID` / `YOUR_OWNER_NUMBER` filled in on the two **Notify** nodes.

---

## Notes & gotchas
- **Verify node params on import.** n8n occasionally renames parameters between node
  versions. If a node shows a validation warning after import (most likely the WhatsApp
  media `mediaUrlGet` field or the Extract-from-File operation), open it and re-pick the
  operation from the dropdown — the wiring stays intact.
- **WhatsApp free-form replies** are only allowed inside the 24-hour customer service
  window. Outside it you must send an approved template; add a template message node if
  you need proactive outreach.
- **Voice**: WhatsApp delivers voice notes as `.ogg` (Opus). Whisper accepts it directly.
- **Cost control**: lower `contextWindowLength`, `topK`, or switch to `gpt-4.1-mini` /
  `text-embedding-3-small` (remember to change the index `numDimensions` to 1536).
- **Long documents**: extraction + big context handles most files; for very large PDFs,
  route them through the ingestion workflow instead so they're chunked into RAG rather
  than stuffed into one prompt.
