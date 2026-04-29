# Section 10 — Non-Governed Document Controls
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.1 (In-Scope Document types — Policies, Frameworks, Standards,
> Enterprise Procedures), Section 2.2 (Document Hierarchy), Section 2.3 (Out-of-Scope documents),
> Section 2.7 (Document Ownership — owner accountability extends to non-governed supporting docs)

---

## 10.1 Overview

The DocArchitect governance model applies its full gate sequence — tiered approvals, template enforcement, CODEOWNERS review, audit log, annual review cron — exclusively to **In-Scope Documents** as defined in PoP Section 2.1: Policies, Frameworks, Standards, and Enterprise Procedures. A substantial body of supporting documentation exists across Block Risk & Compliance that does not meet this definition.

Section 10 defines:
- The complete document hierarchy, including the non-governed sub-procedural tier
- Taxonomy of non-governed document types with Block examples
- What controls apply (light-touch) and what does not (full governance gate sequence)
- The bright-line test for governed vs. non-governed classification
- Storage and owner accountability expectations per category
- The `non-governed-registry.yaml` artifact — an informational index with no workflow automation

---

## 10.2 Full Document Hierarchy

```
Block, Inc. Compliance Document Hierarchy
══════════════════════════════════════════════════════════════════
 GOVERNED  ──  Full DocArchitect gate sequence applies
──────────────────────────────────────────────────────────────────
  Tier 1    Policies                    Board-approved; annual review
  Tier 2    Standards / Frameworks      Committee-approved; annual review
  Tier 3    Enterprise Procedures       Owner-approved; triennial review
──────────────────────────────────────────────────────────────────
 NON-GOVERNED  ──  Section 10 light-touch controls only
──────────────────────────────────────────────────────────────────
  Sub-procedural    Desktop Manuals (DMs)       ┐  Implement and
  (operational)     Desktop Procedures          ┤  support Tier 3
                    Work Instructions           ┘  procedures

  Reference /       Job Aids / Quick Reference Cards
  support           FAQ Documents
                    Regulatory Mapping Matrices
                    Glossaries / Definitions Registers

  Training /        Training Materials and Decks
  communications    Awareness Communications

  Project /         Implementation Plans  ← explicitly excluded from GitHub
  process           Project Plans / Charters
                    Working Group Notes / Meeting Minutes

  Evidence /        Audit Workpapers
  audit             Control Testing Evidence
                    Board / Committee Presentation Decks
══════════════════════════════════════════════════════════════════
```

### Sub-procedural tier — definitions

The sub-procedural tier sits immediately below Enterprise Procedures in the hierarchy. These documents implement procedural requirements at the task or system level. They are owned by operational teams (not the Policy Team) and are not subject to the PoP approval matrix.

| Type | Definition | Typical length | Review cycle |
|---|---|---|---|
| **Desktop Manual (DM)** | End-to-end operational guide for a process area; may span multiple systems or roles; implements one or more Enterprise Procedures | 5–30 pages | Annual or as-needed when underlying procedure changes |
| **Desktop Procedure** | Narrower than a DM; documents a single process or sub-process; step-by-step format; may include decision trees | 1–5 pages | Annual or on process change |
| **Work Instruction** | Most granular; single-task instructions; system-navigation steps, screenshots, field-by-field guidance; directly tied to a specific system or tool | 1–2 pages | On system change |

---

## 10.3 Bright-Line Test: Governed vs. Non-Governed

Before classifying a document, apply this test in order:

| Question | Yes → | No → |
|---|---|---|
| **1. Obligation test:** Does this document impose a requirement or obligation on Block employees (thou shalt / thou shalt not)? | **Governed** | Continue |
| **2. Approval matrix test:** Does PoP 2.13 specify a named approval body for this document type? | **Governed** | Continue |
| **3. Implementation test:** Does this document implement or support a governed document without adding new requirements? | **Non-governed** (link to parent) | Continue |
| **4. Evidence test:** Is this document primarily evidence of control execution rather than a control definition? | **Non-governed** (evidence category) | Continue |
| **5. Escalate:** If still unclear | Policy Team classifies; document-registry.yaml is the authoritative record | |

**Edge case — DMs that add requirements:** If a Desktop Manual imposes obligations beyond what the parent Enterprise Procedure already requires (e.g. adds a new approval step, creates a new reporting obligation), the additional requirement must be incorporated into the parent Procedure via a governed change, not documented in the DM alone. DMs implement; they do not originate requirements.

---

## 10.4 Controls Matrix

The table below defines what applies to each non-governed category. "Full governance" refers to the complete DocArchitect gate sequence: validate → role-enforce → policy-team-qc → tiered-approval → publish, with CODEOWNERS, audit log, and annual review cron.

| Category | Named owner | Storage | Access | Retention | Annual review | Parent doc link | GitHub tracking | Portal visible |
|---|---|---|---|---|---|---|---|---|
| **Desktop Manual (DM)** | Yes — operational team lead | Google Drive | Team + compliance-read | 3 years post-retirement | Recommended (on parent Procedure review) | Required (`parent_doc_id`) | `non-governed-registry.yaml` | No (Phase 2: linked from parent doc detail) |
| **Desktop Procedure** | Yes — operational team lead | Google Drive | Team + compliance-read | 3 years post-retirement | Recommended | Required (`parent_doc_id`) | `non-governed-registry.yaml` | No |
| **Work Instruction** | Yes — system/process owner | Google Drive / Confluence | Team | 1 year post-retirement | On system change | Optional | Optional | No |
| **Job Aid / QRC** | Yes — Policy Team or team lead | Google Drive | Broad read | 1 year post-retirement | On parent change | Optional | Optional | No |
| **FAQ Document** | Yes — team lead | Google Drive / Confluence | Broad read | 1 year | As-needed | Optional | No | No |
| **Regulatory Mapping Matrix** | Yes — Regulatory Affairs | Google Drive | compliance-read + legal | 5 years | Annual | No | Optional | No |
| **Training Material** | Yes — Learning & Development | LMS / Google Drive | All employees | 3 years | Annual | Optional | No | No |
| **Implementation Plan** | Yes — project DRI | Google Drive | Project team | Duration of project + 3 years | No | Optional | Link only (no content) | No |
| **Working Group Notes** | Yes — meeting facilitator | Google Drive | Working group members | 3 years | No | No | No | No |
| **Audit Workpaper** | Yes — Internal Audit / compliance DRI | GRC tool / SharePoint | Restricted | 7 years | No | Optional | No | No |
| **Control Testing Evidence** | Yes — control owner | GRC tool | Restricted | 7 years | No | Optional | No | No |
| **Board / Committee Deck** | Yes — Policy Team / Legal | Google Drive | Board / committee access | Permanent | No | Optional | No | No |

### What never applies to non-governed documents

- `validate.yml` frontmatter and section structure checks
- `role-enforce.yml` two-role model enforcement
- `route-approval.yml` tiered label assignment
- `approval-gate.yml` required status check
- `publish.yml` frontmatter updates and audit log
- `review-alerts.yml` 90/60/30-day cron review reminders
- CODEOWNERS gates
- GitHub branch protection rules

### What always applies

- **Named owner** — every non-governed document must have a named individual accountable for its accuracy and currency. "The team owns it" is not sufficient.
- **Storage in an approved location** — Google Drive (preferred), Confluence, LMS, or GRC tool. No personal storage (local drives, personal Dropbox, etc.).
- **Access consistent with content sensitivity** — documents containing PII, regulatory findings, or legal strategy must be access-controlled; broad-read is appropriate only for general operational guidance.
- **Parent doc link for sub-procedural types** — Desktop Manuals, Desktop Procedures, and Work Instructions must identify the Enterprise Procedure(s) they implement. If the parent procedure is retired, the supporting sub-procedural docs must be reviewed for retirement as well.

---

## 10.5 Sub-Procedural Document Lifecycle

Sub-procedural documents (DMs, Desktop Procedures, Work Instructions) are the only non-governed type with a defined lifecycle tied to the governance system, because they implement governed documents.

```
Parent Enterprise Procedure under review (annual or triggered)
        │
        └─ Procedure Owner notifies DM / Desktop Procedure authors
                │
                ├─ Authors review their supporting docs for accuracy
                │
                ├─ No changes needed → author attests in review notes
                │
                └─ Changes needed → author updates doc in Google Drive
                        │
                        └─ Updated storage link recorded in
                           non-governed-registry.yaml (manual update)
```

**Retirement cascade:** When an Enterprise Procedure is retired, the Policy Team checks `non-governed-registry.yaml` for entries with `parent_doc_id` matching the retired procedure. All linked sub-procedural docs must be reviewed by their owners and either:
- Updated to reference the replacement procedure, or
- Retired (removed from Google Drive or moved to an archive folder)

---

## 10.6 Non-Governed Registry (`non-governed-registry.yaml`)

An informational YAML file listing known non-governed documents. No GitHub Actions workflows read or write this file — it is a manual tracking artifact updated by the Policy Team.

**Purpose:** Gives the Policy Team visibility into the non-governed document landscape; enables the retirement cascade check; provides storage links for staff trying to locate supporting docs.

**Not a replacement for Google Drive organization.** The registry supplements but does not replace proper folder structure and naming conventions in Google Drive.

**Schema:**

```yaml
- ng_id: NG-001
  title: "..."
  category: sub-procedural          # sub-procedural | reference | training | project | evidence
  doc_type: desktop-manual          # desktop-manual | desktop-procedure | work-instruction |
                                    # job-aid | faq | regulatory-mapping | training | implementation-plan |
                                    # working-group-notes | audit-workpaper | evidence | presentation
  parent_doc_id: GOV-025            # governed doc this supports (null if not applicable)
  owner: "Name, Title"
  owner_team: "@squareup/team-slug" # Phase 2
  storage_location: google-drive    # google-drive | confluence | sharepoint | lms | grc-tool
  storage_link: "https://drive.google.com/..."
  last_reviewed: "2026-01-15"       # ISO date; null if never formally reviewed
  status: active                    # active | retired
  notes: "..."
```

**`ng_id` assignment:** Sequential `NG-NNN` identifiers assigned by the Policy Team. Not auto-generated — this is a manual registry.

---

## 10.7 Phase 2 Enhancements

| Enhancement | Description |
|---|---|
| Portal: parent doc detail panel | Document detail drawer in policy library shows linked DMs / Desktop Procedures for each Enterprise Procedure (reads `non-governed-registry.yaml`) |
| Retirement cascade automation | When a governed doc is retired via PR, a GitHub Action checks `non-governed-registry.yaml` for linked sub-procedural docs and opens Issues to notify their owners |
| Non-governed registry validation | `validate.yml` extended to lint `non-governed-registry.yaml` schema on change (required fields, valid `parent_doc_id` references, valid enum values) |
| DM / Desktop Procedure review reminders | Optional: `review-alerts.yml` extended to surface sub-procedural docs whose parent procedure review is upcoming, notifying DM/DP owners |
