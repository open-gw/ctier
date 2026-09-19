# Headers at the trust boundary

Specification 1.22.0.

These are the HTTP headers that cross into software ctier does not
control: the backend, and the agent. Document keys in
[`extensions.md`](extensions.md) are not this contract. Flow
variables that never leave the enforcement point are not this
contract.

These headers are **provenance, not authorisation.** A backend MUST NOT treat
the presence, absence or value of any ctier header as evidence that a request
was authorised, approved, or classified at any tier. They exist so a backend
can correlate a request with the deployment and the decision that produced it,
after the fact.

A backend that grants access because a ctier header is present has been
compromised by anything that can set a header.

---

## The namespace is stripped on ingress

**No header in the `x-ctier-*` namespace that reaches a backend may
be agent-supplied.** The enforcement point MUST strip the entire
namespace on ingress, regardless of which names this target emits.
A closed list of names known at generate time is a mechanism. The
property is the namespace.

**The strip must run before any other component of the enforcement
point reads such a header.** "On ingress" is temporal, not only
spatial. 1.5.0 stated that no agent-supplied `x-ctier-*` header may
reach a backend. It did not say the strip had to precede a read.
A strip that still keeps those headers off the backend after CORS,
jwt, or a request transformer has already seen them satisfies that
letter and not the property the letter was protecting: another
component may have acted on an agent-supplied value. The named
aliases (`X-Correlation-Id`, `X-Ctier-Tier`, and the inbound
idempotency keys) are in the same temporal scope.

This order is free in bundle mode, because ctier owns the path.
Where ctier's configuration coexists with configuration it did not
generate, the deployment owes requirement 3
([`README.md`](README.md#what-ctier-requires-of-the-deployment)).
An implementation that guarantees the order for some class of
coexisting configuration MUST declare that class, including what
the class excludes. The deployment remains responsible for every
component outside it. C11 is the honour-rule: the agent does
not choose its tier. Classification ignores a header an agent
set. It is not restated here as a plugin-order criterion.

Kong's prefix strip satisfies the namespace by construction. Apigee
and APISIX now prefix-walk as well; `proxy-rewrite` remove is aliases
only (A2). Inbound aliases that carry the same values outside the
namespace MUST be stripped too. Today that is `X-Correlation-Id`
(a prefix of `x-ctier-` will not catch it) and `X-Ctier-Tier` (a
prefix will). `Idempotency-Key` and `X-Idempotency-Key` are outside
the namespace and are stripped as the C7 inbound rule, not as
aliases of a ctier name.

**Exemption: custody's execute path.** Custody calls the backend
directly. Nothing is stripped there. That is safe because custody
**builds** the request from persisted context (C6's named
persist-and-execute mechanism: method, target, query, body)
rather than forwarding a request the agent composed.
There is no agent-supplied header on that hop to forge. The
namespace strip protects the forwarding path; construction protects
the other. Custody MUST still not copy agent-supplied `x-ctier-*`
values from stored inbound headers onto the execute request.
What compromise of that path yields is in the threat model
([`README.md`](README.md#threat-model)).

---

## Upstream — to the backend

### `x-ctier-deployment`

| | |
|---|---|
| Direction | Request → backend, on a forwarded execute |
| Value | The deployment digest in force (`sha256:` + 64 lowercase hex) |
| Encoding | ASCII. Same spelling as `deploymentDigest` on the decision record |
| Requirement | **MUST** be set by the enforcement point on a request it forwards |

Identifies the compiled configuration that classified the call. It
does not mean the call was permitted.

Apigee's generated policies write this digest to a flow variable
(`ctier.deployment`) and do not set the HTTP header. Kong and APISIX
set the header. A backend that requires the header will not see it
from an Apigee hop unless the deployment adds that write.

### `x-ctier-correlation-id`

| | |
|---|---|
| Direction | Request → backend, on every execute that reaches it |
| Value | Identifies the decision record for this call |
| Encoding | ASCII. Same value as `correlationId` on that record, and on a 202 body when one exists |
| Requirement | **MUST** be set on any request the enforcement point forwards, and on custody's execute request |

This is the join key. Attribution of agent and approver lives on the
decision record, not on this request. A backend recovers them as:
correlation id → decision record → agent and approver. Auditability
is a headline claim; it applies to ordinary Tier 1 and Tier 2
traffic as well as approved execute. If this header does not arrive,
that recovery has nothing to join.

The reference implementation emits this on the forwarding path as
well as on custody's execute. Where a ledger exists, the value is
the decision record's `correlationId`. Apigee has no local runtime;
that hop is artefact-only.

### `Idempotency-Key`

| | |
|---|---|
| Direction | Request → backend, on an execute that reaches it |
| Value | The derived key, when the implementation satisfies C7 by derivation |
| Encoding | ASCII |
| Requirement | An implementation that satisfies C7 by derivation MUST emit the derived key on every execute, under this header. An implementation satisfying C7 by another mechanism states that mechanism instead, and this requirement does not apply to it. An inbound `Idempotency-Key` or `X-Idempotency-Key` is agent-supplied and MUST NOT reach the backend. |

C7 is the duplicate-execution property. Derivation is one way to
satisfy it. This header is a requirement of that mechanism, not of
a deployment level. Level 2 in the reference lacks the key because
it lacks the mechanism. The inbound strip still applies: an
agent-supplied key MUST NOT be the identifier the backend honours.

### Other upstream names

These also cross. They are provenance under the same paragraph. A
backend MUST NOT authorise on them.

| Name | Value | Requirement |
|---|---|---|
| `x-ctier-tier` | Applied tier, decimal 1–4 | **MAY** |
| `x-ctier-operation` | Compiled `operationId` | **MAY** |
| `x-ctier-composition` | Same digest as `x-ctier-deployment` | **MAY** until engine 0.3.0, then MUST NOT |

`Authorization` on custody's execute request is the policy service
authenticating as itself (C6: the agent is not the client).
Pass-through of the agent's inbound headers (including
`X-Agent-Id`) is not a ctier emit.

### `X-Delegation-Chain`

| | |
|---|---|
| Direction | Request → enforcement point. MAY reach a backend; the name is not stripped |
| Who may set | The agent. The enforcement point does not emit it. An estate component MAY inject it |
| Strip | The enforcement point does not strip it. The name is outside `x-ctier-*`. Requirement 3's 16c bound is the namespace and the named aliases (`X-Correlation-Id`, `X-Ctier-Tier`, the inbound idempotency keys). This name is not in that bound |
| Value | Ordered ancestry the agent asserts, root first |
| Requirement | **Provenance, not entitlement.** The backend rule is deployment requirement 5. The enforcement point MUST NOT classify from it |

1.3.0 restated `standard` as compiled client entitlement, not
token lineage. This header carries lineage. It is not that
entitlement. Believing it — classifying from it, or granting
because it is present — is a path around that replacement.
1.3.0 removed the lineage mechanism; it did not deprecate it.
The name remains so an agent can still send it. The name is
not a second mechanism.

---

## Downstream — to the agent

Already under C5 and C9. The example 202 and 423 responses
are the named mechanisms, not the criteria. Listed here
because they cross.

The join key has **one** wire name: `x-ctier-correlation-id`, both
directions. `X-Correlation-Id` is the name the 1.4.0 example used. It
sits outside the namespace a prefix strip catches. Downstream, 1.5.0
uses `x-ctier-correlation-id`. Inbound `X-Correlation-Id` is stripped
under the alias rule above. The reference implementation still
relays `X-Correlation-Id` on 202 today.

| Name | Value | Requirement |
|---|---|---|
| `X-Agent-Action` | Machine-readable instruction (`continue_task_without_this_step`, `halt_and_hand_off`, `retry_later`) | An implementation that satisfies C5 by the `202` mechanism MUST emit it on 202. An implementation that satisfies C9 by the `423` mechanism MUST emit it on 423. Another mechanism states itself instead. |
| `x-ctier-correlation-id` | Pending `correlationId` | **MUST** on 202 when that is the C5 mechanism; **MUST NOT** on a C9 response (no resume handle) |
| `X-Ctier-Tier` | Applied tier | **MAY**. Inbound, stripped as an alias |

`Retry-After` is **MUST NOT** on 202 when that is the C5
mechanism, and is present on 503 when custody is unavailable
(ADR-005). `Cache-Control: no-store` on agent-facing faults
is ordinary HTTP.

### `X-Incident-Id`

| | |
|---|---|
| Direction | Response → agent, on a C9 refusal. MAY appear inbound if the agent sends it |
| Who may set | The enforcement point MAY emit it on a C9 response. The agent MAY send it inbound |
| Strip | Inbound, the enforcement point does not strip it. The name is outside `x-ctier-*` and is not in the 16c bound |
| Value | Incident identifier for the refusal. Not the join key |
| Requirement | **Not a join key.** A backend MUST NOT correlate or authorise on it. The join key is `x-ctier-correlation-id`. An inbound value is agent-supplied. C9 forbids a correlation identifier on this disposition; this header is not one |

An agent can present any value. That does not forge the
join: inbound `x-ctier-correlation-id` is stripped and the
enforcement point writes its own. Forging this name forges
an incident label, not the decision record.

The conditions an adopter still owes — the backend reachable only
from the enforcement point and from custody, custody treated as a
trusted caller by some means ctier does not provide, and a backend
that does not treat `X-Delegation-Chain` as a grant — are in
[`README.md`](README.md#what-ctier-requires-of-the-deployment).
