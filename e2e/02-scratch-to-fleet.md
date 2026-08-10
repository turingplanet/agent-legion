# E2E 02 — Scratch → fleet member

Proves: scaffold (v0.0.19+), local smoke test, the gate, **self-service
registration** (register.yml + `/register`), and `/review` after admission.

## Who does what (jump to any command)

| stage | what happens | 🧑‍🚀 member (agent owner) | 🛰️ platform admin |
|---|---|---|---|
| 1 · Scaffold | generate the agent, test locally | [steps 1–3](#step-1) | — |
| 2 · Declare intent | registration answer recorded in the manifest | [step 4](#step-4) | — |
| 3 · Publish | first push to main fires registration automatically | [step 5](#step-5) | — |
| 4 · Verdict | read the registrar's outcome (PR opened / App missing / …) | [step 6](#step-6) | — |
| 5 · **Admission** | members.yaml PR merged — membership granted | wait, or nudge with `/register` on any PR | [step 7](#step-7) — **merge = the decision** |
| 6 · Ship code | branch → PR → the gate decides | [step 8](#step-8) | — |
| 7 · AI review | `/review` posts a platform-paid review | [step 9](#step-9) | quota via members.yaml |
| 8 · Stay current | template updates arrive as PRs — never auto-merged | [merge the bot's PR, or self-update](#optional-staying-up-to-date-two-paths-same-destination) | — (cuts releases; the bot opens PRs) |
| 9 · Hosting *(optional)* | live at `slug.agents.turingplanet.ai` | ask the admin (self-service `/deploy` is a future item) | [hosting section](#optional-platform-hosting-admin-gated) |
| 10 · Teardown | leave the fleet / delete the repo — one script | [`scripts/teardown.sh`](#teardown) | **merge the removal PR** ([teardown](#teardown)) |

The split is the platform's core deal: **members act, admins admit (and release)** — every
admin cell is a PR merge, never a manual setup task. Entry and exit are symmetric:
manifest intent → platform-authored PR → admin merge.

## Steps

<a id="step-1"></a>
1. Name it (used throughout):
```bash
export SLUG=e2e-scratch-test
```

<a id="step-2"></a>
2. Scaffold — answer **n** to "Adopting an EXISTING project?" (or pass the flag
as below to be safe); answer **y** to fleet registration:
```bash
cd ~/Claude_Projects && copier copy --trust --data adopt_mode=false gh:turingplanet/agent-template ./$SLUG && cd $SLUG
```

<a id="step-3"></a>
3. Local smoke test:
```bash
poetry install && poetry run pytest
```
→ `4 passed`

<a id="step-4"></a>
4. Confirm your registration intent landed in the manifest:
```bash
grep -A1 "^fleet:" agent.manifest.yaml
```
→ `register: true`

<a id="step-5"></a>
5. Push — this fires `register.yml`, which asks the registrar to open your
members.yaml PR:
```bash
git init && git add -A && git commit -m "Scaffold" && gh repo create enochhz/$SLUG --public --source . --push
```

<a id="step-6"></a>
6. Watch the registration workflow's verdict (wait ~30s first). Guard against
an unset `$SLUG` (new terminal?) — the composed command fails silently otherwise:
```bash
test -n "$SLUG" || echo "SLUG is unset — export SLUG=<your-repo-name> first"
```
```bash
RID=$(gh run list --repo enochhz/$SLUG --workflow register --limit 1 --json databaseId -q '.[0].databaseId') && gh run view --repo enochhz/$SLUG "$RID" --log | grep "registrar says" || echo "no verdict line — inspect the full log: gh run view --repo enochhz/$SLUG $RID --log"
```
→ `registrar says: {"status":"pr_opened","pr":"https://github.com/turingplanet/agent-registry/pull/N"}`
(or `app_not_installed` with the install link if the App doesn't cover this repo — install and push again)

> **Behind the scenes:** that PR is opened **on agent-registry (the platform's
> repo), not on yours** — authored by the platform with its own credentials,
> because your CI can't touch the private registry (secrets never cross
> accounts). Your repo only sent its name; the registrar verified the App
> install (keys) and your manifest's `fleet.register: true` (consent) before
> writing anything. The PR is **inert until an admin merges it** — no command
> or push can make you a member; only the merge can. List pending ones:
> ```bash
> gh pr list --repo turingplanet/agent-registry
> ```

<a id="step-7"></a>
7. **Admin moment**: merge that PR on agent-registry. Membership = your merge.

<a id="step-8"></a>
8. Open a PR and use the perks — note the PR template's "Platform perks" section:
```bash
git checkout -b test-pr && echo "# t" >> README.md && git add README.md && git commit -m "test" && git push -u origin test-pr && gh pr create --title "e2e" --body "x" --head test-pr --base main
```

<a id="step-9"></a>
9. The gate runs automatically (green in ~1 min). Then:
```bash
gh pr comment 1 --repo enochhz/$SLUG --body "/review"
```
Wait ~90s:
```bash
gh pr view 1 --repo enochhz/$SLUG --comments | tail -25
```
→ a real security review with the quota footer (default 2/week).


## Optional: staying up to date (two paths, same destination)

Template improvements reach your repo **only ever as a PR you merge** — the
platform's automation has no merge step, by founding design. Pick either path:

**Path A — the platform updates you (do nothing):** when a new template version
ships, the fleet bot opens a version-scoped PR on your repo
(`chore/template-sync-vX.Y.Z`). Your own gate runs on it; you review and merge —
in the GitHub UI, or:
```bash
gh pr list                       # see the waiting sync PR
```
```bash
gh pr merge <N> --squash --delete-branch
```
The bot never force-pushes over your manual commits on that branch, never
auto-merges, and a new version always gets a fresh PR.

**Path B — update yourself (don't wait for the bot):**
```bash
git switch -c chore/manual-sync && copier update --defaults --trust --conflict inline
```
```bash
git add -A && git commit -m "chore: sync to the latest agent-template" && git push -u origin chore/manual-sync && gh pr create --fill
```
Same 3-way merge the bot performs (your code preserved, conflicts as `<<<<<<<`
markers), through the same gate, merged by the same person: you. The bot's next
run will see you're current and skip you.

> **The invariant, explicitly:** nothing the platform runs can land commits on
> your `main`. Every change arrives as a PR; merging is yours alone. Want that
> structurally enforced too? Add branch protection on `main` in your repo
> settings — the platform works identically either way.

## Optional: platform hosting (admin-gated)

Once admitted, your agent can be hosted at `$SLUG.agents.turingplanet.ai`.
The request is an entry in the registry's `deployments.yaml` — **only an admin
can merge it** (today the admin adds it directly; a self-service `/deploy`
command is a future item). If you ARE the admin:

```bash
cd /Users/haozheng/Claude_Projects/agent-军团/agent-registry && git pull -q && printf '  - slug: %s\n    repo: enochhz/%s\n    host: platform\n' "$SLUG" "$SLUG" >> deployments.yaml && git commit -qam "deploy: $SLUG" && git push
```

~2–3 min later (the push IS the deploy trigger — service created, routed, TLS included), verify from your agent repo — one command checks REST, does a real MCP handshake, and lists your tools (ships in the scaffold, template ≥ v0.0.22):

```bash
./scripts/test_platform_mcp.sh
```
→ `✅ /api/health` · `✅ /mcp handshake OK` · `✅ tools/list → tool_say_hi, tool_run` + a ready-to-paste `claude mcp add` line. (Manual fallback: `curl https://$SLUG.agents.turingplanet.ai/api/health` → `{"ok":true,"agent":"<slug>"}`.) Pushes to your repo's main now auto-redeploy.

Full walkthrough incl. the kill switch: [05-platform-hosting.md](05-platform-hosting.md).

## Teardown

**One script, mirroring the join** (ships with every v0.0.21+ scaffold):

Interactive — two questions (delete repo entirely? / leave the fleet?):
```bash
bash scripts/teardown.sh
```
→ flips `fleet.register: false` (your consent — the platform verifies it), pushes,
and prints `registrar says: {"status":"pr_opened",...}` with the **removal PR** link.
That one PR removes your members.yaml entry AND any platform-hosting entry.

**Admin moment (the mirror of admission):** merge the removal PR — membership
ends, and if you were platform-hosted, the deploy reconciler tears the service
and route down on that same merge. Closing the PR = you stay.

Non-interactive, for automation:
```bash
bash scripts/teardown.sh --unregister          # leave the fleet, keep the repo
```
```bash
bash scripts/teardown.sh --delete-repo --yes   # leave AND delete the GitHub repo
```
Repo deletion needs the scope once: `gh auth refresh -h github.com -s delete_repo`.
The script is idempotent — rerunning reports the current state (`pr_pending`,
`not_registered`) instead of duplicating anything. Local folder is never touched.

<details>
<summary>Manual fallback (pre-v0.0.21 scaffolds)</summary>

```bash
gh repo delete enochhz/$SLUG --yes && rm -rf ~/Claude_Projects/$SLUG
```
Then ask the admin to remove the member entry from `agent-registry/members.yaml`
(and `deployments.yaml` if hosted) — or run `copier update --trust` first to get
the script.
</details>
