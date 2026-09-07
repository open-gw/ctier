# ctier extension reference

Specification 1.1.9. Flat keys — the OpenAPI namespace format is `x-{namespace}-`,
and a single-key overlay action targeting one operation is then trivial to write
and to validate.

**Naming.** 1.0.0 used `x-consequence-*`. 1.1.0 renames to the `ctier`
namespace. 1.1.1 requires all five criteria fields and adds `x-ctier-reverses`.
1.1.2 permits self-reversal where idempotency is `safe` or `inherently-safe`.
1.1.3 requires `x-ctier-recommended-at` whenever `x-ctier-status` is
`recommended`, and forbids it on `declared`. 1.1.4 splits criteria: a
declaration must be complete; a proposal may be a non-empty subset. 1.1.5
adds two reference operations that exercise the two divergence directions,
and documents those signals. 1.1.6 defines the bound comparison as
decimal-numeric. 1.1.7: the withheld response MUST NOT carry a polling
affordance; the policy service constructs the body; `authority` and
`expiresAt` are required; a policy service never persists an agent
credential; Tier 3 and Tier 4 execute under different identities; C6
scopes byte-identity to method, target, query and body; C8's
rejection is observed, not announced. 1.1.8: no agent-facing response
carries a URL; C7 names `Idempotency-Key` and excludes headers from
derivation. 1.1.8 also records which criteria have been exercised.
1.1.9 restates C10 as a property — no operation executes without a
decision having been made — rather than as a request-time write. The
record is evidence that a decision was made, MAY be written
asynchronously, and MUST be durable. The 1.0.0 archive
(DOI 10.5281/zenodo.22020288) is unaltered and remains resolvable.

The JSON Schemas in `schemas/` check shape. Semantic constraints are in
`validation.md`.

---

## Document level

Appear on the OpenAPI root.

### `x-ctier-defaults`

Object. Optional. Fail-closed values apply when a field is omitted.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `undeclaredTier` | integer 1–4 | no | Tier applied to a reachable operation with no `x-ctier-tier`. Default `4`. Applies only inside a granted scope. |
| `escalateAbove` | integer ≥ 0 | no | Accumulated-weight threshold at which the scope floor rises. Default `9`. |
| `decaySeconds` | integer ≥ 0 | no | Window after which accumulated weight is forgotten. Default `1800`. |
| `recommendationTtlDays` | integer ≥ 0 | no | How long a `recommended` status remains current, aged against `x-ctier-recommended-at`. Default `90`. |

---

## Operation level

Appear on an Operation Object.

### `x-ctier-tier`

Integer 1–4. Optional. The declared tier. Absence is Tier 4
(`x-ctier-defaults.undeclaredTier`). Not used with `x-ctier-exclude: true`.

| Tier | Who executes |
|---|---|
| 1 | The agent, under its own credential. |
| 2 | The agent, under its own credential, with a verified compensating action. |
| 3 | The policy service, under its own identity. |
| 4 | The human, under the human's own credential. The agent does not execute. |

**Tier 3** executes under the policy service's own identity, with the
originating agent and the authorising person recorded as attribution.
**Tier 4** executes under the human's own credential.

In Tier 3 a named person authorised an action. In Tier 4 a named person
performed one. The two records say different things, and the difference is
the reason both tiers exist.

### `x-ctier-criteria`

Object. Optional. **Where it appears changes what completeness means.**

| Where | Shape | Why |
|---|---|---|
| Description or qualified spec (`$defs/criteria`) | all five required | it is a **declaration** |
| Recommendation set (`$defs/criteriaProposal`) | any subset, at least one | it is a **proposal** |

A declaration is a completed judgement and must be complete. A proposal is an
observation about what the description makes visible, and the fields it leaves
empty are the ones a human has to supply. Requiring a proposal to be complete
would force the recommender to invent the two criteria — data sensitivity and
compliance trigger — that the interface description cannot express and that
carry the most consequence when wrong.

The enums are shared. Only the required-set differs.

In a description or qualified spec, all five fields are required when the
object is present. Recorded so a later reclassification is reviewable. The
gateway resolves the declared tier; it does not recompute from these at
request time. A validator may compute a tier from them when checking the
single-target rule.

| Field | Values |
|---|---|
| `reversibility` | `not-applicable`, `undo-restores-effect`, `undo-restores-record-only`, `none` |
| `blast-radius` | `single-record`, `single-customer`, `population`, `system-wide` |
| `data-sensitivity` | `public`, `internal`, `personal`, `regulated` |
| `compliance-trigger` | `none`, `adjacent`, `direct` |
| `idempotency` | `inherently-safe`, `safe`, `key-required`, `unsafe` |

There are no criteria defaults. In a declaration the five criteria are the
design-time judgement. Half of that judgement is not a weaker version of it —
it is an author who has not finished. A schema `required` says so with an
error naming the missing field; a fail-closed default says so with a Tier 4
the author cannot account for. A proposal is allowed to be that unfinished
object, because the missing fields are the human's.

Fail-closed is unaffected. C2 operates one level up: an operation carrying no
`x-ctier-*` declaration at all is Tier 4. Completeness *within* a declaration
is a validation concern, not a classification one.

Values are taken from the reference implementation's enumerations so the schema
and the code cannot drift.

#### Reversibility is not the record

Two operations with the same method, the same shape and the same data can
classify differently. The question is not whether the record can be restored
but whether anything acted on the state while it stood. Where the answer
depends on downstream consumers rather than on the operation itself, the
classification belongs to whoever knows those consumers — which is rarely the
API developer.

The reference document's `updateDisplayPreferences` and
`updateCommunicationPreferences` are that pair.

#### Divergence is two findings

A recommender classifies from the description alone and produces a
**structural floor**: the minimum tier the visible shape supports, filling
unknown criteria at lowest consequence. The confirmed tier is compared to
that floor. The two directions are not the same kind of finding.

**Confirmed well above the floor** — the description understates the
operation. This is a defect in the interface description, and it affects
every automated reader equally, agents included. The reference document's
`getPaymentAuthorization` is that case: a GET of one resource floors at
Tier 1; releasing funds is Tier 4.

**Confirmed below the floor** — a human overrode structure downward.
Usually because they know something the description cannot express;
occasionally because a tier is being lowered that should not be. Always
worth a second pair of eyes, never an accusation on its own. The
reference document's `deleteSavedSearch` is that case: a DELETE with a
declared reversal floors at Tier 3; nothing acts on a saved search's
absence, so restoring it restores the effect and Tier 2 is correct.

### `x-ctier-reverses`

String. Optional. The `operationId` this operation reverses. Declares a
compensating action. Do not infer reversal from naming.

The named `operationId` must exist in the same composed description. An
operation declaring `x-ctier-tier: 2`, or criteria resolving to Tier 2, MUST
be named by an `x-ctier-reverses` — another operation's, or its own. See
`validation.md`.

Self-reversal is valid only where `idempotency` is `inherently-safe` or
`safe`. `key-required` and `unsafe` cannot reverse themselves.

### `x-ctier-exclude`

Boolean. Optional. Default `false`. The agent must never hold this scope. Not a
tier — an authorisation decision, taken before any tier is considered. Must not
appear with `x-ctier-tier`.

### `x-ctier-bounds`

Object. Optional. The request bound the declared tier assumes.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `parameter` | string | yes | Name of a parameter on this operation |
| `max` | integer | yes | Inclusive upper bound |

`parameter` must name a parameter that exists on the operation. That check is
validator-enforced; see `validation.md`.

The comparison is **numeric**. The parameter's value is interpreted as a
decimal number, not as an integer. `101.0` and `1.01e2` both exceed a `max` of
`100`.

A value that does not parse as a decimal number does not escalate. A request
omitting the parameter entirely does not escalate. Neither is an error: the
declared tier stands and the operation proceeds under it.

`max` itself is declared as an integer.

### `x-ctier-handoff`

Object. Optional. The authority class. For Tier 3, the class of person who
can authorise the withheld operation. For Tier 4, the class a handoff is
routed to. The class is declared; the person is resolved by the consuming
application. The withheld response carries this class as `authority`.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `authority` | string | yes | Class, not a person. Examples: `account-holder`, `clinician`, `operator`, `risk-approver` |

### `x-ctier-status`

Enum: `declared` | `recommended`. Optional.

- `declared` — a human confirmed the assignment
- `recommended` — awaiting review

Absence on an otherwise unclassified operation is `absent` for coverage
purposes. `recommended` MUST carry `x-ctier-recommended-at`. `declared` MUST
NOT. Full provenance — input text and digest, rule or model version, confidence
— stays on the recommendation-set overlay. The date is here because expiry
operates on the composed document.

### `x-ctier-recommended-at`

String, RFC 3339 date-time. When this tier was proposed. Required whenever
`x-ctier-status` is `recommended`. Forbidden when status is `declared` — a
confirmed tier has no proposal date, and carrying one invites the reader to
think it expires.

This is the minimum a qualified spec carries about a recommendation. There is
no missing-timestamp case: a recommendation without a date is invalid, the
same way a criteria object missing a field is invalid.

### `x-ctier-rationale`

String. Optional. Why this tier. What survives an audit.

### `x-ctier-weight`

Integer ≥ 0. Optional. Contribution to accumulated consequence. **Zero is
meaningful** — "does not accumulate" — and is distinguishable from absent.
Absent means the default weight for the applied tier.

### `x-ctier-escalate-to`

Integer 1–4. Optional. Per-operation accumulation target. May never resolve
below the tier already reached. That check is validator-enforced; see
`validation.md`.

---

## Overlay actions

An overlay action whose `update` sets any `x-ctier-*` field that resolves to a
tier below 4 — whether by declaring `x-ctier-tier` or by supplying
`x-ctier-criteria` that compute one — MUST select exactly one operation. An
action MAY select many operations only when it assigns Tier 4.

See `validation.md`.
