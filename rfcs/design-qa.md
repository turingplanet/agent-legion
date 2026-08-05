# Design Q&A — questions already answered (reference before re-asking)

Settled answers from design sessions (mostly 2026-08-02/03). Each entry: the question, the answer, and the principle behind it. If a new situation seems to contradict one of these, re-read the principle first — most new questions are old questions wearing different clothes.

## Membership & registry

**Q: How does the registry know a contributor's repo?**
It's *told*, never discovers: `members.yaml` is the only roster, populated by (1) the opt-in scaffold question → `register-in-fleet.sh` opens a PR, or (2) by hand. Discovery is impossible cross-account (no GitHub API enumerates template-derived repos) and self-report-via-PR doubles as consent. *(Future: deferred auto-registration — RFC 001 §10.)*

**Q: Member is registered but the bot fails `Not Found` on their repo — why?**
**Roster ≠ keys.** `members.yaml` is knowledge; the fleet App installation is access — and an App cannot grant itself access, so only the repo owner can click it. Also check the repo still *exists*: two "Not Found" members (my-agent, my-agent3) turned out to be deleted repos still on the roster.

**Q: The member's repo owner isn't `enochhz` — extra setup needed?**
For **sync**: one self-service click — install `turing-fleet-bot` on their account, select their repo (scaffolded README documents it). For **platform hosting**: none for the member — but the platform's v1 deploy can't see foreign repos; the fix (fleet-App checkout + `railway up`, plan M6) uses the same App grant they already made.

**Q: A member updated their repo — how does the registry redeploy it?**
It doesn't, on purpose. Three flows: **code update** = member merges → Railway's GitHub integration rebuilds their service directly (registry uninvolved); **lifecycle** (join/leave hosting, slug change) = `deployments.yaml` diff, admin-merged; **template sync** = bot PR through their gate. Caveat until M6: the gate guards the PR path only — a direct push to main deploys unchecked.

## Migration (existing projects → platform)

**Q: Should migration create a new repo or adopt in place?**
**In place, on a branch, by default** — preserves history/issues/stars/secrets/identity, and the migration itself is a PR through their own gate (same trust shape as SYNC). New-repo "extraction" only for: agent-is-a-module-of-a-monorepo, or secret-laden history that must go public.

**Q: Does adoption move/rewrite my code into `api/`/`mcp_server/`?**
Never. Copier only **adds** files; there is no "correct folder" — **the manifest is the contract, folders are convention** (the gate runs whatever commands the manifest declares). Wiring your functions into MCP tools is ~10 lines of imports written by you or your AI following MIGRATE.md, verified by the gate. Additive = automated; bespoke = assisted; everything = verified.

**Q: Is bringing git/PR history along unsafe?**
The platform never receives it — the gate runs in the member's CI, members can stay private (test-agent-2 proves it). The only real risk is pre-existing secrets in history **at the moment a repo goes public**: gitleaks scan + rotate credentials (rotation, not history rewriting) as a deliberate checklist step.

**Q: What does a Java (or TS/Go) project get from migrating, given the template is Python and "policies is pytest"?**
Correction: policies isn't pytest — the gate runs the *manifest's* commands, so `mvn verify` works fine. Non-Python members get **the rails without the kit**: gate, versioned sync (review.yml bumps are language-neutral), fleet membership, deploy convention — but none of the Python scaffold code. Seam-only adoption; if demand appears, build a sibling `agent-template-<lang>`, never stretch the Python one. First non-Python member should be us (the gate has never run a non-Python manifest).

**Q: Copier or a bash setup.sh for scaffolding?**
Copier, always — provenance (`.copier-answers.yml`) and 3-way `copier update` are what the entire SYNC path stands on; bash can only create, never maintain (same flaw that retired the "Use this template" button). Bash = glue only: `detect.sh`, bootstrap one-liner, registration script.

**Q: Who pays for AI assistance during migration?**
Configurable, default self-sovereign: **own Claude** (free to platform, code stays local) · **platform advisor** (hosted MCP tool, opt-in with disclosure that code is sent to us, rate-limited) · none. Platform-paid LLM can never run in member CI — **secrets never cross accounts** (same rule as the AI-review key, Railway tokens, and registry write access).

## Deployment & hosting

**Q: Can deployment to platform Railway + automatic domain be fully automated?**
Yes — live since 2026-08-03: `deployments.yaml` is the interface; merging an entry creates the service and routes `<slug>.agents.turingplanet.ai`; removing it is the kill switch. Per-agent domains do NOT scale (per-domain TXT, unique CNAME targets, plan limit ~3) — the answer is **one wildcard domain on one edge router** (Caddy → `<slug>.railway.internal`), which also seeds Architecture 3.

**Q: Can the wildcard DNS records be automated?**
Don't — they're one-time by design (2 CNAMEs: wildcard + ACME delegation; no per-domain TXT ever). Namecheap's API is a footgun (`setHosts` replaces the whole zone; IP whitelisting). If DNS-as-code ever truly matters: migrate the zone to Cloudflare. The architecture already achieved "zero DNS per agent" — that's the automation that counts.

**Q: Whatever lands in a member repo auto-deploys?**
Pushes to **main** only, and only if the service has a **deployment trigger**. Gotcha (cost us a day of misdiagnosis): `serviceCreate(source:{repo})` sets the source but creates **no trigger**, so API-created services have silently dead CD — it is *not* a GitHub App permission problem. Fix: `deploymentTriggerCreate(provider:"github", repository, branch:"main", checkSuites:true)`; deploy-fleet.py now does this for every new service.

**Q: A PR shows "no checks reported" and the gate never runs — Actions broken?**
Check `mergeable` first: a **CONFLICTING** PR has no merge ref, and `pull_request` workflows run on the merge ref — so GitHub creates no check runs at all. Fix the conflict (merge base into the branch); checks appear immediately. Cost us an hour of ghost-hunting on 2026-08-05.

## Platform AI review (RFC 002)

**Q: Why doesn't `/review` show up in GitHub's slash-command autocomplete?**
It can't — that menu is GitHub's own commands only; third-party Apps cannot register into it. Commands are plain comment text. Hence discoverability lives in the **PR template** (ships with the scaffold), **`/review help`**, and later a sticky PR-open greeting (M6).

**Q: Should the bot post the available commands after every push?**
No — one comment per push is the classic noisy-bot failure (a 10-commit PR gets 10 identical comments and members learn to ignore the bot, burying real reviews). Post **once at PR-open** and **edit in place** (sticky comment) when it needs updating. Static hints belong in the PR template, which costs nothing at runtime.

**Q: Do declined/help replies consume review quota?**
No. They carry a different hidden marker (`fleet-ai-help` vs `fleet-ai-review`) because quota counts LLM calls, and neither runs one. This was a real bug caught during the help build.

## Site & tooling

**Q: Dark or light theme?** Dark — dev-tool audience, code-heavy content; trust is carried by verifiable claims, not background color. Docs (long-form reading), when they exist, should be light-first/theme-aware.

**Q: Mintlify or a docs platform?** Not yet — it's a landing page, not docs; self-contained beats hosted-elsewhere for the trust story. MkDocs Material / VitePress when real multi-page docs exist.

**Q: Delete the bot's auto-filed failure issues?** Close with a root-cause comment, don't delete — they're the incident log the next failure gets pattern-matched against.
