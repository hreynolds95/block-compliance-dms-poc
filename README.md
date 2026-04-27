# Block Compliance DMS — PoC

A GitHub-native document change management and approval system for Block compliance policies. This proof-of-concept demonstrates branch-per-review workflows, tiered approval gates, and a full audit trail — all without a backend.

**Repo:** `hreynolds95/block-compliance-dms-poc`

---

## What this proves out

- Every document review happens on its own branch with a pull request
- Approval requirements are automatically determined by document tier
- Merging to `main` is blocked until the correct approval label is applied
- Every merge triggers an immutable audit log entry
- The entire change history is in git — no external system needed

---

## Repository structure

```
docs/
  tier-1/          # Board-approved (GOV-001, EE-001)
  tier-2/          # Committee-approved (GOV-011, FC-017)
  tier-3/          # Owner-approved (GOV-025, FC-032)
audit/
  audit-log.jsonl  # Append-only publish log
document-registry.yaml   # Central metadata index
.github/
  workflows/
    annual-review.yml    # Trigger a review cycle (manual dispatch)
    validate.yml         # Frontmatter lint on every PR
    route-approval.yml   # Detect tier, post checklist, apply label
    approval-gate.yml    # Required status check — blocks merge until approved
    publish.yml          # On merge to main: update frontmatter + write audit log
  pull_request_template.md
CODEOWNERS
```

---

## Tiered approval model

| Tier | Approval type | Required approver | Documents |
|------|---------------|-------------------|-----------|
| 1 | Board-approved | Board of Directors / CCO | GOV-001, EE-001 |
| 2 | Committee-approved | Compliance Committee | GOV-011, FC-017 |
| 3 | Owner-approved | Document Owner | GOV-025, FC-032 |

The highest tier of any document changed in a PR determines the approval requirement for that PR.

---

## End-to-end workflow

```
1. Initiate review  →  annual-review.yml creates branch + opens PR
2. Validate         →  validate.yml checks frontmatter on every push to the PR
3. Route approval   →  route-approval.yml detects tier, applies tier-N-review label,
                        posts approval checklist comment
4. Approve          →  Reviewer applies approved-tN label to the PR
5. Gate check       →  approval-gate.yml re-runs, passes when approved-tN label is present
6. Merge            →  Merge button unlocks; publish.yml fires on push to main
7. Publish          →  publish.yml updates effective_date + last_reviewed_date in frontmatter,
                        appends entry to audit/audit-log.jsonl
```

---

## How to initiate an annual review

1. Go to **Actions → Initiate Annual Review → Run workflow**
2. Fill in the inputs:
   - `doc_id` — e.g. `FC-017`
   - `change_type` — `immaterial` or `material`
   - `notes` — optional description (appears in PR body)
3. The workflow creates branch `review/{doc-id}-{year}`, updates frontmatter, and opens a PR

---

## How to approve (Phase 1 — solo simulation)

Branch protection requires the `approval-gate` status check to pass. The gate passes when the correct `approved-tN` label is present on the PR.

To simulate sign-off without a second GitHub account:

1. Complete all tollgate checklist items in the PR
2. Go to the PR → **Labels** → add `approved-t1`, `approved-t2`, or `approved-t3` (matching the document tier)
3. The `approval-gate` check re-runs automatically and passes
4. Merge is now unblocked

**Labels:**

| Label | Tier | Meaning |
|-------|------|---------|
| `tier-1-review` | 1 | PR contains Tier 1 document changes |
| `tier-2-review` | 2 | PR contains Tier 2 document changes |
| `tier-3-review` | 3 | PR contains Tier 3 document changes |
| `approved-t1` | 1 | Board / CCO sign-off simulated |
| `approved-t2` | 2 | Committee sign-off simulated |
| `approved-t3` | 3 | Document Owner sign-off simulated |

---

## Document frontmatter schema

Every document in `docs/` uses YAML frontmatter. Required fields are enforced by `validate.yml` on every PR.

```yaml
---
doc_id: FC-017                          # Required. Domain prefix + sequential number
title: "SEL Financial Crimes Program"   # Required
version: 1.0.0                          # Required
status: published                       # Required: draft | in-review | published | retired
tier: 2                                 # Required: 1 | 2 | 3
domain: financial-crimes                # Required
owner: "Ryan Goldstone"                 # Required
approval_type: committee                # Required: board (T1) | committee (T2) | owner (T3)
doc_type: "Standard"                    # Required: Policy | Standard | Procedure
next_review_date: "2026-05-31"          # Required
retention_years: 5                      # Required
effective_date: ""                      # Set automatically by publish.yml on merge
last_reviewed_date: ""                  # Set automatically by publish.yml on merge
change_type: null                       # Set by annual-review.yml: immaterial | material
review_cycle: null                      # Set by annual-review.yml: annual | triggered
legal_entity: "Squareup Europe Ltd."    # Optional
business: "Square"                      # Optional
published_pdf: "https://..."            # Optional — link to published PDF in Drive
logicgate_record_id: "ha9ZQN82"        # Optional — LogicGate PWF record ID
---
```

`approval_type` must be consistent with `tier`: board (T1), committee (T2), owner (T3). `validate.yml` enforces this and rejects mismatches.

---

## document-registry.yaml

Central metadata index at the repo root. Mirrors key frontmatter fields for quick lookup by humans and workflows. Update this file whenever a document is added, removed, or its tier/owner changes.

---

## Audit log

Every merge to `main` that touches a doc appends a JSON line to `audit/audit-log.jsonl`:

```json
{
  "timestamp": "2026-04-27T18:00:00+00:00",
  "sha": "abc123...",
  "doc_id": "FC-017",
  "title": "SEL Financial Crimes Program",
  "tier": 2,
  "action": "published",
  "change_type": "immaterial",
  "review_cycle": "annual",
  "effective_date": "2026-04-27",
  "actor": "hreynolds95"
}
```

The file is append-only and committed by `github-actions[bot]` so it cannot be altered without a visible git history entry.

---

## Phase 2 upgrade path

When teammates join, three changes convert this from solo simulation to real enforcement:

1. **Update `CODEOWNERS`** — replace `@hreynolds95` with real teammate handles per tier path
2. **Enable required PR reviews** in branch protection (1 required reviewer) — CODEOWNERS enforcement kicks in automatically
3. **Remove label-gate dependency** — the `approved-tN` label step becomes optional; CODEOWNERS review becomes the gate

The workflows themselves require no changes.

---

## Documents in scope

| Doc ID | Title | Tier | Owner | Next Review |
|--------|-------|------|-------|-------------|
| GOV-001 | Block, Inc. Compliance Management System (CMS) Policy | 1 | Tyler Hand | 2026-10-31 |
| EE-001 | Block, Inc. Code of Business Conduct & Ethics | 1 | Matt Latham | 2026-10-31 |
| GOV-011 | Block, Inc. Compliance Policy on Policies | 2 | Stevi Winer | 2026-06-06 |
| FC-017 | SEL Financial Crimes Program | 2 | Ryan Goldstone | 2026-05-31 |
| GOV-025 | Afterpay US Inc. Regulatory Change Management Standard | 3 | Corey Chamberlain | 2026-03-31 |
| FC-032 | Cash App US Know Your Business Program | 3 | Vanessa Simpson | 2026-03-31 |
