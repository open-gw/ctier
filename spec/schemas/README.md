# Document schemas

JSON Schema 2020-12. Each instance carries `ctierSchema` of the form
`<name>/<major.minor.patch>`. Each file's `$id` includes that version.

Pre-1.0 these formats may break. Consumers MUST ignore fields they do not
recognise.

| Schema | Document |
|---|---|
| `ctier-extensions-0.1.0.json` | Reusable `x-ctier-*` subschemas |
| `classification-rules-0.1.0.json` | Recommender rule document |
| `composition-manifest-0.1.0.json` | Composition inputs and C13 digest |
| `recommendation-set-0.1.0.json` | Overlay 1.1.0 plus per-action provenance |
| `coverage-report-0.1.0.json` | Declared / recommended / absent |

The qualified spec is OpenAPI plus the extensions in `../extensions.md`. It has
no schema of its own.

**Deferred:** the conformance corpus format (task 08) and the ledger record
formats (task 11). ADR-009 §7 warns against specifying formats before the
implementation has taught us what they hold.
