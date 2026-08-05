# E2E 02 — Scratch → fleet member

Proves: scaffold (v0.0.19+), local smoke test, the gate, **self-service
registration** (register.yml + `/register`), and `/review` after admission.

## Steps

1. Name it (used throughout):
```bash
export SLUG=e2e-scratch-test
```

2. Scaffold — answer **n** to "Adopting an EXISTING project?" (or pass the flag
as below to be safe); answer **y** to fleet registration:
```bash
cd ~/Claude_Projects && copier copy --trust --data adopt_mode=false gh:turingplanet/agent-template ./$SLUG && cd $SLUG
```

3. Local smoke test:
```bash
poetry install && poetry run pytest
```
→ `4 passed`

4. Confirm your registration intent landed in the manifest:
```bash
grep -A1 "^fleet:" agent.manifest.yaml
```
→ `register: true`

5. Push — this fires `register.yml`, which asks the registrar to open your
members.yaml PR:
```bash
git init && git add -A && git commit -m "Scaffold" && gh repo create enochhz/$SLUG --public --source . --push
```

6. Watch the registration workflow's verdict (wait ~30s first):
```bash
gh run view --repo enochhz/$SLUG $(gh run list --repo enochhz/$SLUG --workflow register --limit 1 --json databaseId -q '.[0].databaseId') --log | grep "registrar says"
```
→ `registrar says: {"status":"pr_opened","pr":"https://github.com/turingplanet/agent-registry/pull/N"}`
(or `app_not_installed` with the install link if the App doesn't cover this repo — install and push again)

7. **Admin moment**: merge that PR on agent-registry. Membership = your merge.

8. Open a PR and use the perks — note the PR template's "Platform perks" section:
```bash
git checkout -b test-pr && echo "# t" >> README.md && git add README.md && git commit -m "test" && git push -u origin test-pr && gh pr create --title "e2e" --body "x" --head test-pr --base main
```

9. The gate runs automatically (green in ~1 min). Then:
```bash
gh pr comment 1 --repo enochhz/$SLUG --body "/review"
```
Wait ~90s:
```bash
gh pr view 1 --repo enochhz/$SLUG --comments | tail -25
```
→ a real security review with the quota footer (default 2/week).

## Teardown
```bash
gh repo delete enochhz/$SLUG --yes && rm -rf ~/Claude_Projects/$SLUG
```
Then remove the member entry from `agent-registry/members.yaml` (one-line PR/commit).
