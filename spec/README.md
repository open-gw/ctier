# The `x-consequence` extension

Field reference for the annotations used in `consequence-tiered-api.yaml`.

Two extension points: `x-consequence-defaults` at the document root, holding rules the gateway
applies globally, and `x-consequence` on each operation, holding that operation's classification.

---

## `x-consequence-defaults`

Document-level governance rules, read by the gateway at specification load.

| Field | Purpose |
|---|---|
| `undeclared-operation-tier` | Tier applied to a reachable operation carrying no `x-consequence` block. Set to `4`. Applies only within authority the agent already holds — it is the net for a missed classification, not a statement about operations never granted. |
| `on-excluded-operation-reached` | What to do when a live credential reaches an operation annotated `tier: excluded`. Set to `refuse-and-alert`: this is a provisioning defect, not a governance event, and should surface as one. |
| `enforcement-point` | Where classification occurs. Always before routing, never inside the agent. |
| `runtime-classification` | `resolve-then-escalate`. The five criteria are evaluated by a human at design time; at runtime the gateway resolves the recorded tier by lookup and may then escalate on context. |
| `runtime-may-lower-declared-tier` | `false`. Escalation is one-directional; the declared tier is a floor. |
| `human-in-request-path` | `false`. Tier 3 and Tier 4 authorisation is always out-of-band. |
| `session-accumulation` | Weights per tier, escalation threshold, correlation scope and decay. See below. |
| `delegation` | Whether credential scope reduction is required at each hop, and what to do on authority expansion. |
| `tier-controls` | The control bundle applied at each tier. |
| `agent-contract` | Obligations on the *calling agent*, not on the gateway. See below. |

### `session-accumulation`

A per-call control is defeated by decomposing a high-consequence outcome into a sequence of
low-consequence calls. Accumulation is the counter-measure.

```yaml
session-accumulation:
  enabled: true
  weights: { tier1: 1, tier2: 3, tier3: 9, tier4: null }
  escalate-above: 9
  correlation-scope: [session, task, delegation-chain]
  decay: PT30M
```

Setting `escalate-above` to the weight of one Tier 3 operation states that a sequence whose
aggregate consequence equals one high-tier action will be governed as one. `decay` matters:
without it, governance overhead accumulates monotonically until a well-behaved agent is treated
as hostile by mid-afternoon.

A scalar sum is a first approximation. Pattern detection over the operation sequence — repeated
partial reads reconstructing a whole record, unusual write fan-out from a single task — is
stronger and is where the remaining work is.

### `agent-contract`

The pending-authorisation response is part of the interface contract, not an edge case handled
later. An agent that treats it as a failure and retries defeats the design: the operation was
withheld deliberately, and a retry either produces a duplicate submission or burns an approval
the reviewer already gave.

Recorded as `MUST` / `MUST-NOT` / `MAY` lists for both the Tier 3 pending response and the
Tier 4 suspension response. Conformance should be asserted in agent integration tests **before**
the agent is granted any scope reaching a Tier 3 operation.

---

## `x-consequence` (per operation)

```yaml
x-consequence:
  tier: 3
  criteria:
    reversibility: undo-restores-record-only
    blast-radius: single-customer
    data-sensitivity: personal
    compliance-trigger: adjacent
    idempotency: safe
  approval:
    mode: async-out-of-band
    route-to: customer-operations
    reviewer-context: [before-state, after-state, delegation-chain, initiating-task]
    expires-in: PT2H
    on-timeout: deny
  rationale: >
    Plain-language justification. Read by humans in review, not by the gateway.
```

### `tier`

`1` | `2` | `3` | `4` | `excluded`

`excluded` is not a tier. It records that agents are deliberately kept out of an operation
entirely; enforcement is the absence of the scope grant, not this annotation. Declaring it
distinguishes a decision that was made from one that was missed, and gives the gateway something
to assert against.

### `criteria`

| Criterion | Values |
|---|---|
| `reversibility` | `not-applicable` · `undo-restores-effect` · `undo-restores-record-only` · `none` |
| `blast-radius` | `single-record` · `single-customer` · `population` · `system-wide` |
| `data-sensitivity` | `public` · `internal` · `personal` · `regulated` |
| `compliance-trigger` | `none` · `adjacent` · `direct` |
| `idempotency` | `inherently-safe` · `safe` · `key-required` · `unsafe` |

`reversibility` carries three meaningful states rather than two, and the middle one decides most
Tier 2 / Tier 3 boundaries. `undo-restores-effect` means nothing persists from the interval in
which the operation stood. `undo-restores-record-only` means the record can be restored but
actions taken by others in that interval cannot be recalled — a price put back does not unplace
the orders taken at the wrong price.

The criteria are recorded even though the gateway resolves the tier rather than recomputing it.
They are what makes a later reclassification reviewable instead of arbitrary.

### `compensating-action` (Tier 2)

```yaml
compensating-action:
  operation-ref: updatePreferences
  verified: true
```

`verified: true` asserts that a working undo path exists. If it does not, the operation was
never Tier 2 — it is Tier 3. Note the key is `operation-ref` rather than `operationId`:
extension blocks should not reuse OpenAPI's reserved key names, because linters and generators
key off them positionally.

### `accumulation` (per operation)

Overrides the document defaults for one operation.

```yaml
accumulation:
  contributes-weight: 2
  escalate-if: repeated-across-session
  escalate-to: 3
```

Use where an operation is individually low-consequence but cumulatively is not — paginated reads
over sensitive collections being the common case.

### `approval` (Tier 3)

| Field | Purpose |
|---|---|
| `mode` | Always `async-out-of-band`. |
| `route-to` | The team that owns the system, in tooling they already watch. |
| `reviewer-context` | What the approval event must carry for the decision to be real rather than a rubber stamp. |
| `expires-in` | ISO 8601 duration on the persisted context. |
| `on-timeout` | `deny`, or escalate to Tier 4. Never `allow` — that converts a gate into a delay, which is worse than no gate because it looks like one. |

### `takeover` (Tier 4)

| Field | Purpose |
|---|---|
| `route-to` | Team performing the action. |
| `human-tooling` | The system the human acts in. **Not** an approve button on the agent's request — if the human's only action is approving the agent's payload, the operation belongs in Tier 3. |
| `auto-resume` | Always `false`. The withheld operation is never released back to the agent. |
| `incident-context` | Attached to the incident so the human inherits the agent's work rather than starting over. |

---

## Response contracts

**Tier 3 — `202`, `PendingAuthorization`.** A success outcome, not a failure. Carries
`correlationId`, `expiresAt`, `retryable: false`, and a machine-readable `agentAction` mirrored
in an `X-Agent-Action` header. `Retry-After` is deliberately absent — emitting one would invite
the retry the response exists to prevent. `202` is chosen because it sits outside the classes
client resilience implementations retry by default.

**Tier 4 — `423`, `AgentSuspended`.** Carries `incidentId`, `autoResume: false`, and
`agentAction: halt_and_hand_off`. No correlation identifier, because nothing is pending and there
is nothing to poll for. The agent must not seek an alternative route to the same outcome; that
attempt is itself a recordable incident.

**Excluded — `403`.** Refused at credential validation, before consequence evaluation. If this
is reached by a live token, a scope has been over-granted.
