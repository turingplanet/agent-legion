# E2E 03 — Migrate an existing project (adopt in place)

Proves: `detect.sh`, `adopt_mode` (nothing moved/rewritten/deleted), MIGRATE.md
as an AI runbook, and the gate as the migration verifier.

## Steps

1. Create a fake "existing project" with its own layout:
```bash
cd ~/Claude_Projects && rm -rf my-legacy-app && mkdir -p my-legacy-app/src/services && cd my-legacy-app && git init && echo "# my-legacy-app" > README.md && printf '[project]\nname = "my-legacy-app"\ndependencies = ["fastapi"]\n' > pyproject.toml && printf 'def lookup_order(order_id: str) -> str:\n    return f"order {order_id}: shipped"\n' > src/services/orders.py && git add -A && git commit -m "existing code"
```

2. Preflight (deterministic, nothing leaves your machine):
```bash
curl -fsSL https://raw.githubusercontent.com/turingplanet/agent-template/main/detect.sh | bash
```
→ `language: python · MCP SDK present: no` → recommends **Scenario B**.

3. Adopt — answer the manifest-command questions (defaults are fine here):
```bash
copier copy --trust gh:turingplanet/agent-template . --data adopt_mode=true
```

4. **The promise check** — none of your files modified, only seam files added:
```bash
git status --short
```
→ only `??` additions (manifest, workflows, mcp_server/, config.py, railpack.json);
`README.md`, `pyproject.toml`, `src/` show **no** `M` lines.

5. Wire a tool — the intended way: open this repo in Claude Code and say
*"Follow MIGRATE.md from github.com/turingplanet/agent-template — wire
`lookup_order` from src/services/orders.py as an MCP tool."*
(Manual alternative: edit `mcp_server/server.py` per its TODOs, then
`poetry add mcp fastapi uvicorn` or your stack's equivalent.)

6. The gate is the definition of done:
```bash
git checkout -b adopt-platform && git add -A && git commit -m "Adopt platform seam" && gh repo create enochhz/my-legacy-app --public --source . --push && git push -u origin adopt-platform && gh pr create --title "Adopt platform" --body "migration e2e" --head adopt-platform --base main
```
→ gate runs **your** manifest commands; green = migrated.


## Optional: keep going — member perks + hosting

A migrated repo is a first-class member: registration (`/register` on any PR,
or `fleet.register: true` + push), then `/review`, then optional platform
hosting at `my-legacy-app.agents.turingplanet.ai` — identical to the scratch
path from here on. Follow [02-scratch-to-fleet.md](02-scratch-to-fleet.md)
steps 6–9 and its hosting section with `SLUG=my-legacy-app`.

## Teardown
```bash
gh repo delete enochhz/my-legacy-app --yes && rm -rf ~/Claude_Projects/my-legacy-app
```
