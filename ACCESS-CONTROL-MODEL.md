# Section 3 — Access Control Model
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.4 (Two-Role Document Model), Section 2.7.3 (Policy Team QC Gate),
> Section 2.13 (Enterprise Document Approval Matrix), Section 2.11.1 fn.9 (Owner Self-Approval Prohibition)

---

## 3.1 GitHub Teams Structure

All access in the production squareup org is managed via GitHub Teams. The table below defines each team, its PoP role mapping, and repository permission level.

| GitHub Team | PoP Role | Repo Permission | Notes |
|---|---|---|---|
| `@squareup/compliance-policy-team` | Policy Team (QC reviewers) | Write | CODEOWNERS gate on all `docs/` paths; required reviewer on every PR |
| `@squareup/compliance-leadership` | Leadership approvers | Write | CODEOWNERS on Tier 1, 2, 3 paths; approve on behalf of leadership chain |
| `@squareup/compliance-committee` | Committee approvers | Write | CODEOWNERS on Tier 2 paths |
| `@squareup/board-delegates` | Board / CCO delegates | Write | CODEOWNERS on Tier 1 paths |
| `@squareup/document-owners` | Document Owners | Write | Approve changes and exceptions for their documents; cannot self-approve off-cycle Material Changes |
| `@squareup/document-submitters` | Document Submitters | Write | Open PRs and draft changes; registered per-document in `document-registry.yaml` |
| `@squareup/compliance-read` | All Block employees (portal read) | Read | Read-only access to published documents via GitHub Pages portal |

> **Phase 1 (PoC):** All team slots are filled by `@hreynolds95`. Replace with real handles/teams when migrating to `squareup` org.

---

## 3.2 Repository Permission Matrix

| Path | Policy Team QC | Tier 1 Approver | Tier 2 Approver | Tier 3 Approver | Document Owners | Submitters | All Employees |
|---|---|---|---|---|---|---|---|
| `docs/tier-1/` | Required review | Required review | — | — | Informed | Write | Read (portal) |
| `docs/tier-2/` | Required review | — | Required review | — | Informed | Write | Read (portal) |
| `docs/tier-3/` | Required review | — | — | Required review | Informed | Write | Read (portal) |
| `docs/enterprise-procedures/` | Required review | — | — | Required review | Informed | Write | Read (portal) |
| `audit/` | Required review | — | — | — | No access | No direct access | No access |
| `.github/workflows/` | Required review | — | — | — | No access | No access | No access |
| `document-registry.yaml` | Required review | — | — | — | No access | No access | No access |

---

## 3.3 CODEOWNERS Gate Sequence

Branch protection on `main` enforces gates in this order. All must pass before merge is permitted.

```
PR opened
    │
    ├─ 1. validate (automated)
    │       Frontmatter lint — required fields, valid values, tier/path consistency
    │       Hard gate: fails block PR advancement to QC
    │
    ├─ 2. role-enforcement (automated)
    │       Verifies PR author is a registered Document Submitter
    │       Blocks owner self-approval of off-cycle Material Changes (PoP 2.11.1 fn.9)
    │
    ├─ 3. policy-team-qc (CODEOWNERS — @squareup/compliance-policy-team)
    │       Content QC for revisions; validity assessment for new document requests
    │       Mandatory per PoP Section 2.7.3; cannot be bypassed
    │
    ├─ 4. approval-gate (automated label check)
    │       Verifies correct tier approval label is present:
    │         approved-t1 → Board/CCO sign-off
    │         approved-t2 → Committee sign-off
    │         approved-t3 → Owner + Leadership sign-off
    │
    └─ Merge to main → publish.yml fires
            Updates effective_date, last_reviewed_date in frontmatter
            Appends entry to audit/audit-log.jsonl
```

---

## 3.4 Two-Role Document Model (PoP Section 2.4)

Every In-Scope Document has exactly two associated roles, registered per-document in `document-registry.yaml`:

| Role | Field in registry | Responsibility | GitHub action |
|---|---|---|---|
| **Document Submitter** | `document_submitter_github` | Drafts and submits changes | Opens PR; pushes commits |
| **Document Owner** | `document_owner_github` | Approves changes and exceptions; holds ultimate accountability | Applies approval label; approves exception Issues |

### Self-Approval Prohibition (PoP 2.11.1, footnote 9)

Off-cycle Material Changes (`change_type: material`, `review_cycle` not `annual`/`scheduled`) may not be submitted by the Document Owner. The `role-enforce.yml` workflow detects this condition and fails the `role-enforcement` status check, blocking PR advancement.

**Allowed combinations:**

| Change type | Review cycle | Owner = Submitter? | Allowed? |
|---|---|---|---|
| `immaterial` | any | Yes | ✅ Allowed |
| `material` | `annual` / `scheduled` | Yes | ✅ Allowed (scheduled review attestation) |
| `material` | `off-cycle` / null | Yes | ❌ **Prohibited** — `role-enforce.yml` blocks |
| `material` | `off-cycle` / null | No | ✅ Allowed |

---

## 3.5 Portal Access Tiers (GitHub Pages)

The staff-facing portal at `hreynolds95.github.io/block-compliance-policies` enforces three access tiers:

| Access tier | Who | Capability |
|---|---|---|
| **Read-only** | All Block employees (`@squareup/compliance-read`) | Browse and search published document library; view metadata |
| **Change request** | Document Submitters + Owners (`@squareup/document-submitters`, `@squareup/document-owners`) | Initiate change requests; open review PRs via annual-review dispatch |
| **New document request** | Any Block employee | Submit self-service new document request form (routed to Policy Team for QC validity evaluation) |
| **Admin** | Policy Team (`@squareup/compliance-policy-team`) | Full write access; manage registry; configure workflows |

> **Phase 1:** Portal is publicly accessible (GitHub Pages on personal account). Phase 2: gate behind GitHub SSO/SAML when migrated to squareup org.

---

## 3.6 Exception Request Access (PoP Section 2.6.3)

Exception requests are filed as GitHub Issues using the `exception-request` template (to be built in Phase 2). Auto-assignment to the Document Owner is enforced by GitHub Actions:

1. Any authorized user opens an exception Issue referencing a `doc_id`
2. GitHub Actions reads `document_owner_github` from `document-registry.yaml`
3. Issue is automatically assigned to the Document Owner
4. Document Owner approves or rejects via Issue label transition
5. Approved exceptions are committed to `EXCEPTIONS.md` (tamper-evident via commit history)
6. Daily cron workflow checks expiry dates and opens renewal Issues

---

## 3.7 GitHub Capability Gap Analysis

| Requirement | PoP Section | GitHub Status | Gap | Interim Mitigation |
|---|---|---|---|---|
| Team-based CODEOWNERS gates | 2.7.3, 2.13 | ✅ Native (org) | Requires squareup org — unavailable on personal account | Individual handles in Phase 1 |
| Two-role model enforcement | 2.4 | ✅ Via Actions | `role-enforce.yml` deployed | None — fully mitigated |
| Owner self-approval prohibition | 2.11.1 fn.9 | ✅ Via Actions | `role-enforce.yml` deployed | None — fully mitigated |
| Sub-directory read restrictions | 2.7 | ⚠️ Partial | No native per-path read restriction in monorepo; all org members with repo access can read all paths | Publish to separate Pages site for read-only access; restrict repo access to contributors only |
| Portal role-gating (SSO) | BRD: Access | ⬜ Not yet | Requires GitHub SSO/SAML on squareup org | GitHub Pages is public in Phase 1; acceptable for PoC |
| Exception request auto-assignment | 2.6.3 | ⬜ Not yet | Issue template + assignment workflow not yet built | Manual assignment in Phase 1 |
| Orphaned document detection | 2.4 | ⬜ Not yet | Org membership event monitoring not yet built | Policy Team manual monitoring in Phase 1 |
| Signed commits | Audit | ⬜ Not yet | Commit signing not enforced | Git history provides tamper-evident record; acceptable for PoC |

---

## 3.8 Phase 2 Upgrade Checklist

When migrating to `squareup` org:

- [ ] Create GitHub Teams: `compliance-policy-team`, `compliance-leadership`, `compliance-committee`, `board-delegates`, `document-owners`, `document-submitters`, `compliance-read`
- [ ] Replace `@hreynolds95` in CODEOWNERS with real team slugs (comments in CODEOWNERS show exact replacements)
- [ ] Update `document-registry.yaml` — replace placeholder `document_owner_github` and `document_submitter_github` handles with real squareup GitHub handles
- [ ] Enable branch protection: require CODEOWNERS review + all status checks on `main`
- [ ] Enable GitHub SSO/SAML on the org to gate portal access
- [ ] Set `compliance-read` team to repo Read — all other employees access portal via GitHub Pages only
- [ ] Enable required commit signing on `main`
- [ ] Build exception request Issue template + auto-assignment workflow (Section 5)
- [ ] Build orphaned document detection workflow (Section 9)
