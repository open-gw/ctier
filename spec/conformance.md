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

The measured window, from the implementation that first built the
poller (`ctier-engine` `docs/findings/12a.md`):

> With a poller the true window is **threshold-crossing plus up to one
> poll interval (1000 ms)**.

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
- **Constrained profile** — no cases. Nothing has been built for it.
  It is now the only profile with no implementation.
- **C8's rejection branch** — not exercisable until the authorisation
  layer exists.

The three record schemas (`decision-record/0.2.0`,
`attempt-record/0.2.0`, `outcome-record/0.1.0`) have held a Decision,
an Attempt, and an Outcome through a full withhold-approve-execute
cycle on the same Kong stack. Same status: normative, not 1.0.0.
