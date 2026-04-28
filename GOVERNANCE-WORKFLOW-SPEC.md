# Section 4 — Governance Workflow Specification
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.4 (Two-Role Document Model), Section 2.6 (Review Cycles),
> Section 2.7.3 (Policy Team QC Gate), Section 2.11.1 (Change Types + fn.9),
> Section 2.13 (Enterprise Document Approval Matrix)

---

## 4.1 Change Type Taxonomy

Three change types govern how a document moves through the approval workflow. All are declared in the document frontmatter before the PR is opened.

| Change Type | PoP Ref | `change_type` | `review_cycle` | Approval required |
|---|---|---|---|---|
| **Scheduled Review** | 2.6 | `immaterial` or `material` | `annual` / `triennial` / `scheduled` | Per-tier approval matrix (Section 4.5) |
| **Off-Cycle Material Change** | 2.11.1 | `material` | `off-cycle` | Per-tier approval matrix; owner cannot self-approve (PoP 2.11.1 fn.9) |
| **Immaterial Change** | 2.11.1 | `immaterial` | any or omitted | Owner attestation only; tier approval not required per PoP |

### Scheduled Review Cycles (PoP Section 2.6)

| Document type | Default cycle | `review_cycle` value |
|---|---|---|
| Tier 1, 2, 3 Policies / Standards / Frameworks | Annual | `annual` |
| Enterprise Procedures | Triennial (3 years) | `triennial` |

> **Phase 1:** Triennial cycle is declared in frontmatter but `review-alerts.yml` and `annual-review.yml` treat it identically to `annual`. Phase 2 adds a dedicated `triennial-review.yml` dispatch and separate alert cadence.

---

## 4.2 Document State Machine

Every governed document moves through four states. State transitions are enforced by GitHub Actions — no manual frontmatter edits to `status`, `effective_date`, or `last_reviewed_date` are permitted outside the automated publish step.

```
┌─────────────────────────────────────────────────────────┐
│  DRAFT                                                  │
│  branch: review/<doc_id>-<year>                         │
│  frontmatter: status: draft                             │
│  Actor: Document Submitter edits content                │
└──────────────────────────┬──────────────────────────────┘
                           │  PR opened to main
                           ▼
┌─────────────────────────────────────────────────────────┐
│  IN-REVIEW                                              │
│  PR open; all 4 status checks must pass                 │
│  frontmatter: status: in-review                         │
│  Actors: Policy Team QC + tier approvers                │
└──────────────────────────┬──────────────────────────────┘
                           │  All gates pass + merge
                           ▼
┌─────────────────────────────────────────────────────────┐
│  PUBLISHED                                              │
│  publish.yml fires on merge to main:                    │
│    - effective_date = today                             │
│    - last_reviewed_date = today                         │
│    - status → published                                 │
│    - audit/audit-log.jsonl entry appended               │
└──────────────────────────┬──────────────────────────────┘
                           │  Owner/Policy Team action
                           ▼
┌─────────────────────────────────────────────────────────┐
│  RETIRED                                                │
│  frontmatter: status: retired                           │
│  Document remains in repo at path; portal hides from    │
│  active library. Audit history preserved.               │
└─────────────────────────────────────────────────────────┘
```

---

## 4.3 Gate Sequence

Branch protection on `main` requires all four status checks to pass before merge is permitted. They run in this logical order on every PR touching `docs/**/*.md`.

| Order | Status check | Workflow file | Hard block? | What it enforces |
|---|---|---|---|---|
| 1 | `validate` | `validate.yml` | Yes | Frontmatter lint — all required fields present, valid values, tier/path consistency |
| 2 | `role-enforcement` | `role-enforce.yml` | Yes | PR author is a registered Document Submitter; blocks owner self-approval of off-cycle Material Changes (PoP 2.11.1 fn.9) |
| 3 | `policy-team-qc` | CODEOWNERS | Yes | Policy Team content QC (PoP 2.7.3) — mandatory for all changes, all tiers; cannot be bypassed |
| 4 | `approval-gate` | `approval-gate.yml` | Yes | Correct tier approval label present (`approved-t1` / `approved-t2` / `approved-t3`) |
| — | `publish` | `publish.yml` | Post-merge | Fires on push to `main`; updates frontmatter + appends audit log |

> **Phase 1:** `policy-team-qc` (Gate 3) is simulated by a single CODEOWNERS reviewer (`@hreynolds95`).
> **Phase 2:** Replace with `@squareup/compliance-policy-team` team slug.

---

## 4.4 Approval Label Matrix

The `approval-gate.yml` workflow reads the highest tier in a PR and requires the corresponding label before merge is allowed.

| Tier | Required label | Who applies it | PoP approver |
|---|---|---|---|
| Tier 1 | `approved-t1` | Board/CCO delegate | Block/US Subsidiary Board (PoP 2.13) |
| Tier 2 | `approved-t2` | Committee delegate | Compliance Committee (PoP 2.13) |
| Tier 3 | `approved-t3` | Document Owner | Document Owner + Leadership (PoP 2.13) |

> **Phase 1:** Labels are applied manually by `@hreynolds95` simulating all approver roles.
> **Phase 2:** CODEOWNERS requires a review from the designated approver GitHub Team before the label can be applied, enforcing the approval chain natively.

---

## 4.5 Scheduled Review Workflow

Triggered by the Document Owner or Policy Team via `workflow_dispatch` on `annual-review.yml`.

```
Trigger: Policy Team or Owner runs annual-review.yml dispatch
  Inputs: doc_id, change_type (immaterial|material), notes
         │
         ├─ Locates doc file by doc_id in docs/
         ├─ Creates branch: review/<doc_id>-<year>
         ├─ Stamps frontmatter:
         │     change_type: <input>
         │     review_cycle: annual
         │     review_status_note: "Annual review initiated <date>"
         ├─ Pushes branch
         └─ Opens PR with:
               - Document table (doc_id, title, tier, owner)
               - Tier-specific approval checklist
               - "Add approved-t{tier} label to unlock merge" instruction
                      │
                      └─ Gate sequence (Section 4.3) fires
                               │
                               └─ Merge → publish.yml
                                    - effective_date = today
                                    - last_reviewed_date = today
                                    - status: published
                                    - Audit log appended
```

**Automated triggering:** `review-alerts.yml` opens a GitHub Issue at 90, 60, and 30 days before `next_review_date`, notifying the Document Owner and linking to the dispatch trigger URL. See Section 4.8.

---

## 4.6 Off-Cycle Material Change Workflow

Initiated manually by the Document Submitter. The Document Owner **cannot** be the PR author for this change type (PoP 2.11.1 fn.9) — `role-enforce.yml` hard-blocks the PR if this condition is detected.

```
Document Submitter (≠ Document Owner for this change type):
         │
         ├─ Creates branch: offcycle/<doc_id>-<description>
         ├─ Makes content changes
         ├─ Stamps frontmatter:
         │     change_type: material
         │     review_cycle: off-cycle
         └─ Opens PR to main
                  │
                  ├─ validate.yml — frontmatter lint (Gate 1)
                  │
                  ├─ role-enforce.yml — Gate 2:
                  │     BLOCKS if PR author == document_owner_github
                  │     (off-cycle material self-approval prohibited)
                  │
                  ├─ policy-team-qc — CODEOWNERS Gate 3
                  │     Policy Team reviews content change
                  │
                  ├─ approval-gate.yml — Gate 4
                  │     approved-t{tier} label required
                  │
                  └─ Merge → publish.yml
```

**Key enforcement:** `role-enforce.yml` detects the combination of `change_type: material` + `review_cycle: off-cycle` + `PR author == document_owner_github` and fails with the message:
> "Owner self-approval of off-cycle Material Changes is prohibited (PoP Section 2.11.1, footnote 9)."

---

## 4.7 Immaterial Change Workflow

Minor corrections, formatting updates, or non-substantive clarifications. The Document Owner may be the PR author (self-approval is permitted for immaterial changes per PoP 2.11.1).

```
Document Submitter (may be Document Owner):
         │
         ├─ Creates branch: immaterial/<doc_id>-<description>
         ├─ Makes change
         ├─ Stamps frontmatter:
         │     change_type: immaterial
         └─ Opens PR to main
                  │
                  ├─ validate.yml — frontmatter lint (Gate 1)
                  │
                  ├─ role-enforce.yml — Gate 2:
                  │     Submitter check only; self-approval ALLOWED for
                  │     immaterial changes (PoP 2.11.1)
                  │
                  ├─ policy-team-qc — CODEOWNERS Gate 3
                  │     Policy Team QC applies to ALL changes (PoP 2.7.3)
                  │
                  ├─ approval-gate.yml — Gate 4
                  │     approved-t{tier} label required
                  │     (Phase 1: same label used for all change types)
                  │
                  └─ Merge → publish.yml
```

> **Phase 2 enhancement:** PoP distinguishes immaterial changes (owner attestation only) from material changes (full tier approval chain). Phase 2 should add `change_type: immaterial` handling in `approval-gate.yml` so immaterial changes require only an `attested-owner` label rather than the full tier approval label. See Section 4.10.

---

## 4.8 Review Date Alerting

`review-alerts.yml` is a daily cron workflow that reads `next_review_date` from `document-registry.yaml` for all published documents and opens GitHub Issues at defined thresholds before the review is due.

### Alert thresholds

| Days until due | Alert level | Issue label | Assigned to |
|---|---|---|---|
| 90 days | Early notice | `review-due-90d` | `document_owner_github` |
| 60 days | Reminder | `review-due-60d` | `document_owner_github` |
| 30 days | Urgent | `review-due-30d` | `document_owner_github` |
| 0 (overdue) | Escalation | `review-overdue` | `document_owner_github` |

### Issue content

Each alert Issue includes:
- Document title, doc_id, tier, `next_review_date`
- Direct link to trigger `annual-review.yml` dispatch
- Instructions: input `doc_id`, select `change_type`

**Deduplication:** Before opening a new Issue, the workflow queries for an existing open Issue with the same `doc_id` and label. If one already exists, it is skipped.

> **Phase 2:** Route Issue creation to Slack `#compliance-policy-help` via webhook in addition to GitHub Issue (PoP 2.9 awareness campaign integration). DM the Document Owner on the 30-day threshold.

---

## 4.9 New Document Request Workflow (Phase 2)

Not yet implemented. Defined here for design completeness.

```
Any Block employee:
         │
         ├─ Opens GitHub Issue using "new-document-request" template
         │     Fields: proposed title, doc_type, tier, scope, business entity,
         │              rationale, conflict check attestation, related docs
         │
         ├─ Goose-assisted semantic search runs against existing docs
         │     Appends: list of potentially overlapping documents
         │
         └─ Policy Team evaluates (PoP 2.7.3):
                  ├─ Conflict check: does doc duplicate / overlap existing?
                  ├─ Scope review: does doc fit governance model?
                  └─ Decision:
                       ├─ Approved → create doc_id in registry, scaffold branch
                       └─ Rejected → close Issue with reasoning
```

---

## 4.10 Phase 2 Enhancements

| Enhancement | PoP Section | Description |
|---|---|---|
| Immaterial change bypass | 2.11.1 | `approval-gate.yml` accepts `attested-owner` label (not full tier label) for `change_type: immaterial` PRs |
| New document request workflow | 2.7.3 | GitHub Issue form → conflict check → Policy Team QC → registry scaffolding |
| Review alert Slack routing | 2.9 | `review-alerts.yml` posts to `#compliance-policy-help`; DMs Document Owner at 30d threshold |
| Triennial cycle support | 2.6 | `review-alerts.yml` and `annual-review.yml` respect `review_cycle: triennial` on Enterprise Procedures |
| Enterprise Procedures dispatch | 2.13 | Dedicated `triennial-review.yml` dispatch for `docs/enterprise-procedures/` |
| Stakeholder notifications on publish | 2.9 | `publish.yml` reads `stakeholder_groups` from registry; dispatches Slack + email on merge |
| CODEOWNERS approver enforcement | 2.13 | Phase 2 GitHub Teams replace individual handles; tier approvers must review via CODEOWNERS, not just label |
