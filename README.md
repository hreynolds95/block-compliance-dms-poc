# Block Compliance DMS — PoC

A GitHub-native document change management and approval system for Block compliance policies.
This proof-of-concept implements DocArchitect Sections 3–11 — tiered approvals, two-role
enforcement, exception lifecycle, awareness campaign, orphaned document detection, and a full
audit trail — all without a backend.

**Repo:** `hreynolds95/block-compliance-dms-poc`
**Portal (policy library):** `https://hreynolds95.github.io/block-compliance-policies/`

---

## What this proves out

- Every document review happens on its own branch with a pull request
- Approval requirements are automatically determined by document tier (T1/T2/T3)
- Merging to `main` is blocked until the correct approval label is applied
- Two-role model enforcement: Document Submitter drafts; Document Owner approves
- Owner self-approval prohibition on off-cycle material changes (PoP 2.11.1 fn.9)
- Every merge triggers an immutable audit log entry
- Exception requests are structured, triaged, lifecycle-managed, and expiry-tracked
- Publication notifications are dispatched to stakeholder groups on merge
- Monthly portfolio digest opens a GitHub Issue on the 1st of each month
- Orphaned documents (no active owner) are detected weekly and flagged for reassignment
- Non-governed sub-procedural documents (DMs, Desktop Procedures, Work Instructions) are
  tracked in a separate registry without full governance automation
- The entire change history is in git — no external system needed

---

## Repository structure

```
docs/
  tier-1/                  # Board-approved documents (GOV-001, EE-001)
  tier-2/                  # Committee-approved documents (GOV-011, FC-017)
  tier-3/                  # Owner-approved documents (GOV-025, FC-032)
  _templates/              # Official document templates — EXEMPT from validate.yml gates
    policy-standard-template.md   # Markdown mirror of official Policy/Standard template
    procedure-template.md         # Markdown mirror of official Procedure template

audit/
  audit-log.jsonl          # Append-only publication log (CODEOWNERS-protected)

exceptions/
  exceptions.jsonl         # Append-only approved exception record (CODEOWNERS-protected)

document-registry.yaml     # Central metadata index for workflow lookups (owner, tier,
                           # stakeholder_groups, document_owner_github, …)
                           # Mirrors key frontmatter fields — CODEOWNERS-gated on all changes
non-governed-registry.yaml # Informational index of non-governed supporting documents
                           # (DMs, Desktop Procedures, Work Instructions, job aids, etc.)
                           # No workflow automation reads or writes this file

.github/
  workflows/
    validate.yml            # Frontmatter lint + section structure check (Gate 1)
    role-enforce.yml        # Two-role model + self-approval prohibition (Gate 2)
    route-approval.yml      # Detect tier, apply label, post approval checklist (Gate 3)
    approval-gate.yml       # Required status check — blocks merge until approved (Gate 4)
    annual-review.yml       # Trigger a review cycle (manual workflow_dispatch)
    publish.yml             # On merge: update frontmatter, write audit log, send notifications
    review-alerts.yml       # Daily cron — open Issues at 90/60/30d + overdue
    exception-triage.yml    # On exception-request Issue open: triage + assign
    exception-lifecycle.yml # On exception-approved/rejected label: write jsonl / close
    exception-expiry.yml    # Daily cron — open renewal/expired Issues at 30d + expiry
    monthly-digest.yml      # 1st of month cron — portfolio digest Issue + Slack
    orphan-detection.yml    # Weekly cron + workflow_dispatch — detect unowned docs
  ISSUE_TEMPLATE/
    exception-request.yml   # Structured exception request form
    new-document-request.yml # Self-service new document request form (PoP 2.7.3)
  pull_request_template.md

CODEOWNERS

# Architecture spec documents (DocArchitect Sections 1–14)
ARCHITECTURE-OVERVIEW.md      # Section 1
# Section 2 — Repository & Directory Taxonomy is covered by this README
ACCESS-CONTROL-MODEL.md       # Section 3
GOVERNANCE-WORKFLOW-SPEC.md   # Section 4
EXCEPTION-HANDLING-SPEC.md    # Section 5
TEMPLATE-ENFORCEMENT-SPEC.md  # Section 6
PORTAL-SPEC.md                # Section 7
AWARENESS-CAMPAIGN-SPEC.md    # Section 8
ORPHANED-DOCUMENT-SPEC.md     # Section 9
NON-GOVERNED-CONTROLS.md      # Section 10
CAPABILITY-GAP-ANALYSIS.md    # Section 11
LEGACY-ARCHIVE-SPEC.md        # Section 12
LEGACY-ARCHIVE.md             # Section 12 — pre-migration link registry
IMPLEMENTATION-ROADMAP.md     # Section 13
OPEN-DECISIONS.md             # Section 14
```

---

## Naming conventions

### `doc_id` format

Every In-Scope Document has a unique identifier: `{DOMAIN_PREFIX}-{NNN}`

| Domain prefix | Compliance domain |
|---|---|
| `GOV` | Governance |
| `FC` | Financial Crimes |
| `EE` | Ethics and Employee Conduct |
| `PRV` | Privacy and Data |
| `REG` | Regulatory Affairs |
| `OPR` | Operational Risk |
| `TPR` | Third-Party Risk |

Numbers are sequential within each domain prefix, assigned by the Policy Team at document
creation. Once assigned, a `doc_id` never changes — even if the document is retired.

**Examples:** `GOV-001`, `FC-017`, `EE-001`

### File naming convention

Document files follow the pattern `{doc_id}-{kebab-slug}.md`:

```
docs/tier-2/FC-017-sel-financial-crimes-program.md
docs/tier-1/GOV-001-cms-policy.md
docs/tier-3/GOV-025-regulatory-change-management.md
```

The slug is a lowercase, hyphen-separated abbreviation of the document title. It is
informational only — `doc_id` is the authoritative identifier used by all workflows
and the registry.

### Non-governed document IDs

Non-governed documents tracked in `non-governed-registry.yaml` use `NG-{NNN}` identifiers
(e.g. `NG-001`). These are assigned manually by the Policy Team and carry no tier prefix.

---

## DocArchitect implementation status

| Section | Title | Status |
|---|---|---|
| 3 | Access Control Model | Complete |
| 4 | Governance Workflow Specification | Complete |
| 5 | Exception Handling Specification | Complete |
| 6 | Template Enforcement Specification | Complete |
| 7 | GitHub Pages Portal Specification | Complete |
| 8 | Awareness Campaign & Communications | Complete |
| 9 | Orphaned Document Management | Complete |
| 10 | Non-Governed Document Controls | Complete |
| 11 | GitHub Capability Gap Analysis | Complete |
| 12 | Legacy Archive & Migration Strategy | Complete |
| 13 | Implementation Roadmap | Complete |
| 14 | Open Decisions & Trade-offs | Complete |
| 1 | Architecture Overview | Complete |
| 2 | Repository & Directory Taxonomy | Complete (this README) |

---

## Tiered approval model

| Tier | Approval type | Approval body | Documents |
|---|---|---|---|
| 1 | Board-approved | Block Board of Directors / CCO | GOV-001, EE-001 |
| 2 | Committee-approved | Compliance Committee | GOV-011, FC-017 |
| 3 | Owner-approved | Document Owner | GOV-025, FC-032 |

The highest tier of any document changed in a PR determines the approval requirement for that PR.

---

## End-to-end governance workflow

```
1. Initiate review     →  annual-review.yml creates branch + opens PR
                           (or Document Submitter opens a branch directly for off-cycle changes)

2. Validate (Gate 1)   →  validate.yml: frontmatter required fields + section structure
                           (4 required headings; docs/_templates/ exempt)

3. Role enforce (Gate 2) → role-enforce.yml:
                           Rule 1: only Document Submitter may open the PR
                           Rule 2: Document Owner may not self-approve off-cycle material changes

4. Route approval (Gate 3) → route-approval.yml: detects tier, applies tier-N-review label,
                              posts approval checklist comment

5. Policy Team QC      →  Policy Team reviews content; CODEOWNERS gate in Phase 2
                           Phase 1: manual review by @hreynolds95

6. Approve (Gate 4)    →  Reviewer applies approved-tN label to the PR
                           approval-gate.yml re-runs and passes

7. Merge               →  Merge button unlocks

8. Publish             →  publish.yml fires on push to main:
                           - Updates effective_date + last_reviewed_date in frontmatter
                           - Appends entry to audit/audit-log.jsonl
                           - Sends publication notifications to stakeholder_groups channels
```

---

## Exception handling workflow

```
1. Request    →  Any employee opens an Issue using the exception-request template
                 Label auto-applied: exception-request

2. Triage     →  exception-triage.yml fires:
                 - Parses structured Issue body
                 - Looks up Document Owner in document-registry.yaml
                 - Assigns Issue to @hreynolds95 (Phase 1)
                 - Applies exception-pending label
                 - Posts triage comment with doc metadata

3. Decision   →  Policy Team applies exception-approved or exception-rejected label

4. Lifecycle  →  exception-lifecycle.yml fires on label:
                 Approved → generates EX-NNN ID, appends to exceptions/exceptions.jsonl,
                            commits, closes Issue as completed
                 Rejected → posts comment with reasoning, closes Issue as not planned

5. Expiry     →  exception-expiry.yml daily cron:
                 30 days before expiry → opens renewal Issue (exception-renewal label)
                 On expiry day        → opens expired Issue (exception-expired label)
```

---

## Awareness campaign

**Publication notification** — fires as the final step in `publish.yml` on every merge to `main`:
- Reads `stakeholder_groups` from `document-registry.yaml` for each changed doc
- Deduplicates channels across multi-doc merges
- Phase 1: logs Slack payload to Actions run
- Phase 2: POSTs to `SLACK_WEBHOOK_URL` secret (no code changes needed to activate)

**Monthly portfolio digest** — `monthly-digest.yml` cron, 1st of each month:
- Sections: portfolio summary, published last 30 days, coming due in 30 days, overdue, open exceptions
- Always opens a GitHub Issue titled `[Monthly Digest] Compliance Policy Portfolio — {Month YYYY}`
- Deduplicates: skips if open digest issue for current month already exists
- Phase 1: Slack payload logged; Phase 2: posts to `#compliance-policy-help`

---

## Orphaned document detection

`orphan-detection.yml` runs every Monday at 09:00 ET. A document is orphaned when its
`document_owner_github` in the registry is null or empty (Phase 1) or when the owner handle
is no longer a member of the GitHub org (Phase 2).

**To simulate a departure for PoC demo:**
1. Go to **Actions → Orphaned Document Detection → Run workflow**
2. Enter a `doc_id` (e.g. `FC-017`)
3. Workflow opens a `[Orphaned Document]` Issue regardless of current owner state

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

Branch protection requires the `approval-gate` status check to pass. The gate passes when
the correct `approved-tN` label is present on the PR.

1. Complete all tollgate checklist items in the PR
2. Go to the PR → **Labels** → add `approved-t1`, `approved-t2`, or `approved-t3` (matching tier)
3. The `approval-gate` check re-runs automatically and passes
4. Merge is now unblocked

**Labels:**

| Label | Tier | Meaning |
|---|---|---|
| `tier-1-review` | 1 | PR contains Tier 1 document changes |
| `tier-2-review` | 2 | PR contains Tier 2 document changes |
| `tier-3-review` | 3 | PR contains Tier 3 document changes |
| `approved-t1` | 1 | Board / CCO sign-off simulated |
| `approved-t2` | 2 | Committee sign-off simulated |
| `approved-t3` | 3 | Document Owner sign-off simulated |
| `exception-request` | — | Structured exception request filed |
| `exception-pending` | — | Exception under review |
| `exception-approved` | — | Exception approved; record written to jsonl |
| `exception-rejected` | — | Exception rejected |
| `exception-renewal` | — | Exception expiring within 30 days |
| `exception-expired` | — | Exception past expiry date |
| `review-due-90d` | — | Document review due in 90 days |
| `review-due-60d` | — | Document review due in 60 days |
| `review-due-30d` | — | Document review due in 30 days |
| `review-overdue` | — | Document review is overdue |
| `orphaned-document` | — | Document has no active owner |
| `monthly-digest` | — | Monthly portfolio digest issue |
| `new-doc-request` | — | New document request filed |

---

## Document frontmatter schema

Every document in `docs/` uses YAML frontmatter. Required fields are enforced by `validate.yml`.

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

`approval_type` must be consistent with `tier`. `validate.yml` rejects mismatches.
Documents must include 4 required section headings (Overview, Compliance & Enforcement,
Questions & Contact Information, Document Details & Revisions). `docs/_templates/` is exempt.

---

## Audit log

Every merge to `main` that touches a doc appends a JSON line to `audit/audit-log.jsonl`:

```json
{
  "timestamp": "2026-04-27T18:00:00+00:00",
  "sha": "abc123...",
  "short_sha": "abc123ab",
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

The file is append-only and committed by `github-actions[bot]`.

---

## Phase 1 simulation constraints

- All GitHub assignments (Issues, PRs) go to `@hreynolds95` only — no other handles
- `document_owner_github` values in the registry are placeholder handles (e.g. `@tyler-hand-block`)
  and do not exist as real GitHub users
- `role-enforce.yml` Rule 2 (self-approval prohibition) will not fire because PR author
  (`hreynolds95`) never matches placeholder owner handles; temporarily set a doc's owner
  to `@hreynolds95` to test that gate
- `SLACK_WEBHOOK_URL` is not configured — notification payloads are logged to Actions runs

---

## Phase 2 upgrade path

| Change | Effect |
|---|---|
| Migrate repo to `squareup` org | Enables GitHub SSO/SAML, org Teams, org-level branch protection |
| Update `CODEOWNERS` with real team handles | Tier gates become hard native enforcement |
| Enable required PR reviews (1 reviewer) | Two-role model becomes technically enforced |
| Set `SLACK_WEBHOOK_URL` secret | Publication notifications and monthly digest activate with no code changes |
| Add `organization: [member_removed]` trigger to `orphan-detection.yml` | Real-time owner departure detection |
| Enable Phase 2 portal features | "Request Review" button, exception register tab, SSO login gate |
| Export audit log + exceptions jsonl to Snowflake | Immutable WORM record; closes audit trail gap identified in Section 11 |

---

## Documents in scope

| Doc ID | Title | Tier | Owner | Next Review |
|---|---|---|---|---|
| GOV-001 | Block, Inc. Compliance Management System (CMS) Policy | 1 | Tyler Hand | 2026-10-31 |
| EE-001 | Block, Inc. Code of Business Conduct & Ethics | 1 | Matt Latham | 2026-10-31 |
| GOV-011 | Block, Inc. Compliance Policy on Policies | 2 | Stevi Winer | 2026-06-06 |
| FC-017 | SEL Financial Crimes Program | 2 | Ryan Goldstone | 2026-05-31 |
| GOV-025 | Afterpay US Inc. Regulatory Change Management Standard | 3 | Corey Chamberlain | 2026-03-31 |
| FC-032 | Cash App US Know Your Business Program for Cash for Business Customers | 3 | Vanessa Simpson | 2026-03-31 |
