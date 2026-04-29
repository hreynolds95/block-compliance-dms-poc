# Section 12 — Legacy Archive & Migration Strategy
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.1 (In-Scope Documents — migration scope),
> Section 2.13 (Enterprise Document Approval Matrix — approval history must be preserved),
> Section 2.7 (Document Ownership — owner accountability transfers on migration)
>
> **Migration constraint:** LogicGate must remain the system of record until all GitHub
> readiness criteria in Section 12.4 are met. No cutover until all criteria are satisfied.

---

## 12.1 Overview

The Block Compliance DMS migration moves the system of record for In-Scope Documents from
LogicGate to GitHub. The strategy has three guiding principles:

1. **Link, don't copy.** Pre-migration history (signed approval PDFs, Google Doc redlines,
   LogicGate records) stays where it is. GitHub holds a link registry (`LEGACY-ARCHIVE.md`),
   not a content copy. This avoids format conversion, avoids duplicating controlled documents
   in a new location, and keeps the audit trail anchored to its original system.

2. **LogicGate stays live until cutover criteria are met.** Dual-system operation is a
   temporary state with a defined exit. The readiness checklist in Section 12.4 is the gate.
   Until all items are checked, LogicGate is authoritative and GitHub is a shadow system.

3. **Migrate in cohorts by tier.** The 6 PoC documents prove the workflow end-to-end before
   the full portfolio (~190 documents) is migrated. Tier 1 documents migrate first
   (highest visibility, most scrutiny); Tier 3 last (largest volume, lowest approval overhead).

---

## 12.2 Current State (Pre-Migration)

| System | What it holds | Status |
|---|---|---|
| **LogicGate** | Authoritative document register; status, owner, review dates, exception tracking | System of record until cutover |
| **Google Drive** | Published PDFs; historical versions; Google Doc source files; signed approval records | Stays in Drive permanently — link-only from GitHub |
| **Snowflake** | `LOGICGATE_SF.PUBLIC.POLICY_DASHBOARD` — pipeline read by compliance portal | Continues feeding portal until GitHub becomes data source |
| **GitHub (this repo)** | 6 pilot documents with full frontmatter; all DocArchitect workflows; spec documents | Shadow system during Phase 1 — not yet authoritative |
| **GitHub Pages portal** | Staff-facing policy library and metrics dashboard | Currently reads from Snowflake/LogicGate; Phase 2 reads from GitHub frontmatter |

### What LogicGate holds that GitHub does not (pre-migration)

- Approval history records (who approved, when, in which committee/board session)
- Exception tracking records prior to GitHub exception workflow going live
- Full document change history predating October 2025 (LogicGate go-live)
- Pre-LogicGate history predating the Google Drive era (paper records, legacy systems)
- LogicGate workflow state (active routing, pending approvals in flight)

---

## 12.3 Migration Scope

### What migrates to GitHub

| Asset | Migration action |
|---|---|
| Document content (Policies, Standards, Procedures) | Author/export to Markdown; frontmatter populated from LogicGate metadata |
| Document metadata (title, tier, domain, owner, review dates, status) | Frontmatter fields; `document-registry.yaml` entry |
| PDF link | `published_pdf` frontmatter field → Drive link (no PDF content copied) |
| LogicGate record cross-reference | `logicgate_record_id` frontmatter field |
| Pre-migration approval history summary | Entry in `LEGACY-ARCHIVE.md` with Drive link to signed approval PDF |

### What never migrates to GitHub

| Asset | Stays in | Reason |
|---|---|---|
| Published PDFs | Google Drive | Binary files; Drive is the distribution surface |
| Historical PDF versions | Google Drive | Linked from `LEGACY-ARCHIVE.md`; no content copy |
| Google Doc source files | Google Drive | Authoring surface stays in Drive; GitHub holds Markdown mirror |
| Signed approval records (board/committee minutes, attestations) | Google Drive / LogicGate | Legal record; must stay in original signed format |
| Implementation plans | Google Drive | Explicitly excluded from GitHub per DocArchitect constraints |
| Audit workpapers | GRC tool / SharePoint | Evidence records stay in evidence management system |
| Pre-GitHub exception records | LogicGate | Historical exceptions not re-entered; linked from `LEGACY-ARCHIVE.md` |
| Board / committee presentation decks | Google Drive | Read-only reference; not In-Scope Documents |

---

## 12.4 GitHub Readiness Criteria

**All criteria must be satisfied before GitHub becomes the system of record.** The Policy
Team reviews this checklist and formally approves cutover.

### Technical readiness

- [ ] All In-Scope Documents have valid frontmatter (all required fields, passing `validate.yml`)
- [ ] All documents have `published_pdf` links pointing to current PDFs in Google Drive
- [ ] All documents have `logicgate_record_id` values for bidirectional cross-reference
- [ ] `document-registry.yaml` is complete for all documents (owner, tier, stakeholder_groups)
- [ ] `LEGACY-ARCHIVE.md` is fully populated — every document has a pre-migration history entry
- [ ] At least one complete end-to-end governance cycle completed in GitHub (initiate → approve → merge → publish; FC-017 is the target)
- [ ] Exception handling workflow tested end-to-end (submit → triage → approve/reject → expiry)
- [ ] `review-alerts.yml` confirmed generating correct Issues on test docs
- [ ] `publish.yml` notification step confirmed logging correct Slack payloads
- [ ] Branch protection confirmed: `approval-gate` required status check blocking merge without correct label
- [ ] CODEOWNERS confirmed gating commits on all tier paths, `audit/`, `document-registry.yaml`, `.github/workflows/`

### Organisational readiness (Phase 2 / squareup org)

- [ ] Repo migrated to `squareup` org
- [ ] GitHub SSO/SAML enabled — all Block employees can authenticate
- [ ] `@squareup/compliance-policy-team`, `@squareup/document-owners`, `@squareup/document-submitters` teams created with correct membership
- [ ] CODEOWNERS updated with real `@squareup/` team handles
- [ ] Branch protection updated: required PR reviews enabled (1 non-author reviewer)
- [ ] `SLACK_WEBHOOK_URL` secret configured — publication notifications and digest activated
- [ ] Portal "Request Review" button live and calling `annual-review.yml` dispatch via GitHub API
- [ ] Portal exception register tab live (reads `exceptions/exceptions.jsonl`)
- [ ] Quincy RAG chat system prompt updated to reflect GitHub as primary source

### Process readiness

- [ ] Policy Team sign-off on migration plan
- [ ] Document Owners briefed on GitHub workflow (annual-review dispatch, PR approval, portal usage)
- [ ] Document Submitters trained on branch/PR workflow and frontmatter standards
- [ ] LogicGate records updated with `github_url` cross-reference field for each migrated document
- [ ] LogicGate decommission plan agreed with IT (or retention-only mode confirmed)
- [ ] Communication sent to all Block compliance staff announcing GitHub as new SoR

---

## 12.5 Migration Approach

### Phase 1 — PoC pilot (6 documents, this repo)

The 6 PoC documents are already scaffolded with full frontmatter and section structure.
They serve as the proof that the technical workflow is sound. No full portfolio migration
occurs until the squareup org is ready.

**Pilot validation test:** Complete a full annual review cycle for FC-017 (most urgent,
due 2026-05-31):
- Actions → Initiate Annual Review → doc_id=FC-017, change_type=immaterial
- Review branch created; PR opened; validate → role-enforce → route-approval fire
- Apply `approved-t2` label to simulate committee sign-off
- Merge → publish.yml fires; audit log entry confirmed; notification payload logged

### Phase 2 — Full portfolio migration (~190 documents)

**Step 1 — LogicGate metadata export**

Export all In-Scope Document records from LogicGate. Required fields per document:
`title`, `doc_id` (or assign new), `tier`, `domain`, `doc_type`, `owner`, `owner_email`,
`next_review_date`, `status`, `legal_entity`, `business`, `logicgate_record_id`,
`published_pdf` (Google Drive link).

**Step 2 — Batch frontmatter generation**

Script generates a Markdown file per document with frontmatter populated from the export.
Document body is seeded with the correct template (`policy-standard-template.md` or
`procedure-template.md`) and a placeholder body pointing to the Google Drive PDF.

```
scripts/
  migrate_from_logicgate.py   # Reads LogicGate export CSV/JSON; generates docs/**/*.md stubs
```

**Step 3 — Cohort PRs by tier**

Documents are migrated in three PRs, one per tier, in reverse tier order (Tier 3 first —
largest volume, lowest approval overhead; Tier 1 last — smallest volume, highest scrutiny).

| PR | Cohort | Approx. count | Gate |
|---|---|---|---|
| Migration PR 1 | Tier 3 — Owner-approved | ~100 docs | `approved-t3` |
| Migration PR 2 | Tier 2 — Committee-approved | ~60 docs | `approved-t2` |
| Migration PR 3 | Tier 1 — Board-approved | ~30 docs | `approved-t1` |

`validate.yml` runs on every document in each PR. Frontmatter errors block merge. All errors
must be resolved before the PR can advance.

**Step 4 — `LEGACY-ARCHIVE.md` population**

For each migrated document, add an entry to `LEGACY-ARCHIVE.md` linking to:
- The pre-migration PDF in Google Drive
- The LogicGate record
- Any historical versions available in Drive

**Step 5 — LogicGate cross-reference update**

After each cohort PR merges, update the corresponding LogicGate records with a `github_url`
field pointing to the document's path in the GitHub repo. This maintains bidirectional
linkage during the dual-system transition period.

**Step 6 — Cutover announcement**

Once all three cohort PRs are merged and the readiness checklist is fully checked:
- Policy Team sends all-hands communication to compliance staff
- Portal data source switches from Snowflake/LogicGate to GitHub frontmatter
- LogicGate moves to read-only / retention mode

---

## 12.6 Dual-System Operation (Transition Period)

During the period between Phase 2 launch and full cutover, both GitHub and LogicGate hold
document records. The following rules govern this period:

| Scenario | Authoritative system | Action |
|---|---|---|
| New document published | LogicGate (until cutover) | Update LogicGate; mirror frontmatter to GitHub |
| Document under annual review | LogicGate (until cutover) | Run review in LogicGate; update GitHub frontmatter after approval |
| Exception request received | LogicGate (until cutover) | Track in LogicGate; GitHub exception workflow available for pilot docs only |
| Review date missed | LogicGate generates alert | GitHub `review-alerts.yml` also fires for pilot docs |

**Divergence risk:** The highest-severity gap identified in Section 11. Mitigated by:
- Field-level authority mapping: LogicGate is authoritative for status and dates during transition
- `logicgate_record_id` / `github_url` cross-references maintained in both systems
- Weekly manual reconciliation by Policy Team until cutover

---

## 12.7 `LEGACY-ARCHIVE.md` — Purpose and Format

`LEGACY-ARCHIVE.md` is a Markdown file in the repo root. It is the human-readable index
of all pre-GitHub document history. It does not contain document content — it contains links.

**When to add an entry:**
- When a document is migrated to GitHub (its pre-migration history gets an entry)
- When a document is retired (its full history, including GitHub history, gets an entry)

**Schema per entry:**

```markdown
### {doc_id} — {title}

| Field | Value |
|---|---|
| Current GitHub path | `docs/{tier}/{filename}.md` |
| LogicGate record | [Record link](https://block.logicgate.com/records/{id}) |
| Published PDF (current) | [Drive link](https://drive.google.com/...) |
| Pre-migration history | [Drive folder](https://drive.google.com/...) |
| Approval history (pre-GitHub) | [Approval record](https://drive.google.com/...) |
| Notes | Free text — migration notes, known gaps, retired date if applicable |
```

See `LEGACY-ARCHIVE.md` for the populated example using the 6 PoC documents.
