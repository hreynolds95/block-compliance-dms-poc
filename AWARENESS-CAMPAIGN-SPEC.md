# Section 8 — Awareness Campaign & Communications Specification
## Block Compliance DMS | DocArchitect Architecture

> **PoP references:** Section 2.9 (Awareness Campaign),
> Section 2.4 (Two-Role Model — Document Owner defines stakeholder groups),
> Section 2.6.3 (Exception expiry notifications covered in Section 5)

---

## 8.1 Overview

The awareness campaign ensures that when a compliance document is published or materially changed, the right stakeholders are notified automatically — without the Policy Team manually tracking distribution lists. It also provides the Policy Team with a monthly digest of portfolio health.

Two automation components implement this:

| Component | Trigger | Workflow | Channels |
|---|---|---|---|
| **Publication notification** | Document merged to `main` | `publish.yml` (new step) | Per-doc `stakeholder_groups` from `document-registry.yaml` |
| **Monthly policy digest** | 1st of each month, 10:00 ET | `monthly-digest.yml` | `#compliance-policy-help` + GitHub Issue |

Both components use the same `SLACK_WEBHOOK_URL` repository secret. If the secret is not configured (Phase 1), the notification payload is logged to the Actions run and a GitHub Issue is opened instead.

---

## 8.2 Stakeholder Groups Configuration

Each document in `document-registry.yaml` carries a `stakeholder_groups` list — Slack channel names that should be notified on publication or material change. The Document Owner is responsible for keeping this list accurate.

```yaml
# document-registry.yaml
- doc_id: GOV-001
  stakeholder_groups:
    - "#compliance-all-hands"
    - "#compliance-policy-help"

- doc_id: FC-017
  stakeholder_groups:
    - "#financial-crimes"
    - "#compliance-sel"
```

**Deduplication:** If a single PR publishes multiple documents that share a channel (e.g. both notify `#compliance-all-hands`), one consolidated message is sent to that channel listing all published documents.

**Phase 2 extension:** `stakeholder_groups` may also include email distribution list identifiers (e.g. `compliance-all@block.xyz`). The notification step routes by prefix: `#` → Slack, `@` → email.

---

## 8.3 Publication Notification (`publish.yml` — new step)

Added as the final step in `publish.yml`, after the frontmatter commit. Fires for every doc merged to `main` under `docs/**/*.md`.

### Message content (per channel)

```
📄 *Compliance Document Published*

The following document(s) were published to the Block Compliance library:

• *{title}* ({doc_id})
  Tier {tier} · {domain} · {doc_type}
  Change type: {change_type} · Owner: {owner}
  Effective: {effective_date}
  {pdf_link}

View the full library: https://hreynolds95.github.io/block-compliance-policies/
```

If multiple docs are published to the same channel in one merge, they are listed as bullet points in a single message.

### Workflow behaviour

```
publish.yml fires (on push to main with docs/** changes)
        │
        ├─ Step 1: Update frontmatter + write audit log (existing)
        ├─ Step 2: Commit frontmatter + audit log (existing)
        └─ Step 3: Send publication notifications (new)
                │
                ├─ Re-reads changed docs (HEAD~1..HEAD)
                ├─ Looks up stakeholder_groups in document-registry.yaml
                ├─ Deduplicates channels across multiple docs
                ├─ Builds consolidated message per channel
                │
                ├─ SLACK_WEBHOOK_URL configured?
                │     Yes → POST to Slack webhook
                │     No  → Log payload to Actions run (Phase 1 simulation)
                │
                └─ Logs: "Would notify: #channel-name (N doc(s))"
```

---

## 8.4 Monthly Policy Digest (`monthly-digest.yml`)

Runs on the 1st of each month at 10:00 ET. Provides the Policy Team with a structured summary of portfolio health.

### Digest sections

| Section | Source | Logic |
|---|---|---|
| Published last 30 days | `audit/audit-log.jsonl` | Entries with `timestamp` within last 30 days |
| Coming due (30 days) | `document-registry.yaml` | `next_review_date` between today and today+30 |
| Overdue | `document-registry.yaml` | `next_review_date` < today (published docs only) |
| Open exceptions | GitHub Issues API | Open issues with `exception-request` label |
| Portfolio summary | `document-registry.yaml` | Count by status |

### Delivery

| Channel | Phase 1 | Phase 2 |
|---|---|---|
| Slack `#compliance-policy-help` | Logged to Actions run (no webhook) | POSTed to Slack via `SLACK_WEBHOOK_URL` |
| GitHub Issue | Always opened — title: `[Monthly Digest] Compliance Policy Portfolio — {Month YYYY}` | Same |

The GitHub Issue is opened regardless of whether Slack is configured, so the digest is always accessible in the repo's issue history. It is assigned to `@hreynolds95` in Phase 1.

### Issue deduplication

Before opening a new digest Issue, the workflow checks for an existing open Issue with `monthly-digest` label and the current month in the title. If one exists, it is skipped.

---

## 8.5 Slack Webhook Configuration

### Phase 1 (PoC)

No `SLACK_WEBHOOK_URL` secret is configured. Both workflows detect the missing secret and fall back gracefully:
- Log the full payload they would have sent
- `monthly-digest.yml` still opens the GitHub Issue

### Phase 2 (production)

1. Create an incoming webhook in the Block Slack workspace for `#compliance-policy-help`
2. Add as a repository secret: `Settings → Secrets → Actions → SLACK_WEBHOOK_URL`
3. Both `publish.yml` and `monthly-digest.yml` activate automatically — no code changes required

**Multiple channel support (Phase 2):** For publication notifications, the `SLACK_WEBHOOK_URL` secret is a general-purpose webhook. To post to different channels (e.g. `#financial-crimes` vs `#compliance-all-hands`), Phase 2 should use the Slack API with a bot token (`SLACK_BOT_TOKEN`) and a `chat.postMessage` call specifying the `channel` parameter, rather than a single fixed-channel webhook.

---

## 8.6 Phase 2 Enhancements

| Enhancement | PoP Section | Description |
|---|---|---|
| Per-channel Slack routing | 2.9 | Replace single webhook with Slack bot token + `chat.postMessage`; routes each notification to the correct `stakeholder_groups` channel |
| Email notifications | 2.9 | Extend `stakeholder_groups` to support email DLs; send via SendGrid or equivalent on publication |
| Material change flag | 2.11.1 | Notification message highlights `change_type: material` with prominent indicator |
| Digest Slack formatting | 2.9 | Replace plain text digest with Slack Block Kit rich formatting |
| Digest email to Policy Team | 2.9 | Monthly digest also emailed to Compliance Policy distribution list |
| On-demand awareness blast | 2.9 | `workflow_dispatch` input to re-send notification for a specific `doc_id` (for late joiner distribution) |
