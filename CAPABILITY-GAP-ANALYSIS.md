# Section 11 — GitHub Capability Gap Analysis
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.13 (Enterprise Document Approval Matrix — platform must enforce
> tiered approvals), Section 2.4 (Two-Role Model — platform must enforce role separation),
> Section 2.6.3 (Exception tracking — platform must support structured exception lifecycle)

---

## 11.1 Overview

This section provides a structured assessment of GitHub's native capabilities against the
functional requirements of the Block Compliance DMS. For each capability area it records:

- What GitHub supports natively
- The gap between native capability and DMS requirements
- The mitigation built in this PoC (if any)
- Residual risk and the Phase 2 path to close it

The analysis covers the **Phase 1 configuration** (personal account, `hreynolds95`) and notes
where the **Phase 2 configuration** (squareup org, GitHub Enterprise) closes gaps automatically.

**Summary verdict:** GitHub is a viable system of record for compliance document management.
The gaps are real but addressable — most through custom GitHub Actions automation (already built
in this PoC) or through GitHub Enterprise features available in Phase 2. The platform's
strengths in versioning, audit trail, and PR-based workflow outweigh its gaps relative to
purpose-built GRC tools for the DMS use case.

---

## 11.2 Access Control

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Role-based repo access (read / write / admin) | GitHub Teams + repo permissions | None at org level | CODEOWNERS assigns write-path ownership per tier and file type | Phase 2: squareup org Teams enforce role separation technically |
| Document-level access control (restrict specific docs to specific users) | Not supported — all users with repo read can read all files | **Gap:** Cannot restrict individual documents within a repo by sensitivity or tier | None in Phase 1 | Phase 2 option: separate repos per sensitivity tier (heavy); or accept org-read parity and control PDF access in Google Drive separately |
| Two-role enforcement (Submitter cannot merge without Owner approval) | Branch protection requires PR review; CODEOWNERS gates review to teams | **Gap:** Can't enforce that the *specific document owner* (not just any team member) approves | `role-enforce.yml` checks PR author vs. `document_owner_github` in registry; posts blocking comment | `role-enforce.yml` is advisory in Phase 1 (can't block merge without a second real reviewer); Phase 2: branch protection + CODEOWNERS creates a hard gate |
| Prevent self-approval on off-cycle material changes | Not natively — a PR author can approve their own PR if they have write access | **Gap:** Owner self-approval prohibition (PoP 2.11.1 fn.9) has no native enforcement | `role-enforce.yml` Rule 2 detects author == `document_owner_github` + `change_type: material` and posts blocking comment | Comment is advisory only in Phase 1 simulation; Phase 2: squareup org enforces "dismiss stale reviews" + required non-author approvals |
| Attribute-based access control (ABAC) | Not supported | **Gap:** Cannot gate access by document domain, tier, or entity without separate repos | Out of scope for Phase 1 | Phase 2: SSO/SAML gates portal access at org level; per-doc ABAC deferred to post-Phase 2 |

---

## 11.3 Approval Workflows

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Tiered approval (T1/T2/T3 different approval bodies) | Branch protection can require teams as code owners | **Gap:** A single branch protection rule can't conditionally require different reviewers based on which files changed | `route-approval.yml` detects tier from changed docs, applies `tier-{n}-review` label; `approval-gate.yml` requires `approved-t{n}` label | Label-based gate is robust; Phase 2: CODEOWNERS per tier path (`docs/tier-1/` → `@squareup/board-approvers`, etc.) makes this a hard native gate |
| Board-level approval (Tier 1) cannot be simulated by a non-board member | Not enforceable — any user with write access can apply a label | **Gap:** Platform cannot verify the identity or authority of the person applying the `approved-t1` label | Phase 1 simulation: `@hreynolds95` applies all labels; process constraint only | Phase 2: restrict `approved-t1` label to `@squareup/board-approvers` team via GitHub org label permissions (not yet a native feature) — mitigated by audit trail showing who applied the label |
| Policy Team QC gate (PoP 2.7.3) — mandatory before tiered approval | CODEOWNERS can require a team review | No native position enforcement in review sequence | Gate order is process-enforced in Phase 1; CODEOWNERS covers Phase 2 | Phase 2: `@squareup/compliance-policy-team` as CODEOWNERS on all `docs/**/*.md` — QC is required before merge regardless of tier |
| Prevent merge without all required labels | Not native — branch protection checks status checks, not labels | **Gap:** GitHub has no native label-required-for-merge feature | `approval-gate.yml` runs as a required status check; fails until correct `approved-t{n}` label is present | Solid mitigation; no Phase 2 change needed |

---

## 11.4 Document Versioning and Redlining

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Full change history per document | Git commit log — immutable, timestamped, actor-attributed | None | Audit log frontmatter + `audit-log.jsonl` supplement git history with business-level metadata | N/A |
| Redline (tracked changes) equivalent | GitHub PR diff view shows line-level changes between branches | **Gap:** Non-technical users unfamiliar with unified diffs; no side-by-side rich-text redline | Markdown source diffs are readable for structured documents; portal renders diff view in Phase 2 | Phase 2: portal renders before/after PDF or highlighted diff; `pandoc` + `latexdiff` option for side-by-side redline |
| Point-in-time snapshot retrieval | Git tag + commit SHA = point-in-time retrieval of any file at any historical state | None | Audit log records SHA for every publication event | N/A |
| Pre-LogicGate history | Not in GitHub | **Gap:** Documents published before October 2025 have no GitHub history | Legacy history stays in Google Drive; linked from `LEGACY-ARCHIVE.md` (Section 12) | No change needed — Google Drive is the authoritative pre-migration record |
| Protection against history rewrite (force push) | Branch protection `--no-force-push` on main | None | Branch protection enabled on main | Phase 2: squareup org enforces this at the organization policy level |

---

## 11.5 Audit Trail

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Immutable publication record | Git commit history is append-only under branch protection | **Gap:** Branch protection `--no-force-push` is configurable, not immutable — an admin can disable it | `audit/audit-log.jsonl` is a secondary record; branch protection on main | Phase 2: squareup org org-level policy locks branch protection; audit log exported to Snowflake provides a third record |
| Actor attribution on all changes | GitHub commit author + PR actor are always recorded | None | `audit-log.jsonl` captures `actor: github.actor` on every publication | N/A |
| GitHub Actions audit log retention | GitHub retains Actions logs for 90 days (free/Team plan) | **Gap:** Workflow run logs purge after 90 days — post-90d debugging is limited | `audit-log.jsonl` in the repo is permanent; workflow logs are supplementary | Phase 2: GitHub Enterprise retains Actions logs for 400 days; Splunk/Datadog ingest for permanent retention |
| Exception approval record | Issues + labels provide actor + timestamp for label application | **Gap:** No cryptographic signature on label application; anyone with write access can relabel | `exception-lifecycle.yml` records actor + timestamp in `exceptions/exceptions.jsonl` on approved label application | Solid for PoC; Phase 2: CODEOWNERS on exceptions path + restricted label write |
| WORM (write-once, read-many) storage | Not supported natively | **Gap:** `audit-log.jsonl` is a flat file; a privileged user could edit it | File is append-only by convention + branch protection; no cryptographic guarantee | Phase 2: export to Snowflake (immutable via time-travel) or S3 with Object Lock |

---

## 11.6 Notifications and Awareness

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Slack notification on document publication | Not native | **Gap:** No built-in Slack integration | `publish.yml` notification step: reads `stakeholder_groups`, deduplicates channels, POSTs to `SLACK_WEBHOOK_URL` (Phase 1: logs payload) | Phase 2: activate `SLACK_WEBHOOK_URL`; upgrade to Slack bot token + `chat.postMessage` for per-channel routing |
| Monthly portfolio digest | Not native | **Gap:** No native scheduled digest | `monthly-digest.yml` cron opens GitHub Issue + Slack payload | Phase 2: webhook activation; Slack Block Kit formatting |
| Single webhook posts to one channel only | Incoming webhook is channel-bound | **Gap:** One `SLACK_WEBHOOK_URL` secret can only reach the channel it was created for; `stakeholder_groups` may list multiple channels | Phase 1: payload logged per channel; Phase 2: upgrade to bot token | Phase 2: `SLACK_BOT_TOKEN` + `chat.postMessage` with `channel` param routes each notification correctly |
| Email notifications to DLs | Not native | **Gap:** No email dispatch from GitHub Actions without a third-party service | Out of scope for Phase 1 | Phase 2: SendGrid or AWS SES; `stakeholder_groups` extended with `@dl` prefix routing |
| GitHub native notifications (watchers, mentions) | Available — users can watch repo or be @mentioned | **Gap:** Requires GitHub accounts; not accessible to staff who don't use GitHub | Portal provides the non-GitHub-user interface | Phase 2: SSO enrollment + Slack notifications cover both surfaces |

---

## 11.7 Exception Management

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Structured exception request intake | GitHub Issue Forms (YAML) provide structured, machine-parseable bodies | None | `exception-request.yml` issue template built | N/A |
| Auto-assignment to Document Owner | Not native — assignment requires manual action or API call | **Gap:** No native "assign based on label" rule | `exception-triage.yml` parses Issue body, looks up `document_owner_github`, assigns | Phase 1 assigns to `@hreynolds95`; Phase 2: real owner handles work once placeholder handles are live GitHub users |
| Expiry date tracking and alerts | Not native | **Gap:** GitHub Issues have no native due-date or expiry field | `exception-expiry.yml` daily cron reads `exceptions.jsonl`, computes days until expiry, opens renewal/expired Issues | Robust mitigation; no Phase 2 change needed |
| Append-only exception record | Not native | **Gap:** JSONL is a flat file, editable by privileged users | `exceptions/exceptions.jsonl` append-only by convention; branch protection prevents force-push | Phase 2: export to Snowflake with immutable retention |
| Exception register in portal | Not yet built | **Gap:** `exceptions.jsonl` is not surfaced in the GitHub Pages portal | Portal Phase 2 item (PORTAL-SPEC.md Section 7.4.1) | Phase 2: exception register tab in portal reads `exceptions.jsonl` at runtime |

---

## 11.8 Search and Discovery

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Staff-facing document discovery | Not native on GitHub.com — code search is developer-oriented | **Gap:** Compliance staff cannot perform metadata-faceted search on GitHub.com | GitHub Pages portal with client-side full-text + metadata search | N/A — portal is the designed access surface |
| Filter by domain, tier, owner, review status | Not native | **Gap:** GitHub has no document metadata index | `docs-data.json` generated from frontmatter; portal filters all run client-side | N/A |
| URL-shareable filtered views | Not native | **Gap:** GitHub.com has no URL-param filter state for file listings | Portal deep links (`?tier=1&domain=governance`) — all filters URL-param driven | N/A |
| AI-assisted document Q&A | Not native | **Gap:** No built-in RAG over repo content | Quincy RAG chat in portal (Phase 2 — production portal only) | Phase 1 PoC: Quincy lives in the compliance-dms portal repo, not this PoC repo |

---

## 11.9 Orphaned Document and Owner Management

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Detect departed Document Owners | Org `member_removed` webhook event (org-level only) | **Gap:** Personal account (Phase 1) has no org membership events | `orphan-detection.yml` cron scans for null/empty `document_owner_github`; `workflow_dispatch` simulation | Phase 2: add `organization: [member_removed]` trigger; org API membership check per handle |
| Real-time owner departure alerting | Not available Phase 1 | **Gap:** Phase 1 can only catch already-null handles, not new departures | Weekly cron is best available; `workflow_dispatch` for manual simulation | Phase 2: real-time via org event |

---

## 11.10 Template and Structure Enforcement

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| Required document section headings | Not native | **Gap:** GitHub has no document structure validation in PRs | `validate.yml` section structure step checks 4 required headings via regex before PR advances | Heading presence is enforced; heading content quality is not (a section titled "Overview" with one sentence passes) |
| Required frontmatter fields | Not native | **Gap:** GitHub has no YAML schema validation built into PR checks | `validate.yml` frontmatter lint step checks required fields and valid enum values | Solid mitigation |
| Table structure within sections | Not native | **Gap:** Cannot validate that required tables (e.g. Approval History, Document Details) are present and correctly structured | Out of scope for Phase 1 — heading presence is the enforced gate | Phase 2 option: extended regex checks for table headers within required sections |
| Template selection by document type | Not native | **Gap:** GitHub can't enforce that a Policy uses the Policy template vs. a Procedure using the Procedure template | `validate.yml` could check `doc_type` frontmatter vs. template path prefix | Not yet built; Phase 2 enhancement |

---

## 11.11 External System Integration

| Requirement | Native GitHub | Gap | PoC mitigation | Residual risk / Phase 2 |
|---|---|---|---|---|
| LogicGate as parallel system of record | Not native — GitHub and LogicGate are separate systems | **Gap:** No automatic sync between GitHub frontmatter and LogicGate records; dual-entry risk | `logicgate_record_id` field in frontmatter links outbound; no inbound sync | Phase 2 readiness criterion: define authoritative source for each field before migration; LogicGate → GitHub migration plan in Section 12 |
| Snowflake data pipeline | Not native | **Gap:** No native GitHub → Snowflake connector | `generate_docs_data.py` script generates `docs-data.json` from frontmatter; downstream Snowflake pipeline reads this | Phase 2: GitHub Actions cron replaces manual script execution |
| Google Drive PDF storage | Not native | Markdown is the authoritative source; PDF is generated output | `publish.yml` generates PDF via `pandoc` on merge (Phase 2); `published_pdf` holds Drive link to generated file; portal surfaces it in document detail drawer | Phase 2: Block-branded `pandoc` template + Drive API upload step in `publish.yml` |
| LogicGate GooseMCP conflict search | Not native | **Gap:** Semantic conflict search on new document requests requires external AI tool | Phase 1: Policy Team performs manual conflict check | Phase 2: Goose-assisted search appends related docs to new document request Issues |

---

## 11.12 Gap Severity Summary

| Gap | Severity | Phase 2 closes? |
|---|---|---|
| Document-level access control | Low — PDFs in Drive are separately access-controlled; metadata sensitivity is limited | Partial (SSO gates org access) |
| Two-role self-approval prohibition enforcement | Medium — advisory comment only in Phase 1; a motivated actor could merge without remediation | Yes — branch protection + non-author required review |
| Board-level label authority verification | Low — audit trail records who applied label; process control | Partial — label write restriction not yet a native GitHub feature |
| Single Slack webhook channel limitation | Medium — Phase 1 logs payloads; no multi-channel delivery | Yes — Slack bot token |
| WORM audit storage | Low for PoC — `audit-log.jsonl` + branch protection is sufficient | Yes — Snowflake export |
| Real-time orphan detection | Low for PoC — weekly cron catches null handles; departure lag is acceptable | Yes — org event trigger |
| LogicGate / GitHub dual system of record | High until migration — risk of divergence between systems | Yes — Phase 2 migration criteria in Section 12 |
| Actions log 90-day retention | Low — `audit-log.jsonl` in repo is permanent | Yes — GitHub Enterprise 400-day retention |
| Non-technical staff access to diffs | Medium — staff navigating GitHub.com diffs is a UX barrier | Yes — portal abstracts GitHub entirely; generated PDF redline closes the remaining gap in Phase 2 |
