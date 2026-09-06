# Proposed OAI Namespace Registry entry

Prepared for a pull request to `spec.openapis.org` following that repository's
`CONTRIBUTING.md`. **Not opened upstream.** Registry entry records that a
prefix is in use. It is not a proposal to include ctier in the OpenAPI
Specification proper.

Checked 6 September 2026: the Namespace Registry listed `fdx`, `jsonschema`,
`ms`, `oai`, `oas-draft`, `oas`, `sap`, `scalar`. `ctier` was free.

---

## Proposed table row

| Value | Prefix | Description | Documentation |
|---|---|---|---|
| `ctier` | `x-ctier-` | Consequence-tiered API governance for autonomous agents | https://github.com/open-gw/ctier/blob/main/spec/extensions.md |

## Proposed registry page text

# Extension Field Namespace Registry

## ctier — Consequence-tiered API governance for autonomous agents

The `x-ctier-` prefix is reserved for extensions defined by the ctier
specification. ctier classifies API operations by the consequence of incorrect
execution and declares that classification in an OpenAPI document.

The official list of keys is in the ctier specification:

https://github.com/open-gw/ctier/blob/main/spec/extensions.md

Archived 1.0.0 (DOI 10.5281/zenodo.22020288) used `x-consequence-*`. 1.1.0
renames to this namespace. The archive is unaltered.

---

## Also not in this request

- Registration of individual keys in the Extensions Registry (`x-ctier-tier`,
  `x-ctier-criteria`, …). Do those later, once the 1.1.0 set is stable.
- Any change to the OpenAPI Specification itself.
