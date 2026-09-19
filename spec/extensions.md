# ctier extension reference

Specification 1.22.0. Flat keys — the OpenAPI namespace format is `x-{namespace}-`,
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
asynchronously, and MUST be durable. Names the three ledger record
types — decision, attempt, outcome — without schematising them. Closes
the maturity note: C5, C6, C7, C9 and C10 have each been built against.
1.2.0 promotes the conformance corpus (`conformance-corpus/0.3.0`) and
the three record schemas (`decision-record/0.1.0`, `attempt-record/0.1.0`,
`outcome-record/0.1.0`). They are normative, not 1.0.0. 1.3.0 restates
the `standard` profile as client entitlement, not token lineage, and
records the deploy-time delegation bound that follows from it. The
`profile` enum is unchanged. 1.4.0 renames the ledger digest to
`deploymentDigest` — composed description, tier declarations, and
declared agent identities — so a record written under `composition`
is obviously old. Decision and attempt records are `0.2.0`. The
outcome record never carried a digest and stays `0.1.0`. The 1.0.0
archive (DOI 10.5281/zenodo.22020288) is unaltered and remains
resolvable. 1.5.0 specifies the HTTP headers that cross the trust
boundary (`spec/headers.md`). Document keys in this file are not
those headers. 1.6.0 records strip-before-read as deployment
requirement 3. 1.7.0 restates it as a property with a declared
bound: a deployment MUST still ensure the strip precedes any
component that reads a reserved header; where an implementation
guarantees that for a class of coexisting configuration, it MUST
state the class precisely, including what the class excludes.
1.8.0 restates C7 as a property: a duplicate attempt MUST NOT
produce a second execution, and no party can suppress another's
operation by choosing the identifier a honouring backend uses.
Derivation is one satisfying mechanism, not the criterion.
1.9.0 attaches the derived-key header MUST to that mechanism, not
to a deployment level. 1.10.0 removes "tier on spans" from the
adoption ladder — that capability has never existed in the
reference — and adds the fourth deployment requirement: ctier does
not guarantee durability of gateway-log decision records.
1.11.0 replaces the Level 4 qualification: accumulation is on both
live targets; the remaining differences (timer start, dictionary
declaration, unmeasured memory pressure, Apigee artefact-only) are
stated as exclusions. The escalation window is Derived from the
configured poll interval, not measured. The threat model records
custody's execute path: it does not traverse the enforcement
point, and compromise of custody yields direct backend access
under the policy service's own identity.
1.12.0 restates C12 as coverage, not the generator: no reachable
operation executes without a tier assignment in force; a miss is
refused and recorded with the applied tier undetermined.
1.13.0 restates C5 as the property a status and a constructor
were standing in for: a withheld agent does not retry and does
not stay engaged. The `202` field list remains one way. An
implementation MUST state which it relies on. A response that
ends the exchange without instructing the agent to continue
the rest of the task does not satisfy it. It restates C6 as
two properties that must both hold: the bytes that execute
are the bytes that were approved, and the agent is not the
client. Persist-and-execute remains one way. An
implementation MUST state which it relies on. Loosening the
store does not loosen either half. It restates C9 as the
identity prohibition: a Tier 4 operation does not execute
under the agent's identity, or under one derived from it.
The `423` field list remains one way. An implementation MUST
state which it relies on. It restates C11 as the honour-rule
at the enforcement point: the agent does not choose its tier.
Strip and integrity-protection remain named ways. An
implementation MUST state which it relies on. The 16c bound
is requirement 3's, not this criterion's.
1.14.0 scopes C12 to the governed set: those present in the
composed description from which the enforcement configuration
was generated. A path reachable at the enforcement point but
absent from that composition is outside this criterion. An
implementation MUST make the governed set discoverable. An
undetermined refusal records that no assignment was found; a
Tier 4 decision records that a maximally consequential
operation was refused. C2's undeclared default applies to
operations within the composed description that carry no
explicit tier assignment, not to paths outside the
composition. It records that neither live target satisfies
C12's miss clause: Kong assigns Tier 4 via a global plugin;
APISIX does nothing; the criterion requires an undetermined
refusal.
1.15.0 scopes C10 to the governed set: those present in the
composed description from which the enforcement configuration
was generated. A path reachable at the enforcement point but
absent from that composition is outside this criterion.
It titles C3: Unidirectionality in validation.md is the
criterion, wording unaltered. The escalate-to field remains
the field-level echo. It states C1 and C4. C1 is the
property: no governed operation reaches its target
unclassified; "recorded" was the PRD's mechanism. C4 is
the property: no decision path blocks on human input,
deliberately unbounded; occupancy is C5 and the
suspension rule. It corrects the criteria accounting: of
twelve named criteria, three were never stated.
1.16.0 is the identifier pass. It titles C2: Fail-closed
in validation.md is the criterion, wording unaltered. It
titles C13: Digest discipline in validation.md is the
criterion, wording unaltered. It attaches
`refuse-provisioning` to Level 2's provisioning refusal.
Of 195 identifiers across 33 classes, 22 had no
requirement sentence and 3 were stated under a
different name.
1.17.0 states what a C-series criterion is: a
requirement a second implementation must satisfy,
testable against a running deployment or a compiled
artefact. A validator-checked rule is a requirement a
document must satisfy. The specification mints the
letter; an implementation proposes. It admits C13
as the thirteenth criterion. C13 states a
mechanism: how the digests are computed. The
property — an assignment cannot be applied to a
composition other than the one it was computed
from — is not restated here. The digest is over
the composition; that composition is the governed
set by definition. C13's declared evidence is none.
The accounting is thirteen. The audit of tasks 25,
26, 39 and 42 covered twelve criteria. C13 was not
among them; its classification was recorded in
1.17.0, after the audit closed. Conforming to
twelve is not conforming to thirteen. Digest
discipline has been in force since 1.4.0, so
nothing an implementation must do has changed.
A count change breaks downstream iteration. Any
tooling that walks C1 through C12 now silently
skips C13 and reports conformance. Thirteen is
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
different set.
1.18.0 defines the governed set (`governed-set`)
once. C1, C10 and C12 each carried the same
scope paragraph; the statement moves to that
definition. Those three now reference the
identifier. No change in meaning.
C2 was compared and is not collapsed. It
names the undeclared default's subject —
operations in the composition that carry no
explicit tier — not the criterion boundary.
That difference is not a moved statement.
C13's 1.17.0 sentence — the digest is over
the composition; that composition is the
governed set by definition — is not a fifth
copy of the scope paragraph. C4 remains
deliberately unbounded.
1.19.0 states what a record sink must
provide. The record survives failure of
the component whose behaviour it
records. It is retrievable by someone
who was not present when it was
written. Absence of a record is
detectable. A custody ledger and a
gateway log both satisfy that
property. An implementation MUST
state which it relies on. C10's
requirement sentence is unaltered.
The sink obligation is on every
record the implementation writes;
coverage remains the governed set.
The log and custody are not
equivalent: the log does not give a
ledger you can query for gaps.
C10-via-gateway-log is Observed on
both live engines.
1.20.0 states trust status for
`X-Delegation-Chain`. The header
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
trust rule. It admits C14: no
credential persistence. The
body is unaltered. Property.
Scope is the policy service, as
written — not the enforcement
point alone, and not everything
ctier specifies. Declared
evidence is none. The reference
policy service strips
`Authorization` from persisted
context. It satisfies. It admits
C15: agent suspension is an
authorisation concern.
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
C8 already recorded. It rejects
"recorded before acted on" as a
specification requirement. C10
already states the property: the
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
1.21.0 titles Additive evolution:
the orphan MUST 43 found, in
validation.md and at the close of
the README version history, wording
unaltered. It does not take a
letter. The admission rule's scope
is implementation behaviour that
mints a C-series criterion. A
consumer of records is a second
implementation's client or auditor;
this is a named validation rule
about how those readers treat
unrecognised fields, not
enforcement-point behaviour. The
accounting remains fifteen. The
three remaining 1.17.0 candidates
are unchanged. It states the
contradiction: consumers MUST
ignore fields they do not
recognise, and the ledger schemas
set additionalProperties: false.
An unrecognised field on a closed
record is invalid. Both cannot
hold of the same object. The
schemas stay closed. The rule is
scoped to unknown x-ctier-*
headers at the trust boundary,
and to unrecognised keys in a
named extensions object where a
document type defines one. The
MUST is unaltered. Withdrawing it
because the schemas were closed
would let the mechanism mint the
format. attempt-record moves to
0.3.0. declaredTier is no longer
required. A present declaredTier
is the declaration that was made.
Absence is the declaration that
was not. These are different
facts. 0.2.0 required an integer
1–4 and could not say unknown; 4
was the worst value and was
written. A 0.2.0 validator
rejects a 0.3.0 Attempt that
omits the field. Writers emit
attempt-record/0.3.0.
decision-record stays 0.2.0: the
field was already optional.
outcome-record stays 0.1.0. An
auditable record does not carry
its own sequence.
Absence-detectable is a property
of the sink, not of the record
object. epoch and seq on the log
line sit on the sink. A consumer
that extracts the JSON and
discards the line envelope has
left the sink. 50's arrangement
is conformant-by-statement.
Records carried away from
their sink must carry the
sink's sequencing with them,
or gap detection does not
travel. That is deployment
requirement 6: a condition
ctier states and does not
enforce.
1.22.0 splits Additive
evolution. The document
half is the reader of a
document: consumers MUST
ignore fields they do not
recognise, in a named
extensions object where a
document type defines one.
It does not take a letter.
The request half is the
namespace strip in
headers.md: unknown
x-ctier-* headers at the
trust boundary. The two
must not share a name. A
reader-of-documents rule
and an enforcement-point
rule are not the same
requirement — the same
principle as deployment
requirement 5. The
accounting remains
fifteen. Admission of the
request half is on its
own merits.
The request half states
one action: strip. 51
scoped Additive evolution
to those headers with the
word ignore. Nobody asked
the permit-question:
scoping felt like it
cannot loosen. The scope
landed where the rule was
never written. Ignore at
the trust boundary newly
permits forwarding an
attacker-chosen header.
The action is strip, the
mechanism C11 already
names. This rule does not
restate it. C11 is
unaltered.
It admits C16: the
namespace is stripped on
ingress. A second
implementation must strip
the entire x-ctier-*
namespace on ingress.
Testable against a
running deployment. Not a
document rule. Not a
deployment condition the
implementation cannot
make unavoidable. The
action is strip. C11 is
the honour-rule and is
not this criterion. The
accounting is sixteen. A
count change breaks
downstream iteration.
Any tooling that walks
C1 through C15 now
silently skips C16 and
reports conformance.
The 1.17.0 namespace
strip candidate — the
MUST strip in headers.md,
not requirement 3 — is
resolved as C16. It is
not a sixth candidate
and it is not absorbed
without a letter. Of the
five 1.17.0 candidates,
three now have letters.
Two remain: bound
comparison; no
agent-facing URL.
Declared evidence is
none.

The JSON Schemas in `schemas/` check shape. Semantic constraints are in
`validation.md`.

---

## Document level

Appear on the OpenAPI root.

### `x-ctier-defaults`

Object. Optional. Fail-closed values apply when a field is omitted.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `undeclaredTier` | integer 1–4 | no | Tier applied to an operation in the composed description with no `x-ctier-tier`. Default `4`. Applies only inside a granted scope; it is not a default for paths outside the composition. |
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

Fail-closed is unaffected. C2 operates one level up: an operation in the
composed description carrying no `x-ctier-*` declaration at all is Tier 4.
It is not a default for paths outside the composition. Completeness
*within* a declaration is a validation concern, not a classification one.

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
`validation.md` (C3).

---

## Overlay actions

An overlay action whose `update` sets any `x-ctier-*` field that resolves to a
tier below 4 — whether by declaring `x-ctier-tier` or by supplying
`x-ctier-criteria` that compute one — MUST select exactly one operation. An
action MAY select many operations only when it assigns Tier 4.

See `validation.md`.
