# E2E 04 — The /review command family

Proves: every reply path of the platform reviewer. Run on any **fleet-member**
repo with an open PR (hello-fleet works; or the repo from E2E 02 after admission).

```bash
export REPO=enochhz/hello-fleet PR=1   # adjust to your member repo + open PR
```

| comment | expected reply |
|---|---|
| `/review help` | command table + your live quota (e.g. `2/5 used`) — costs nothing |
| `/review` | full **security** review + quota footer (consumes 1) |
| `/review perf` | performance-lens review (consumes 1) |
| `/review nonsense` | decline listing valid types — costs nothing |
| `/review` on a **non-member** repo | decline + "Reply `/register` to request membership" |
| `/register` on a non-member repo | 📬 registration PR opened (or ⏳ pending / 🔑 install-the-App / cooldown) |

Post one like this, wait ~60–90s, read:
```bash
gh pr comment $PR --repo $REPO --body "/review help"
```
```bash
gh pr view $PR --repo $REPO --comments | tail -25
```

Quota accounting check — count the hidden markers yourself (GitHub is the database):
```bash
gh api repos/$REPO/issues/$PR/comments -q '[.[] | select(.body | contains("fleet-ai-review"))] | length'
```
→ matches the "used" number in the footer. Help/declines carry a different
marker (`fleet-ai-help`) and never count.

Notes: GitHub has **no autocomplete** for third-party commands — type them as
plain comments. Only repo collaborators get replies. Quota window is a rolling
7 days; the exhausted-decline names the members.yaml governance path.
