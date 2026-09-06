# Examples

Valid instances pass the corresponding JSON Schema. Invalid instances are
**schema-valid** and **validator-invalid**: each violates one semantic rule in
`../validation.md`. They are the corpus a validator in any language is written
against.

| File | Schema | Expected |
|---|---|---|
| `classification-rules.valid.yaml` | `classification-rules/0.1.0` | schema pass |
| `composition-manifest.valid.json` | `composition-manifest/0.1.0` | schema pass |
| `recommendation-set.valid.yaml` | `recommendation-set/0.1.0` | schema pass |
| `coverage-report.valid.json` | `coverage-report/0.1.0` | schema pass |
| `invalid/single-target.overlay.yaml` | Overlay + extensions | violates **single-target** against `consequence-tiered-api.yaml` |
| `invalid/unidirectionality.operation.yaml` | extensions | violates **unidirectionality** |
| `invalid/bounds-coherence.operation.yaml` | extensions | violates **bounds coherence** |
| `invalid/exclusion-exclusivity.operation.yaml` | extensions | violates **exclusion exclusivity** |
