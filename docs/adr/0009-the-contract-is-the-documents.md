## ADR-009 — The contract is the documents, not the code

**Status:** proposed · **Date:** 5 September 2026
**Depends on:** ADR-002 (compiled targets), ADR-004 (library-first, distribution
surfaces), ADR-008 (recommender)

---

## 1. Why this is the fundamental one

ADR-004 says ctier is "a library with a stable programmatic API" so that a
Spectral ruleset or a Backstage plugin can embed it later. That sentence does not
survive contact with the facts: **Spectral is JavaScript, Backstage is
TypeScript, the Kong and APISIX adapters are Lua.** None of them can call a
Python function. A stable Python API buys nothing for the thing it was written to
buy.

But this is not an isolated slip. Four commitments already made, independently,
all require the same missing thing:

| Commitment | What it actually requires |
|---|---|
| **C13** — assignment binds to the digest of the composition | A digest is taken over *bytes*. There must be a canonical document to digest |
| **ADR-002** — the reference classifier is the conformance oracle across heterogeneous targets | Comparing a Python decision with an Apigee policy's decision can only happen over *data*. There is no shared call stack |
| **C1–C13** — "the requirements a second implementation would have to meet" | A second implementation can only be conformant against a specification of *documents*, not of functions |
| **ADR-004** — distribution into other people's tooling | The interop surface is files in a pipeline, not an importable package |

Four separate lines of reasoning arriving at one conclusion is the signal that
this is load-bearing rather than a preference.

**Decision.** ctier's stable public surface is a set of **versioned document
formats and the transformations between them**. The Python package is one
implementation of those transformations. Its function signatures are not the
contract and carry no stability promise.

---

## 2. What is and is not a document

The failure mode of this decision is over-formalising — every internal step
becoming a serialised artefact, the implementation turning slow and verbose, and
the schemas ossifying before anyone knows what they need to hold.

The line that prevents it:

> A step produces a document **only when it crosses a boundary**: between tools,
> between languages, between build time and run time, or between the system and a
> human who must read or edit it. Everything else stays an in-process structure
> with no schema and no stability promise.

Applying that test honestly gives six documents, not sixteen. `compute_tier`'s
internals, the parsed operation model, the accumulator's state — none of these
cross a boundary, so none of them are documents.

---

## 3. The catalogue

| Document | Boundary it crosses | Standard |
|---|---|---|
| **Interface description** | the world → ctier | OpenAPI 3.x, unchanged |
| **Governance overlay** | governance team → build | Overlay 1.1.0 + our extensions |
| **Classification rules** | ctier → org, and Python → JS | new |
| **Qualified spec** | compose → generate, and CI → audit | OpenAPI + `x-ctier-*` |
| **Composition manifest** | build → run time (via the digest) | new |
| **Recommendation set** | recommender → human → compose | Overlay 1.1.0 + our extensions |
| **Coverage report** | build → CI gate → dashboards | new |
| **Conformance corpus** | reference implementation → every target | new |
| **Ledger records** | run time → audit | new |

Nine, of which three are existing standards we extend rather than invent.

### The transformations, as document-in / document-out

```
compose    (description, overlay*, rules?)  → qualified spec, composition manifest
recommend  (description, rules)             → recommendation set  [an Overlay]
cover      (qualified spec)                 → coverage report
generate   (qualified spec, target, mode)   → gateway artefacts, conformance corpus
decide     (qualified spec, request context)→ decision record
```

Every one is pure and total: same inputs, same outputs, no clock, no network, no
hidden state. `decide` is the oracle — it is the only transformation a compiled
gateway target also implements, which is exactly why it must be expressible as
documents.

---

## 4. Three consequences worth taking deliberately

### 4.1 Rules become data, not code

The classification rules — *a `GET /{id}` returning one object is Tier 1, a
`DELETE` is Tier 3 at minimum* — become a **rule document**, not a Python module.

This is the same move as ADR-002 made for the tier itself, for the same reason:
a Spectral ruleset and a Python composer interpreting one rule file cannot
diverge, whereas two implementations of the same rules in two languages
inevitably will. It also makes an organisation's own rules a first-class artefact
they own and review, rather than a fork of our source.

### 4.2 Recommendations are an overlay

The recommender emits an **Overlay document**. A human edits it to accept or
override. It composes like any other overlay. No second concept, no new state to
plumb through the composer, and "recommended versus declared" becomes a property
carried in the overlay rather than a parallel mechanism.

For the inline-first team — the default path under ADR-001 — `ctier recommend
--apply` writes the accepted recommendations into the description instead. One
implementation, both paths.

### 4.3 The schemas live in the specification repository, not the engine

This one is structural and easy to get wrong. The document formats are the
**normative** artefact; the engine is an implementation of them. So the JSON
Schemas belong in `open-gw/ctier` under CC BY 4.0, and `ctier-engine` depends on
them.

That split is what makes a second implementation possible at all, and it is what
lets the conformance corpus be published alongside the specification rather than
buried in one implementation's test directory.

---

## 5. Versioning and stability

Every ctier-authored document carries a schema version. Semantic versioning, with
one rule that does the real work:

> **Consumers MUST ignore fields they do not recognise.**

That is the same invariant as C11's requirement that the enforcement point ignore
unknown `x-ctier-*` headers, and it is what makes additive evolution safe. Adding
a field is a minor version; removing one, or changing what an existing field
means, is major.

**Pre-1.0 the formats may break**, and that must be stated loudly rather than
implied — the alternative is committing to shapes before the implementation has
taught us what they need to hold. Freeze at v1.0.0 alongside the demo, not
before.

**What JSON Schema can and cannot do.** Schema validates *shape*. It cannot
express the single-target rule, C3's unidirectionality, or "an overlay action
setting criteria that compute below Tier 4 must select exactly one operation."
Those are semantic constraints enforced by a validator, and the specification must
say which constraints are schema-checked and which are validator-checked, or
implementers will assume the schema is the whole contract.

---

## 6. Worked example — the composition manifest

Concrete enough to build against, minimal enough not to over-commit:

```json
{
  "ctierSchema": "composition-manifest/0.1.0",
  "composedAt": "2026-09-05T20:14:02Z",
  "compositionDigest": "sha256:9f2b...c41e",
  "inputs": [
    { "role": "description", "path": "openapi.yaml",
      "digest": "sha256:1a7d...88b0" },
    { "role": "overlay", "path": "governance/pharmacy.overlay.yaml",
      "digest": "sha256:44c1...02fa", "order": 0 }
  ],
  "rules": { "path": "ctier-rules.yaml", "digest": "sha256:7e01...9d33" },
  "operations": 214,
  "coverage": { "declared": 186, "recommended": 22, "absent": 6 },
  "generator": { "target": "apigee", "mode": "bundle", "version": "0.2.0" }
}
```

`compositionDigest` is over a **canonical JSON serialisation** of the composed
document — keys sorted, no insignificant whitespace — not over the source YAML.
Otherwise reformatting a file changes the digest and C13 degrades into noise. The
per-input digests are over raw bytes, because those answer a different question:
*which file, exactly, did this come from.*

Note what the manifest already carries: enough for the runtime to stamp every
decision with the composition it was governed by, and enough for an auditor to
reconstruct which artefact assigned which tier. That is C13 satisfied by a
document rather than by a convention.

---

## 7. What this costs

**Schemas before code in M1.** More upfront work, and it will feel slower for
about a week.

**Formats are harder to change than functions.** That is the point, and it is
also the risk. Mitigated by 0.x and by formalising only the six boundary-crossing
artefacts.

**A discipline that has to be held.** The temptation to make the next internal
structure "just a small document" will recur. The boundary test in §2 is the
answer, and it should be quoted in the contributing guide.

---

## 8. What it buys

**The conformance suite becomes language-neutral.** A corpus of *(input
documents, expected output documents)* pairs. Any implementation, in any
language, passes or fails against it. C1–C13 stop being prose that a reviewer
interprets and become a test run — which is what "the requirements a second
implementation would have to meet" was always claiming to be.

**Gateway-agnosticism becomes a test result.** Same corpus, run against the
reference `decide` and against each compiled target. Identity of decisions is the
claim, and now it is checkable.

**The distribution surfaces get cheap.** A Spectral ruleset generated from the
rule document; a Backstage plugin reading coverage reports; Postman reading a
qualified spec. None of them reimplement classification, and none of them need
Python.

**And the strongest new-matter claim gets a mechanism.** NM-002 asserts that the
reference implementation acts as conformance oracle across heterogeneous
enforcement points. Without a document contract that is an aspiration. With one it
is a described mechanism with a corpus, a canonicalisation and a comparison rule.

---

## 9. Consequences for the plan

Task 03 changes shape. It was "core library + composer"; it becomes:

| | |
|---|---|
| **03a** | The document contract: JSON Schemas for the six ctier-authored formats, in `open-gw/ctier`, versioned 0.1.0, with the schema-checked versus validator-checked split written down |
| **03b** | The composer in `ctier-engine`, implementing `compose` against those schemas, with golden-file tests — input documents in, expected documents out |
| **03c** | The rule document format and the default rule set, replacing what would have been a rules module |

`decide` is not re-implemented here — the existing `classifier.py` becomes its
reference implementation, and task 08's conformance harness is where it is
pinned to the corpus.

**One thing to check before 03a.** Every `x-ctier-*` extension we publish should
be registered in the OpenAPI Specification Extension Registry under the `ctier`
namespace, which is still free. Cheaper to do while there are six extensions than
after there are twenty.
