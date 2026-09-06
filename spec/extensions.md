# ctier extension reference

Specification 1.1.2. Flat keys — the OpenAPI namespace format is `x-{namespace}-`,
and a single-key overlay action targeting one operation is then trivial to write
and to validate.

**Naming.** 1.0.0 used `x-consequence-*`. 1.1.0 renames to the `ctier`
namespace. 1.1.1 requires all five criteria fields and adds `x-ctier-reverses`.
1.1.2 permits self-reversal where idempotency is `safe` or `inherently-safe`.
The 1.0.0 archive (DOI 10.5281/zenodo.22020288) is unaltered and remains
resolvable.

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
| `recommendationTtlDays` | integer ≥ 0 | no | How long a `recommended` status remains current. Default `90`. |

---

## Operation level

Appear on an Operation Object.

### `x-ctier-tier`

Integer 1–4. Optional. The declared tier. Absence is Tier 4
(`x-ctier-defaults.undeclaredTier`). Not used with `x-ctier-exclude: true`.

### `x-ctier-criteria`

Object. Optional. When present, all five fields are required. Recorded so a
later reclassification is reviewable. The gateway resolves the declared tier;
it does not recompute from these at request time. A validator may compute a
tier from them when checking the single-target rule.

| Field | Values |
|---|---|
| `reversibility` | `not-applicable`, `undo-restores-effect`, `undo-restores-record-only`, `none` |
| `blast-radius` | `single-record`, `single-customer`, `population`, `system-wide` |
| `data-sensitivity` | `public`, `internal`, `personal`, `regulated` |
| `compliance-trigger` | `none`, `adjacent`, `direct` |
| `idempotency` | `inherently-safe`, `safe`, `key-required`, `unsafe` |

There are no criteria defaults. The five criteria are the design-time
judgement. Half of that judgement is not a weaker version of it — it is an
author who has not finished. A schema `required` says so with an error naming
the missing field; a fail-closed default says so with a Tier 4 the author
cannot account for.

Fail-closed is unaffected. C2 operates one level up: an operation carrying no
`x-ctier-*` declaration at all is Tier 4. Completeness *within* a declaration
is a validation concern, not a classification one.

Values are taken from the reference implementation's enumerations so the schema
and the code cannot drift.

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

### `x-ctier-handoff`

Object. Optional. The authority class a Tier 4 handoff is routed to. The class
is declared; the person is resolved by the consuming application.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `authority` | string | yes | Class, not a person. Examples: `account-holder`, `clinician`, `operator`, `risk-approver` |

### `x-ctier-status`

Enum: `declared` | `recommended`. Optional.

- `declared` — a human confirmed the assignment
- `recommended` — awaiting review

Absence on an otherwise unclassified operation is `absent` for coverage
purposes. Provenance for a recommendation lives on the recommendation-set
overlay, not here.

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
