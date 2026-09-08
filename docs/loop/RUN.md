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

---

## Loop final report — tasks 14–17

Written 8 September 2026, after the first hard stop. 15 ran later and is BLOCKED; that is not a loop halt. 14's security finding is. No push. M6 demo is task 18, not 14.

- **Completed:** none of 14–17.
- **Stopped:** 14, Part 0. Custody approved-execute (`engine/ctier/custody/execute.py` `build_outbound`) emits `x-ctier-approver` (named approver) and `x-ctier-agent` (unverified identity claim) to the backend. Approver identity crossing the trust boundary. Spec stays 1.4.0. No `spec/headers.md`. Commit `707ff99` `docs: findings for 14 — custody emits an approver identity upstream`. Findings: `docs/findings/14.md`.
- **Blocked:** 15 (engine). `CTIER_AGENT_BASE_URL` and `CTIER_AGENT_KEY` absent. Experiment not run. Commit `82837fb` `docs: task 15 blocked pending agent credentials`. `engine/docs/loop/RUN.md` records the gate. No `engine/docs/findings/15.md` (correct for BLOCKED).
- **Not started:** 16 (fragment mode; waiting on 14). Engine tree clean of fragment work. 17 (APISIX) must not start — header names unsettled; 16 did not change generation.
- **Acceptance items not satisfied:**
  - **14:** `spec/headers.md`; provenance-not-authorisation paragraph; emit ⊆ strip (Apigee closed strip omits `x-ctier-agent`, `x-ctier-approver`, `x-ctier-correlation-id`; Kong prefix strip would catch them on ingress); `spec/conformance.md` row; spec 1.5.0. Stopped before those artefacts. Inventory table was written in findings.
  - **15:** experiment acceptance N/A — live `--agent` suite not invoked. Arms A/D not run. No retry / decomposition / stop distribution.
  - **16:** not started — fragment-mode acceptance never reached.
  - **17:** not started — APISIX acceptance never reached.
- **Questions whose answer was stop and ask:**
  - 14 Part 0: custody emits an approver identity upstream (the hard stop).
  - 14 Part 3: emit ⊈ strip would have been next (Apigee closed set vs custody names; Kong prefix disagrees). Reached in findings, then stopped.
  - 15: never asked (blocked on credentials).
  - 16: corpus dimension never reached.
  - 17: A1–A4 never reached.
- **NEW-MATTER:** none. `docs/findings/14.md` reports none (existing emit, not a new mechanism). 15 wrote no findings. `NEW-MATTER.md` was not created.
- **Next:** author review of the custody upstream identity headers (`x-ctier-approver`, `x-ctier-agent`, and the related `x-ctier-correlation-id`). Decision: allowed (then specify and strip) or must be removed. Then 14 can resume as a spec 1.5.0 headers task. 16 and 17 wait on that. 15 can run independently if credentials are provided. Do not push.
