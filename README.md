# Consequence-Tiered API Governance

A runtime governance model for autonomous AI agents calling production APIs.

Existing access controls answer questions about the **caller**. OAuth scopes describe which
operations a client may invoke. RBAC describes what role it holds. Rate limits describe how
often it may call. None of them answers the question that matters once the caller is an
autonomous agent: **what happens if this particular call is wrong?**

An agent can be fully authenticated, fully authorised, within its rate limit, and still
initiate a payment that should never have been initiated. Identity is not consequence.

This repository holds a specification and a worked reference for closing that gap: classify
each operation by the consequence of getting it wrong, declare the classification in the API
specification, and enforce it at the gateway.

---

## The model in one page

**Three layers, not two.**

| Layer | Question | Mechanism | Describes |
|---|---|---|---|
| Identity | Who is calling? | mTLS, tokens, agent ID, delegation chain | the caller |
| Authorisation | What are they permitted to do? | OAuth scopes, RBAC, scope reduction | the caller |
| **Consequence** | **What happens if this call is wrong?** | **consequence tier, enforced at the gateway** | **the action** |

**Five criteria classify any operation.**

1. **Reversibility** — does the undo restore the *effect*, restore only the *record*, or not exist?
2. **Blast radius** — one record, one customer, a population, or system-wide?
3. **Data sensitivity** — public, internal, personal, or regulated?
4. **Compliance impact** — none, adjacent, or a direct regulatory trigger?
5. **Idempotency** — safe to run twice, safe only with a key, or unsafe?

Reversibility is three states rather than two, and the middle state is where most consequential
operations sit. You can put a price back; the orders placed at the wrong price were still
placed. You can revoke an export; the data has left. The reversal stops the bleeding without
restoring the state.

**Four tiers follow.**

| Tier | Character | Control | Human role |
|---|---|---|---|
| 1 — Low | Read-only, no compliance surface | Log only; logging does not gate | None |
| 2 — Medium | Undo restores the effect | Audit trail, before/after state, verified compensating action | Decides whether it *stands* |
| 3 — High | Undo restores the record, not the effect | Gateway withholds; async approval; approver recorded | Out-of-band approval |
| 4 — Critical | No undo exists | Agent suspended before the action; no auto-resume | Performs it personally |

**Tier 3 is a delay. Tier 4 is a handover.** In Tier 3 the operation is deferred and the
agent's operation eventually executes with a named approver against it. In Tier 4 the operation
is withdrawn — the agent never executes it, and the human performs the action themselves under
their own credentials.

---

## The architectural claim

**The human is never in the API request path.**

This is the part that makes the model deployable rather than theoretical. A human placed inside
the request path holds an HTTP connection open for minutes while somebody reads a message queue.
Gateway timeouts fire, connection pools drain, and — worst — the agent's own retry logic fires
against the pending request, so a control built to *prevent* an operation causes it to happen
twice.

Instead, for Tier 3 the gateway:

1. **Withholds** the operation. Nothing is routed.
2. **Persists** the full request, the delegation chain, an idempotency key and an expiry.
3. **Emits** an asynchronous authorisation event to an orchestration layer.
4. **Returns** a terminal, deliberately non-retryable response to the agent, which is then free
   to continue with the rest of its task.
5. **Executes from the persisted context** on approval — so the operation that runs is
   byte-identical to the one the reviewer saw, rather than whatever the agent would have
   re-planned in the interim.

The human decides on human timescales. No connection, worker, or client timeout is consumed.

---

## Declaration and resolution

Tiers are evaluated **at design time** by the interface owner and recorded in the OpenAPI
document. At runtime the gateway **resolves** the recorded tier — a lookup, not an evaluation,
which is why it fits inside a production latency budget — and then applies what the
specification could not know:

- accumulated consequence across the session, task or delegation chain
- delegation chain integrity, where a child agent holds broader authority than its parent
- request content exceeding the bounds the declaration assumed

**Escalation is one-directional.** Runtime information can raise a tier and can never lower it.
The declared tier is a floor, not a determination — which also means under-declaring an
endpoint buys nobody anything.

```yaml
paths:
  /accounts/{accountId}/balance:
    get:
      operationId: getBalance
      x-consequence-tier: 1

  /payments:
    post:
      operationId: initiatePayment
      x-consequence-tier: 4

  /customers/{customerId}/profile:
    put:
      operationId: updateProfile
      # no tier declared -> the gateway treats this as Tier 4
```

**No tier declared = Tier 4.** Unclassified operations fail closed, so coverage gaps surface
instead of hiding.

---

## Exclusion is not Tier 4

If an agent has no business anywhere near an operation, do not classify it — do not grant the
scope. That is exclusion, and it happens at the authorisation layer, before any tier is
considered.

Tier 4 is for operations the agent genuinely participates in but must not complete. Both are
needed, because scopes are always coarser than operations: `accounts:write` covers a Tier 2
preference update and a Tier 4 regulatory determination, and you cannot withhold one without
losing the other.

**Exclude at the scope; tier within it.**

---

## What's in this repository

| Path | What it is |
|---|---|
| `spec/consequence-tiered-api.yaml` | Complete worked OpenAPI 3.1 example — ten operations across all four tiers, plus one explicitly excluded and one deliberately unclassified |
| `spec/README.md` | Field-by-field reference for the `x-consequence` extension |
| `docs/` | Classification worksheet, domain walkthroughs, FAQ |

The example specification is deliberately built so the argument is visible in its structure:
the OAuth scopes are drawn at the granularity real systems use, so the mismatch between scope
and consequence shows on the page. `accounts:read` reaches both a Tier 1 balance query and an
unclassified bulk export. `payments:write` would reach a small refund and a large transfer
identically.

Two operations in it are worth reading first:

- **`updateFeatureFlags`** is Tier 4 with `data-sensitivity: internal`. No personal data
  anywhere in the operation, and it is still critical — blast radius and irrecoverability carry
  it. Classification by data sensitivity alone misses the most dangerous operations on a surface.
- **`listTransactions`** is Tier 1 individually and carries an accumulation rule that escalates
  it on repetition. A sequence of individually low-consequence reads that reconstructs a
  complete record is the obvious attack on any per-call control, and it has to be handled at
  session level or the whole model is decorative.

---

## Applying it

Five steps, roughly a week, no new platform required.

1. **Inventory what your agents can actually call.** Pull the scopes on agent service accounts
   and enumerate the operations those scopes reach. The list is usually longer than expected.
2. **Classify the top twenty by hand.** Enough to reveal the shape of your risk, small enough to
   finish in an afternoon. Resist building tooling first.
3. **Find the ones that are wrong** — operations you assumed were safe that turn out to be
   irreversible, and operations you have been gating that are provably Tier 1. The second group
   is where you win credibility, because you are giving time back.
4. **Declare tiers in the spec.** Costs nothing before you enforce anything, and it forces the
   conversation with interface owners while it is still cheap.
5. **Enforce one tier, at one gateway, for one agent.** You will learn more from one working
   async approval path than from a taxonomy covering everything and enforcing nothing.

What not to do first: build a classification service, write a policy document, or try to
classify every operation before enforcing any of them. All three feel like progress and none
produce a governed operation.

---

## Status

Specification and reference example. A reference implementation is in preparation.

Subject matter described here is the subject of US provisional patent application 64/137,066
(filed 19 August 2026). The specification and documentation are licensed CC BY 4.0; that licence
covers copyright and grants no patent licence. See `LICENSE`.

Tier boundaries are intended to be configured per organisation — if your refund threshold
differs from the one in the example, the model is working as designed. Issues and pull requests
proposing different boundaries, additional criteria, or gaps in the accumulation logic are
welcome and are the reason this is public.

## Citation

See `CITATION.cff`.

## Licence

Specification and documentation: CC BY 4.0. See `LICENSE`.
