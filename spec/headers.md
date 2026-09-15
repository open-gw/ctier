# Headers at the trust boundary

Specification 1.5.0.

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

## Upstream — to the backend

### `x-ctier-deployment`

| | |
|---|---|
| Direction | Request, enforcement point → backend, on executing tiers |
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
| Direction | Request, custody → backend, on approved execute |
| Value | The pending context's `correlationId` |
| Encoding | ASCII string; the same value as `correlationId` on the 202 body and the decision record |
| Requirement | **MUST** be set by custody on the execute request |

This is the join key. Attribution of agent and approver lives on the
decision record, not on this request. A backend recovers them as:
correlation id → decision record → agent and approver. If this header
does not arrive, that recovery has nothing to join.

The enforcement point does not set this header on a request it
forwards. Tier 1 and Tier 2 execute with no pending context; there
is no decision record to join. The load-bearing path is custody's.

### Other upstream names the engine currently emits

These also cross. They are provenance under the same paragraph. A
backend MUST NOT authorise on them.

| Name | Value | Requirement | Emitted by |
|---|---|---|---|
| `x-ctier-tier` | Applied tier, decimal 1–4 | **MAY** | Kong and APISIX, executing tiers |
| `x-ctier-operation` | Compiled `operationId` | **MAY** | Kong (when the instance has `operation`) and APISIX |
| `x-ctier-composition` | Same digest as `x-ctier-deployment` | **MAY** until engine 0.3.0, then MUST NOT | Kong only, dual-emit window (13h) |

`Idempotency-Key` on custody's execute request is C7, not a ctier
claim. `Authorization` on that request is the policy service
authenticating as itself (C6). Pass-through of the agent's inbound
headers (including `X-Agent-Id`) is not a ctier emit.

---

## Downstream — to the agent

Already under C5, C9, and the example 202/423 responses. Listed here
because they cross.

| Name | Value | Requirement |
|---|---|---|
| `X-Agent-Action` | Machine-readable instruction (`continue_task_without_this_step`, `halt_and_hand_off`, `retry_later`) | **MUST** on 202 and 423; C5 |
| `X-Correlation-Id` | Pending `correlationId` | **MUST** on 202; **MUST NOT** on 423 (C9: no correlation identifier) |
| `X-Ctier-Tier` | Applied tier | **MAY** |

`Retry-After` is **MUST NOT** on 202 (C5) and is present on 503 when
custody is unavailable (ADR-005). `Cache-Control: no-store` on
agent-facing faults is ordinary HTTP.

---

## Emit is a subset of strip

Every header ctier emits upstream on the **forwarding** path MUST be
in the enforcement point's ingress strip set. Otherwise an agent
sets it itself and the backend cannot tell the two apart. Adding a
header and forgetting the strip is how this relation decays.

Kong satisfies it by prefix-stripping `x-ctier-*` and then setting
the names it emits. Apigee satisfies it vacuously for HTTP: the
generated policies strip a closed set and do not set `x-ctier-*` on
the target request. APISIX cannot list a name in both
`proxy-rewrite` remove and set (same-name remove drops the set). It
overwrites the three names it emits. The inbound value does not
survive. That is not membership in the remove list; it is the
outcome the subset exists to produce, under a platform constraint.

**Exemption: custody's execute path.** Custody calls the backend
directly. Nothing is stripped there. That is safe because custody
**builds** the request from persisted context (C6: method, target,
query, body) rather than forwarding a request the agent composed.
There is no agent-supplied header on that hop to forge. The
invariant protects the forwarding path; construction protects the
other.

Names emitted only on the custody path (`x-ctier-correlation-id`)
are still forgeable on the forwarding path if a closed strip omits
them. An enforcement point that strips by exact name MUST include
every `x-ctier-*` name a backend might treat as provenance, including
names it does not itself set. Prefix strip already does. A closed
strip that omits `x-ctier-correlation-id` forwards an agent-supplied
join key.

The two conditions an adopter still owes — the backend reachable only
from the enforcement point and from custody, and custody treated as a
trusted caller by some means ctier does not provide — are in
[`README.md`](README.md#what-ctier-requires-of-the-deployment).
