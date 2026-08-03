# RFC 001 — Migration wizard & platform deploy (Architecture 2)

**Status:** draft for discussion · **Author:** Hao + Claude · **Date:** 2026-08-02
**Implements:** onboarding for existing projects (migration/adoption) + post-merge deployment (the roadmap's Architecture 2).

---

## 1. Problem & goals

Community members fall into two groups: those with **no repo** (served today by `copier copy`) and those with an **existing project** they want to bring into the fleet. For the second group we need a migration path that:

- never destroys or silently rewrites their code,
- works across repo shapes (full-stack, API-only, MCP-already, non-Python),
- gives optional AI assistance — configurable between **their own LLM** (free to us, code stays local) and a **platform-paid LLM** (opt-in, we pay),
- ends in the same place as every member: gate-green, optionally registered, optionally deployed,
- and, for deployment, lets members choose **their own Railway** or **platform-hosted Railway with a `*.agents.turingplanet.ai` subdomain** — where the **registry controls who may use platform assets**.

## 2. Principles this design inherits (do not re-litigate)

1. **The manifest is the contract, not folder layout.** The gate runs whatever `agent.manifest.yaml` declares; nothing needs relocating.
2. **The gate decides, the AI advises.** Detection and validation are deterministic; LLMs only assist.
3. **Additive is automated; bespoke is assisted; everything is verified.** Copier adds files, never rewrites code. Wiring functions into tools is judgment — done by the member (or their AI following `MIGRATE.md`), verified by the gate.
4. **Secrets never cross accounts.** Platform API keys and Railway tokens cannot live in member repos. Anything platform-paid must run platform-side (hosted advisor, registry-driven deploys).
5. **Provenance is a file.** `.copier-answers.yml` makes migrated repos indistinguishable from scaffolded ones — the bot needs no changes.
6. **Every authorization is a PR.** Fleet membership (`members.yaml`) and platform hosting (`deployments.yaml`) are granted by admin-merged PRs, revoked by one-line diffs.

## 3. Scenarios

| # | You have | The move | You get | Notes |
|---|---|---|---|---|
| 0 | Nothing — starting fresh | `copier copy` scaffold | Runnable agent in minutes | Today's quickstart, unchanged |
| A | Full-stack repo (frontend + backend) | Adopt in place scoped by manifest, **or** extract agent module to new repo | Backend gains `/mcp`; frontend untouched | Extract when the agent is a *component* of a product, or when secret-laden history must go public |
| B | API-only (Python) | `adopt_mode` scaffold; wire funcs as tools | Same app, two doors: `/api` kept, `/mcp` added | The mainstream case |
| C | Own MCP server (Python) | Seam-only adoption (manifest + review.yml + provenance) | Instant member, zero rewrite | Must rename any folder literally named `mcp/` (shadows the SDK) |
| D | MCP in TS / Go / Java / … | Seam-only; manifest declares their stack's commands | Rails without the kit: gate + sync + fleet | Rails are language-agnostic; scaffold is Python-first. First non-Python member should be us (test the gate!) |

**Key asymmetry:** the more MCP-native a repo already is, the *fewer* files adoption adds. Scratch is just the degenerate case that skips preflight entirely.

## 4. Master flow

```mermaid
flowchart TB
    A["Existing repo<br/>any shape · any language"] --> P["Preflight — detect.sh<br/>stack · MCP deps · secrets scan (gitleaks)<br/>prints scenario + recommended path"]
    Z["No repo yet — start fresh"] --> S["Scaffold new<br/>copier copy gh:turingplanet/agent-template ./my-agent"]
    P --> D1{"Agent = whole repo?"}
    D1 -- "yes" --> IP["Adopt in place<br/>copier adopt_mode — adds files, deletes nothing"]
    D1 -- "no — agent is a module, or<br/>dirty history going public" --> EX["Extract to new repo<br/>fresh scaffold + copy agent module<br/>original repo untouched"]
    IP --> C["Config: AI assist<br/>own Claude (default) · platform advisor (opt-in, we pay) · none"]
    EX --> C
    C --> W["Write / wire your MCP tools<br/>MIGRATE.md for migrants · placeholders for fresh starts"]
    S --> W
    W --> G["Gate runs on your PR<br/>manifest · install · tests · security"]
    G --> D2{"Gate green?"}
    D2 -- "no — fix & push" --> W
    D2 -- "yes" --> M["Member agent<br/>register + deploy optional"]
```

Notes:

- **Preflight is deterministic** (`detect.sh`, ships in the template): greps for MCP deps, frontend frameworks, language, an `mcp/` dir; runs `gitleaks detect`. Prints the scenario, recommended destination, and a security summary — free, local, no LLM.
- **Both destination paths are non-destructive.** In-place adoption only *adds* files (copier collisions surface as conflict markers, resolved by the member). "Extract" copies code *out*; the original repo is never modified.
- **The AI-assist knob** (`own | platform | none`): "platform advisor" cannot run in member CI (principle 4) — it is a hosted MCP tool on a platform agent, opt-in with explicit disclosure that code is sent to us, rate-limited. Default is the member's own AI: `MIGRATE.md` is written to be handed to Claude Code as a runbook.
- **Migration itself is a PR through the member's own gate** — same trust shape as SYNC; reviewable, revertible, nothing new to trust.
- **Going-public checklist:** history is only a risk at the moment a private repo flips public. `detect.sh`'s gitleaks report + "rotate, don't rewrite" guidance gates that step.

## 5. Deploy phase (Architecture 2)

```mermaid
flowchart TB
    M["Merged — gate green"] --> D{"Deploy where?"}
    D -- "own Railway" --> OWN["Connect repo in your Railway<br/>CD built in: merge = deploy"]
    OWN --> LO["Live on your account<br/>your-app.up.railway.app<br/>you pay · you control"]
    D -- "platform Railway" --> REQ["Request platform hosting<br/>PR adds entry to deployments.yaml<br/>registry = the allowlist"]
    REQ --> AP{"Admin approves?"}
    AP -- "no" --> OWN
    AP -- "yes" --> DEP["Registry deploys<br/>Railway API from platform CI<br/>gate-green main only"]
    DEP --> LP["Live on platform<br/>slug.agents.turingplanet.ai<br/>platform pays · kill switch stays"]
```

### 5.1 Why the platform lane is a *pull*, not a push

Platform Railway tokens cannot be given to member repos (principle 4 — a member could exfiltrate the token and deploy anything on our bill). So platform deploys run **platform-side, in agent-registry CI**, where the token lives as a repo secret and never leaves. The registry *pulls* approved repos and deploys them — the same shape as the migration bot, pointed at Railway.

### 5.2 `deployments.yaml` — the allowlist is the infrastructure

```yaml
# agent-registry/deployments.yaml
- repo: enochhz/hello-agent
  host: platform            # platform | dns-only
  domain: hello.agents.turingplanet.ai
  limits: { memory: 512MB, replicas: 1 }
```

- Requesting hosting = a PR adding your entry; **admin merge = authorization**. No entry, no deploy.
- The deploy workflow reads this file, checks out each repo's **main at a gate-green commit only**, and drives the Railway API. Declarative YAML + API calls = enough IaC for v1; Terraform can come later if state management ever warrants it.
- Removing an entry undeploys: a **one-line-diff kill switch**.
- `host: dns-only` grants a branded subdomain to a *self-hosted* member (CNAME to their Railway) — brand + discoverability at zero compute cost, still registry-controlled.

### 5.3 Domains

One wildcard CNAME in Namecheap (`*.agents.turingplanet.ai` → Railway), added once; the deploy workflow attaches `slug.agents.turingplanet.ai` per service via API.

### 5.4 CD triggers

- **Own lane:** already exists — Railway's GitHub integration redeploys on every push to main. Document, don't build.
- **Platform lane v1:** cron + manual `workflow_dispatch` (same rhythm as the migration bot). Fast-follow: fleet GitHub App push-webhooks → instant deploys.

### 5.5 Risks

Platform hosting runs **member code on our account**: egress wears our name, costs are ours. Mitigations: approval is a genuine human trust decision, per-service resource limits are mandatory in the schema, start with known members, kill switch stays one diff away.

## 6. Configurable knobs (all recorded in `.copier-answers.yml` / registry files)

| Knob | Values | Default | Who controls |
|---|---|---|---|
| `destination` | in-place · new-repo | recommended by `detect.sh` | member |
| `ai_assist` | own · platform · none | own (code stays local) | member; platform side rate-limited |
| `deploy` | own · platform · dns-only · none | own | member requests; **registry approves** |

Every default is the self-sovereign option; every platform option is granted by a registry PR.

## 7. Tooling decision: Copier stays the engine

`setup.sh` does **not** replace Copier — a bash scaffold has no provenance (`.copier-answers.yml`) and no 3-way-merge update path, the exact flaws that retired the "Use this template" button. Roles:

- **Copier** — scaffold + update engine (questions, conditional `adopt_mode` generation, Jinja, `copier update`).
- **bash** — glue only: `detect.sh`, `register-in-fleet.sh`, and the bootstrap one-liner
  `curl -fsSL https://agents.turingplanet.ai/setup.sh | bash`
  (checks prereqs, installs copier via pipx, runs scaffold — or `detect.sh` when run inside an existing repo).
- **`MIGRATE.md`** — the AI-runnable runbook (scenario decision table up top).
- **The existing gate** — verification. No new validator.

## 8. Components to build, in order

1. **agent-template v0.0.18** — `adopt_mode` (conditional generation) · destination/AI-assist copier questions · `detect.sh` · `MIGRATE.md` · `fleet.register` manifest field + `register.yml` push-workflow (§10; the `github_repo` copier question is deleted) · the pending `railpack.json` startCommand fix (Railpack's poetry runtime broke; legion-demo carries the fix, the template doesn't yet).
2. **`setup.sh`** hosted on the site (bootstrap + detect).
3. **Registrar endpoint** on a platform-hosted service (§10) — receives `{repo}` from member CI, opens the members.yaml PR platform-side. First version can piggyback on legion-demo's FastAPI.
4. **agent-registry: `deployments.yaml` + deploy workflow** (Railway API, wildcard domain attach, cron + dispatch). *First registry change in this design.*
5. **Platform advisor** MCP tool on a platform-hosted agent — only if members actually ask for platform-paid assist; it has ongoing cost + abuse surface.
6. **Nothing in policies.**

## 9. Registry control plane — one release, N update PRs

How the registry keeps every registered repo current: membership is **declared, never discovered** (`members.yaml` is the single roster; entries arrive only as admin-merged PRs — see §10 for how they get opened). When the platform ships a template or policies release, one bot run fans out to the whole fleet:

```mermaid
flowchart TB
    REL["Platform ships a release<br/>agent-template vN+1 — or policies vN+1"] --> RUN["Bot run — agent-registry CI<br/>cron Mon 06:00 UTC · or manual dispatch"]
    RUN --> ROSTER["Read members.yaml — the roster"]
    ROSTER --> M1["mint App token for alice<br/>copier update → PR on alice/agent-a"]
    ROSTER --> M2["mint App token for bob<br/>copier update → PR on bob/agent-b"]
    ROSTER --> M3["… one PR per registered repo"]
    M1 --> G1["alice's own gate runs<br/>her pinned policies@vN"]
    M2 --> G2["bob's own gate runs"]
    G1 --> A1["alice reviews + merges — or ignores"]
    G2 --> A2["bob reviews + merges — or ignores"]
```

Properties of the control plane:

- **One release → one bot run → N PRs.** The bot mints a short-lived GitHub App installation token *per member account* at runtime; no long-lived credentials exist.
- **Never auto-merges.** Every PR passes through that member's own pinned gate; merging (or ignoring) is the member's call. A member on `policies@v5` stays on v5 until *they* merge the bump.
- **Two layers, both required:** `members.yaml` = the roster (knowledge) · fleet App installation on the repo = the keys (access). A registered repo without App access is skipped — this is a live failure mode today and the motivation for §10's v2.
- **`deployments.yaml` (§5) rides the same plane:** platform-hosted members get redeployed from gate-green main after they merge — same roster model, same admin-merge authorization, same kill-switch-by-diff.

## 10. Deferred auto-registration

Today registration fires at scaffold time and asks for the GitHub repo name **before the repo exists** (the script guesses the owner via `gh api user`). This section defers execution to the moment a real repo appears — while keeping consent explicit.

**Design:**

- **The copier question stays, as an explicit yes/no** — registration is always a conscious choice at scaffold, never a silent default. The answer is recorded in `agent.manifest.yaml` as `fleet: { register: true | false }` — visible and auditable in the member's own repo (the manifest is the seam every tool reads). Flip it anytime; the next push acts on the new value.
- **Local scaffold does nothing.** No repo, no registration activity, nothing to worry about.
- **First push executes the intent.** A small `register.yml` workflow (`on: push` to main) reads the flag; the repo address needs no detection at all — `github.repository` is `owner/name`, CI knows where it's running.
- **The write happens platform-side.** Member CI cannot open PRs on the (private) registry — the default `GITHUB_TOKEN` is repo-scoped, and secrets never cross accounts. So member CI sends only public metadata (`{repo}`) to a **registrar endpoint** on a platform-hosted service, which validates the repo's manifest actually says `register: true`, then opens the members.yaml PR itself with platform credentials that never leave home. Idempotent (checks roster + open PRs), rate-limited, and **inert until the admin merges** — spam costs a PR-close click, never a fleet entry.
- `register-in-fleet.sh` survives only as a manual fallback.

```mermaid
flowchart TB
    Q["Scaffold — copier asks explicitly:<br/>register this agent in the fleet? yes / no"] --> F["Answer recorded in agent.manifest.yaml<br/>fleet.register: true | false"]
    F --> LOCAL["Local development<br/>no repo yet — nothing fires"]
    LOCAL --> PUSH["First push to GitHub<br/>register.yml reads the flag +<br/>github.repository (auto-detected)"]
    PUSH -- "fleet.register: false" --> OFF["No-op<br/>flip the flag later → next push registers"]
    PUSH -- "fleet.register: true" --> REG["Registrar — platform-side service<br/>validates manifest · idempotent · rate-limited"]
    REG --> PR["Opens PR on agent-registry<br/>adding repo to members.yaml"]
    PR --> ADM{"Admin merges?"}
    ADM -- "no" --> OUT["Closed — not in fleet"]
    ADM -- "yes" --> IN["Registered — on the roster"]
    IN --> APP["Member grants fleet App access<br/>one browser click — the keys"]
    APP --> DONE["Fully synced member<br/>bot update PRs now reach this repo"]
```

**v2 — collapse the last two steps:** GitHub Apps emit an `installation` webhook when added to a repo. "Register" could simply *be* "install the fleet App": one browser click grants access **and** signals intent; the webhook hits the registrar, which checks the manifest flag and opens the PR. Roster and keys in one action — the §9 skipped-member failure mode disappears. Needs an always-on webhook receiver, so it's a fast-follow, not v1.

## 11. Open questions

- Extract path: auto-create the new GitHub repo (`gh` magic) or generate locally and let the member push? **Recommendation: manual push for v1** — consistent with "you own every outward action."
- Non-Python: when demand appears, sibling `agent-template-<lang>` sharing the same manifest schema — never stretch the Python template.
- Platform-lane instant CD + App-install-as-registration (§10 v2) share the same webhook receiver — build once, serve both.
- Advisor privacy wording ("code sent to us, not stored") — needs exact copy before launch.
- Registrar spam posture: is PR-close-on-junk enough, or should the endpoint require the caller to prove repo control (e.g. a challenge file)? v1 ships simple; revisit if abused.
