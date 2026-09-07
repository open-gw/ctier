# ctier specification 1.1.9

The stable public surface is a set of versioned document formats and the
transformations between them (`docs/adr/0009-the-contract-is-the-documents.md`).

**Naming.** Specification 1.0.0 used `x-consequence-*`. 1.1.0 renames to the
`ctier` namespace. The 1.0.0 archive
([doi.org/10.5281/zenodo.22020288](https://doi.org/10.5281/zenodo.22020288)) is
unaltered and remains resolvable.

| Path | What it is |
|---|---|
| [`extensions.md`](extensions.md) | Flat `x-ctier-*` keys: meaning, location, type, required |
| [`validation.md`](validation.md) | Schema-checked shape versus validator-checked semantics |
| [`schemas/`](schemas/) | JSON Schema 2020-12 for the ctier-authored formats, 0.1.0 |
| [`examples/`](examples/) | Valid instances, invalid instances one per semantic rule, and bound-comparison request cases |
| [`consequence-tiered-api.yaml`](consequence-tiered-api.yaml) | Worked OpenAPI 3.1 example |

`exportStatements` is unclassified **on purpose**. It is the one `absent`
operation in the reference document. Absence is Tier 4 by C2 — fail-closed —
and completing that classification as tidying would remove the example of the
default the model rests on.

The qualified spec is an OpenAPI document carrying the Part 2 extensions. It
has no schema of its own; it has a validation profile in `validation.md`.

**Deferred**, because the implementation has not yet taught us what they hold:
the conformance corpus format (task 08). Ledger record formats exist in the
reference implementation as `0.1.0-draft` (decision, attempt, outcome); they
are named in `validation.md` and promote on the same condition as the corpus.

Pre-1.0 the ctier-authored formats may break. Freeze at v1.0.0 alongside the
demo, not before. Consumers MUST ignore fields they do not recognise.

---

## Response contracts

These are interface behaviour, not extension keys.

**C5 — The withheld response is terminal and non-retryable.** Status `202`,
outside conventionally retried classes. No `Retry-After`. `retryable: false`.
A machine-readable `agentAction`, mirrored in an `X-Agent-Action` header.

The withheld response is subject to the no-URL rule in
`spec/validation.md`. An implementer who adds a `pollUrl` has not retried
the operation — they have left the agent engaged with it. The effect is
the one the design exists to prevent: custody acquires a caller for the
length of a human review, and the connection occupancy that out-of-band
approval was meant to eliminate reappears under another name.

The agent's involvement with a withheld operation ends when it receives the
response. What happens next happens to the operation, not to the agent.

The policy service constructs the complete response body. The enforcement
point relays it unchanged and MUST NOT construct, augment, or reformat it.
Resolve it once, in the place that knows, and let the edges carry it — the
same reasoning as compiling the decision rather than the declaration.

Required contents:

| Field | Meaning |
|---|---|
| `correlationId` | Identifies the pending context. Not a status URL. |
| `retryable` | `false`. Present so generic clients can branch on it. |
| `agentAction` | `continue_task_without_this_step`: continue the rest of the task; do not resubmit this step. |
| `authority` | The authority class from `x-ctier-handoff` — the class of person who can authorise this, not a person. |
| `expiresAt` | When the pending authorisation lapses, derived as `min(credential remaining lifetime, configured review period)`. |

`expiresAt` is derived from the credential's **expiry**, not from the
credential. An implementation that stores a token in order to compute a
deadline has stored a token. The enforcement point passes the `exp` claim
as a value at persist time; the policy service records the deadline and
never sees anything that authenticates.

`202` sits outside the classes client resilience implementations retry by
default.

**C6 — Execution is from persisted context.** On approval, the operation
executed is byte-identical to the operation described in the approval event
in **method, request target, query and body**. The agent does not resubmit
and is not involved.

Byte-identity does **not** extend to the credential. The executed request
carries the policy service's own identity; the originating agent and the
authorising person are recorded as attribution. A policy service that
replays the agent's credential has stored one, which `spec/validation.md`
forbids. A store of live agent credentials awaiting replay is a larger
liability than the one custody exists to manage, and credentials expire on
a schedule unrelated to human review.

**C7 — Idempotency keys are derived, not generated.** Derived from the
declared `operationId` (or method + path template) plus a canonical
serialisation of the request specifics, so a resubmission resolves to the
pending context rather than creating a second one.

Request specifics for the purpose of derivation are the **request target,
query, body, agent identity and task identity**. Headers are not included.
A derivation that includes a credential produces a key that changes when
the credential rotates, which defeats the guarantee.

The derived key is transmitted to the target in the `Idempotency-Key`
request header.

The key is derived once, when the operation is first classified, and stored
with the persisted context. It is **not** re-derived at execution time.
Re-deriving from a reconstructed request risks deriving from something
subtly different from the original, which defeats the guarantee it exists
to provide. A second implementation choosing `X-Idempotency-Key` produces
a backend that silently does not deduplicate, with no error raised
anywhere.

**C8 — Expiry and rejection differ.** Rejection pauses the agent for the
operation class. Expiry refuses the single operation and leaves the agent
working.

The effect of a rejection is **observed, not announced**. A paused agent
discovers its state on its next attempt at an operation of that class, which
is refused. No notification, callback or status affordance is created for
it. The pause is state, not a message.

**C9 — Tier 4 has no resume path.** No state transition exists by which a
Tier 4 operation executes under the agent's identity. Status `423`,
`AgentSuspended`. Carries `incidentId`, `autoResume: false`, and
`agentAction: halt_and_hand_off`. No correlation identifier. The agent
must not seek an alternative route to the same outcome.

The no-URL rule in `spec/validation.md` applies. Tier 4's content is the
handover: a human performs the operation under their own credential, and
the agent does not. A location in the agent's response makes the agent
the mediator of a handover it was excluded from. The exclusion is the
control.

**C10 — No operation executes without a decision having been made.** The
decision MUST precede the action. That precedence may arise from the
decision being bound into the enforcement point's configuration at
deploy time, or from a synchronous evaluation before the request
proceeds; both satisfy this criterion and an implementation MUST state
which it relies on.

The **record** of a decision is evidence that it was made, not the
decision itself. It MUST carry the declared tier, the applied tier, the
escalation reason and the composition digest. It MAY be written
asynchronously, and MUST be durable — an implementation whose records
can be lost has not recorded them.

Where the policy service is unavailable and the enforcement point
refuses without classifying, the refusal MUST itself be recorded,
marking the applied tier as undetermined. A refusal to decide is a
decision.

If the record sink is unavailable alongside the policy service, that
window has no record. ADR-005 accepts this rather than implying a
guarantee the design cannot make.

**Excluded — `403`.** Refused at credential validation, before consequence
evaluation. If this is reached by a live token, a scope has been over-granted.
