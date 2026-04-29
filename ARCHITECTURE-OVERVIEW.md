# Section 1 — Architecture Overview
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.1 (In-Scope Documents — system scope), Section 2.4 (Two-Role
> Model — architectural separation of Submitter and Owner), Section 2.9 (Awareness Campaign —
> notification architecture), Section 2.13 (Enterprise Document Approval Matrix — tiered
> enforcement architecture)

---

## 1.1 Design Rationale

Block's compliance document management requirement has three properties that make GitHub
a strong fit as the system of record:

**Every meaningful change should leave an immutable, actor-attributed record.** Git's commit
history is exactly this — append-only, timestamped, tied to an authenticated identity, and
cryptographically linked (each commit references its parent). No separate audit database
is needed; the version control system is the audit trail.

**Review and approval is a structured workflow, not a social process.** GitHub's pull request
model enforces that all changes to `main` pass through a reviewable, commentable, gate-checked
branch workflow. GitHub Actions automates the gate sequence (validate → role enforce → route
approval → approval gate → publish) without requiring manual coordination by the Policy Team
on each document change.

**Staff need a read interface that is not a code repository.** GitHub.com is not accessible
to non-technical compliance staff as a document library. The GitHub Pages portal provides
the read surface — metadata search, filters, document detail, audit log — while the GitHub
repo remains the write surface. Staff never interact with git directly.

---

## 1.2 System Architecture

```
                    ┌─────────────────────────────────────────┐
                    │         WRITE SURFACE (GitHub)          │
                    │                                         │
  Document          │  block-compliance-dms-poc               │
  Submitter  ──PR──▶│  ┌─────────────────────────────────┐   │
                    │  │  docs/tier-{1,2,3}/**/*.md       │   │
  Document          │  │  document-registry.yaml          │   │
  Owner    ──label──│  │  non-governed-registry.yaml      │   │
                    │  │  audit/audit-log.jsonl            │   │
  Policy    ──merge─│  │  exceptions/exceptions.jsonl     │   │
  Team               │  └─────────────────────────────────┘   │
                    │               │                         │
                    │    12 GitHub Actions workflows           │
                    │    (validate → publish → notify)        │
                    └───────────────┬─────────────────────────┘
                                    │
                    ┌───────────────▼─────────────────────────┐
                    │         READ SURFACE (Portal)           │
                    │                                         │
                    │  block-compliance-policies (GH Pages)   │
                    │  ┌─────────────────────────────────┐   │
  Block      ──────▶│  │  /index.html  Policy library    │   │
  employees          │  │  /dashboard.html  Metrics       │   │
                    │  │  /section-logic.html  Reference  │   │
                    │  │  Quincy RAG chat                 │   │
                    │  └────────────┬────────────────────┘   │
                    └───────────────┼─────────────────────────┘
                                    │ reads
                    ┌───────────────▼─────────────────────────┐
                    │         DATA PIPELINE                   │
                    │                                         │
                    │  Phase 1: LogicGate → Snowflake →       │
                    │           generate_docs_data.py →       │
                    │           site/docs-data.json           │
                    │                                         │
                    │  Phase 2: GitHub frontmatter →          │
                    │           generate_docs_data.py →       │
                    │           site/docs-data.json           │
                    └─────────────────────────────────────────┘

External systems (linked, not migrated):
  Google Drive  — published PDFs, pre-migration history, implementation plans
  LogicGate     — current SoR (Phase 1); retention-only after Phase 4 cutover
  Slack         — notification delivery (SLACK_WEBHOOK_URL / bot token Phase 2)
```

---

## 1.3 Component Inventory

| Component | Repo / Location | Purpose | Who interacts |
|---|---|---|---|
| **Document backend** | `hreynolds95/block-compliance-dms-poc` → `squareup/block-compliance-policies` | System of record for all In-Scope Documents; all workflow automation | Document Submitters (PR), Document Owners (review/label), Policy Team (merge/admin) |
| **GitHub Actions workflows** | `.github/workflows/` in backend repo | Gate sequence enforcement, audit log, notifications, cron jobs | Triggered automatically; Policy Team configures |
| **Staff portal** | `hreynolds95/block-compliance-policies` → `squareup/block-compliance-policies` (Pages) | Read-only document library, metrics dashboard, change request initiation, new doc request | All Block employees (read); Document Owners/Submitters (request review Phase 2) |
| **`docs-data.json`** | `site/docs-data.json` in portal repo | Generated metadata snapshot powering the portal; no backend API required | Generated by `generate_docs_data.py`; read by portal JavaScript at runtime |
| **`document-registry.yaml`** | Backend repo root | Central metadata index for workflow lookups (owner, tier, stakeholder_groups); mirrors key frontmatter fields | Read by all workflows; updated by Policy Team via PR |
| **`audit-log.jsonl`** | `audit/` in backend repo | Append-only publication record; every merge-to-main event | Written by `publish.yml`; read by portal audit panel |
| **`exceptions.jsonl`** | `exceptions/` in backend repo | Append-only approved exception record | Written by `exception-lifecycle.yml`; read by portal (Phase 2) |
| **Google Drive** | External | Published PDFs; pre-migration history; implementation plans | Document Owners (upload PDFs); all employees (read via portal link) |
| **LogicGate** | External | Current SoR (Phase 1 transition); retention-only post-cutover | Policy Team (current operations); IT (decommission) |
| **Slack** | External | Publication notifications; monthly digest; orphan alerts | Stakeholder groups (receive); Policy Team (configure webhook) |

---

## 1.4 PoP Alignment

Every technical component maps to a specific requirement in Block's Policy on Policies. This
table is the traceability record — the evidence that the architecture is not a best-guess
interpretation but a deliberate implementation of the PoP.

| PoP Section | Requirement | Technical implementation |
|---|---|---|
| **2.1** | Define In-Scope Document types (Policies, Frameworks, Standards, Enterprise Procedures) | `doc_type` frontmatter field + `validate.yml` enum check; `docs/tier-{1,2,3}/` directory hierarchy |
| **2.2** | Document hierarchy: Policy → Standard → Procedure → sub-procedural | Directory structure (tier-1/2/3) for governed docs; `non-governed-registry.yaml` for sub-procedural docs (Section 10) |
| **2.3** | Define out-of-scope documents | `NON-GOVERNED-CONTROLS.md` taxonomy; `non-governed-registry.yaml` informational registry |
| **2.4** | Two-role model: Submitter drafts/submits; Owner approves | `role-enforce.yml` Gate 2: checks PR author vs. `document_owner_github` in registry; CODEOWNERS Phase 2 |
| **2.4** | Owner self-approval prohibition on off-cycle material changes (fn.9) | `role-enforce.yml` Rule 2: blocks when author == `document_owner_github` AND `change_type: material` |
| **2.6.3** | Exception expiry notifications | `exception-expiry.yml` daily cron: opens renewal Issues at 30d, expired Issues on expiry day |
| **2.7** | Every document must have an accountable Document Owner | `document_owner_github` required field in `document-registry.yaml`; `orphan-detection.yml` weekly scan |
| **2.7.3** | Policy Team mandatory QC gate before tiered approval | CODEOWNERS gate on all `docs/**/*.md` assigned to `@squareup/compliance-policy-team` (Phase 2); manual in Phase 1 |
| **2.9** | Awareness campaign: notify stakeholder groups on publication | `publish.yml` notification step reads `stakeholder_groups` from registry; posts to Slack channels |
| **2.9** | Monthly portfolio health digest for Policy Team | `monthly-digest.yml` cron: opens GitHub Issue + Slack on 1st of each month |
| **2.11** | Change type taxonomy: Scheduled Review / Off-Cycle Material / Immaterial | `change_type` frontmatter field (`immaterial` \| `material`); set by `annual-review.yml` dispatch |
| **2.11.1** | Material changes require full stakeholder review cycle | `route-approval.yml` checklist comment includes stakeholder review step; `change_type` recorded in audit log |
| **2.13** | Enterprise Document Approval Matrix: T1 Board, T2 Committee, T3 Owner | `approval-gate.yml` required status check; passes only when `approved-t{tier}` label present; `route-approval.yml` routes by tier |
| **2.13** | Approval history must be preserved | `audit-log.jsonl` records every publication event with actor, SHA, tier, change_type; PR history in git |

---

## 1.5 Gate Sequence

Every document change to `main` passes through a linear gate sequence. Gates are implemented
as GitHub Actions status checks; a failing gate blocks merge.

```
PR opened (branch → main)
        │
        ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Gate 1: validate.yml                                       │
  │  • Required frontmatter fields present and valid            │
  │  • approval_type consistent with tier                       │
  │  • 4 required section headings present in document body     │
  │  • docs/_templates/ exempt                                  │
  └─────────────────────────┬───────────────────────────────────┘
                             │ pass
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Gate 2: role-enforce.yml                                   │
  │  • Rule 1: PR author must be document_submitter_github      │
  │  • Rule 2: if off-cycle material change, PR author must     │
  │            NOT be document_owner_github (self-approval ban) │
  └─────────────────────────┬───────────────────────────────────┘
                             │ pass
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Gate 3: route-approval.yml                                 │
  │  • Detects tier of changed documents                        │
  │  • Applies tier-{n}-review label                            │
  │  • Posts approval checklist comment                         │
  └─────────────────────────┬───────────────────────────────────┘
                             │ (advisory — no block)
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Policy Team QC (PoP 2.7.3)                                 │
  │  Phase 1: manual review by @hreynolds95                     │
  │  Phase 2: CODEOWNERS requires @squareup/compliance-policy-  │
  │           team review before merge                          │
  └─────────────────────────┬───────────────────────────────────┘
                             │ approved
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Gate 4: approval-gate.yml  (required status check)         │
  │  • Fails until approved-t{n} label is present on PR         │
  │  • Label must match tier detected by route-approval.yml     │
  │  • Merge button stays locked until this check passes        │
  └─────────────────────────┬───────────────────────────────────┘
                             │ pass
                             ▼
                        MERGE TO MAIN
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  publish.yml (post-merge)                                   │
  │  • Updates effective_date + last_reviewed_date in           │
  │    frontmatter; sets status: published                      │
  │  • Appends entry to audit/audit-log.jsonl                   │
  │  • Sends publication notifications to stakeholder_groups    │
  └─────────────────────────────────────────────────────────────┘
```

---

## 1.6 Cron-Driven Automation

Three workflows run on a schedule independently of document changes. Together they provide
the continuous monitoring layer — the system does not wait for a human to notice a missed
review or an expiring exception.

| Workflow | Schedule | Purpose |
|---|---|---|
| `review-alerts.yml` | Daily 09:00 ET | Opens Issues at 90/60/30 days before `next_review_date`; opens overdue Issue on breach |
| `exception-expiry.yml` | Daily 10:00 ET | Opens renewal Issue 30 days before exception expiry; opens expired Issue on expiry day |
| `monthly-digest.yml` | 1st of month 10:00 ET | Opens portfolio health digest GitHub Issue; logs/sends Slack summary |
| `orphan-detection.yml` | Monday 09:00 ET | Scans registry for documents with no active `document_owner_github`; opens orphan Issues |

---

## 1.7 Phase 1 vs Phase 2 Architecture

The PoC is intentionally designed so Phase 2 activation requires configuration changes,
not code rewrites. Every gap between Phase 1 simulation and Phase 2 enforcement is bridged
by a setting, a secret, or a team handle — not a new workflow.

| Capability | Phase 1 (PoC — personal account) | Phase 2 (squareup org — production) |
|---|---|---|
| Approval enforcement | Label applied by `@hreynolds95`; process constraint | CODEOWNERS + required non-author review; hard technical gate |
| Role enforcement | Advisory comment; cannot block merge | Branch protection prevents self-merge; CODEOWNERS enforces team separation |
| Slack notifications | Payload logged to Actions run | `SLACK_WEBHOOK_URL` secret set; zero code change |
| Org membership events | Not available (personal account) | `organization.member_removed` trigger in `orphan-detection.yml` |
| Portal access | Public; no auth | GitHub SSO/SAML; Block employee login required |
| Owner assignment | `@hreynolds95` hardcoded (`POC_ASSIGNEE`) | Dynamic lookup from `document_owner_github` in registry |
| Document Owners | Placeholder handles (`@tyler-hand-block`) | Real `@squareup/` user handles |
| Portal data source | LogicGate → Snowflake → `docs-data.json` | GitHub frontmatter → `docs-data.json` (same script, different input) |
| Exception tracking | GitHub Issues + `exceptions.jsonl` | Same; enhanced with portal exception register tab |
| WORM audit record | `audit-log.jsonl` + branch protection | Same + Snowflake export with immutable retention |
