# Conformance corpus

The artefact a second implementation is held to. It cannot serve that
purpose while it lives in one implementation's test directory; 1.2.0
promotes it as `conformance-corpus/0.3.0`.

It is **normative, not 1.0.0**. This format reaches 1.0.0 when an
implementation **other than the reference one** has been held to a
corpus in it. Until then it defines what conformance means, but it has
only ever been read by its author, which is a weaker claim than
stability. A format used by one implementation is a serialisation; a
format used by two is an interface, and only the second has been tested
as one.

---

## What it establishes

**Classification agreement.** Given a composed description and a
request, two implementations resolve the same applied tier. Under a
declared `deployment` level, they select the same disposition.

That is the whole claim.

## What it does not

That either implementation is **correct**. Both can agree and both be
wrong. The corpus compares them to each other and to an expected pair;
it does not prove the pair.

Nor anything about custody's internals, the authorisation layer, or an
agent's behaviour on receiving a response. Those are other contracts.

The header contract in [`headers.md`](headers.md) is specified as of
1.12.0. **No conformance case asserts it.** The corpus compares
classifications and dispositions. It does not observe which headers
reach a backend, and it does not observe whether another component
read one first.

Against the 1.5.0 namespace property — no agent-supplied `x-ctier-*`
header reaches a backend — the reference prefix-walks on all three
targets and strips the aliases a prefix misses (`X-Correlation-Id`,
and the idempotency names outside the namespace). That closed-set
failure is gone.

The join key `x-ctier-correlation-id` is emitted on the forwarding
path. Where a ledger exists, live tests compare it to
`correlationId` on the decision record. At Levels 1 and 2, Kong and
APISIX write a schema-valid decision to the gateway log; the join
key joins that line. Apigee remains artefact-only: generated
`print()` is the Trace tool, not a production log.

**Declared gap — C7 at Level 2.** The reference does not satisfy C7
at Level 2: inbound agent keys are stripped, and nothing is emitted
in their place. That is a choice, recorded in
[`README.md`](README.md#what-ctier-requires-of-the-deployment) and
on the adoption ladder, not a level-scoped exemption from C7. A
stored derived key is emitted when the implementation satisfies C7
by derivation (custody present). The header MUST in
[`headers.md`](headers.md) follows that stated mechanism. Same
class of evidence as Apigee's unproven differential.

A new case shape that observed what the backend received would make
those failures visible in the corpus. This document does not take
that decision.

---

## The dimensions a case declares

| Field | Why |
|---|---|
| `profile` | `standard` bounds an exchanged token by the requesting client's entitlement; `constrained` does not assume that, and puts the check at the enforcement point |
| `deployment` | `level-2` has no custody, so Tier 3 is a provisioning refusal; `level-3` has it, so Tier 3 withholds. Same classification, different disposition |
| `sequence` | Accumulation is stateful. A single request cannot express a decomposition |

A single-request case is a sequence of one. Existing cases did not need
rewriting when the schema grew; the shape stayed valid.

`profile` and `deployment` are properties of the run as well as of a
case. A case may override the document `profile`. `deployment` is
declared once, on the document, because it is how the system is
installed, not how one operation is classified.

**`standard`** — the authorisation server bounds an exchanged token by
the **entitlement of the client requesting the exchange**. An
operation's scope cannot be obtained by a client to which it was never
granted, whatever the exchange requests and whatever the subject token
carried.

This is a weaker requirement than reduction by token lineage, and
deliberately so: it is satisfied by authorisation servers that
implement no lineage enforcement at all, which is most of them at the
time of writing. A reader who knows RFC 8693 will assume the omission
is an oversight unless the document says it is a choice.

Delegation is bounded to identities that exist at deploy time. An
agent exchanges into an identity someone compiled; it cannot mint a
sub-agent at runtime with an arbitrary narrower scope. Any
implementation of this profile inherits that bound — it follows from
the mechanism, not from a choice the reference implementation made.

The digest identifies the deployment, from which the **declared**
delegation topology can be recovered. It does not record which parent
initiated a given exchange. In the reference implementation those are
the same thing, because a sub-agent identity may be declared under at
most one parent; an implementation permitting a shared sub-agent
identity recovers an ambiguous topology from the same digest and MUST
NOT claim chain evidence from it.

---

## The bounded imprecision

An escalating case asserts `appliedTierAtLeast` rather than an exact
tier on a specific request, because the accumulated floor is published
out of band and read from gateway-local state. Escalation is guaranteed
within threshold-crossing plus one poll interval, not on a nominated
request.

The Derived bound, from the configured poll interval of the
implementation that first built the poller:

> Escalation is guaranteed within **threshold-crossing plus one poll
> interval**. In the reference both live targets poll at 1000 ms, so
> the bound is 1000 ms. That figure is Derived: it follows from the
> interval. It is not a measurement, and a sample from a run does not
> stand in for it.

An exact expectation on the escalating request would encode that
interval into the corpus and make it flaky. The assertion that matters
is that escalation happened within the bound.

---

## What has been run against it

Keep this section current. A reader deciding whether to trust the
corpus needs to know which targets have been run and which have been
reasoned about.

- **Kong** — differentially proven at level-2 and level-3, standard
  profile, for **stateless** cases (24/24). Sequential cases have run
  live against Kong at level-3 (decomposition, under-threshold,
  interleaved scopes, decay, unavailable floor). They are not
  differential against the reference adapter: that adapter constructs a
  fresh classifier per request and would escalate on the crossing
  call — a different mechanism, not a disagreement.
  **Fragment mode** is differentially proven at level-2 and level-3
  over a populated existing proxy (same 24/24). Bundle and fragment
  agree on classification and disposition. Strip precedence against
  coexisting configuration is not a corpus claim; it is
  estate-dependent (requirement 3) and recorded below.
- **APISIX** — differentially proven at level-2 and level-3, standard
  profile, for **stateless** cases (24/24), bundle mode. Sequential
  cases have run live against APISIX at level-3 (the same five
  properties as Kong). They are not differential against the
  reference adapter, for the same reason as Kong. **Fragment
  mode** is the same 24/24 over a populated proxy. Precedence is
  likewise estate-dependent, not a general fragment result.
- **Standard profile** — implemented and live-verified against Keycloak
  26.3.3 in the reference rig. The deploy-time bound applies. Until
  13c, neither profile had an implementation of delegation-chain
  integrity: standard was undeployable on this pin, constrained was
  deliberately not built. A reader of the earlier evidence section
  could not have discovered that. A specification that quietly
  acquires an implementation reads the same as one that always claimed
  to have one.
- **Apigee** — golden-file correct, differentially **unproven**. No
  local runtime exists; verification waits on a real organisation.
  **Fragment mode is artefact merge only.** It prepends strip steps
  onto an existing ProxyEndpoint and does not inherit Kong or APISIX
  live results. A PreProxy FlowHook is outside the artefact.
- **Constrained profile** — no cases. Nothing has been built for it.
  It is now the only profile with no implementation.
- **C8's rejection branch** — not exercisable until the authorisation
  layer exists.

**Strip precedence (requirement 3).** A precedence result depends on
the estate it ran against. The boundary report names the estate.
These rows are what has been run:

| Target | Mode | Estate | Precedence |
|---|---|---|---|
| Kong | fragment | readers below 10000 | passes |
| Kong | fragment | `pre-function` at 1e6 | fails |
| APISIX | fragment | reader in ctier's slot | passes |
| APISIX | fragment | reader outside the slot | fails |
| Apigee | fragment | — | untestable, no local runtime |

The corpus still does not assert precedence. Live observation is
outside the classification corpus.

The three record schemas (`decision-record/0.2.0`,
`attempt-record/0.2.0`, `outcome-record/0.1.0`) have held a Decision,
an Attempt, and an Outcome through a full withhold-approve-execute
cycle on the same Kong stack. Same status: normative, not 1.0.0.
