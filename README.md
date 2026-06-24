# 图灵星球 Agent 军团 — Agent Legion

A platform for building and running lots of small AI agents (served over **MCP**). The whole idea in one sentence:

> **You write your agent's code in your own repo. The platform gives you the shared machinery — a starter kit, an automatic reviewer, and a place to run.**

If a word here ever feels like jargon, jump to the **[plain-language glossary](#plain-language-glossary)** at the bottom.

## 1. The big picture: three repos

There are only **three** repositories, and each has one job:

```mermaid
flowchart TB
    subgraph Platform["🏗️ Platform owns the rails — github.com/turingplanet"]
        TPL["agent-template<br/>the starter kit you copy once"]
        POL["policies<br/>the rulebook + the robot reviewer"]
    end
    subgraph You["👤 You own the cargo — your own account"]
        MR["your-agent<br/>your business logic + a manifest"]
    end
    TPL -- "COPY — once, at creation" --> MR
    MR -- "REFERENCE — live, by version (@v4)" --> POL
```

The platform team owns the first two (**the rails**); you own the third (**the cargo**).

## 2. The two ways things connect: COPY vs REFERENCE

This is the one idea that makes everything else click. Repos connect to the shared ones in **two different ways**:

```mermaid
flowchart TB
    subgraph COPY["📋 COPY — happens once, then frozen"]
        direction LR
        T["agent-template"] -- "copied in at creation,<br/>then detaches" --> Y["your repo"]
    end
    subgraph REF["🔗 REFERENCE — live, by version"]
        direction LR
        Y2["your repo<br/>uses policies@v4"] -- "follows" --> P["policies@v4"]
        P -- "platform ships @v5;<br/>you adopt with a one-line bump" --> Y2
    end
```

- **COPY** = a one-time photocopy. When you start, the **template** is photocopied into your repo. After that your copy is *yours* — if the template changes later, your copy does **not** change on its own.
- **REFERENCE** = a live link by version number. Your repo *points at* **policies** with a label like `@v4`. The platform can ship `@v5`; you pick it up by changing that one label.

**Everyday version:** COPY is like photocopying a recipe card — your copy won't update when the original does. REFERENCE is like following a magazine by issue number — you choose which issue you read, and moving to the next issue is one small step.

## 3. How you build an agent (start here)

Each agent is its **own repo that you own**, scaffolded from `agent-template` with **[Copier](https://copier.readthedocs.io)**:

```mermaid
flowchart TB
    A["You want to build an agent"] --> B["Scaffold with Copier:<br/>copier copy gh:turingplanet/agent-template ./my-agent"]
    B --> C["A new repo YOU own, with a .copier-answers.yml<br/>recording which template version it came from"]
    C --> D["Replace the placeholder code in /api + /mcp,<br/>edit agent.manifest.yaml if needed"]
    D --> E["Open a Pull Request inside your repo<br/>(your branch → your main)"]
    E --> F["The review flow runs (from policies@vN)"]
    F --> G{"🚦 Gate: did the hard checks pass?"}
    G -- "yes ✅" --> H["Merge into your main"]
    G -- "no ❌" --> D
```

**Two things worth knowing:**

- **Copier remembers where your repo came from.** Scaffolding writes a `.copier-answers.yml` that pins the template version — which is what lets you later run **`copier update`** to pull new template changes as a PR (a 3-way merge: your code is preserved, conflicts come out as markers to resolve).
- **"Open a PR" = a PR inside *your own* repo.** You don't save changes straight to your `main`. You make a branch, then **open a pull request** (your branch → your `main`). Opening it starts the automatic review, and the gate decides whether it merges. It all happens in your repo — nothing is sent to the platform. (The same "review before you merge" habit you'd use with a human reviewer.)

## 4. What happens when you open a PR

Opening a PR runs the **review flow** — the steps come from `policies`, live:

```mermaid
flowchart TB
    PR["You open a Pull Request"] --> M["1. Read the manifest<br/>(your repo's instruction card)"]
    M --> S["2. Set up declared tools<br/>(e.g. Python + poetry)"]
    S --> CH["3. Run the HARD checks:<br/>install · tests · lint · security"]
    AI["🤖 AI reviewer<br/>reads your diff, writes advice"] -. "comments only — never blocks" .-> G
    CH --> G{"🚦 4. Gate:<br/>did the hard checks pass?"}
    G -- "yes ✅" --> OK["PR can merge"]
    G -- "no ❌" --> BACK["Sent back with the findings"]
```

The important part: **the gate decides, the AI only advises.** The automated checks (install, tests, lint, security) are the real pass/fail. The 🤖 AI reviewer reads your change and leaves comments, but it can *never* block your PR on its own. Green checks = you can merge.

## 5. How platform improvements reach you

Because your repo **references** `policies` by version, rolling out an improvement is simple:

```mermaid
flowchart LR
    DEV["Platform improves a<br/>check or the reviewer"] --> PUB["Publishes policies@v5<br/>(v1…v4 stay frozen)"]
    PUB --> ADOPT["Each member adopts with a<br/>one-line bump: @v4 → @v5"]
    ADOPT --> MA["member A ✓"]
    ADOPT --> MB["member B ✓"]
    ADOPT --> MC["member C ✓"]
```

Old versions (`v1…v4`) stay frozen forever, so nothing breaks under you. You upgrade when you're ready by bumping one line. Template changes (the **COPY** side) are handled in the same spirit, automatically: the [agent-registry](https://github.com/turingplanet/agent-registry) **migration bot** runs `copier update` across the fleet and opens a PR per repo — never auto-merging, so each change still passes through your gate.

## Plain-language glossary

| You'll see… | It means… |
| --- | --- |
| **the platform / the rails** | the shared system everyone builds on (run by `turingplanet`) |
| **contributor / member** | you, building one agent on the system |
| **manifest** (`agent.manifest.yaml`) | an index card on your repo telling the system how to build & test it |
| **the anchor** | the fixed name + location of that card, so every tool can find it |
| **COPY** | photocopy the starter kit once; your copy then goes its own way |
| **REFERENCE** | link to the shared rules by version; bump the version to get updates |
| **`agent-template`** | the Copier template you scaffold a new agent from |
| **`policies`** | the shared rulebook + robot reviewer your repo links to |
| **member repo** | your own repo, with your code |
| **PR** (pull request) | a request to merge a change; checks run before it can merge |
| **the gate** | the automatic pass/fail (the tests are the judge; the AI just advises) |
| **`@v4`** | which frozen version of the shared rules your repo follows |

## Where to go next

- **Want the rules / the reviewer?** → [policies](https://github.com/turingplanet/policies)
- **Want the starter kit?** → [agent-template](https://github.com/turingplanet/agent-template)
- **Want a worked example?** → [hello-agent](https://github.com/enochhz/hello-agent)
- **The fleet inventory + auto-migration bot?** → [agent-registry](https://github.com/turingplanet/agent-registry)

## Status

Architecture 1 (build → review → gate) works end-to-end today: a PR runs the full flow and the gate decides. Architecture 2 (release / deploy) and Architecture 3 (the MCP gateway) are next.

## Full design

The complete architecture design lives in Notion: _<add your Notion share link here>_.
