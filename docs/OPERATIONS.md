# Omniagent — Operations Manual

How to run this as a business, not a project. One operator can run ~20 Growth-tier
clients with the cadence below; capacity is the honest constraint on The System.

## Client onboarding (target: live in 5 business days)

**Day 0 — Sale closes.** Send the intake form (below). Invoice setup fee.
**Day 1 — Accounts.** Client creates/hands over: n8n instance, OpenAI key, MongoDB
Atlas cluster, Meta WhatsApp Business app. You never hold their keys in your own infra —
credentials live in *their* n8n. This is a selling point and a liability shield.
**Day 2 — Knowledge.** Load everything from the intake form via `ingest-knowledge-base.json`.
Rule of thumb: a business is well-covered at 10–30 documents (menus, price lists, policies,
FAQs, hours, service descriptions).
**Day 3 — Agents + tools.** Import the chosen workflows, wire tool webhooks
(booking system, ticket endpoint, CRM, QuickBooks realm), replace every `YOUR_*`
placeholder, set `YOUR BUSINESS` and `YOUR_REVIEW_LINK` in prompts.
**Day 4 — Rehearsal.** Run the smoke suite (below) on a test number. Fix retrieval gaps
by adding documents, not by loosening the grounding rules.
**Day 5 — Go live.** Point the production webhook, activate, send the client their
"first week" note: what to expect, how CONFIRM works, who to call.

### Intake form (send verbatim)
1. Your WhatsApp Business number and Meta Business Manager access.
2. Every document a great new employee would need on day one (menus, prices, policies, FAQs, hours).
3. The 10 questions customers ask most — with the answers you *want* given.
4. What the agent must NEVER do or promise.
5. Where bookings/tickets/leads should land (system + a webhook or a spreadsheet).
6. Your public review link (Google/Yelp) for the Reputation Loop.
7. Owner's WhatsApp number for the morning brief and escalations.

### Smoke suite (every go-live, every major change)
- [ ] Known-answer question → correct, **with citation**.
- [ ] Out-of-scope question → declines and escalates; **no invention**.
- [ ] Voice note → transcribed and answered.
- [ ] Photo of a receipt/product → read and acted on.
- [ ] PDF → extracted and answered.
- [ ] Write action → proposes summary, waits for CONFIRM, executes once, reports id.
- [ ] CONFIRM typed *without* a pending proposal → agent asks what to confirm, does nothing.
- [ ] Escalation → ticket lands with summary + number.
- [ ] Night Auditor dry run → brief arrives, numbers match the seeded reports.
- [ ] Chaser dry run on one fake invoice → correct tone for its age, log written.

## Weekly cadence (per client, ~30 min)
1. Skim the week's transcripts for wrong/awkward answers → fix with documents or one
   prompt line. Never widen what the agent may invent.
2. Check Chaser weekly report went out; sanity-check "recovered" numbers.
3. Check review count/rating movement (Reputation Loop scorecard).
4. Send the client a 5-line WhatsApp summary: volume handled, actions taken, money
   chased, anything needing their decision. **This message is why they don't churn** —
   it's the invoice justifying itself.

## Monthly cadence
- Knowledge refresh call (15 min): what changed — prices, menu, staff, policies?
- Cost check: OpenAI + WhatsApp spend vs. norm (~$10–30/mo per small client). Alert at 2×.
- Tune: retrieval `topK`, memory window, model tier if costs or quality drift.

## SLAs (publish these)
| Tier | First response (human support) | Incident (agent down) |
|---|---|---|
| Starter | 2 business days, email | next business day |
| Growth | same business day | 4 business hours |
| Scale / System | 2 business hours, named contact | 1 hour, phone |

## Escalation & failure policy
- Agent down = WhatsApp webhook failing or n8n workflow deactivated. Detection: the
  client notices, or the Night Auditor brief doesn't arrive (its absence is itself an alarm).
- On repeated hallucination reports: switch that client's agent read-only (delete write
  tools) until diagnosed. Grounding failures are always treated as sev-1 — the whole
  brand is "invent nothing."

## Churn playbook
Churn risk shows up as: owner stops reading the weekly summary → schedule the monthly
call, bring one number ("the Chaser recovered $X since you joined") and one new
document to load. If a client leaves: deactivate workflows, they keep their n8n/keys/data
(it was always theirs), delete your copies of their documents, confirm in writing.

## Unit economics (per Growth client, monthly)
- Revenue: $399 · Model+WhatsApp cost: paid by client directly ($0 to you)
- Your time at steady state: ~2.5 h/mo (weekly 30 min + monthly call) → at $50/h ≈ $125
- Gross margin ≈ **69%** at steady state; setup fee ($900) covers the ~8–10 h of onboarding.
