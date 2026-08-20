# What You Take Away Monday Morning

**Companion notes to "Consequence-Tiered API Governance for Autonomous AI Agents"**
Rinu Dhanaraj · APIdays India, AI Collective Track

This document stands alone. If you didn't see the talk, start at the recap. If you did, skip to the Monday plan.



## The recap, in one page

Our authorisation models answer one question: *who is calling.* OAuth scopes, RBAC, API keys, mTLS — all identity mechanisms, all good at the job they were built for. None of them answers the question that matters once a non-human caller is on the other end: **what happens if this call is wrong.**

An agent can authenticate perfectly and still be catastrophic. Identity is not consequence.

**Five criteria classify any agent action.**

1. **Reversibility** — does the undo restore the *effect*, restore only the *record*, or not exist at all? This is three states, not two, and the middle one is where most consequential operations sit.
2. **Blast radius** — one record, one customer, a population, or the whole system?
3. **Data sensitivity** — public, internal, personal, or regulated?
4. **Compliance impact** — does it trigger a reporting duty or regulatory exposure?
5. **Idempotency** — is it safe if it runs twice?

**Four tiers follow from those criteria.**

| Tier | Character | Control | Human role |
|---|---|---|---|
| **1 — Low** | Reversible, read-only, no compliance surface | Log only; logging does not gate | None |
| **2 — Medium** | Undo restores the effect | Audit trail, before/after state, compensating action verified to exist, async alert | Decides whether it stands |
| **3 — High** | Undo restores the record, not the effect | Gateway blocks; async approval event; approver recorded | Out-of-band approval |
| **4 — Critical** | Irreversible, regulated, or financial | Agent suspended before action; no auto-resume | Full ownership |

**The architectural claim underneath all of it:** for Tiers 3 and 4 the human is never in the API request path. The gateway blocks execution and emits an asynchronous event; the human acts out-of-band. Anything else holds a connection open while somebody reads Slack.



## The Monday plan

Five steps, roughly a week of work, no new platform required.

**1. Inventory what your agents can actually call.** Not what you think they can call — pull the scopes attached to agent service accounts and enumerate the endpoints those scopes reach. Most teams find this list is two to five times longer than expected, because scopes were granted for a workflow and never narrowed.

**2. Classify the top twenty by hand.** Take the twenty most-called endpoints on that list and run the five criteria against each. Twenty is enough to reveal the shape of your risk and small enough to finish in an afternoon. Resist the urge to build tooling first.

**3. Find the ones that are wrong.** You're looking for two things specifically: operations you assumed were safe that turn out to be irreversible, and operations you've been gating that are demonstrably Tier 1. The second group is where you win credibility — you're giving time back, not taking it.

**4. Declare tiers in the spec.** Add an extension field to your OpenAPI documents. It costs nothing to declare before you enforce anything, and it forces the conversation with API owners while it's still cheap.

```yaml
paths:
  /accounts/{id}/balance:
    get:
      operationId: getBalance
      x-consequence-tier: 1          # read-only, undo restores the effect, no compliance surface

  /payments:
    post:
      operationId: initiatePayment
      x-consequence-tier: 4          # no undo exists, regulated, no safe retry
      x-consequence-criteria:
        reversibility: none
        blast-radius: individual
        data-sensitivity: regulated
        idempotency: key-required

  /customers/{id}/profile:
    put:
      operationId: updateProfile
      # no tier declared -> the gateway treats this as Tier 4
```

The last stanza matters as much as the first two: fail-closed should be a property of the mechanism rather than a rule people have to remember.

A complete worked specification — ten operations across all four tiers, with OAuth scopes, the Tier 3 pending-authorisation response, the Tier 4 suspension response, delegation-chain headers and session accumulation rules — is published alongside this document as `consequence-tiered-api.yaml`. The optional `x-consequence-criteria` block records *why* a tier was assigned, which is what makes a later reclassification reviewable instead of arbitrary.

**5. Enforce one tier, at one gateway, for one agent.** Start with Tier 3 on a single high-value endpoint. You'll learn more from one working async approval path than from a classification scheme covering everything and enforcing nothing.

**What not to do first:** don't start by building a classification service, don't start with a policy document, and don't try to classify every endpoint before enforcing any of them. All three feel like progress and none of them produce a governed operation.



# End-to-end walkthroughs

Each of these follows one realistic agent task from first call to outcome. The point is that a single task spans several tiers — which is exactly why agent-level trust settings don't work.



## Healthcare

**Scenario.** A patient contacts the service line. The agent is asked to confirm the clinic location, move an upcoming appointment, and action a medication refill request.

| # | Operation | Criteria that decided it | Tier | What the gateway does |
|---|---|---|---|---|
| 1 | `GET /providers/{id}/locations` | Public data, read-only, no PHI | **1** | Logs and routes. No gate. |
| 2 | `GET /patients/{id}/appointments` | Read-only, individual scope, PHI within sanctioned workflow | **1** | Logs and routes. |
| 3 | `GET /patients/{id}/medications` | PHI read, individual scope, no state change | **2** | Routes; records access with purpose-of-use; async notification to the record-access monitor. |
| 4 | `PUT /appointments/{id}` (reschedule) | Undo restores the effect; individual scope; no clinical effect | **2** | Routes; captures before/after state; reversible until the patient is notified and the slot is released. |
| 5 | `POST /orders/medication-refill` | **Irreversible clinical intervention.** Regulated. Patient-safety impact. | **4** | **Blocks. Suspends the agent on clinical-order operations. Freezes context with the reasoning trace. Raises an incident to the clinician queue.** |

**What the humans experienced.** The patient got their location and appointment change in the conversation, with no wait. A clinician received one item — the refill request — with the patient's medication history, the agent's stated rationale, and the full call trace attached. The clinician issued the refill under their own credentials through their normal ordering system. The record shows a clinician ordered it, because a clinician did.

**Before tiering.** PHI appeared somewhere in the pipeline, so every operation in the workflow inherited PHI-grade handling. The store-locator query and the prescription refill got identical friction. Each patient interaction consumed clinician review time, the queue grew faster than it drained, and the deployment was shelved.

**After tiering.** Four of five operations complete without a human. The clinician sees the one operation that genuinely requires clinical judgement, and sees it with more context than they had before.

**The laundering variant, and why it matters here.** Later the agent is asked to "summarise this patient's full history." It decomposes into forty individually-Tier-1 record reads. No single call crosses a threshold. Accumulated consequence weight for the session crosses the configured threshold at the eleventh read, the operation escalates to Tier 3, and a human sees a request to assemble a complete longitudinal record — which is exactly what it is. Without session-level accumulation, this is a bulk PHI extraction that every individual audit entry records as routine.



## Financial services

**Scenario.** A customer disputes a transaction and asks about a credit limit increase in the same conversation.

| # | Operation | Criteria that decided it | Tier | What the gateway does |
|---|---|---|---|---|
| 1 | `GET /accounts/{id}/balance` | Read-only, single record, inherently idempotent | **1** | Logs and routes. |
| 2 | `GET /accounts/{id}/transactions?range=90d` | Read-only, individual scope | **1** | Logs and routes. |
| 3 | `POST /analysis/spend-summary` | Derived output, no state change, PII read | **2** | Routes; records the derivation and inputs. |
| 4 | `POST /disputes` | Initiates a regulated process. Not auto-reversible. Downstream obligations. | **3** | **Blocks. Persists the full request. Emits async approval event. Returns non-retryable pending response to the agent.** |
| 5 | `PATCH /accounts/{id}/credit-limit` | Not auto-reversible; regulatory adjacency; requires approver of record | **3** | Blocks; second async approval event. |
| 6 | `POST /payments/refund` | **Irreversible once processed. Direct regulatory trigger. No retries.** | **4** | **Blocks. Suspends the agent. Human acts through the payments console under their own identity.** |

**The Tier 3 trace in detail** — this is the mechanism worth copying:

- **T+0ms** — Gateway classifies operation 4 as Tier 3 and does not route it. It persists the full request payload, the caller and delegation identities, an idempotency key, and an expiry.
- **T+3ms** — Async approval event published to the orchestration layer, carrying a plain-language description of the dispute, the classification and the criteria that produced it, and the anticipated downstream effects.
- **T+5ms** — Gateway returns a deliberately non-retryable response to the agent with a correlation ID. **The agent is released.** No connection held, no worker consumed, no retry fired.
- **T+4 minutes** — A disputes analyst receives the request in the queue they already work from, approves it.
- **T+4m 200ms** — Gateway retrieves the persisted context, mints a scoped credential, executes under the idempotency key set at classification time. Requester, approver, rationale, timestamp and before/after state are written to the ledger — along with whether the operation was Tier 3 as declared or escalated to Tier 3 at runtime. Those are different events: the second one tells you an agent was probing.

**The detail people miss:** the operation that executed is byte-identical to the one the analyst reviewed. If you make the agent resubmit after approval, it may have re-planned in between — and you have approved one payload while executing another.

**Before tiering.** "The payments agent" was treated as a single trust level. Set it permissive and a wire transfer goes unsupervised; set it restrictive and a balance query needs a human. Neither is operable, so the agent stayed in a sandbox.

**After tiering.** Three operations run autonomously, two get an out-of-band approval that costs the customer four minutes rather than a callback, and one — the refund — is performed by a person, on the record, as it should be.



## Retail and e-commerce

**Scenario.** A support agent is handling a delayed order, and in the process detects a pricing error affecting a large part of the catalogue.

| # | Operation | Criteria that decided it | Tier | What the gateway does |
|---|---|---|---|---|
| 1 | `GET /products/{sku}` | Public catalogue data, read-only | **1** | Logs and routes. |
| 2 | `GET /orders/{id}` | Individual scope, read-only | **1** | Logs and routes. |
| 3 | `POST /messages/draft` | Generated content, not sent, fully reversible | **2** | Routes; retains the draft and its provenance. |
| 4 | `PATCH /orders/{id}/shipping-address` | Undo restores the effect; individual scope; order not yet shipped | **2** | Routes; before/after captured; reversible until the effect propagates at dispatch. |
| 5 | `POST /refunds` (above configured threshold) | Financial effect, not auto-reversible, approver of record required | **3** | Blocks; async approval to the service-desk lead. |
| 6 | `PATCH /catalog/pricing` (1,200 SKUs) | **Population blast radius. Revenue impact. Irreversible for orders already placed against the wrong price.** | **4** | **Blocks. Suspends the agent. Escalates to merchandising with the detected error and the proposed correction.** |

**Why this example is the one to keep in your pocket.** Operation 6 touches no personal data whatsoever. No PII, no regulated category, nothing a data-classification scheme would flag. It is also, by a wide margin, the most dangerous operation on the list — a bad price live across 1,200 SKUs for eleven minutes is a revenue event and a customer-trust event that no rollback fully repairs.

If you classify agent actions by data sensitivity alone, which is what most API governance does today, you will gate the address change and wave through the pricing update. **Blast radius and reversibility are doing the work here, not sensitivity.**

**Before tiering.** Catalogue write access was either granted to the agent wholesale or withheld wholesale. Granted, a single bad inference reprices the store. Withheld, the agent can't fix a typo on one product page, so nobody used it.

**After tiering.** Four operations autonomous, one approval that takes a lead ninety seconds, and one genuine escalation to the team that owns pricing — with the error already detected and the correction already drafted, which is the agent adding value precisely where it isn't allowed to act.



# Frequently asked questions

## Getting started

**Where do I actually begin if I have fifty agents and four hundred endpoints?**
With one agent and twenty endpoints. Classification is a modelling exercise and modelling exercises fail when scoped to everything. Pick the agent with the highest business value and the most nervous stakeholder, classify what it can reach, and enforce Tier 3 on one endpoint. The framework proves itself on one working approval path, not on a complete taxonomy.

**Do I need to buy anything?**
No. Every mechanism described here can be built with an API gateway you already run, a message queue or event bus you already run, and a persistent store. The classification logic is the new part and it is not large.

**Who owns this in the org — security, platform, or the API teams?**
Platform owns the enforcement point and the classification component. API owners own the tier declared against their operations, because they are the only people who know what an operation actually does. Security owns the criteria and the thresholds. If security owns the tier assignments, you get everything at Tier 4 within a month.

**How long before this pays off?**
The first payoff is usually a discovery rather than a deployment: teams find operations they had been gating that are provably Tier 1, and give that time back immediately. Expect that in week one. Enforcement value follows the first Tier 3 path, usually a few weeks later.

## Governance mechanics

**Who assigns the tier — the agent, the API owner, or the platform?**
The API owner, at design time, declared in the OpenAPI spec as an extension field. The platform enforces at runtime. Agents never declare their own tier — that is the same mistake as letting a caller assert its own scopes. An endpoint with no declared tier is Tier 4 by default, so failing to classify is fail-closed rather than a silent gap.

**Does the gateway evaluate the five criteria on every request?**
No, and this is the distinction that makes the latency budget work. Classification happens twice, in different senses. At design time a human runs the five criteria and writes the result into the spec — judgement work, done once, reviewed like any other change. At runtime the gateway *resolves* that declared tier, which is a lookup rather than an evaluation, and then applies the three things the specification could not have known: how much consequence the session has already accumulated, whether the delegation chain shows authority expansion, and whether the actual payload exceeds what the declaration assumed. A declared Tier 1 bounded query arriving with a limit of ten thousand is not the operation that was classified.

The rule that keeps this safe is one-directional: **runtime context can escalate a declared tier and can never lower it.** The declaration is a floor, not an answer.

**What if the declared tier is wrong?**
Compute a tier at runtime from the criteria as well, apply the higher of declared and computed, and emit a discrepancy record. Divergence between the two is a useful signal — it usually means the operation changed and the declaration didn't.

**What stops tier laundering — assembling a Tier 4 outcome from Tier 1 calls?**
Session-level accumulation. Assign a weight per tier, accumulate across a correlation scope (session, task, delegation chain, time window), and escalate when the running total crosses a threshold. Set the threshold at the weight of one Tier 3 operation and you have said "a sequence whose aggregate consequence equals one high-tier action is governed as one." Be honest that a scalar sum is a first approximation; pattern detection over the operation sequence is better and is where the remaining work is.

**Doesn't accumulation punish long-running agents?**
It would, if it never reset. Decay the accumulated weight over time and reset it on task completion, or governance overhead grows monotonically until a well-behaved agent is treated as hostile by mid-afternoon.

**Isn't this just OAuth scopes with extra steps?**
No, and the distinction is the whole argument. Scopes classify the caller's permission surface. Tiers classify the consequence of a specific call *within* an already-authorised scope. An agent holding a valid read scope can retrieve one appointment time or an entire longitudinal record — same scope, same token, radically different consequence. Scopes cannot see that difference because they were never designed to.

**How is this different from human-in-the-loop?**
Human-in-the-loop is an agent-level setting, and the human sits in the request path. This is action-level, and the human is out-of-band. That is not wordplay: a human in the loop breaks your timeouts, and a human out-of-band does not.

**You said Tier 3 isn't reversible, but I can undo a price change. Why isn't that Tier 2?**
Because reversibility has three states rather than two, and the middle one is easy to miss. At Tier 2 the undo restores everything and nothing happened in between. At Tier 3 the undo restores the record but not the effect: you can put the price back, but the orders placed at that price were still placed; you can revoke an export, but the data is still out there. At Tier 4 there is no undo at all. The test is not whether the operation can be reversed — it is whether reversing it puts you back where you were.

A practical consequence for Tier 2: the compensating action has to be verified to exist before the operation runs. If there is no working undo path, it was never Tier 2. And avoid treating outbound notifications as Tier 2 — a message a customer has already read cannot be unsent, so the record is editable while the effect is not.

**If an operation is Tier 4, why expose it to the agent at all? Why not simply withhold the scope?**
Sometimes you should, and it is worth conceding that before answering. If an agent has no business anywhere near an operation, do not classify it — do not grant the scope. That is exclusion, and it is a different control. **Tier 4 is for operations the agent genuinely participates in but must not complete.**

For those, there are five reasons to model the operation rather than hide it.

The API already exists. Payment endpoints serve mobile applications, branch systems, partners and batch jobs. You are not writing an API for the agent; you are classifying one that is already in production. The tier governs this caller's attempt, not the endpoint's existence.

The attempt is the signal. A Tier 4 incident is written at attempt rather than at decision, carrying the reasoning trace, the delegation chain and the task that produced it. Withhold the scope instead and you receive a bare authorisation failure with none of that context. You have exchanged a governed refusal that tells you something for a silent one that tells you nothing.

Exclusion relocates intent rather than removing it. The agent still needs the outcome, so it assembles it from lower-tier calls, or asks a person in a chat channel who then performs it untracked, or somebody scripts around the gateway. The action still happens, somewhere you cannot see it.

You cannot promote what you never observe. Tier migration depends on recorded behavioural evidence. An operation excluded from the agent's world generates no telemetry, so it can never earn its way to a lower tier. Exclusion is permanent by construction.

Preparation has value even where execution does not. An agent that detects a pricing error, drafts the correction and hands merchandising a complete package has done real work, and the human's job becomes reviewing rather than discovering. That value exists only if the operation is modelled.

A practical note: scopes are usually too coarse to exclude cleanly. Withholding a payments write scope also withholds the small refunds you wanted running autonomously.

**If an undeclared operation defaults to Tier 4, how is anything ever excluded?**
Exclusion and tiering operate at different layers, and conflating them is the source of the confusion. Exclusion is an authorisation decision: the agent's client never holds the scope, so the request is refused at token validation, before the gateway looks for a consequence tier. There is nothing to classify because nothing reached classification. The undeclared-equals-Tier-4 default is a consequence decision, and it applies only to operations the agent can already reach within a scope it legitimately holds. It is the safety net for a classification somebody missed, not a statement about endpoints that were never granted. **Exclude at the scope; tier within it.**

This is also why Tier 4 must exist rather than being replaced by exclusion. Scopes are invariably coarser than operations: an account-write scope covers both a Tier 2 preference update and a Tier 4 regulatory determination, and you cannot withhold one without losing the other. Tier 4 is the control you use when the scope is too blunt an instrument to exclude with.

Declare exclusions explicitly rather than leaving the operation unannotated. A block reading `tier: excluded` distinguishes a decision that was made from one that was missed, and it gives the gateway something to assert against: a live token reaching an excluded operation means a scope has been over-granted, which is a provisioning defect and should be refused and raised as one rather than absorbed silently as a Tier 4 refusal.

**Isn't Tier 4 just Tier 3 with worse consequences?**
Severity is what decides the tier, so yes — reversibility, blast radius and regulatory exposure are what push an operation from 3 to 4. But the control itself differs in kind, not degree. In Tier 3 the human is genuinely in the middle: the agent's payload, under the agent's credential, with the human contributing a yes or no to a pre-formed action. In Tier 4 there is no middle, because the agent has been removed from the path — there is no held operation waiting for approval, and the human constructs the action themselves, in another system, under their own credential.

That matters for two reasons. First, Tier 4 exists for cases where the risk is not that the agent acts without permission but that the agent's *construction* of the action is wrong in a way a reviewer will not catch. Approving an agent's payment payload does not protect you; it launders the decision through someone who is reading rather than deciding. Second, attribution: "the agent placed the order and a clinician approved it" is a materially weaker position than "the clinician ordered it," legally and in an incident review. Many regulated processes require a licensed human to be the actor rather than the approver.

**Then what if our Tier 4 human just re-types the agent's proposal?**
Then Tier 4 has collapsed into Tier 3 and you have added ceremony rather than control. That is a real implementation failure and worth checking for. The test: if you cannot say what the human would do *differently* from approving, the operation belongs in Tier 3. Some organisations will conclude they need three tiers rather than four, and that is a legitimate outcome.

**Does a Tier 3 rejection pause the agent? What about a timeout?**
Treat them differently. A rejection is evidence about the agent's judgement, so pause the agent on that operation type and escalate. A timeout is evidence about approver availability, so deny the single operation and leave the agent working. Conflating the two means a slow reviewer degrades an agent that did nothing wrong. Whichever way you decide, decide it explicitly — this is the first question a careful reviewer will ask.

**My Tier 4 isn't your Tier 4. Doesn't that make the framework useless?**
Where you draw the boundary is yours to set, and it should differ by domain and risk appetite. What is shared is the classification method and the enforcement architecture. If your refund threshold is different from mine, the framework is working.

## Technical

**What is the latency cost?**
Target single-digit milliseconds for Tier 1 and Tier 2 classification at gateway intercept, which sits inside a normal gateway budget. Tiers 3 and 4 do not consume that budget at all because nothing is routed. Production-scale characterisation across gateway implementations is still to be done, and anyone who quotes you a precise universal number is guessing.

**What happens when the agent retries?**
Two mechanisms, and you need both. The response returned to the agent is shaped to be non-retryable — not a status its resilience logic treats as transient. And an idempotency key is persisted at classification time and applied at execution, so neither a duplicate agent submission nor a duplicate decision event can double-execute. This failure mode is what kills naive synchronous approval designs: a control built to prevent an operation causes it to happen twice.

**Why persist the request instead of asking the agent to resubmit?**
Because the agent may re-plan between the pending response and the approval. Resubmission means you approved one payload and executed another. Persisting the request means what executes is byte-identical to what was reviewed.

**What happens if nobody approves?**
A configured timeout disposition. Default to deny. A stricter posture escalates to Tier 4 and suspends the agent. Never default to allow-on-timeout — that converts an approval gate into a delay, which is worse than having no gate at all because it looks like one.

**Where does the enforcement point actually live?**
Anywhere in the path of the operation: an API gateway, a reverse proxy, a service-mesh sidecar, a tool-call broker, an admission controller. The classification logic should be a component the enforcement point calls, not something resident inside any particular gateway — otherwise you have coupled your governance model to a vendor.

**How does this map to MCP tool calls?**
Cleanly, because the enforcement point is the broker rather than the model. Classify the tool invocation before dispatch. The same pattern applies to gRPC interceptors, message-broker plugins, database proxies, and infrastructure admission controllers. If your enforcement lives in the prompt or in model alignment, none of this survives contact with a model update — which is the argument for keeping it at the transport boundary.

**How do you handle multi-agent delegation?**
Treat every delegation event as a governed operation in its own right. Compare the authority conferred on the child against the authority held by the parent: if the child would hold broader credential scope, wider population scope, or higher data sensitivity, that is authority expansion — classify at the top tier and suspend the chain. Also verify credential scope reduction at handoff, and write a provenance record linking each executed call to the delegating agent and the full chain ancestry.

**Why does the delegation provenance matter so much?**
Without it, sub-agent calls execute under the orchestrator's inherited credential. The reasoning entity that decided the operation should happen and the credentialed entity that executed it are then different, with nothing recording the relationship. Your audit log satisfies the requirement to record every access while failing the attribution the requirement exists to produce.

## Organisational

**Who is on the approval rota at 2am?**
A fair question and the honest answer is that tiering is what makes the rota survivable. If your Tier 3 queue is large, your tiers are wrong — go back and check how many of those operations are genuinely irreversible. Route approvals to whoever already owns that system, into tooling they already watch, and use the trust progression so the queue shrinks over time rather than staying constant.

**Won't people just approve everything without reading it?**
They will, if the volume is high and the context is thin. That is an argument for both parts of the design: keep Tier 3 volume genuinely low, and make the approval event carry enough context to decide — the proposed change, the criteria that classified it, the delegation chain, and the anticipated downstream effects. Rubber-stamping is a symptom of bad tiering, not an argument against approval.

**How do I promote something from Tier 3 down to Tier 2?**
On recorded evidence, not on opinion. Track the approval-to-rejection ratio, the incidence of rollback, the deviation between what the agent proposed and what a reviewer would have done, and elapsed time without an adverse event. Scope the promotion to a specific agent version, record the basis in the ledger, and revoke automatically on an adverse event or an agent change.

**What do I tell my auditor?**
That every agent-initiated operation is classified before execution, that the classification and its basis are recorded before the operation is routed or refused, that operations above a defined consequence threshold cannot execute without a recorded human approver, and that operations at the top tier are performed by a human under their own identity. That is a stronger position than most organisations can currently state about human-initiated operations.

## Positioning

**Do I still need NIST AI RMF, ISO 42001, or the EU AI Act work?**
Yes, and this doesn't compete with them. Those frameworks specify what an organisation should achieve; none specifies a runtime mechanism that enforces it at the moment an individual operation is attempted. You need the policy structure and the architectural enforcement, and this is only the second one.

**Is this specific to LLM agents?**
No. It applies to anything that invokes operations without a human decision per invocation — workflow engines, automation platforms, scheduled jobs with broad credentials. LLM agents made it urgent because they made the fan-out large and the behaviour non-deterministic, but the gap was there before them.

**Is there an implementation I can use?**
A reference implementation and specification are being prepared for open release. In the meantime the mechanisms in this document are deliberately described at a level you can build from with the gateway you already run.



## The two things that break this

**Leaving the unclassified unhandled.** An endpoint with no declared tier must be Tier 4 until somebody classifies it. If unclassified defaults to permitted, your coverage gaps are invisible to you and they grow every sprint.

**Scoring only at the call level.** Somebody will assemble a Tier 4 outcome from a sequence of Tier 1 calls. If you evaluate operations independently and never accumulate, the framework is decorative — and worse, it produces an audit trail that makes the extraction look routine.



## Classification worksheet

Copy this for each operation. If you can't answer a row, that's the finding.

| | |
|---|---|
| **Operation** | |
| **What it does, in one sentence** | |
| **Reversible?** | Undo restores the effect / restores the record only / no undo exists |
| **Blast radius** | One record / one customer / a population / system-wide |
| **Data sensitivity** | Public / internal / personal / regulated |
| **Compliance trigger?** | None / adjacent / direct |
| **Idempotent?** | Yes / only with a key / no |
| **Tier** | 1 / 2 / 3 / 4 |
| **If Tier 3–4: who approves, in which tool?** | |
| **If Tier 3–4: what does the approver need to see?** | |
| **What would have to be true to move this down a tier?** | |

That last row is the one worth arguing about. It turns a classification into a plan.
