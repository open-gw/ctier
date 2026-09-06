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
- Where `x-ctier-criteria` is present, all five fields are required. There are
  no criteria defaults. A missing field is a schema error naming that field.
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
sorted lexicographically, no insignificant whitespace, UTF-8). Per-input
digests are over raw bytes. Schema checks the string form, not the bytes.

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

---

## What a validator is

A program that loads a description, an overlay, a rule document, or a
composition, and rejects inputs that fail the rules above. This repository does
not ship one. The invalid examples are the corpus it is written against.
