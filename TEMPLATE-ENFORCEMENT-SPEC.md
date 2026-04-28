# Section 6 — Template Enforcement Specification
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.5 (Document Templates), Section 2.7.3 (Policy Team QC Gate)
> **Template source:**
> - Policy / Standard: https://docs.google.com/document/d/13OH51rvZhq3RbKkt0WJQFKf1B7cOx0sSnCrdP0bUNkk/template/preview
> - Procedure: https://docs.google.com/document/d/18yn5l2ZrBDOxWb1BoQl0NWcxpcedkWbKj3vExN1X-tU/template/preview

---

## 6.1 Overview

All In-Scope Documents must conform to the official Block Compliance document template. Template compliance is a hard gate enforced by `validate.yml` as Gate 1 in the PR pipeline — a PR with missing required sections cannot advance to Policy Team QC.

Markdown equivalents of both templates live in `docs/_templates/` for author reference. The Google Docs links above remain the authoritative source for brand-formatted originals.

---

## 6.2 Template Structure

### Policy / Standard Template

Applies to `doc_type: Policy` and `doc_type: Standard`.

| # | Section | Required? | Notes |
|---|---|---|---|
| 1 | **Overview** | Required | Introduction & Purpose, Scope, Definitions table |
| 2 | **Block, Inc. Requirements** | Optional | Delete if not applicable |
| 3 | **[Brand/Entity]-Specific Requirements** | Optional | Delete section and/or sub-brand sections not applicable; sub-brands: Square, Cash App, TIDAL, Afterpay, Spiral |
| 4 | **Compliance & Enforcement** | Required | Sub-sections below |
| 4.1 | → Roles and Responsibilities | Required | Block, Inc. Roles + Subsidiary Roles |
| 4.2 | → Monitoring and Reporting | Required | Standard language per CMS Policy |
| 4.3 | → Training | Optional | Required if linked to Mandatory Training Program |
| 4.4 | → Record Retention | Required | Standard language referencing Data Classification & Handling Policy |
| 4.5 | → Exceptions | Required | Owner authority, written documentation, Policy Governance Team submission |
| 4.6 | → Confidentiality | Optional | |
| 5 | **Questions & Contact Information** | Required | Owner contact + related documents list |
| 6 | **Document Details & Revisions** | Required | Approval History table + Document Details table |

### Procedure Template

Applies to `doc_type: Procedure`.

| # | Section | Required? | Notes |
|---|---|---|---|
| 1 | **Overview** | Required | Introduction & Purpose, Scope, Definitions table |
| 2 | **Procedure Requirements** | Optional | Delete if not applicable; describes implementation steps |
| 3 | **Compliance & Enforcement** | Required | Same sub-sections as Policy/Standard; Exceptions is Optional for Procedures |
| 4 | **Questions & Contact Information** | Required | |
| 5 | **Document Details & Revisions** | Required | Review cycle: triennial (vs. annual for Policy/Standard) |

---

## 6.3 Required Sections (Enforced by Gate 1)

`validate.yml` checks that the following four section headings are present in every changed document, regardless of `doc_type`. Optional sections (Block Inc. Requirements, Brand-Specific Requirements, Procedure Requirements) are not checked — their absence is valid.

| Required heading | Match logic | Applies to |
|---|---|---|
| `## Overview` | Heading containing `overview` (case-insensitive) | All doc types |
| `## Compliance & Enforcement` | Heading containing both `compliance` and `enforcement` | All doc types |
| `## Questions & Contact Information` | Heading containing both `questions` and `contact` | All doc types |
| `## Document Details & Revisions` | Heading containing `document details` | All doc types |

**Files exempt from section validation:**
- `docs/_templates/**` — template source files
- Files that do not exist at HEAD (deletions)

---

## 6.4 Validation Implementation (`validate.yml`)

Section validation runs as a second step in `validate.yml`, after frontmatter lint. Both steps must pass for the `validate` status check to succeed.

```
PR opens with docs/**/*.md changes
         │
         ├─ Step 1: Frontmatter lint
         │     Required fields present
         │     Valid status / tier / approval_type / doc_type values
         │     Tier ↔ approval_type consistency
         │     Tier ↔ path consistency
         │
         └─ Step 2: Section structure validation
               For each changed doc (excluding docs/_templates/):
                 Read document body (after frontmatter)
                 Extract all Markdown heading lines
                 Check each of 4 required headings is present
                 ❌ FAIL → lists missing sections, blocks PR
                 ✅ PASS → advances to Gate 2 (role-enforcement)
```

**Error message format:**
```
❌ Section validation failed:

  • GOV-999: missing required section "Compliance & Enforcement"
    Expected a heading containing "compliance" and "enforcement"

Refer to docs/_templates/ or the Block Compliance template Google Docs for the required structure.
```

---

## 6.5 Markdown Template Files (`docs/_templates/`)

Local Markdown templates mirroring the official Google Docs. Authors copy the appropriate template when creating a new document.

| File | Applies to | Google Doc source |
|---|---|---|
| `docs/_templates/policy-standard-template.md` | Policy, Standard | [Policy/Standard Template](https://docs.google.com/document/d/13OH51rvZhq3RbKkt0WJQFKf1B7cOx0sSnCrdP0bUNkk/template/preview) |
| `docs/_templates/procedure-template.md` | Procedure | [Procedure Template](https://docs.google.com/document/d/18yn5l2ZrBDOxWb1BoQl0NWcxpcedkWbKj3vExN1X-tU/template/preview) |

Template files are **exempt** from section and frontmatter validation — they contain placeholder frontmatter and are not governed documents.

---

## 6.6 Frontmatter Template

All governed documents must open with a YAML frontmatter block. The full required field set:

```yaml
---
doc_id: GOV-999                        # Assigned by Policy Team on new doc request
title: "Document Title"
version: 1.0.0
status: draft                          # draft | in-review | published | retired
tier: 3                                # 1 | 2 | 3
domain: governance                     # compliance domain slug
legal_entity: "Block, Inc."
business: "Block"
owner: "Full Name"
approval_type: owner                   # board (T1) | committee (T2) | owner (T3)
doc_type: Policy                       # Policy | Standard | Procedure
next_review_date: "2027-04-28"
retention_years: 5
change_type: null                      # material | immaterial (set at review time)
review_cycle: null                     # annual | triennial | off-cycle (set at review time)
published_pdf: null                    # Google Drive link to PDF version (optional)
logicgate_record_id: null              # LogicGate PWF record ID (optional)
---
```

---

## 6.7 Phase 2 Enhancements

| Enhancement | Description |
|---|---|
| Sub-section validation | Check that `Roles and Responsibilities`, `Record Retention`, and `Exceptions` headings are present within `Compliance & Enforcement` for Policy/Standard docs |
| Definitions table check | Verify a Markdown table is present in the Overview section |
| Approval History table check | Verify the Approval History table is present and has at least one non-placeholder row in `Document Details & Revisions` |
| PR comment on failure | Post a structured PR comment (like `role-enforce.yml`) listing missing sections with links to the relevant template |
| Template scaffolding | `annual-review.yml` pre-populates the review branch with the correct template headings if the doc body is empty |
