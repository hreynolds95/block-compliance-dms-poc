# Section 5 — Exception Handling Specification
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.6.3 (Exception Handling),
> Section 2.4 (Two-Role Document Model — Document Owner approves exceptions)

---

## 5.1 Overview

Exceptions are formal acknowledgements that a specific requirement in a governed document cannot currently be met, together with documented mitigating controls and a defined expiry. They are not waivers — they require Owner approval, written justification, and active monitoring.

All approved exceptions are logged in `exceptions/exceptions.jsonl` (append-only, tamper-evident via commit history). Exception requests and renewals are managed entirely through GitHub Issues.

---

## 5.2 Exception Request Fields

All fields are collected via the GitHub Issue form (`.github/ISSUE_TEMPLATE/exception-request.yml`).

| Field | Required | Description | PoP requirement |
|---|---|---|---|
| `doc_id` | Yes | Document the exception applies to | Identifies the governed requirement |
| Justification | Yes | Written explanation of why the requirement cannot be met | PoP 2.6.3 |
| Mitigating Controls | Yes | Controls in place to reduce risk during the exception period | PoP 2.6.3 |
| Scope | Yes | Which entities, products, or business lines are affected | PoP 2.6.3 |
| Effective Date | Yes | Date the exception takes effect (YYYY-MM-DD) | PoP 2.6.3 |
| Expiry Date | Yes | Date the exception expires (YYYY-MM-DD); maximum 12 months | PoP 2.6.3 |
| Evidence | No | Link to supporting documentation, risk assessment, or control testing | PoP 2.6.3 |

---

## 5.3 Exception Lifecycle State Machine

```
Any authorized user opens an Issue
using the exception-request template
         │
         │  exception-request label auto-applied by template
         ▼
┌────────────────────────────────┐
│  PENDING                       │
│  label: exception-pending      │
│  exception-triage.yml fires:   │
│  - Parses fields from body     │
│  - Looks up doc in registry    │
│  - Assigns to @hreynolds95     │
│  - Posts triage comment        │
└──────────┬───────────────┬─────┘
           │               │
    Owner applies    Owner applies
    exception-      exception-
    approved         rejected
           │               │
           ▼               ▼
┌──────────────┐   ┌───────────────┐
│  APPROVED    │   │  REJECTED     │
│              │   │               │
│ lifecycle.yml│   │ lifecycle.yml │
│ fires:       │   │ fires:        │
│ - Appends to │   │ - Posts       │
│   jsonl      │   │   rejection   │
│ - Commits    │   │   comment     │
│ - Comments   │   │ - Closes      │
│ - Closes     │   │   (not        │
│   (completed)│   │    planned)   │
└──────┬───────┘   └───────────────┘
       │
       │  Daily cron (exception-expiry.yml)
       │  monitors expiry_date
       ▼
┌────────────────────────────────┐
│  RENEWAL REQUIRED              │
│  label: exception-renewal      │
│  30 days before expiry:        │
│  - New Issue opened            │
│  - Instructions to resubmit    │
│    or confirm remediation      │
└──────────────┬─────────────────┘
               │
               │  If not renewed
               ▼
┌────────────────────────────────┐
│  EXPIRED                       │
│  label: exception-expired      │
│  On expiry date:               │
│  - New Issue opened            │
│  - Urgent remediation notice   │
└────────────────────────────────┘
```

---

## 5.4 GitHub Issue Form

`.github/ISSUE_TEMPLATE/exception-request.yml`

- Auto-applies the `exception-request` label on submission
- Structured fields produce a machine-parseable body that `exception-triage.yml` reads
- Title prefix: `[Exception] ` — helps filter exception Issues in the issue list

---

## 5.5 Triage Workflow (`exception-triage.yml`)

**Trigger:** `issues: [opened]` where issue has `exception-request` label

**What it does:**

1. Parses `doc_id` and all other fields from the Issue body using `### Field Name` header extraction
2. Looks up the document in `document-registry.yaml` to retrieve title, tier, and Document Owner
3. Applies `exception-pending` label
4. Assigns Issue to `@hreynolds95` (Phase 1; Phase 2 uses `document_owner_github` from registry)
5. Posts a structured triage comment with:
   - Document metadata table (title, tier, owner)
   - Quoted justification and mitigating controls
   - Approval/rejection instructions

**Label creation:** Creates all 6 exception labels on first run if they don't exist.

---

## 5.6 Lifecycle Workflow (`exception-lifecycle.yml`)

**Trigger:** `issues: [labeled]` where issue has `exception-request` label AND the label just applied is `exception-approved` or `exception-rejected`

### Approval path (`exception-approved`)

1. Parses all fields from the Issue body
2. Checks `exceptions/exceptions.jsonl` for an existing entry with the same `issue_number` (deduplication — prevents double-commit if label is removed and re-added)
3. Generates a sequential `exception_id` (`EX-001`, `EX-002`, …) by counting existing entries
4. Builds and appends a JSON entry to `exceptions/exceptions.jsonl`
5. Commits with message: `feat(exceptions): log approved exception {exception_id} — {doc_id} (issue #{n})`
6. Posts confirmation comment with the logged entry fields and renewal notice
7. Closes the Issue as `completed`

### Rejection path (`exception-rejected`)

1. Posts a rejection comment instructing the requester to open a new request if needed
2. Closes the Issue as `not planned`

---

## 5.7 `exceptions/exceptions.jsonl` Schema

Append-only. One JSON object per line. No deletions or edits — superseded entries are addressed via new exceptions (renewals).

```json
{
  "exception_id":        "EX-001",
  "doc_id":              "FC-017",
  "title":               "SEL Financial Crimes Program",
  "status":              "active",
  "scope":               "Cash App US only",
  "justification":       "The standard requires X but we are doing Y because Z...",
  "mitigating_controls": "1. Control A\n2. Control B",
  "effective_date":      "2026-04-28",
  "expiry_date":         "2026-10-28",
  "evidence":            "https://drive.google.com/...",
  "approved_by":         "hreynolds95",
  "issue_number":        12,
  "issue_url":           "https://github.com/hreynolds95/block-compliance-dms-poc/issues/12",
  "timestamp":           "2026-04-28T14:00:00+00:00"
}
```

| Field | Type | Description |
|---|---|---|
| `exception_id` | string | Sequential identifier (`EX-NNN`); generated at approval time |
| `doc_id` | string | Document the exception applies to; must match registry |
| `title` | string | Document title at time of approval (denormalized for readability) |
| `status` | string | Always `active` at write time; no status updates in file (append-only) |
| `scope` | string | Entities/products/lines affected |
| `justification` | string | Full justification text from Issue form |
| `mitigating_controls` | string | Full mitigating controls text from Issue form |
| `effective_date` | string | ISO 8601 date |
| `expiry_date` | string | ISO 8601 date; `exception-expiry.yml` monitors this field |
| `evidence` | string | URL or empty string |
| `approved_by` | string | GitHub handle of the approver |
| `issue_number` | integer | GitHub Issue number; used for deduplication |
| `issue_url` | string | Full URL to the originating Issue |
| `timestamp` | string | ISO 8601 UTC timestamp of approval |

---

## 5.8 Expiry Alerting (`exception-expiry.yml`)

**Trigger:** Daily cron at 10:00 ET (offset from `review-alerts.yml` at 09:00 ET)

**What it does:**

1. Reads all entries from `exceptions/exceptions.jsonl`
2. Filters to `status: active` entries
3. For each entry, calculates `days_until = expiry_date − today`

| Condition | Label | Issue title prefix |
|---|---|---|
| `days_until == 30` | `exception-renewal` | `[Renewal Required]` |
| `days_until <= 0` | `exception-expired` | `[Expired]` |

4. Deduplicates: skips if an open Issue with the matching label and `exception_id` in the title already exists
5. Opens Issue assigned to `@hreynolds95` with:
   - Exception metadata table
   - Two options: renew (open new request) or close (confirm remediation)
   - Link to the original approval Issue

> **Note:** `status: active` in `exceptions.jsonl` is a write-time snapshot; it is never updated in the file (append-only). The expiry cron uses the `expiry_date` field directly — if a renewal is approved, a new entry with a new `exception_id` and later `expiry_date` is appended, and the cron will track the new entry going forward.

---

## 5.9 Exception Labels

All labels are created automatically by `exception-triage.yml` on first run.

| Label | Color | Description |
|---|---|---|
| `exception-request` | `#bfd4f2` | Applied by Issue template on submission |
| `exception-pending` | `#e4a10c` | Applied by triage workflow; awaiting Owner review |
| `exception-approved` | `#0e8a16` | Applied by Owner to approve; triggers lifecycle workflow |
| `exception-rejected` | `#b60205` | Applied by Owner to reject; triggers lifecycle workflow |
| `exception-renewal` | `#1d76db` | Applied by expiry cron at 30-day threshold |
| `exception-expired` | `#b60205` | Applied by expiry cron when past expiry date |

---

## 5.10 Phase 2 Enhancements

| Enhancement | PoP Section | Description |
|---|---|---|
| Auto-assign to Document Owner | 2.6.3 | Replace `POC_ASSIGNEE = 'hreynolds95'` in triage and expiry workflows with `document_owner_github` from registry |
| Slack notification on approval | 2.6.3 | `exception-lifecycle.yml` posts to `#compliance-policy-help` when an exception is approved |
| Slack notification on expiry | 2.6.3 | `exception-expiry.yml` DMs Document Owner and posts to channel on expired exceptions |
| Exception register on portal | 2.6.3 | GitHub Pages portal renders active exceptions by doc, filtered by domain/tier |
| Status update on renewal | — | On renewal approval, patch the original entry's `status` field to `superseded` and add `superseded_by: EX-NNN`; requires moving from append-only to an update model or a separate status-patch file |
