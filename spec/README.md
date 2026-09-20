# ctier specification 1.23.0

The stable public surface is a set of versioned document formats and the
transformations between them (`docs/adr/0009-the-contract-is-the-documents.md`).

**Naming.** Specification 1.0.0 used `x-consequence-*`. 1.1.0 renames to the
`ctier` namespace. The 1.0.0 archive
([doi.org/10.5281/zenodo.22020288](https://doi.org/10.5281/zenodo.22020288)) is
unaltered and remains resolvable.

| Path | What it is |
|---|---|
| [`headers.md`](headers.md) | HTTP headers at the trust boundary: provenance, not authorisation |
| [`validation.md`](validation.md) | Schema-checked shape versus validator-checked semantics |
| [`conformance.md`](conformance.md) | What the corpus establishes, the case dimensions, and which targets have been run |
| [`schemas/`](schemas/) | JSON Schema 2020-12 for the ctier-authored formats |
| [`examples/`](examples/) | Valid instances, invalid instances one per semantic rule, and bound-comparison request cases |
| [`consequence-tiered-api.yaml`](consequence-tiered-api.yaml) | Worked OpenAPI 3.1 example |

`exportStatements` is unclassified **on purpose**. It is the one `absent`
operation in the reference document. Absence of a declaration on an
operation in the composed description is Tier 4 by C2 — fail-closed —
and completing that classification as tidying would remove the example of
the default the model rests on.

The qualified spec is an OpenAPI document carrying the Part 2 extensions. It
has no schema of its own; it has a validation profile in `validation.md`.

1.2.0 promotes the conformance corpus (`conformance-corpus/0.3.0`) and the
three ledger record types (`decision-record/0.1.0`, `attempt-record/0.1.0`,
`outcome-record/0.1.0`). They are normative, not 1.0.0: a format used by
one implementation is a serialisation; a format used by two is an
interface.

1.3.0 restates the `standard` profile as entitlement of the client
requesting the exchange, not reduction by token lineage, and records
the deploy-time bound that follows from that mechanism. The `profile`
enum is unchanged.

1.4.0 renames the ledger digest to `deploymentDigest` and widens it
to cover declared agent identities. Decision and attempt records move
to `0.2.0`. The outcome record never carried a digest and stays
`0.1.0`. The corpus does not assert a digest value and is not bumped.

1.5.0 specifies the HTTP headers that cross into software ctier does
not control ([`headers.md`](headers.md)): provenance, not
authorisation; `x-ctier-deployment` and `x-ctier-correlation-id` as
the load-bearing pair; the entire `x-ctier-*` namespace stripped on
ingress, regardless of what this target emits; custody's execute path
exempt because it constructs rather than forwards. A closed strip set
is a conformance failure against that property. The join key is MUST
on every request that reaches the backend; the reference now emits
it on the forwarding path. The C7 derived `Idempotency-Key` is MUST
on every such request as written; the reference emits it only when
custody is present (Level 3). Level 2 strips inbound keys and emits
none. Declared.

1.6.0 adds the third deployment requirement: where ctier's
configuration coexists with configuration it did not generate, the
ingress strip MUST run before any component that reads a request
header. Bundle mode has that order because it owns the path. Fragment
mode, as then measured, did not, on any of three gateways tested. The
strip-before-read property is stated in [`headers.md`](headers.md);
1.5.0 had stated only that an agent-supplied header MUST NOT reach a
backend.

1.7.0 restates requirement 3 as a property with a declared bound.
Implementations can guarantee strip-before-read for a class of
coexisting configuration; the class differs per target. A deployment
MUST still ensure the property. Where an implementation guarantees
it for some class, it MUST state that class precisely, including
what the class **excludes**, and the deployment remains responsible
for every component outside it. Coverage without exclusion is not
a description.

1.8.0 restates C7 as the property a derived key was standing in for:
a duplicate attempt MUST NOT produce a second execution, and no
party can suppress another's operation by choosing the identifier a
honouring backend uses to recognise a duplicate. Derivation remains
one way to satisfy it. An implementation MUST state which it relies
on. The 1.5.0 header MUST in [`headers.md`](headers.md) is not
scoped here.

1.9.0 attaches that MUST to the stated mechanism: an implementation
that satisfies C7 by derivation MUST emit the derived key on every
execute; another mechanism states itself instead, and the header
requirement does not apply to it. The requirement is not scoped by
deployment level. Level 2 of the reference does not satisfy C7; the
declined alternative is named there and on the adoption ladder.

1.10.0 removes "tier on spans" from the adoption ladder. That
capability has never existed in the reference. Level 1 is classify
and emit a decision record, enforcing nothing. The same release
names what the other rungs do not give: Level 2's Tier 3 is a
provisioning refusal, not a withhold; Level 4's accumulation is
target-dependent. The join key at Levels 1 and 2 joins a gateway-log
decision where that sink is the gateway's own log. Adds the fourth
deployment requirement: ctier does not guarantee durability of
those log records.

1.11.0 replaces that Level 4 qualification. Accumulation is on both
live targets. What still differs is how the timer starts, where the
shared dictionary is declared, that memory-pressure behaviour of
that dictionary is unmeasured, and that Apigee remains artefact-only
for this rung. Coverage without those exclusions is not a
description. The same release restates the escalation window as
Derived: the bound is threshold-crossing plus one poll interval.
The 1000 ms figure follows from the configured interval; it is not
a measurement. It records the standing exception to the
enforcement point as chokepoint: custody's execute path does not
traverse that point, and compromise of custody yields direct
backend access under the policy service's own identity.

1.12.0 restates C12 as coverage, not as a generator: no operation
reachable through the enforcement point executes without a tier
assignment in force. A compile-time refuse-to-emit and a
request-time refuse of an unassigned operation both satisfy it.
An implementation MUST state which it relies on. Absence of an
assignment MUST NOT be permissive: the operation is refused and
the refusal is recorded with the applied tier undetermined.

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

1.14.0 scopes C12 to the governed set: the operations present
in the composed description from which the enforcement
configuration was generated. A path reachable at the
enforcement point but absent from that composition is outside
this criterion. An implementation MUST make that set
discoverable. An undetermined refusal and a Tier 4 decision
are different facts; a ledger that cannot distinguish them
cannot tell a provisioning defect from a governance outcome.
C2's undeclared default applies to operations within the
composed description that carry no explicit tier assignment,
not to paths outside the composition. It records that
neither live target satisfies C12's miss clause: Kong
assigns Tier 4 via a global plugin; APISIX does nothing;
the criterion requires an undetermined refusal.

1.15.0 scopes C10 to the governed set: the operations present
in the composed description from which the enforcement
configuration was generated. A path reachable at the
enforcement point but absent from that composition is outside
this criterion. It titles C3: Unidirectionality in
validation.md is the criterion, wording unaltered. The
escalate-to field remains the field-level echo. It states
C1 and C4. C1 is the property: no governed operation
reaches its target unclassified; "recorded" was the PRD's
mechanism. C4 is the property: no decision path blocks on
human input, deliberately unbounded; occupancy is C5 and
the suspension rule. It corrects the criteria accounting:
of twelve named criteria, three were never stated.
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
1.21.0 titles Additive
evolution: the orphan MUST
43 found, in validation.md
and at the close of this
history, wording unaltered.
It does not take a letter.
The admission rule's scope
is implementation behaviour
that mints a C-series
criterion. A consumer of
records is a second
implementation's client or
auditor; this is a named
validation rule about how
those readers treat
unrecognised fields, not
enforcement-point behaviour.
The accounting remains
fifteen. The three remaining
1.17.0 candidates are
unchanged. It states the
contradiction: consumers MUST
ignore fields they do not
recognise, and the ledger
schemas set
additionalProperties: false.
An unrecognised field on a
closed record is invalid.
Both cannot hold of the same
object. The schemas stay
closed. The rule is scoped
to unknown x-ctier-* headers
at the trust boundary, and
to unrecognised keys in a
named extensions object
where a document type
defines one. The MUST is
unaltered. Withdrawing it
because the schemas were
closed would let the
mechanism mint the format.
attempt-record moves to
0.3.0. declaredTier is no
longer required. A present
declaredTier is the
declaration that was made.
Absence is the declaration
that was not. These are
different facts. 0.2.0
required an integer 1–4 and
could not say unknown; 4
was the worst value and
was written. A 0.2.0
validator rejects a 0.3.0
Attempt that omits the
field. Writers emit
attempt-record/0.3.0.
decision-record stays
0.2.0: the field was
already optional.
outcome-record stays
0.1.0. An auditable record
does not carry its own
sequence. Absence-detectable
is a property of the sink,
not of the record object.
epoch and seq on the log
line sit on the sink. A
consumer that extracts the
JSON and discards the line
envelope has left the sink.
50's arrangement is
conformant-by-statement.
Records carried
away from their sink
must carry the sink's
sequencing with them,
or gap detection does
not travel. That is
deployment requirement
6: a condition ctier
states and does not
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
The action is strip.
C11 names strip as one
way to meet the
honour-rule. This rule
does not restate C11.
C11 is unaltered.
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
C16 is the namespace
property: nothing the
agent sends under this
prefix reaches a
backend. Besides C11
it protects the join
key, the deployment
digest, unknown names,
and later emits. C11
depends on it only when
C11 chooses strip.
Never-read still
satisfies C11. That is
not circularity.
The admission rule
gains a clause: a
criterion is not another
criterion's mechanism.
Applied to the letters
already minted, it newly
excludes none. C13, C14,
C15 and C16 stay.
1.23.0 records C4's
evidence. Observed
witnesses of a property
that cannot be
exhaustively observed
(taxonomy 33). Three on
Kong and on APISIX: a
withhold is 202 with no
pollUrl; Level 2 Tier 3
is 403
provisioning_defect — a
decision, not a wait;
Tier 4 is 423
halt_and_hand_off and
that correlationId is
not in custody pending.
Passing does not
establish C4. C13's
declared evidence
remains none. Engine 55
has not landed on
origin.
It ran the retroactive
check 53a asserted.
Applied to C1 through
C16, the clause newly
excludes none. C8 and
C15 are the shape the
clause is about:
compiled-absence is how
C8's rejection has
effect; C15 is the
placement, which C8
does not require. C13
and C12 are different
facts. The clause was
not sharpened.
The observations in
conformance.md are
relative to
ctier-engine
455c5e66937ca3431f3859a205e8dca04f8b234f
(origin/main after 54).
C12's live miss
observation is that
commit's, not 38's.

Pre-1.0 the ctier-authored formats may break. Freeze at v1.0.0 alongside the
demo, not before. **Additive evolution.** Consumers MUST ignore fields they do not recognise.

---

## The governed set

`governed-set`

The operations ctier governs: those present in the
composed description from which the enforcement
configuration was generated. A path reachable at the
enforcement point but absent from that composition is
outside this set.

A change to this definition is a change to every
criterion that references it: C1, C10, C12.

---

## Response contracts

These are interface behaviour, not extension keys.

**C1 — Classification precedes routing.** No governed
operation reaches its target unclassified. C1 applies to
the `governed-set`.

That precedence may arise from the classification being
bound into the enforcement point's configuration at
deploy time, or from a classification at request time
before the request is routed. Both satisfy this
criterion. An implementation MUST state which it relies
on.

The **record** of a classification is evidence that it
was made, not the classification itself.

**C2 — Fail-closed.** Fail-closed is C2: an operation in
the composed description with no `x-ctier-*` declaration
is Tier 4. That default is not a default for paths
outside the composition.

**C3 — Escalation is unidirectional.** `x-ctier-escalate-to`
may never resolve below the tier already reached at that
point in evaluation.

**C4 — No decision path blocks on human input.** A
classification, a withhold, a refuse, or an escalation
MUST complete without waiting for a human. Where a
human must act, that action is a later, separate path
— not a step the decision waits on.

This criterion is deliberately unbounded. It applies to
every decision the implementation takes, including a
refuse of a path outside the composed description. A
bound to the governed set would permit a synchronous
approval of an unknown path, which is the failure this
criterion exists to forbid.

Where a human sits is not the requirement.

**C5 — A withheld agent does not retry and does not stay
engaged.** The agent's involvement with a withheld operation
ends when it receives the response. What happens next happens
to the operation, not to the agent. The response MUST be
terminal, MUST NOT invite retry of this step, and MUST
instruct the agent to continue the rest of the task without
this step.

That property may be satisfied by status `202`, outside
conventionally retried classes, with no `Retry-After`,
`retryable: false`, and a machine-readable `agentAction:
continue_task_without_this_step` mirrored in an
`X-Agent-Action` header, the policy service constructing the
complete body and the enforcement point relaying it unchanged;
or by another response that meets the same property. An
implementation MUST state which it relies on.

A response that ends the exchange without instructing the
agent to continue the rest of the task does not satisfy this
criterion. Silence that merely stops the agent is not the
withheld disposition.

The withheld response is subject to the no-URL rule in
`spec/validation.md`. An implementer who adds a `pollUrl` has not retried
the operation — they have left the agent engaged with it. The effect is
the one the design exists to prevent: custody acquires a caller for the
length of a human review, and the connection occupancy that out-of-band
approval was meant to eliminate reappears under another name.

Where the named `202` mechanism is used, the policy service
constructs the complete response body. The enforcement point
relays it unchanged and MUST NOT construct, augment, or
reformat it. Resolve it once, in the place that knows, and let
the edges carry it — the same reasoning as compiling the
decision rather than the declaration.

Required contents of that mechanism:

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

**C6 — What executes is what was approved, and the agent is
not the client.** On approval, the operation executed is
byte-identical to the operation that was approved in
**method, request target, query and body**. Those bytes MUST
be the bytes that were approved, not reconstructed from a
later submission. The agent does not resubmit and is not the
HTTP client of the execute.

That property may be satisfied by persisting the operation at
withhold and executing from that store under the policy
service's own identity; or by a signed snapshot the agent
never resubmits, executed by a party that is not the agent.
An implementation MUST state which it relies on.

Loosening the store does not loosen either half. An agent
that presents the approved bytes is still the client. A later
submission that happens to match is not the approved bytes.

Byte-identity does **not** extend to the credential. The executed request
carries the policy service's own identity; the originating agent and the
authorising person are recorded as attribution. A policy service that
replays the agent's credential has stored one, which `spec/validation.md`
forbids. A store of live agent credentials awaiting replay is a larger
liability than the one custody exists to manage, and credentials expire on
a schedule unrelated to human review.

**C7 — A duplicate attempt MUST NOT produce a second execution.** An
operation that is the same in declared identity (`operationId`, or
method + path template) and in request target, query, body, agent
identity and task identity MUST NOT execute twice because it was
submitted twice, or because an execution event was applied twice.

That property may be satisfied by deriving a key from those fields
when the operation is first classified, storing it with persisted
context, and transmitting it so a honouring backend recognises the
duplicate; or by another mechanism that meets the same property. An
implementation MUST state which it relies on.

**No party can cause another party's operation to be suppressed by
choosing the identifier a honouring backend uses to recognise a
duplicate.** An inbound `Idempotency-Key` or `X-Idempotency-Key` is
a choice by the requesting agent. It MUST NOT be that identifier.

Headers, including credentials, are not part of the operation's
identity for this criterion. Including a credential produces an
identifier that changes when the credential rotates, which defeats
the guarantee.

If the chosen mechanism derives a key from the request, the key is
derived once, when the operation is first classified, and stored
with the persisted context. It is **not** re-derived at execution
time. Re-deriving from a reconstructed request risks deriving from
something subtly different from the original.

Where a header carries the identifier, its name is `Idempotency-Key`.
A second implementation choosing `X-Idempotency-Key` produces a
backend that silently does not deduplicate, with no error raised
anywhere.

**C8 — Expiry and rejection differ.** Rejection pauses the agent for the
operation class. Expiry refuses the single operation and leaves the agent
working.

The effect of a rejection is **observed, not announced**. A paused agent
discovers its state on its next attempt at an operation of that class, which
is refused. No notification, callback or status affordance is created for
it. The pause is state, not a message.

**C9 — A Tier 4 operation does not execute under the agent's
identity.** No state transition exists by which a Tier 4
operation executes under the agent's identity, or under an
identity derived from it. A service identity minted from the
agent's token is the agent's identity in all but name. The
human performs the operation under their own credential, and
the agent does not.

That property may be satisfied by status `423`,
`AgentSuspended`, carrying `incidentId`, `autoResume: false`,
and `agentAction: halt_and_hand_off`, with no correlation
identifier; or by another response that names no resume
handle and instructs the agent to halt and hand off. An
implementation MUST state which it relies on.

The agent must not seek an alternative route to the same
outcome. A location in the agent's response makes the agent
the mediator of a handover it was excluded from. The
exclusion is the control.

The no-URL rule in `spec/validation.md` applies. Tier 4's
content is the handover. The no-correlation-identifier rule
is the same prohibition: a handle the agent can watch is a
resume path.

**C10 — No operation executes without a decision having been made.**
C10 applies to the `governed-set`.
The decision MUST precede the action. That precedence may arise from the
decision being bound into the enforcement point's configuration at
deploy time, or from a synchronous evaluation before the request
proceeds; both satisfy this criterion and an implementation MUST state
which it relies on.

The **record** of a decision is evidence that it was made, not the
decision itself. It MUST carry the declared tier, the applied tier, the
escalation reason and the deployment digest. It MAY be written
asynchronously, and MUST be durable — an implementation whose records
can be lost has not recorded them.

A **sink** is where that record is written. The record MUST survive
failure of the component whose behaviour it records. It MUST be
retrievable by someone who was not present when it was written.
Absence of a record MUST be detectable — a sink that cannot be
audited for gaps lets this criterion hold for every request that
was recorded.

That property may be satisfied by writing to a custody ledger, or
by writing to the gateway's own structured log. An implementation
MUST state which it relies on. The gateway log qualifies because
it is not the component whose failure a custody-unavailable
Attempt records, and because a later reader can retrieve the line
from the log — or from a collector the operator has configured —
without having been present at the write.

An auditable record does not carry its own sequence.
Absence-detectable is a property of the sink, not of
the record object. `epoch` and `seq` on a log line sit
on the sink. A consumer that extracts the JSON and
discards the line envelope has left the sink. That
arrangement is conformant.
Records carried away from their sink must carry the
sink's sequencing with them, or gap detection does
not travel. That is deployment requirement 6. It
is not this criterion.

C10's coverage — no operation executes without a decision —
remains the `governed-set`. The sink requirement is not that
coverage. It is an obligation on the implementation's own
conduct: every record it writes, including a refusal of a path
outside the composition. A bound to the governed set would
permit a non-qualifying sink for those records. Coverage gets
scoped; the obligation on how the implementation records does
not.

Where the policy service is unavailable and the enforcement point
refuses without classifying, the refusal MUST itself be recorded,
marking the applied tier as undetermined. A refusal to decide is a
decision.

If the record sink is unavailable alongside the policy service, that
window has no record. ADR-005 accepts this rather than implying a
guarantee the design cannot make.

**C11 — The agent does not choose its tier.** A tier arriving
as an inbound client header MUST NOT be honoured. The
enforcement point classifies from its own assignment, not
from a value the agent set.

That property may be satisfied by stripping any
client-supplied tier at the edge before setting the
enforcement point's own; or by integrity-protecting the
value so a gateway-set tier is distinguishable from a
caller-set one; or by binding the assignment at deploy time
and never reading an inbound tier header. An implementation
MUST state which it relies on.

Honouring means treating the inbound value as the assigned
tier. An implementation that never reads the header for
classification still satisfies this criterion. The namespace
strip — no agent-supplied `x-ctier-*` header reaches a
backend — remains independently required by
[`headers.md`](headers.md) and is not this criterion. Plugin
order is deployment requirement 3, bounded per target, and
is not restated here (16a, 16c).

Without this, an agent declares itself Tier 1 and the model
collapses.

**C12 — No reachable operation executes without a tier assignment
in force.** C12 applies to the `governed-set`.
An operation in that set MUST have a tier assignment in force
before it executes.

That assignment may be bound into the enforcement point's
configuration at deploy time — a generator that refuses to emit an
unassigned operation is one way — or evaluated at request time
against a loaded description, where a miss is a refuse. Both
satisfy this criterion. An implementation MUST state which it
relies on.

An implementation MUST make the governed set discoverable — an
operator must be able to determine which operations are covered
without reading generated configuration.

C2's undeclared default is an assignment in force: a known
operation with no `x-ctier-tier` is classified (default Tier 4),
not unassigned. That default applies to operations within the
composed description that carry no explicit tier assignment.
It is not a default for paths outside the composition.

The absence of an assignment MUST NOT resolve to a permissive
outcome. Where no assignment is in force for a governed
operation, the operation MUST be refused, and **that refusal MUST
be recorded**, marking the applied tier as undetermined. An
undetermined refusal records that no assignment was found. A
Tier 4 decision records that a maximally consequential
operation was refused. These are different facts, and a ledger
that cannot distinguish them cannot tell a provisioning defect
from a governance outcome.

**C13 — Digest discipline.** Stated in `spec/validation.md`.

**C14 — No credential persistence.** Stated in `spec/validation.md`.

**C15 — Agent suspension is an authorisation concern.** Stated in `spec/validation.md`. This forbids gateway-local suspension.

**C16 — The namespace is stripped on ingress.** Stated in `spec/validation.md`. The property is the namespace: nothing the agent sends under this prefix reaches a backend. C11 is the honour-rule for one name and is not this criterion.

**Excluded — `403`.** Refused at credential validation, before consequence
evaluation. If this is reached by a live token, a scope has been over-granted.

## What ctier requires of the deployment

ctier compiles configuration into an enforcement point. It cannot
make itself unavoidable. The conditions below have to hold in the
deployment for the guarantees to hold. None is a defect.

1. **The backend MUST accept requests only from the enforcement point
   and from custody.** Network policy or mutual authentication does
   that. A deployment where the backend is reachable directly has the
   classifications and none of the guarantees.
2. **Custody's execute path bypasses the enforcement point by
   design.** The decision was already taken when custody executes.
   The backend MUST treat custody as a trusted caller by some means
   ctier does not provide. What an attacker gains if that caller
   is compromised is in the threat model below.
3. **A deployment MUST ensure the ctier ingress strip precedes any
   component that reads a reserved header.** Where an implementation
   guarantees this for some class of coexisting configuration, it
   MUST state that class precisely, and the deployment remains
   responsible for every component outside it. The declared class
   MUST state what it **excludes**, not only what it covers.
4. **ctier does not guarantee durability of gateway-log decision
   records.** An operator who has not configured a durable collector
   for `[ctier-decision]` lines has not recorded them. Custody, when
   deployed, remains the durable ledger. This requirement does not
   apply to records written there.
5. **A backend MUST NOT treat `X-Delegation-Chain` as a grant.** The
   header is agent-supplied lineage. The enforcement point does not
   strip it. ctier cannot prevent a backend from believing it. A
   deployment that grants because this header is present has the
   header and none of the entitlement.
6. **Records carried away from their sink must carry the sink's
   sequencing with them, or gap detection does not travel.** A
   SIEM, aggregator or compliance export that takes the JSON and
   leaves the line envelope has left the sink. ctier cannot put
   that sequence on the record — the schemas stay closed — and
   cannot make the export carry it. An operator who ships the
   extract without the sink's sequencing has the records and none
   of the gap detection.

An enforcement point where another component reads a reserved header
first has the classifications and not the forgery protection.

Requirements 1, 2 and 5 are satisfied at deployment. Requirement 3 must
be re-established whenever the enforcement point's configuration
changes, whether or not ctier generated the change — including when
the change is a component outside the implementation's declared
class. Requirement 4 is the exclusion for the log sink: a collector
the operator does not run is not a record. Requirement 6 is the
exclusion for export: records that leave the sink without its
sequencing have left gap detection behind.

An implementation SHOULD provide a means of detecting components that
read reserved headers, and a deployment relying on requirement 3
SHOULD re-run that detection whenever the enforcement point's
configuration changes. In the reference implementation this is the
generator's conflict report, which makes regeneration part of the
operating procedure rather than only a build step. The bound an
implementation claims is its own publication; this specification
does not enumerate target-specific thresholds.

A specification that leaves these implicit invites someone to deploy
it and believe more than it does.

## Threat model

The enforcement point is not the only path to the backend. When a
withheld operation is approved, custody executes it directly, and
nothing compiled into the enforcement point applies to that request
— including the ingress strip.

This is safe for a different reason than the forwarding path:
custody constructs the request from persisted context rather than
forwarding one the agent composed, so there is no agent-supplied
header to strip and no agent-chosen parameter to re-validate.

It follows that a deployment must treat custody as a trusted caller
by means ctier does not provide — deployment requirement 2 — and
that compromise of custody yields direct backend access under the
policy service's own identity.

**Level 2 and C7.** Level 2 enforces Tiers 1, 2 and 4 with no ctier
runtime component. It does not satisfy C7: duplicate execution under
ordinary agent retry is not defended against at this level.

A mechanism was available — an agent-supplied key namespaced by a
ctier-supplied identifier, so that no agent can suppress another's
operation — and was declined, because it would place an identity
value on the request that ctier had removed for separate reasons.
An implementation willing to make that trade could satisfy C7 at
Level 2.

## Adoption ladder

Each rung is independently useful. Coverage without exclusion is not
a description.

| Level | Deployed | Gives |
|---|---|---|
| 0 · Classify | nothing | a review queue in risk order, and a number: what share of the estate is unclassified |
| 1 · Observe | proxy config | classify and emit a decision record, enforcing nothing; coverage warns |
| 2 · Enforce 1/2/4 | proxy config + AS flags | first level with teeth, still no new runtime; coverage fails the build |
| 3 · Withhold | custody | Tier 3 end to end |
| 4 · Accumulate | scope floor | decomposition defence, and the cycle back to design |

Level 2's "coverage fails the build" is one C12 mechanism: the
reference generator refuses to emit an unassigned operation
present in the composed description. C12 is the property (no
operation in that set executes without an assignment in force),
not the build failure. A request-time miss that refuses and
records is the other named mechanism. Level 0
deploys nothing, so C12 does not yet apply; the unclassified share
is a coverage-report number, not an enforcement-point assignment.

Level 2 enforces Tiers 1, 2 and 4 with no ctier runtime component.
It does not satisfy C7: duplicate execution under ordinary agent
retry is not defended against at this level.

A mechanism was available — an agent-supplied key namespaced by a
ctier-supplied identifier, so that no agent can suppress another's
operation — and was declined, because it would place an identity
value on the request that ctier had removed for separate reasons.
An implementation willing to make that trade could satisfy C7 at
Level 2.

**Level 1 does not stamp spans.** The ladder used to claim that.
The capability has never existed in the reference. Level 1
classifies and emits a decision record, and enforces nothing.

**Level 2's Tier 3 is a provisioning refusal** (`refuse-provisioning`), not a withhold.
Withhold is Level 3.

**The join key has something to join at Levels 1 and 2** where the
gateway-log sink is the gateway's own structured log (Kong, APISIX).
That is a note, not an exclusion. Apigee's generated `print()` is
the Debug/Trace session (`stepExecution-stdout`), not Cloud Logging
or any production log stream. An evaluator with a trace running can
read the object; an operator without one cannot.

**The gateway log and custody are not equivalent sinks.** The
reference writes to the log at Levels 1 and 2, and to custody at
Level 3. An Attempt that records custody unavailable is written
to the log.

What the log provides. A schema-valid Decision or Attempt that
survives the component whose failure it records — custody down
is the named case. A later reader can retrieve the line from
the gateway's structured log, or from a collector the operator
has configured.

What it does not. A ledger you can query for what is missing.
Custody's records are the set; a missing correlation is a fact.
A log you cannot audit for gaps lets C10 hold for every request
that was recorded. That is the third property, met only as far
as a collector can be joined to an independent request stream.
`epoch` and `seq` on the line are the sink providing that
property, not fields of the record. A consumer that extracts
the JSON has left the sink. Records carried away from their
sink must carry the sink's sequencing with them, or gap
detection does not travel. That is requirement 6.
Durability remains requirement 4.
Outcome records remain
custody's — the log has never carried them.

That is why the ladder has three rungs below Level 4, not one.
Level 1 and 2 emit through the log. Level 3 has custody. An
operator who has the first has classified and recorded. They
have not withheld, and they have not got a ledger.

**Level 4 accumulation is on both live targets, with remaining
differences.** Kong and APISIX both publish a scope floor from an
out-of-band poller into worker-local shared state and read it on
the request path. What still differs:

- How the timer starts. Kong starts it from the plugin worker
  module. APISIX has no equivalent worker-init hook; the generated
  Lua claims one timer per worker with a pid-keyed shared-dict
  add, so copies on every classified route do not each start one.
- Where the dictionary is declared. Kong's is compose configuration
  of the plugin. APISIX's is node config, not the generated route
  deck. An operator who omits it gets no escalation, not a refuse.
- Memory-pressure behaviour of that shared dictionary has never
  been observed.
- Apigee has no live runtime for this rung. A generated bundle is
  not a Level 4 deployment.

"Both targets support Level 4" with nothing beside it is the
positively-stated claim these exclusions exist to prevent.


