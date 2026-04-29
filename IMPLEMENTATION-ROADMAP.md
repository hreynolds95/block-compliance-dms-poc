# Section 13 — Implementation Roadmap
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.1 (In-Scope Documents — migration scope defines Phase 2 volume),
> Section 2.13 (Enterprise Document Approval Matrix — all approval gates must be live before cutover),
> Section 2.9 (Awareness Campaign — staff communications required at each phase transition)

---

## 13.1 Overview

The roadmap moves the Block Compliance DMS from the current proof-of-concept to full production
deployment across five phases. Each phase has a defined objective, milestone set, owner
assignments, and a gated success criterion before the next phase begins.

**Starting point:** Phase 0 — PoC is functionally complete (DocArchitect Sections 3–13 built,
12 workflows deployed, 6 pilot documents). The first end-to-end governance cycle has not yet
been run.

**End state:** GitHub is the authoritative system of record for all ~190 In-Scope Documents
across Block Risk & Compliance. LogicGate is in retention-only mode. All Block compliance staff
access documents via the portal. All governance events (reviews, approvals, exceptions) flow
through GitHub.

---

## 13.2 Phase Summary

| Phase | Name | Objective | Owner | Gate to next phase |
|---|---|---|---|---|
| **0** | PoC Completion | Close the PoC; validate end-to-end workflow; finish spec | Hunter Reynolds (Policy Team) | FC-017 full cycle complete; Sections 13–14 written |
| **1** | squareup Org Setup | Flip from simulation to real enforcement on real org infrastructure | IT + Policy Team | All technical org criteria from Section 12.4 met |
| **2** | Document Migration | All ~190 In-Scope Documents in GitHub with valid frontmatter | Policy Team + Document Owners | Migration readiness checklist fully checked; portal reads GitHub data |
| **3** | Portal Phase 2 | Staff-facing portal fully live with Phase 2 features and SSO | Engineering + Policy Team | All Phase 2 portal features validated; SSO confirmed working for all Block employees |
| **4** | Cutover & Steady State | GitHub becomes SoR; LogicGate decommissioned; process embedded | Policy Team + CCO | Readiness checklist signed off; all-staff communication sent |

---

## 13.3 Phase 0 — PoC Completion

**Objective:** Complete the DocArchitect specification, validate the end-to-end governance
workflow with a real document, and establish the formal baseline from which Phase 1 begins.

**Duration:** 1–2 weeks from DocArchitect spec completion.

**Owner:** Hunter Reynolds, Compliance Policy

### Milestones

| # | Milestone | Owner | Dependency | Status |
|---|---|---|---|---|
| 0.1 | DocArchitect Section 13 (Roadmap) written | Hunter Reynolds | Sections 3–12 complete | In progress |
| 0.2 | DocArchitect Section 14 (Open Decisions) written | Hunter Reynolds | Section 13 | Pending |
| 0.3 | README updated to reflect all sections complete | Hunter Reynolds | Section 14 | Pending |
| 0.4 | FC-017 end-to-end governance cycle completed | Hunter Reynolds | All workflows deployed | Pending |
| 0.5 | `LEGACY-ARCHIVE.md` populated with real Drive + LogicGate links for 6 pilot docs | Hunter Reynolds | Drive/LG links sourced | Pending |
| 0.6 | Phase 0 findings documented; go/no-go recommendation for Phase 1 prepared | Hunter Reynolds | 0.4, 0.5 | Pending |

### FC-017 end-to-end validation test (Milestone 0.4)

```
Actions → Initiate Annual Review
  doc_id = FC-017, change_type = immaterial

Expected sequence:
  ✓ Branch review/FC-017-2026 created
  ✓ PR opened with body populated from registry
  ✓ validate.yml: frontmatter + section structure pass
  ✓ role-enforce.yml: submitter check passes (Rule 1)
  ✓ route-approval.yml: tier-2-review label applied; checklist comment posted
  ✓ approval-gate.yml: fails (no approved-t2 label yet)
  → Apply approved-t2 label (simulate Committee sign-off)
  ✓ approval-gate.yml: passes
  ✓ Merge unblocked; merge to main
  ✓ publish.yml: effective_date + last_reviewed_date updated in frontmatter
  ✓ publish.yml: entry appended to audit/audit-log.jsonl
  ✓ publish.yml: Slack payload logged for #financial-crimes, #compliance-sel
  ✓ Audit log entry visible in portal document detail drawer
```

### Phase 0 success criterion

> FC-017 annual review cycle completes without errors. All expected workflow steps fire in
> sequence. Audit log entry confirmed. Sections 13 and 14 written. `LEGACY-ARCHIVE.md`
> populated. Policy Team go/no-go issued for Phase 1.

---

## 13.4 Phase 1 — squareup Org Setup

**Objective:** Migrate the PoC from a personal GitHub account to the `squareup` org. Enable
SSO/SAML, create real teams, update CODEOWNERS and branch protection. After Phase 1, the
governance gates are technically enforced — not just process-enforced.

**Duration:** 2–4 weeks (IT scheduling dependent).

**Owners:** IT Engineering (org setup, SSO), Hunter Reynolds / Policy Team (team membership,
CODEOWNERS, workflow config).

### Milestones

| # | Milestone | Owner | Dependency |
|---|---|---|---|
| 1.1 | New repo created under `squareup` org: `squareup/block-compliance-policies` | IT | Phase 0 complete |
| 1.2 | PoC repo content pushed to `squareup` org repo | Hunter Reynolds | 1.1 |
| 1.3 | GitHub SSO/SAML enabled on `squareup` org — Block employee login required | IT | 1.1 |
| 1.4 | GitHub Teams created: `compliance-policy-team`, `document-owners`, `document-submitters`, `compliance-read` | IT + Hunter Reynolds | 1.3 |
| 1.5 | Teams populated with initial members (Policy Team + 6 pilot Document Owners) | Hunter Reynolds | 1.4 |
| 1.6 | `CODEOWNERS` updated: all `@hreynolds95` references replaced with `@squareup/` team handles | Hunter Reynolds | 1.4 |
| 1.7 | `document-registry.yaml`: `document_owner_github` placeholder handles replaced with real `@squareup/username` handles for 6 pilot docs | Hunter Reynolds | 1.5 |
| 1.8 | Branch protection updated: required PR reviews (1 non-author reviewer) enabled | Hunter Reynolds | 1.6 |
| 1.9 | `SLACK_WEBHOOK_URL` secret configured — publication notifications and monthly digest activate | Hunter Reynolds | Slack workspace admin |
| 1.10 | Workflow dispatch assignments updated: `POC_ASSIGNEE` replaced with dynamic team-based assignment where applicable | Hunter Reynolds | 1.4 |
| 1.11 | End-to-end governance cycle re-run on `squareup` org repo with real reviewer (GOV-011 or FC-017) | Hunter Reynolds + 1 Document Owner | 1.1–1.10 |
| 1.12 | `role-enforce.yml` Rule 2 (self-approval prohibition) validated with a real owner vs. submitter scenario | Hunter Reynolds | 1.7, 1.8 |
| 1.13 | Portal URL updated: `squareup.github.io/block-compliance-policies` (or custom domain) | IT + Engineering | 1.3 |
| 1.14 | All Block employees confirmed able to authenticate to portal via SSO | IT | 1.3, 1.13 |

### Phase 1 success criterion

> All technical org criteria from `LEGACY-ARCHIVE-SPEC.md` Section 12.4 are met. A real
> two-person governance cycle (Submitter + Owner are different people) completes without errors.
> `role-enforce.yml` Rule 2 blocks self-approval as expected. Slack notifications delivered to
> correct channels on merge. Portal accessible to all Block employees via SSO.

---

## 13.5 Phase 2 — Document Migration

**Objective:** Migrate all ~190 In-Scope Documents from LogicGate to GitHub. Every document
has valid frontmatter, a PDF link, a LogicGate cross-reference, and a `LEGACY-ARCHIVE.md`
entry. `validate.yml` passes for every document.

**Duration:** 4–8 weeks (volume and Policy Team bandwidth dependent).

**Owners:** Hunter Reynolds / Policy Team (migration scripting, PR review, registry update),
Document Owners (content review of migrated stubs), IT (LogicGate export access).

### Milestones

| # | Milestone | Owner | Dependency |
|---|---|---|---|
| 2.1 | LogicGate metadata export completed — all In-Scope Document records (title, tier, domain, owner, review dates, status, LogicGate IDs, PDF links) | IT + Hunter Reynolds | Phase 1 complete |
| 2.2 | `scripts/migrate_from_logicgate.py` written — generates Markdown stubs from export | Hunter Reynolds | 2.1 |
| 2.3 | Migration dry run: script generates all stubs; `validate.yml` run locally against all outputs; errors triaged | Hunter Reynolds | 2.2 |
| 2.4 | Document Owners briefed: GitHub workflow training, frontmatter review expectations, how to complete stub content | Hunter Reynolds | Phase 1 complete |
| 2.5 | **Migration PR 1: Tier 3 (~100 docs)** — stubs generated, content reviewed by owners, `validate.yml` passes, `approved-t3` applied, merged | Policy Team + T3 Owners | 2.3, 2.4 |
| 2.6 | **Migration PR 2: Tier 2 (~60 docs)** — as above for Tier 2 | Policy Team + T2 Owners | 2.5 |
| 2.7 | **Migration PR 3: Tier 1 (~30 docs)** — as above for Tier 1; Board/CCO sign-off required | Policy Team + CCO + Board | 2.6 |
| 2.8 | `LEGACY-ARCHIVE.md` fully populated — every migrated document has Drive folder + LogicGate record + approval history links | Hunter Reynolds | 2.7 |
| 2.9 | LogicGate records updated with `github_url` field for all migrated documents | Hunter Reynolds + IT | 2.7 |
| 2.10 | Portal data pipeline switched: `generate_docs_data.py` reads from GitHub frontmatter (not Snowflake/LogicGate export) | Engineering | 2.7 |
| 2.11 | Portal data refresh moved to GitHub Actions cron — manual download step eliminated | Engineering | 2.10 |
| 2.12 | `non-governed-registry.yaml` populated for all known non-governed supporting documents | Policy Team | 2.7 |
| 2.13 | Review alerts validated: `review-alerts.yml` fires correctly for overdue docs in full portfolio (GOV-025, FC-032 are already overdue) | Hunter Reynolds | 2.5–2.7 |

### Phase 2 success criterion

> All ~190 In-Scope Documents are in GitHub with passing `validate.yml`. Portal reads from
> GitHub frontmatter and shows correct data for full portfolio. `LEGACY-ARCHIVE.md` complete.
> LogicGate cross-references updated. `review-alerts.yml` confirmed generating correct Issues
> for the full portfolio.

---

## 13.6 Phase 3 — Portal Phase 2

**Objective:** Complete the staff-facing portal with Phase 2 features: authenticated access,
"Request Review" button, exception register tab, and Quincy updated to GitHub as source.

**Duration:** 3–5 weeks (Engineering bandwidth dependent).

**Owners:** Engineering (portal development), Hunter Reynolds / Policy Team (content and
workflow validation), IT (SSO).

### Milestones

| # | Milestone | Owner | Dependency |
|---|---|---|---|
| 3.1 | Portal "Request Review" button added to policy library — calls `annual-review.yml` dispatch via GitHub API with authenticated user token | Engineering | Phase 2 complete |
| 3.2 | Portal request review modal: `change_type` selector + notes field; API call confirmed working for T2/T3 owners | Engineering + Hunter Reynolds | 3.1 |
| 3.3 | Portal exception register tab added — renders `exceptions/exceptions.jsonl` client-side; linked from document detail drawer | Engineering | Phase 2 complete |
| 3.4 | Quincy system prompt updated: GitHub as primary source; portal URL updated; `docs-data.json` sourced from GitHub; process-index updated | Hunter Reynolds | Phase 2 complete |
| 3.5 | Goose-assisted conflict search wired to new document request workflow — appends related docs to Issue on submission | Engineering | Phase 2 complete |
| 3.6 | Automated registry scaffolding on new document approval: Policy Team approves new-doc-request Issue → Actions creates doc_id, registry entry, template branch | Engineering | 3.5 |
| 3.7 | Portal Phase 2 regression testing: all existing features validated against full portfolio data | Engineering + Hunter Reynolds | 3.1–3.6 |
| 3.8 | SSO portal access validated: non-GitHub-user Block employees complete SSO enrollment and can access portal | IT + Hunter Reynolds | Phase 1 complete |
| 3.9 | Portal custom domain configured if required (e.g. `compliance-docs.block.xyz`) | IT + Engineering | 3.8 |

### Phase 3 success criterion

> "Request Review" button tested end-to-end by a real Document Owner. Exception register tab
> displays correct data. Quincy answers correctly for documents sourced from GitHub. SSO
> confirmed working for a sample of non-GitHub-user Block employees. All existing portal
> features pass regression test against full portfolio.

---

## 13.7 Phase 4 — Cutover & Steady State

**Objective:** Formally designate GitHub as the system of record. Decommission LogicGate
(or move to retention-only). Embed the GitHub DMS workflow into steady-state compliance
operations.

**Duration:** 2–3 weeks (communication, training, and IT decommission scheduling).

**Owners:** Hunter Reynolds / Stevi Winer (Policy Team), Tyler Hand (CCO sign-off), IT
(LogicGate decommission).

### Milestones

| # | Milestone | Owner | Dependency |
|---|---|---|---|
| 4.1 | Section 12.4 readiness checklist formally reviewed — all items checked and signed off | Hunter Reynolds + Stevi Winer | Phase 3 complete |
| 4.2 | CCO formal approval: GitHub designated as system of record for all In-Scope Documents | Tyler Hand | 4.1 |
| 4.3 | All-staff communication sent to Block compliance community — portal URL, how to request reviews, how to submit exceptions, where to find documents | Hunter Reynolds | 4.2 |
| 4.4 | Document Owner / Submitter training sessions completed | Hunter Reynolds | 4.2 |
| 4.5 | LogicGate moved to retention-only mode — no new records, no updates; existing records preserved for audit history | IT | 4.2 |
| 4.6 | LogicGate → Snowflake pipeline decommissioned or redirected | IT + Engineering | 4.5 |
| 4.7 | `GOV-011 (Compliance Policy on Policies)` updated to reflect GitHub as SoR — PR opened, reviewed, merged through full gate sequence | Hunter Reynolds | 4.2 |
| 4.8 | First post-cutover monthly digest confirmed: `monthly-digest.yml` fires, portfolio reflects full ~190 docs | Hunter Reynolds | 4.3 |
| 4.9 | 30-day post-cutover review: open Issues from `review-alerts.yml`, exception requests, and orphan detections assessed for correctness | Hunter Reynolds | 4.8 |
| 4.10 | Steady-state process documented: Policy Team operating guide for annual review cycle, exception approval, new document onboarding | Hunter Reynolds | 4.9 |

### Phase 4 success criterion

> CCO sign-off received. All-staff communication sent. LogicGate in retention mode. GOV-011
> updated to reference GitHub. First monthly digest opens with correct ~190-doc portfolio.
> 30-day post-cutover review finds no systematic errors in workflow automation.

---

## 13.8 Dependency Map

```
Phase 0 (PoC Completion)
    │
    ├─ FC-017 end-to-end cycle validated ──────────────────────────┐
    ├─ Sections 13 + 14 written                                    │
    └─ LEGACY-ARCHIVE.md real links populated                      │
                                                                   │
Phase 1 (squareup Org Setup)                                       │
    │  requires: Phase 0 complete                                  │
    ├─ SSO/SAML enabled                                            │
    ├─ Teams + CODEOWNERS updated                                  │
    ├─ Branch protection: real reviewer required ──────────────────┤
    └─ SLACK_WEBHOOK_URL configured                                │
                                                                   │
Phase 2 (Document Migration)                                       │
    │  requires: Phase 1 complete                                  │
    ├─ LogicGate export                                            │
    ├─ Migration PRs T3 → T2 → T1 ────────────────────────────────┤
    ├─ LEGACY-ARCHIVE.md complete                                  │
    └─ Portal switched to GitHub data source                       │
                                                                   │
Phase 3 (Portal Phase 2)                                           │
    │  requires: Phase 2 complete                                  │
    ├─ Request Review button                                       │
    ├─ Exception register tab                                      │
    └─ Quincy + Goose conflict search updated ─────────────────────┤
                                                                   │
Phase 4 (Cutover)                                                  │
    │  requires: Phase 3 complete + Section 12.4 checklist ────────┘
    ├─ CCO sign-off
    ├─ All-staff communication
    └─ LogicGate retention mode
```

---

## 13.9 Owner and Resource Summary

| Role | Phase 0 | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|---|---|---|---|---|---|
| **Hunter Reynolds** (Policy Team) | Lead | Lead | Lead | Support | Lead |
| **Stevi Winer** (Head of Compliance Policy) | Reviewer | Approver | Approver | Approver | Co-lead |
| **Tyler Hand** (CCO) | Aware | Aware | Aware | Aware | Sign-off |
| **IT Engineering** | — | Lead (org/SSO) | Support (LG export) | Support (SSO) | Lead (LG decommission) |
| **Portal Engineering** | — | Support | Support | Lead | Support |
| **Document Owners** (6 pilot) | Aware | Enroll | Content review | Test | Train |
| **Document Owners** (full portfolio, ~190) | — | — | Content review | Test | Train |
| **Compliance Legal** | Aware | Aware | Aware | Aware | Review GOV-011 update |

---

## 13.10 Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| IT org setup / SSO takes longer than expected | Medium | Blocks Phase 1 | Early IT engagement; document exact requirements from Section 12.4 technical checklist |
| Document Owner engagement for content review in Phase 2 is low | Medium | Migration PRs stall | Stagger cohort PRs; assign dedicated owner contact per tier; track progress in GitHub Issues |
| LogicGate export is incomplete or poorly structured | Medium | Phase 2 migration script breaks | Request full export early (Phase 0); validate field mapping before scripting |
| ~190 document migration PRs surface large volume of validate.yml errors | High | Phase 2 timeline extends | Run `validate.yml` locally against dry-run output before opening PRs; fix in bulk |
| LogicGate / GitHub divergence during dual-system transition | High | Data integrity risk | Weekly reconciliation; field-level authority mapping (Section 12.6); short transition window |
| Portal Engineering bandwidth unavailable for Phase 3 timeline | Medium | Phase 3 delays Phase 4 | Phase 3 features are independent of Phase 4 cutover for documents; cutover can proceed with portal at Phase 1 feature set if needed |
| Board / CCO availability for Tier 1 migration PR sign-off | Medium | Tier 1 PR stalls | Schedule Tier 1 PR for a known committee meeting window; brief CCO in Phase 1 |
