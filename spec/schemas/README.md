# Document schemas

JSON Schema 2020-12. Each instance carries `ctierSchema` of the form
`<name>/<major.minor.patch>`. Each file's `$id` includes that version.

These formats are **normative, not 1.0.0**. A format reaches 1.0.0 when an
implementation other than the reference one has been held to it. Until
then it defines the contract but has only ever been read by its author,
which is a weaker claim than stability.

Pre-1.0 these formats may break. Consumers MUST ignore fields they do not
recognise.

| Schema | Document |
|---|---|
| `ctier-extensions-0.1.0.json` | Reusable `x-ctier-*` subschemas, including `$defs/operation`, `$defs/criteria`, `$defs/criteriaProposal` |
| `classification-rules-0.1.0.json` | Recommender rule document |
| `composition-manifest-0.1.0.json` | Composition inputs and C13 digest |
| `recommendation-set-0.1.0.json` | Overlay 1.1.0 plus per-action provenance |
| `coverage-report-0.1.0.json` | Declared / recommended / absent |
| `conformance-corpus-0.3.0.json` | Classification-agreement cases; a case is a sequence |
| `decision-record-0.2.0.json` | An operation was classified and acted upon. Carries `deploymentDigest`. |
| `attempt-record-0.2.0.json` | The enforcement point refused without classifying. Carries `deploymentDigest`. |
| `outcome-record-0.1.0.json` | What happened when an approved Tier 3 executed. No digest; joined by `correlationId`. |

The qualified spec is OpenAPI plus the extensions in `../extensions.md`. It has
no schema of its own.

What the corpus establishes, and what has been run against it, is
[`../conformance.md`](../conformance.md).
