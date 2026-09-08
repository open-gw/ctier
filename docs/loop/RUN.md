# Loop run — tasks 14 through 17

Written 8 September 2026. Gates replace in-loop human review.

---

## 14 — trust-boundary headers

- **Date:** 8 September 2026
- **Repo:** `open-gw/ctier`
- **Outcome:** **HARD STOP** (Part 0). Spec remains 1.4.0.
- **Commits:** (this file and `docs/findings/14.md`; no `spec/headers.md`, no version bump)
- **pytest -q:** N/A — this repository has no test suite.
- **Live suite:** N/A — spec-only.
- **Acceptance:**
  - Inventory table: **yes**, both targets, three directions, plus custody execute (the hop that actually reaches the backend on approved Tier 3).
  - No emitted header carries authorisation material: **not satisfied as a pass** — stopped and reported. Custody emits `x-ctier-approver` (approver identity) and `x-ctier-agent` (unverified identity claim) to the backend.
  - `spec/headers.md`: **not written** (stop before normative text).
  - Provenance paragraph: **not written**.
  - Emit ⊆ strip: **would also have failed** — Apigee's closed strip omits `x-ctier-agent`, `x-ctier-approver`, `x-ctier-correlation-id`. Not papered over.
  - `spec/conformance.md` row: **not written**.
  - Spec 1.5.0: **no**.
  - Findings: `docs/findings/14.md`.
- **Reviewer:** Do not start 15–17 on the assumption that header names are settled. 16 depends on 14. The security finding is engine-side (`engine/ctier/custody/execute.py`); this spec task must not fix it.
- **NEW-MATTER:** none.

---

## Summary (first hard stop)

- **Completed:** none of 14–17.
- **Stopped:** 14, Part 0. Custody emits an approver identity (`x-ctier-approver`) and an unverifiable agent identity (`x-ctier-agent`) upstream. Not a naming fix.
- **Blocked:** none.
- **Not started:** 15 (agent-contract), 16 (fragment mode; depends on 14), 17 (APISIX; depends on 14).
- **Acceptance items not satisfied:** `spec/headers.md`; provenance-not-authorisation paragraph in the spec; emit ⊆ strip (Apigee closed set vs custody emit); `spec/conformance.md` row; spec 1.5.0.
- **Questions whose answer was stop and ask:** none beyond the stop condition the task already named. The corpus-dimension question (whether to add header cases) was not reached.
- **NEW-MATTER:** none.
- **Next:** an engine task (or a review decision) for the custody outbound headers — whether they are stripped from the Apigee closed set, renamed, dropped, or specified as attribution with the provenance rule. Until that is settled, 16 and 17 must not assume the header contract. 15 is independent of 14 by the loop table, but the loop also says do not start the next task until the previous gate passes; this gate did not pass.
