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

### C3 — Unidirectionality

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

### C14 — No credential persistence

A policy service MUST NOT persist any credential belonging to the agent. It
receives the credential's expiry as a value, not the credential.

`expiresAt` on the withheld response is derived from that value:
`min(credential remaining lifetime, configured review period)`. An
implementation that stores a token in order to compute a deadline has
stored a token. Schema cannot see a request header; this is a security
property of the design, not an implementation preference.

### C15 — Agent suspension is an authorisation concern

Agent suspension — required by both Tier 4 and by C8's rejection branch —
is enforced at the authorisation layer as a property of the credential. An
implementation without it may record a rejection but cannot produce its
effect, and MUST NOT simulate the effect by consulting the policy service
on every request: that reintroduces into the request path the dependency
the out-of-band design removes.

Compiled-absence is the mechanism: the scope is removed at the
authorisation server. This criterion forbids gateway-local
suspension — a denylist at the enforcement point, a shared
dictionary of suspended agents, or any consult of the policy
service to ask whether an agent is suspended.

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

### C13 — Digest discipline

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

### C2 — Fail-closed

Fail-closed is C2: an operation in the composed description with no
`x-ctier-*` declaration is Tier 4. That default is not a default for
paths outside the composition. That guarantee does not fill in a
half-written criteria object.

### Ledger records

Three types. An attempt is not a decision with nulls. An outcome is the
only record that proves C6 held.

**Decision** — an operation was classified and acted upon. Declared
tier, applied tier, escalation reason, disposition, deployment digest.

**Attempt** — the enforcement point refused without classifying.
The named case is the policy service unavailable. C12 adds another:
an operation in the composed description with no assignment in
force. Both carry the declared tier where known, the deployment
digest and the reason, and mark the applied tier undetermined.

**Outcome** — what happened when an approved Tier 3 executed.
Correlation, approver, executed-at, result, whether it failed. Written
after the fact by definition. No digest; it joins a Decision by
`correlationId`.

Their schemas are `decision-record/0.2.0`, `attempt-record/0.2.0` and
`outcome-record/0.1.0`. They are normative. They reach 1.0.0 when an
implementation other than the reference one has been held to them.

A record is written to a sink. What the sink must provide is C10's,
in `spec/README.md`: the record survives failure of the component
whose behaviour it records, is retrievable by someone not present
at the write, and absence is detectable. A custody ledger and a
gateway log both satisfy. An implementation MUST state which it
relies on.

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

This closes that pattern. Of the twelve named criteria
the audit of tasks 25, 26, 39 and 42 covered, three
were never stated: C1, C3 and C4 — C3's requirement existed
unattached to its name, C1 and C4 not at all. Of the nine
that were stated, seven prescribed a mechanism where a
property was meant. Task 25 classified C1 from the engine
PRD without noticing the sentence was not in this
specification; task 26's conclusion that C1 was not a
mechanism was reached about that same sentence. The method
held; the corpus of documents it ran against did not. **C5,
C6, C9 and C11 are restated as of 1.13.0** as the properties
their prescriptions were standing in for; the named status,
store, and strip remain satisfying mechanisms. **C12 is
restated as of 1.12.0** as coverage, not as a generator; the
compiled refuse-to-emit is one satisfying mechanism. **1.14.0
scopes that coverage to the composed description** from which
the enforcement configuration was generated; a path reachable
at the enforcement point but absent from that composition is
outside the criterion. **1.15.0 scopes C10 to that same
governed set.** **C8's rejection branch is not**, and
cannot be until the authorisation layer exists. Expiry is
distinct and does run.

The same finding now applies to the specification's own artefacts,
not only to its criteria. Four have been corrected after something
was built against them — ADR-006's fail-closed floor, C10's
record-before-act, the `standard` profile, and the composition digest
on the ledger — and each named a mechanism where it meant a
property. 1.8.0 restates C7 the same way, found by asking what the
criterion requires rather than by another implementation colliding
with the letter.

**1.16.0 is the identifier pass.** Of **195** identifiers across
**33** classes, **22** had no requirement sentence and **3**
were stated under a different name. The numbers count the
identifiers and classes the sweep used — a floor for a
reader-complete inventory; a later reader naming another
class raises both.

**1.17.0 admits C13.** The accounting is thirteen —
the adjudicated count, not the total of
paragraphs that meet the admission rule. Five
further paragraphs meet that rule and have no
letter, named here as candidates under review:
bound comparison (interpret a bound as a
decimal); no agent-facing URL; no credential
persistence; agent suspension is an
authorisation concern; and the namespace
strip in headers.md (the MUST strip, not
requirement 3). The document does not state
how many of the thirteen currently state a
mechanism; seven of nine is the audit of a
different set. The audit of tasks 25, 26, 39
and 42 covered twelve criteria. C13 was not
among them; its classification was recorded
in 1.17.0, after the audit closed.
**1.18.0 defines the governed set once**
(`governed-set` in spec/README.md). C1, C10
and C12 reference that identifier. No change
in meaning. C2 was compared and is not
collapsed: it names the undeclared default's
subject, not the criterion boundary.
**1.19.0 states what a record sink
must provide.** The record survives
failure of the component whose
behaviour it records, is retrievable
by someone not present at the write,
and absence is detectable. A custody
ledger and a gateway log both
satisfy. An implementation MUST
state which it relies on. C10's
requirement sentence is unaltered.
The sink obligation is on every
record the implementation writes.
The log and custody are not
equivalent. C10-via-gateway-log is
Observed on both live engines.
**1.20.0 states trust status for
`X-Delegation-Chain`.** The header
does not take a letter. The
admission rule's scope is
implementation behaviour that
mints a C-series criterion; a
header rule is a header rule.
The header is agent-supplied
lineage. The enforcement point
does not strip it; the name is
outside the 16c bound. The
backend rule is deployment
requirement 5: the header is
information; treating it as
authority is a condition ctier
states and does not enforce.
The enforcement point MUST NOT
classify from it. Compiled
client entitlement (1.3.0) is
the bound; believing this header
is a path around that
replacement. Live Kong and
APISIX do not read it. They
satisfy. It states trust status
for `X-Incident-Id`. The header
does not take a letter. It may
be emitted on a C9 response and
may be sent inbound. It is not
the join key. An inbound value
is agent-supplied. A backend
MUST NOT correlate or authorise
on it. Live targets do not emit
it, do not strip it, and do not
believe it. They satisfy the
trust rule. **It admits C14: no
credential persistence.** The
body is unaltered. Property.
Scope is the policy service, as
written — not the enforcement
point alone, and not everything
ctier specifies. Declared
evidence is none. The reference
policy service strips
`Authorization` from persisted
context. It satisfies. **It
admits C15: agent suspension is
an authorisation concern.**
Compiled-absence — the scope
removed at the authorisation
server — is the mechanism. This
forbids gateway-local
suspension. Declared evidence is
none. The reference does not
consult custody for suspension
and does not keep a denylist.
It satisfies the
enforcement-point half. The
authorisation server removing
the scope remains the estate, as
C8 already recorded. **It
rejects "recorded before acted
on" as a specification
requirement.** C10 already
states the property: the
decision precedes the act; the
record MAY be written
asynchronously and MUST be
durable. Re-adopting "before"
would undo 1.1.9. Both live
targets write `[ctier-decision]`
in the access phase before
proxy; a ledger POST that fails
is fail-open, and log durability
is requirement 4. Neither writes
a durable record before the
operation proceeds. Adopting
would make both non-conformant.
The sentence is an engine local
choice. It is to come out of
engine documents; this
specification does not edit
them. The claimed C10 sink is
`[ctier-decision]` in the error
log; an error-log line has no
request sequencing, so 1.19.0's
absence-detectable clause is
unmet there. Named gap; C10 is
unaltered. Of the five 1.17.0
candidates, two now have
letters. Three remain: bound
comparison; no agent-facing
URL; the namespace strip in
headers.md (the MUST strip, not
requirement 3). The accounting
is fifteen. A count change
breaks downstream iteration.
Any tooling that walks C1
through C13 now silently skips
C14 and C15 and reports
conformance.

---

## What a C-series criterion is

A requirement a second implementation must satisfy,
testable against a running deployment or a compiled
artefact. A validator-checked rule is a requirement a
document must satisfy, testable against this
specification's own files. Those are different tests.

Appearing in `conformance.md` with declared evidence is
not the test. That names a document location, and it
would be circular: a paragraph would become a criterion
by being listed. It is a cost of admission. A criterion
so admitted obliges a declaration of evidence there,
including none.

The specification mints the letter. An implementation
may propose a requirement; it does not mint the letter.

This rule is a property, not a location. Its scope is
requirements on an implementation's behaviour. It does
not admit a rule about the specification's own
documents, and it does not admit a deployment condition
the implementation cannot make unavoidable.

C13 meets this rule. It is the thirteenth criterion.
C14 meets this rule. It is the fourteenth criterion.
C15 meets this rule. It is the fifteenth criterion.

---

## What a validator is

A program that loads a description, an overlay, a rule document, or a
composition, and rejects inputs that fail the rules above. This repository does
not ship one. The invalid examples are the corpus it is written against.
