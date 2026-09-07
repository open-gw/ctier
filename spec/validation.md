# Schema-checked versus validator-checked

JSON Schema validates **shape**. Several rules in this specification are
semantic. A second implementation that treats the schemas as the whole contract
will accept documents this specification rejects.

Consumers MUST ignore fields they do not recognise. That is a rule about
behaviour, not shape — the same invariant the runtime already requires of
unknown `x-ctier-*` headers. It is what makes additive evolution safe. Adding a
field is a minor version; removing one, or changing what an existing field
means, is major.

---

## Schema-checked

These are true or false of a document in isolation.

- `ctierSchema` equals the expected `<name>/<version>` string.
- Enumerations: tier 1–4; criteria values as in `extensions.md`; status
  `declared` | `recommended`; rule `confidence` `high` | `medium` | `low`.
- Where `x-ctier-criteria` appears on a description or qualified spec, all
  five fields are required (`$defs/criteria`). A missing field is a schema
  error naming that field, not `criteriaProposal`. There are no criteria
  defaults.
- A recommendation set MAY propose a partial criteria object
  (`$defs/criteriaProposal`: any subset, at least one). An empty criteria
  object is a schema error.
- `propose` contains `tier` or `criteria`, never both.
- `when` contains only the closed predicate keys: `method`,
  `pathEndsWithParameter`, `responseIsCollection`, `hasRequestBody`,
  `securitySchemes`, `tagIn`, `operationIdMatches`,
  `hasDeclaredReversal`, `pathSegmentCount`, `pathParameterCount`.
- `pathSegmentCount` and `pathParameterCount` are an integer or `{min, max}`.
- `operationIdMatches` is a list of exact strings, not a pattern.
- Digests match `sha256:` followed by 64 lowercase hex characters.
- Required fields on each document type, as declared in the schema.
- `x-ctier-weight` is an integer ≥ 0 when present. Zero is a value.
- Overlay documents that are recommendation sets carry `overlay: "1.1.0"` and
  per-action `x-ctier-provenance`.
- An Operation Object validated against `$defs/operation` in
  `ctier-extensions-0.1.0.json`: `x-ctier-status: recommended` requires
  `x-ctier-recommended-at`; any other status, or no status, forbids it.
  Both halves are `if`/`then`/`else` on `x-ctier-status`. Field-by-field
  validation of the leaf `$defs` cannot see the conditional — the validator
  applies `$defs/operation` to each Operation Object.

An operation whose `x-ctier-status` is `recommended` MUST carry
`x-ctier-recommended-at`. An operation whose status is `declared` MUST NOT —
a confirmed tier has no proposal date, and carrying one invites the reader to
think it expires. JSON Schema 2020-12 expresses both halves, including a date
with no status (the `else`). The validator's job is to apply `$defs/operation`
to each Operation Object, not to re-check the conditional.

A recommendation set MAY propose a partial criteria object. The composer MUST
reject a description or qualified spec whose criteria are partial, whether they
arrived inline or through an overlay. Completeness is required at the point of
declaration, not at the point of proposal.

`ctier recommend --apply` writes a recommendation into the description. Applying
a partial proposal inline therefore produces an **invalid description**, by
design. The command must either refuse to apply an incomplete proposal, or
apply it and report the document as incomplete. That behaviour is the
recommender's; the invalidity is intended rather than a defect.

---

## Validator-checked

These require a description, an evaluation context, or both. The schema cannot
see them. Invalid examples in `examples/invalid/` are written against this list.

### Single-target rule

An overlay action whose `update` sets any `x-ctier-*` field that resolves to a
tier below 4 — whether by declaring `x-ctier-tier` or by supplying
`x-ctier-criteria` that compute one — MUST select exactly one operation. An
action MAY select many operations only when it assigns Tier 4.

Two consequences:

1. The check requires evaluating the JSONPath against a specific description.
   The same overlay can be valid against one description and invalid against
   another.
2. The check requires computing a tier from criteria. The validator depends on
   the classification rules, not only on the schema.

### Unidirectionality

`x-ctier-escalate-to` may never resolve below the tier already reached at that
point in evaluation. Schema cannot see the evaluation context.

### Bounds coherence

`x-ctier-bounds.parameter` must name a parameter that exists on the operation.

### Bound comparison

An implementation MUST interpret a bound parameter's value as a decimal number.
An implementation that parses it as an integer will fail to escalate on
fractional values that exceed the bound, which is a fail-open.

This is runtime interpretation of a request, not document shape. Schema sees
`max` as an integer and `parameter` as a string; it cannot see `101.0`. The
request cases in `examples/bound-comparison.yaml` are the corpus a second
implementation is held to.

### No agent-facing URL

**No response emitted to an agent carries a URL.** Not a polling URL, not a
callback, not a status endpoint, not a handoff location, and not a
documentation link that resolves to any of those.

An agent-facing response carries a correlation id, a reason, an
`agentAction` directive, an authority **class**, and — where the
disposition has one — an expiry. Resolving an authority class to a
person, a place or a channel is the consuming application's work, not
the enforcement point's and not the policy service's.

This is a property of every agent-facing disposition, including those
not yet written. C5 and C9 reference it rather than restate it.

### No credential persistence

A policy service MUST NOT persist any credential belonging to the agent. It
receives the credential's expiry as a value, not the credential.

`expiresAt` on the withheld response is derived from that value:
`min(credential remaining lifetime, configured review period)`. An
implementation that stores a token in order to compute a deadline has
stored a token. Schema cannot see a request header; this is a security
property of the design, not an implementation preference.

### Agent suspension is an authorisation concern

Agent suspension — required by both Tier 4 and by C8's rejection branch —
is enforced at the authorisation layer as a property of the credential. An
implementation without it may record a rejection but cannot produce its
effect, and MUST NOT simulate the effect by consulting the policy service
on every request: that reintroduces into the request path the dependency
the out-of-band design removes.

### Tier 2 requires a verified compensating action

An operation declaring `x-ctier-tier: 2`, or criteria resolving to Tier 2,
MUST be named by an `x-ctier-reverses` field in the same composed description.
The namer may be another operation, or the operation itself. An
`x-ctier-reverses` target that does not exist is invalid.

Do not infer reversal from operation names. The author declares it.

### Self-reversal

An operation MAY name itself in `x-ctier-reverses`, asserting that re-invoking
it with the captured prior state restores the effect. Self-reversal is valid
only where `x-ctier-criteria.idempotency` is `inherently-safe` or `safe`. An
operation that is `key-required` or `unsafe` cannot reverse itself:
re-invoking it is not a restoration, it is a second mutation.

The model already requires Tier 2 to carry an audit trail with
before-and-after state. That capture is precisely what makes self-reversal
possible — the compensating action *is* "invoke this operation again with the
captured before-state." The audit requirement and the compensating-action
requirement are the same mechanism seen from two ends.

### Exclusion exclusivity

`x-ctier-exclude: true` with a declared `x-ctier-tier` is a contradiction.
Exclusion is an authorisation decision taken before any tier is considered.

### Coverage sort

`coverage-report` `operations` is ordered by recommended tier descending,
`absent` last. Schema cannot require that order.

### Digest discipline

`compositionDigest` is over canonical JSON of the composed document (keys
sorted lexicographically, no insignificant whitespace, UTF-8). That is
the compose-time binding (C13). `deploymentDigest` is over every input
the generator consumed: the composed description, the tier
declarations, and the declared agent identities with their
entitlements. Regenerating from identical inputs MUST produce an
identical digest. A change to any input MUST change it. Per-input
digests are over raw bytes. Schema checks the string form, not the
bytes.

### Pagination is refused as a predicate

Pagination is not standardised in OpenAPI — `limit`/`offset`, `page`/`size`,
`cursor` and others are all in use. Detecting it means heuristics over
parameter names, which is precisely the kind of inference that would diverge
between a Python composer and a generated Spectral ruleset. A bounded operation
is one whose bound the author has **declared**, via `x-ctier-bounds`. ctier does
not guess.

No pagination predicate exists. If a recommender wants that signal, it uses
`x-ctier-bounds` being present or absent.

### Engine `Criteria` defaults are not the contract

The reference implementation's `Criteria` model retains permissive Python
defaults so tests can construct objects without filling every field. Those
defaults are **not** the document contract and must never be relied upon by a
document. A document that omits a criteria field is invalid.

Fail-closed is C2: an operation with no `x-ctier-*` declaration is Tier 4.
That guarantee does not fill in a half-written criteria object.

### Ledger records

Three types. An attempt is not a decision with nulls. An outcome is the
only record that proves C6 held.

**Decision** — an operation was classified and acted upon. Declared
tier, applied tier, escalation reason, disposition, deployment digest.

**Attempt** — the enforcement point refused without classifying,
because the policy service was unavailable. Carries the declared tier,
the deployment digest and the reason, and marks the applied tier
undetermined.

**Outcome** — what happened when an approved Tier 3 executed.
Correlation, approver, executed-at, result, whether it failed. Written
after the fact by definition. No digest; it joins a Decision by
`correlationId`.

Their schemas are `decision-record/0.2.0`, `attempt-record/0.2.0` and
`outcome-record/0.1.0`. They are normative. They reach 1.0.0 when an
implementation other than the reference one has been held to them.

### On the maturity of these criteria

C5, C6, C7, C9 and C10 have each now been built against. Each turned
out to have a gap that appeared only when something implemented them.

Each of these gaps pointed the unsafe way — add a polling affordance,
store a credential, derive a key from one, hand the agent a location,
describe a mechanism instead of the property it protects. A criterion
written from a design's intent omits prohibitions, because its author
did not consider doing the wrong thing. The remedy is not more careful
writing; it is building something against each criterion and reporting
what the plain reading permits.

This closes that pattern. Of the remainder: C1, C2, C3, C4, C11 and
C12 are exercised by the compiled targets and the corpus. **C8's
rejection branch is not**, and cannot be until the authorisation layer
exists. Expiry is distinct and does run.

---

## What a validator is

A program that loads a description, an overlay, a rule document, or a
composition, and rejects inputs that fail the rules above. This repository does
not ship one. The invalid examples are the corpus it is written against.
