# Section 9 — Orphaned Document Management Specification
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.4 (Two-Role Model — every document must have an accountable Owner),
> Section 2.7 (Document Ownership — Owner is responsible for review cycle and exception approval),
> Section 2.9 (Awareness Campaign — Policy Team notified on orphan detection)

---

## 9.1 Overview

An **orphaned document** is a published In-Scope Document whose Document Owner is no longer able to fulfill ownership responsibilities — typically because they have left Block or changed roles. Orphaned documents pose a direct compliance risk: review cycles are missed silently, exception requests have no approver, and the Policy Team has no visibility until a deadline is breached.

The orphaned document management system detects this condition automatically and opens a GitHub Issue prompting the Policy Team to assign a new owner before the next review cycle is missed.

| Component | Trigger | Workflow |
|---|---|---|
| **Orphan detection** | Weekly cron (Monday 09:00 ET) | `orphan-detection.yml` |
| **Manual simulation** | `workflow_dispatch` with `doc_id` input | `orphan-detection.yml` |
| **Phase 2 real-time detection** | `organization.member_removed` event | `orphan-detection.yml` (extended) |

---

## 9.2 Definition of Orphaned Documents

A document is considered orphaned when any of the following conditions are true:

| Condition | Phase 1 detectable | Phase 2 detectable |
|---|---|---|
| `document_owner_github` is null or empty in `document-registry.yaml` | Yes | Yes |
| `document_owner_github` handle is no longer a member of the GitHub org | No (personal account has no org API) | Yes |
| Document Owner's GitHub account has been deactivated | No | Yes (org membership event) |
| Manually flagged via `workflow_dispatch` `doc_id` input | Yes (test/simulation) | Yes |

**Phase 1 scope:** The workflow can only detect the first condition (null/empty handle) automatically. The `workflow_dispatch` input covers the simulation case — any document can be manually flagged to demonstrate the full orphan lifecycle without requiring a real departure event.

**Scope:** Only `status: published` documents are evaluated. Documents in `draft`, `in-review`, `intake`, or `retired` status are excluded — ownership gaps in non-published documents are a process concern, not an active compliance risk.

---

## 9.3 Detection Mechanism

### Phase 1 — Weekly cron + manual dispatch

```
orphan-detection.yml fires (Monday 09:00 ET OR workflow_dispatch with doc_id)
        │
        ├─ workflow_dispatch mode (doc_id provided)?
        │     Yes → flag that specific doc regardless of owner state
        │     No  → scan all published docs in document-registry.yaml
        │             for null / empty document_owner_github
        │
        ├─ For each orphaned doc:
        │     ├─ Dedup check: open Issue with label 'orphaned-document'
        │     │   AND doc_id in title? → skip
        │     │
        │     ├─ Open Issue: '[Orphaned Document] {doc_id} — {title}'
        │     │   label: orphaned-document
        │     │   assigned: @hreynolds95 (Phase 1)
        │     │
        │     └─ Log Slack payload (Phase 1) or POST to webhook (Phase 2)
        │
        └─ Summary: N orphaned doc(s) found / N issue(s) opened
```

### Phase 2 — Real-time org membership event

When a user is removed from the `squareup` org, GitHub fires an `organization` webhook event (`action: member_removed`). The workflow:

1. Receives the `member_removed` event
2. Extracts the departing user's GitHub handle
3. Scans `document-registry.yaml` for all docs with that handle as `document_owner_github`
4. Opens an orphan Issue for each affected document
5. Posts a consolidated Slack alert to `#compliance-policy-help` listing all affected docs

This provides real-time detection with zero lag — the Policy Team is notified the same day the user departs, with time to act before any review deadline is breached.

---

## 9.4 Orphan Lifecycle

```
Orphaned document detected
        │
        └─ [Orphaned Document] Issue opened
                │
                ├─ Phase 2: Slack alert to #compliance-policy-help
                │
                ├─ Policy Team identifies new Document Owner
                │     (must have same tier approval authority as previous owner)
                │
                ├─ Policy Team updates document-registry.yaml:
                │     - owner: "New Owner Name"
                │     - document_owner_github: "@new-owner-handle"
                │
                ├─ Policy Team updates document frontmatter (owner field)
                │
                ├─ PR opened for registry + frontmatter update
                │   → validate.yml, route-approval.yml, approval-gate.yml fire
                │   → merge → publish.yml fires (frontmatter updated)
                │
                └─ Issue closed with comment confirming new owner and PR link
```

**Urgency escalation:** If an orphaned document's `next_review_date` is within 30 days of the orphan Issue being opened, the Issue body flags this with a prominent warning. The `review-alerts.yml` workflow independently tracks review deadlines — the two systems complement each other.

---

## 9.5 Issue Format

```
Title:  [Orphaned Document] {doc_id} — {title}
Labels: orphaned-document
Assign: @hreynolds95 (Phase 1) / @compliance-policy-team (Phase 2)
```

**Issue body:**

```markdown
## Orphaned Document Alert

**Document:** {title} (`{doc_id}`)
**Tier:** {tier} | **Domain:** {domain} | **Doc Type:** {doc_type}
**Current owner (registry):** {owner} ({document_owner_github})
**Next review date:** {next_review_date}

### Why flagged

{reason}
<!-- e.g.:
     "document_owner_github is null or empty in document-registry.yaml"
     "Manually flagged via workflow_dispatch (Phase 1 simulation)"
     "Owner @handle removed from squareup org on {date}" (Phase 2)
-->

### Required action

The Policy Team must assign a new Document Owner and update the registry
**before {next_review_date}** to avoid a missed review cycle.

**Steps to resolve:**
1. Identify a new Document Owner (same tier approval authority required)
2. Update `document-registry.yaml`: set `owner` and `document_owner_github` for `{doc_id}`
3. Update the document frontmatter: `owner` field in `{path}`
4. Open a PR for the registry + frontmatter change; merge through normal gate sequence
5. Comment on this issue with the new owner's name, handle, and PR link
6. Close this issue once the PR is merged

> **Phase 1 simulation note:** This workflow detects null/empty `document_owner_github`
> values in the registry. Phase 2 will also trigger on `organization.member_removed`
> events for real-time departure detection.
```

---

## 9.6 Registry Update Procedure

When a new owner is assigned, two files must be updated in a single PR:

**`document-registry.yaml`:**
```yaml
- doc_id: {doc_id}
  owner: "New Owner Name"
  document_owner_github: "@new-owner-handle"
  # all other fields unchanged
```

**Document frontmatter (`docs/{tier}/{filename}.md`):**
```yaml
owner: "New Owner Name"
```

The PR follows the standard tiered approval path:
- **Tier 1 doc:** requires `approved-t1` label (Board-level change)
- **Tier 2 doc:** requires `approved-t2` label (Committee-level)
- **Tier 3 doc:** requires `approved-t3` label (Owner-level)

Owner reassignment is treated as an **Immaterial Change** — it does not affect document content or substantive requirements, so the full stakeholder review cycle is not required. The `change_type` field in the PR branch should reflect `immaterial`.

---

## 9.7 Labels

| Label | Color | Created by |
|---|---|---|
| `orphaned-document` | `#e4e669` (yellow) | `orphan-detection.yml` on first run |

---

## 9.8 Phase 2 Enhancements

| Enhancement | PoP Section | Description |
|---|---|---|
| Org membership event trigger | 2.4 | `organization.member_removed` fires workflow; real-time detection of departures |
| Org API membership check | 2.4 | Weekly cron queries GitHub org API to verify each `document_owner_github` handle is still an active org member; catches departures between events |
| Slack alert per orphan | 2.9 | POST consolidated message to `#compliance-policy-help` listing all affected docs with owner name, tier, and next review date |
| Bulk reassignment on departure | 2.4 | If departing owner holds multiple documents, single Issue groups all affected docs; Policy Team resolves in one PR |
| Interim owner designation | 2.4 | Issue template includes field for Policy Team to designate an interim owner (responsible for upcoming review) vs. permanent owner (long-term accountability); registry schema extended with `interim_owner_github` |
