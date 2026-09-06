# ctier specification 1.1.1

The stable public surface is a set of versioned document formats and the
transformations between them (`docs/adr/0009-the-contract-is-the-documents.md`).

**Naming.** Specification 1.0.0 used `x-consequence-*`. 1.1.0 renames to the
`ctier` namespace. The 1.0.0 archive
([doi.org/10.5281/zenodo.22020288](https://doi.org/10.5281/zenodo.22020288)) is
unaltered and remains resolvable.

| Path | What it is |
|---|---|
| [`extensions.md`](extensions.md) | Flat `x-ctier-*` keys: meaning, location, type, required |
| [`validation.md`](validation.md) | Schema-checked shape versus validator-checked semantics |
| [`schemas/`](schemas/) | JSON Schema 2020-12 for the ctier-authored formats, 0.1.0 |
| [`examples/`](examples/) | Valid instances, and invalid instances one per semantic rule |
| [`consequence-tiered-api.yaml`](consequence-tiered-api.yaml) | Worked OpenAPI 3.1 example |

The qualified spec is an OpenAPI document carrying the Part 2 extensions. It
has no schema of its own; it has a validation profile in `validation.md`.

**Deferred**, because the implementation has not yet taught us what they hold:
the conformance corpus format (task 08) and the ledger record formats (task 11).

Pre-1.0 the ctier-authored formats may break. Freeze at v1.0.0 alongside the
demo, not before. Consumers MUST ignore fields they do not recognise.

---

## Response contracts

These are interface behaviour, not extension keys.

**Tier 3 — `202`, `PendingAuthorization`.** A success outcome, not a failure.
Carries `correlationId`, `expiresAt`, `retryable: false`, and a machine-readable
`agentAction` mirrored in an `X-Agent-Action` header. `Retry-After` is
deliberately absent. `202` sits outside the classes client resilience
implementations retry by default.

**Tier 4 — `423`, `AgentSuspended`.** Carries `incidentId`, `autoResume: false`,
and `agentAction: halt_and_hand_off`. No correlation identifier. The agent must
not seek an alternative route to the same outcome.

**Excluded — `403`.** Refused at credential validation, before consequence
evaluation. If this is reached by a live token, a scope has been over-granted.
