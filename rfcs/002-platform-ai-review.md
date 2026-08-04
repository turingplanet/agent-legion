# RFC 002 — Platform-paid on-demand AI security review

**Status:** draft for discussion · **Date:** 2026-08-03
**One-liner:** a member comments `/security-review` on their PR; the platform's own Claude reviews the diff for security issues and replies as a PR comment — platform-funded, quota-controlled by the registry, advisory only.

## 1. Problem

The existing AI reviewer in the gate is **member-key-funded** (`ANTHROPIC_API_KEY` in the member's repo secrets, skipped when unset) — most community members won't have a key. We want a platform-funded alternative that members can invoke on demand, with the platform controlling cost (per-repo weekly quota + global budget).

## 2. Principles applied (all pre-decided — see design-qa.md)

1. **Secrets never cross accounts** → the platform's Anthropic key and App credentials live only in a platform-side service; member CI transmits public metadata only.
2. **The gate decides, the AI advises** → the review is a PR comment, never a blocking check — even though the platform pays for it.
3. **The registry is the entitlement store** → quotas are admin-merged YAML, like `members.yaml` roster and `deployments.yaml` hosting.
4. **GitHub is the database** → usage accounting = counting the service's own marked comments; no state infrastructure.

## 3. Flow

```mermaid
flowchart TB
    C["Member comments '/security-review' on their own PR"] --> W["platform-review.yml — template-shipped,<br/>runs in MEMBER CI, sends public metadata only"]
    W --> E["Reviewer service (platform-side)<br/>POST /api/review {repo, pr, comment_id}"]
    E --> V{"Entitled?"}
    V -- "not a member · quota spent ·<br/>commenter lacks write access · diff too large" --> NO["Polite decline comment on the PR<br/>(reason + when quota resets)"]
    V -- "yes" --> R["Fetch PR diff with a fleet App token<br/>run Claude security review<br/>platform key never leaves the service"]
    R --> P["Post advisory review comment<br/>hidden marker: fleet-ai-review"]
    P --> A["Accounting = GitHub search for markers<br/>per repo, last 7 days — no database"]
```

## 4. Components

### 4.1 Trigger — `platform-review.yml` (template, ~15 lines)
`on: issue_comment (created)`; proceeds only when the comment is on a PR and starts with `/security-review`; POSTs `{repo, pr_number, comment_id}` to the reviewer endpoint; **exits 0 quietly if the endpoint doesn't exist yet** (ships before the service, same forward-compat pattern as `register.yml`). v2: retire it — the M6 webhook receiver subscribes the fleet App to comment events, so the trigger needs nothing in member repos at all.

### 4.2 Reviewer service (platform-side; shares a home with the M4 registrar)
Both endpoints hold privileged credentials and follow "receive public metadata → act platform-side" — build them as **one small platform service** (dedicated, not on legion-demo, per the registrar blast-radius decision). Validation order for `/api/review`:

1. `repo` is in `members.yaml` (registry = who's entitled at all).
2. `comment_id` really exists on that PR, body starts with `/security-review`, and the author's `author_association` is OWNER/MEMBER/COLLABORATOR — closes the "curl the endpoint to burn someone's quota" hole.
3. Quota: count marked comments in that repo over the trailing 7 days < `weekly_limit` (member-specific or platform default). Also enforce a **global weekly cap** across the fleet (platform budget backstop) — same search, unscoped.
4. Diff size cap (e.g. ≤ 3,000 changed lines) — oversized PRs get a "narrow it down" decline rather than a truncated bad review.

Then: fetch the diff via a fleet App installation token → one Claude call (pinned model + max-tokens; security-focused prompt) → post the review comment via the App with the hidden marker → done. Declines are also comments, stating the reason and when the quota resets — silent failure is banned (design-qa).

### 4.3 Quota config — `members.yaml` extension

```yaml
- name: hello-agent
  repo: enochhz/hello-agent
  ai_review:
    weekly_limit: 3     # absent or 0 = feature disabled for this repo
```

Platform default (proposed: 2/week) applies when the key is absent but the feature is globally enabled; per-member overrides are admin-merged PRs. The service needs read access to the (private) registry: grant the fleet App `agent-registry` in the turingplanet installation — config, not code, and the M4 registrar needs the same grant anyway.

## 5. Security & cost posture

- **Reviewed code is untrusted input.** The prompt must treat the diff strictly as data to analyze; instructions inside the diff are findings ("this code attempts prompt injection"), never commands. Advisory-only design bounds the blast radius: worst case is a bad comment.
- **Privacy disclosure** (same as the migration advisor, RFC 001): the command's response footer states "your diff was sent to the platform's LLM for this review; not retained."
- **Cost levers**, cheapest first: per-repo quota → global cap → diff cap → model pin. All four ship in v1.
- The platform's key never appears in any member-visible surface; App tokens are minted per-request and expire.

## 6. Relationship to the existing gate AI reviewer

Two tiers of the same principle:

| | Gate reviewer (exists) | Platform reviewer (this RFC) |
|---|---|---|
| Runs | every PR automatically | on demand (`/security-review`) |
| Pays | member (their key; skipped if none) | platform |
| Cost control | member's own | registry quota + global cap |
| Blocking | never | never |

Members with their own key keep automatic reviews; everyone gets the on-demand platform tier. Same advisory posture, so the community learns one rule: *AI comments, checks decide.*

## 7. Build plan (~1.5 sessions)

1. **Platform service** (FastAPI, scaffolded from own template; Railway in agent-fleet; secrets: `ANTHROPIC_API_KEY`, fleet App creds): `/api/review` with the checks above. *(The M4 registrar's `/api/register` joins it later.)*
2. **App grant** (config): fleet App → turingplanet installation → add `agent-registry`.
3. **Template**: `platform-review.yml` (bundle into v0.0.19 with the M2 features — one release, one fleet sync).
4. **Registry**: `ai_review` schema note in members.yaml comments; set pilot quotas.
5. **Verify**: comment on a test PR → review arrives; burn the quota → decline arrives; curl the endpoint raw → rejected.

## 8. Open questions

- Command surface: just `/security-review`, or a family (`/review`, `/review security|perf|general`)? v1: single command, security-focused — broaden on demand.
- Model: pin the cheaper tier for cost, or the top tier for quality? Proposed: mid-tier pinned, revisit with real usage.
- Should quota-exhausted members see "ask an admin to raise your limit" (PR to members.yaml) in the decline? Proposed: yes — it turns a limit into a governance touchpoint.
- Once the M6 webhook receiver exists, does the member-side trigger workflow retire immediately or stay as fallback? Proposed: retire (one less file in the template).
