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
| `reversal.valid.yaml` | qualified spec | validator pass — Tier 2 named by another operation |
| `self-reversal.valid.yaml` | qualified spec | validator pass — self-reversal, idempotency `safe` |
| `recommended.valid.yaml` | `$defs/operation` | **schema pass** — `recommended` with `x-ctier-recommended-at` |
| `invalid/single-target.overlay.yaml` | Overlay + extensions | violates **single-target** against `consequence-tiered-api.yaml` |
| `invalid/unidirectionality.operation.yaml` | extensions | violates **unidirectionality** |
| `invalid/bounds-coherence.operation.yaml` | extensions | violates **bounds coherence** |
| `invalid/exclusion-exclusivity.operation.yaml` | extensions | violates **exclusion exclusivity** |
| `invalid/criteria-incomplete.yaml` | `classification-rules/0.1.0` | **schema fail** — missing `compliance-trigger` |
| `invalid/tier2-unverified.yaml` | qualified spec | violates **Tier 2 verified compensating action** |
| `invalid/reverses-dangling.yaml` | qualified spec | violates **dangling x-ctier-reverses** |
| `invalid/self-reversal-unsafe.yaml` | qualified spec | violates **self-reversal idempotency guard** |
| `invalid/recommended-missing-date.operation.yaml` | `$defs/operation` | **schema fail** — `recommended` without `x-ctier-recommended-at` |
| `invalid/declared-with-recommended-at.operation.yaml` | `$defs/operation` | **schema fail** — `declared` carrying `x-ctier-recommended-at` |
