# Section 14 — Open Decisions & Trade-offs
## Block Compliance DMS | DocArchitect Architecture

> This is the final section of the DocArchitect specification. It records architectural
> decisions made during the PoC design — what was chosen, what alternatives were considered,
> and why — alongside the questions that remain genuinely open and require stakeholder input
> before Phase 2 can proceed.

---

## 14.1 Open Decisions

These items do not yet have a resolved answer. Each one must be decided before the Phase
it blocks can begin. The responsible decision-maker is noted; the Policy Team owns surfacing
these for resolution.

---

### OD-01 — Who holds `approved-t1` authority in GitHub?

**Phase blocked:** Phase 1
**Decision-maker:** Tyler Hand (CCO) + Compliance Legal + General Counsel

The Board of Directors does not use GitHub. In Phase 1 simulation, `@hreynolds95` applies
`approved-t1` as a proxy for all approval bodies. In Phase 2, applying the `approved-t1`
label in GitHub must be a recognised, legally sufficient act of approval.

**Options:**

| Option | Pros | Cons |
|---|---|---|
| CCO applies `approved-t1` label on behalf of Board after board resolution | Minimal GitHub onboarding; approval record stays with Board minutes in Drive | Label is one step removed from the actual approval body |
| Board Secretary applies label after resolution | Board minutes + GitHub record aligned | Requires GitHub account + SSO enrollment for Board Secretary |
| CCO attestation comment on PR + label application | Full record in GitHub | Label authority still rests with a single individual |

**Recommendation (for discussion):** CCO applies `approved-t1` after confirmed Board resolution. Board minutes in Google Drive remain the legally authoritative record; GitHub label is the operational trigger. `LEGACY-ARCHIVE.md` links to the Board resolution for each Tier 1 document approval.

---

### OD-02 — Is GitHub an appropriate system of record for legally significant compliance documents?

**Phase blocked:** Phase 4 (Cutover)
**Decision-maker:** General Counsel + Tyler Hand (CCO)

GitHub was designed as a code repository. Storing legally significant compliance documents
(Board-approved policies, committee-approved standards) as Markdown files in a git repo is
technically sound but requires legal sign-off that:

1. A Markdown file in GitHub constitutes a valid controlled document under Block's legal and regulatory obligations
2. The GitHub audit trail (commit history + `audit-log.jsonl`) satisfies record-keeping requirements applicable to Block's regulated entities
3. GitHub's data residency and security posture are acceptable for compliance document storage

**Resolution needed before Phase 4 cutover.** No blocking issue is anticipated given GitHub's enterprise security posture and the supplementary audit export to Snowflake, but formal General Counsel sign-off is required.

---

### OD-03 — How are active LogicGate exceptions handled during migration?

**Phase blocked:** Phase 2
**Decision-maker:** Hunter Reynolds + Stevi Winer

At the time of Phase 2 migration, LogicGate will hold active exception records (approved
exceptions with open expiry windows). These records are not currently in `exceptions.jsonl`.

**Options:**

| Option | Pros | Cons |
|---|---|---|
| Back-populate `exceptions.jsonl` from LogicGate export for active exceptions | Full GitHub record from day one | Manual re-entry effort; timestamps will reflect migration date, not original approval date |
| Leave active exceptions in LogicGate until they expire naturally; new exceptions only in GitHub | Zero re-entry effort | Dual exception tracking during transition window (months to years depending on extension lengths) |
| Link only: add `legacy_logicgate_id` and Drive link to each exception entry | Minimal effort; maintains audit trail | `exceptions.jsonl` is incomplete; expiry cron won't track LG exceptions |

**Recommendation (for discussion):** Back-populate active exceptions with original approval dates and a `migrated_from_logicgate: true` flag. Effort is bounded — exception volume is small relative to document count.

---

### OD-04 — Single Slack webhook vs. bot token for publication notifications

**Phase blocked:** Phase 1 (partial) / Phase 2 (full activation)
**Decision-maker:** Hunter Reynolds + Slack workspace admin

The current `publish.yml` notification step uses a single `SLACK_WEBHOOK_URL` which is
channel-bound. `stakeholder_groups` in the registry lists multiple channels per document.
Multi-channel routing requires a Slack bot token (`SLACK_BOT_TOKEN`) and `chat.postMessage`.

**Decision needed:** When is the bot token approach implemented? Phase 1 activation with
a single `#compliance-policy-help` webhook is viable as an interim step. Full per-channel
routing requires bot token before Phase 4 cutover (publication notifications must reach
correct stakeholder groups).

---

### OD-05 — PDF generation: automated from Markdown or manual upload?

**Status: RESOLVED — 2026-04-30**
**Decision:** All compliance document content will be authored and maintained in Markdown
in the GitHub repository. On merge, `publish.yml` generates a Block-branded PDF via `pandoc`
and stores it in Google Drive. The `published_pdf` frontmatter field holds the Drive link to
the generated output. Google Docs authoring is retired as the source of truth.

**Rationale:**

1. **Block intelligence world model** — Markdown in the repo is machine-readable and indexable.
   Quincy's RAG can query actual document content, not just metadata. Any future Block AI
   tooling can consume the full compliance corpus directly.

2. **Single source of truth** — Markdown is authoritative. The PDF is a derived, generated
   output. Markdown and PDF cannot diverge.

3. **Audit and regulator downloads** — Generated PDFs are provably derived from the
   version-controlled source. Every published PDF maps to a specific git SHA — a stronger
   chain of custody than a manually uploaded file of unknown provenance.

4. **Portal rendering** — The portal can render Markdown natively and serve generated PDFs
   for download, eliminating the need for a separate PDF store for staff viewing.

**Phase 2 implementation requirements:**

- Block-branded `pandoc` LaTeX template (logos, fonts, footer with doc_id + effective_date)
- `publish.yml` PDF generation step: `pandoc {path} -o {doc_id}.pdf --template block-policy`
- Google Drive API upload in `publish.yml`; write returned URL to `published_pdf` frontmatter
- Content migration: ~160 existing LogicGate documents converted to Markdown during Phase 2
  cohort PRs (Tier 3 → Tier 2 → Tier 1)
- Authoring guidance for Document Owners (lightweight Markdown authoring guide)

---

### OD-06 — Should Desktop Manuals and Desktop Procedures eventually come under a lightweight governance track?

**Phase blocked:** Post-Phase 4
**Decision-maker:** Stevi Winer + Document Owners

Section 10 classifies DMs, Desktop Procedures, and Work Instructions as non-governed.
As the DMS matures, the Policy Team may want lighter-weight governance for sub-procedural
documents — for example, an annual attestation by the owning team lead without the full
tiered approval gate.

**Question:** Does the PoP need to be amended to define a sub-procedural governance track,
or is the current "named owner + storage standard + parent doc link" sufficient?

---

### OD-07 — Quincy RAG chat: data sensitivity and API boundary

**Phase blocked:** Phase 3
**Decision-maker:** Privacy & Data + IT Security

Quincy sends compliance document content (full text from `search-index.json`) to the
Anthropic API as part of the RAG system prompt. This requires confirmation that:

1. Compliance document content is not classified at a sensitivity level that prohibits
   transmission to third-party AI APIs
2. Anthropic's data processing terms are acceptable under Block's vendor management framework
3. The Quincy system prompt does not inadvertently include PII or legally privileged content

**Resolution needed before Phase 3 portal launch.**

---

## 14.2 Architectural Trade-offs

These decisions were made deliberately during PoC design. The rationale is recorded here
so future maintainers understand what alternatives were considered and why they were rejected.

---

### AT-01 — Markdown as document source vs. Google Docs / Word

**Chosen:** Markdown in GitHub
**Alternative:** Google Docs as primary; GitHub holds link only

**Rationale:** Markdown enables git-native version control, line-level diff (the redline
equivalent), frontmatter-as-metadata, and CI/CD validation — none of which are possible
with Google Docs. The authoring friction for non-technical policy writers is real but
manageable with templates and the portal as the read surface.

**Trade-off accepted:** Policy writers must learn basic Markdown. Google Docs remains the
authoring surface for initial drafts; the Markdown file is the authoritative version after
approval. PDF continues to be generated from Google Docs manually (see OD-05).

---

### AT-02 — Label-based approval gate vs. native GitHub required reviews

**Chosen:** Label-based gate (`approval-gate.yml` required status check + `approved-tN` label)
**Alternative:** GitHub branch protection "required reviewers" with CODEOWNERS

**Rationale:** Personal account (Phase 1) cannot require a second reviewer — branch protection
requires a second GitHub user. Label-based gate provides the equivalent enforcement without
a second account. In Phase 2, CODEOWNERS + required reviews become the primary gate and
labels become supplementary signalling.

**Trade-off accepted:** Label can theoretically be applied by any user with write access in
Phase 1. Mitigated by: (a) audit log records who applied the label, (b) CODEOWNERS closes
this in Phase 2, (c) the PoC is a simulation environment not processing real approvals.

---

### AT-03 — Inline Python scripts vs. composite Actions / reusable workflows

**Chosen:** Inline Python `python3 - <<'EOF' ... EOF` in workflow steps
**Alternative:** Separate `.py` script files under `scripts/`; composite Actions

**Rationale:** Inline scripts keep each workflow self-contained — the full logic is visible
in the workflow file without navigating to separate files. Zero external dependencies.
Easier to audit (one file = full picture of what the workflow does).

**Trade-off accepted:** Code duplication across workflows (e.g. YAML parsing logic repeated
in `validate.yml`, `publish.yml`, `route-approval.yml`). Harder to unit-test individual
functions. Acceptable for a PoC with limited maintainer surface; Phase 2 should extract
shared logic into `scripts/` as volume and complexity grow.

---

### AT-04 — JSONL flat files for audit log and exceptions vs. database

**Chosen:** Append-only `.jsonl` files committed to the repo (`audit/audit-log.jsonl`,
`exceptions/exceptions.jsonl`)
**Alternative:** External database (Postgres, Snowflake table, GitHub Projects)

**Rationale:** JSONL in git is version-controlled, human-readable, queryable with standard
Python, and requires no external infrastructure. The audit log is read by the GitHub Pages
portal at runtime via `fetch()` — no API or database connection needed. Branch protection
on `main` prevents rewriting.

**Trade-off accepted:** No native query capability — all reads are full file scans. File size
grows over time (bounded by document count × review cycle frequency — manageable for ~190
docs at annual cadence). No WORM guarantee (mitigated by branch protection + Phase 2
Snowflake export as described in Section 11).

---

### AT-05 — Single repo for all tiers vs. per-tier repos

**Chosen:** Single repo (`block-compliance-policies`) holding Tier 1, 2, and 3 documents
**Alternative:** Separate repos per tier (`block-compliance-t1`, `block-compliance-t2`, etc.)

**Rationale:** Single repo keeps the full document portfolio visible in one place, simplifies
portal data generation (`docs-data.json` from one source), and avoids cross-repo workflow
coordination. CODEOWNERS per tier path (`docs/tier-1/`, `docs/tier-2/`, `docs/tier-3/`)
enforces approval separation within the single repo.

**Trade-off accepted:** Cannot enforce document-level access control — all users with repo
read can see all documents across all tiers. Mitigated by: PDFs in Google Drive are
separately access-controlled; document metadata sensitivity is low; Phase 2 SSO gates portal
access at the org level.

---

### AT-06 — `document-registry.yaml` as separate metadata index vs. frontmatter-only

**Chosen:** Dual record — frontmatter in each `.md` file + `document-registry.yaml`
**Alternative:** Frontmatter only; workflows parse document files directly

**Rationale:** Registry provides a single fast-lookup file for workflows that need to
cross-reference multiple documents (e.g. `route-approval.yml` checking tier, `exception-triage.yml`
looking up owner, `publish.yml` resolving `stakeholder_groups`). Parsing 190 Markdown files
on every workflow run is slow and fragile.

**Trade-off accepted:** Dual-entry risk — `document-registry.yaml` and individual frontmatter
can diverge. Mitigated by `validate.yml` cross-checking key fields and the CODEOWNERS gate on
`document-registry.yaml` requiring Policy Team review on every change.

---

### AT-07 — GitHub Issues for exception tracking vs. LogicGate workflow

**Chosen:** GitHub Issues with structured Issue Forms + `exceptions.jsonl` audit record
**Alternative:** Continue using LogicGate's exception workflow; GitHub holds link only

**Rationale:** GitHub Issues provide a unified tracking surface with the rest of the DMS
workflow. Exception lifecycle automation (`exception-triage.yml`, `exception-lifecycle.yml`,
`exception-expiry.yml`) is GitHub-native. The audit trail stays in one system. Issues
support the full structured form, label-based state machine, and cron-driven expiry alerting.

**Trade-off accepted:** GitHub Issues are less structured than a purpose-built GRC exception
workflow (no mandatory field validation at submission time beyond the Issue Form required
fields, no native expiry date field). Accepted for Phase 1 PoC; Phase 2 should evaluate
whether LogicGate exception tracking should be retained in parallel until GitHub matures.

---

### AT-08 — GitHub Actions for all automation vs. dedicated orchestration tool

**Chosen:** GitHub Actions for all workflow automation (crons, event-driven, dispatch)
**Alternative:** Prefect / Airflow / AWS Step Functions for workflow orchestration

**Rationale:** GitHub Actions is already the CI/CD platform for the repo. Using it for
compliance workflow automation eliminates a separate orchestration tool, keeps all
automation visible in `.github/workflows/`, and uses the same CODEOWNERS gate as the
documents themselves. No additional vendor relationship required.

**Trade-off accepted:** GitHub Actions has operational limitations (90-day log retention,
6-hour job timeout, limited secrets management relative to HashiCorp Vault). Acceptable
for compliance document workflows given the low-frequency, low-volume nature of the events.

---

## 14.3 Deferred Scope

The following capabilities were explicitly considered and deferred beyond Phase 2.
They are documented here to prevent scope creep — these are not forgotten items,
they are conscious deferrals.

| Capability | Why deferred | Revisit trigger |
|---|---|---|
| Automated PDF generation from Markdown (`pandoc`) | Authoring workflow change management; template formatting effort | OD-05 resolved; authoring model stabilised post-cutover |
| Per-document ABAC (attribute-based access control) | Requires per-tier repos or custom auth layer; complexity outweighs benefit given PDF access control in Drive | Regulatory requirement or audit finding demanding document-level access restriction |
| Sub-procedural governance track (DMs, Desktop Procedures) | PoP does not currently define this tier; requires PoP amendment | OD-06 resolved; PoP next revision cycle |
| Goose-assisted bulk document quality scoring | Scope creep for PoC; AI quality scoring of policy content is a separate capability | Post-cutover continuous improvement initiative |
| Mobile-optimised portal | Staff access primarily on desktop; no mobile requirement identified | Portal usage analytics post-cutover |
| Multi-language document support | No current requirement across Block entities | Regulatory requirement or entity-specific need |
| Real-time LogicGate ↔ GitHub sync during transition | Complexity outweighs benefit for a bounded transition window | If transition window extends beyond 6 months |

---

## 14.4 PoC Completion Checklist

The following items close out the DocArchitect specification and the Phase 0 PoC.

- [x] Section 3: Access Control Model
- [x] Section 4: Governance Workflow Specification
- [x] Section 5: Exception Handling Specification
- [x] Section 6: Template Enforcement Specification
- [x] Section 7: GitHub Pages Portal Specification
- [x] Section 8: Awareness Campaign & Communications Specification
- [x] Section 9: Orphaned Document Management Specification
- [x] Section 10: Non-Governed Document Controls
- [x] Section 11: GitHub Capability Gap Analysis
- [x] Section 12: Legacy Archive & Migration Strategy
- [x] Section 13: Implementation Roadmap
- [x] Section 14: Open Decisions & Trade-offs
- [ ] FC-017 end-to-end governance cycle completed (Milestone 0.4)
- [ ] `LEGACY-ARCHIVE.md` populated with real Drive + LogicGate links (Milestone 0.5)
- [ ] Sections 1 & 2 (Architecture Overview + Repository Taxonomy) written
- [ ] README updated to reflect all 14 sections complete
- [ ] Go/no-go recommendation for Phase 1 prepared and presented
