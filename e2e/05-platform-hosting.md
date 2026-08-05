# E2E 05 — Platform hosting + the kill switch  *(admin only)*

Proves: `deployments.yaml` → live `<slug>.agents.turingplanet.ai` → teardown by
one-line diff. Needs registry write access. Uses the repo from E2E 02 (or any
member repo Railway's GitHub App can see — `enochhz/*` today).

## Deploy

1. Add the entry — **the merge/push IS the deploy trigger**:
```bash
cd /Users/haozheng/Claude_Projects/agent-军团/agent-registry && git pull -q && printf '  - slug: %s\n    repo: enochhz/%s\n    host: platform\n' "$SLUG" "$SLUG" >> deployments.yaml && git commit -qam "deploy: $SLUG" && git push
```

2. Watch the workflow (creates the Railway service + CD trigger, sets PORT/HOST,
regenerates the router table, rebuilds the router pinned to that commit):
```bash
gh run watch --repo turingplanet/agent-registry $(gh run list --repo turingplanet/agent-registry --workflow deploy-fleet --limit 1 --json databaseId -q '.[0].databaseId')
```

3. ~2–3 min later — TLS included, zero DNS work performed:
```bash
curl https://$SLUG.agents.turingplanet.ai/api/health
```
→ `{"ok":true,"agent":"<slug>"}`

4. Member CD is live too: push any change to the member repo's main → Railway
rebuilds it automatically (no registry involvement).

## Kill switch

5. Remove the three lines and push:
```bash
cd /Users/haozheng/Claude_Projects/agent-军团/agent-registry && python3 -c "
import os, pathlib
s = os.environ['SLUG']; p = pathlib.Path('deployments.yaml')
p.write_text(p.read_text().replace(f'''  - slug: {s}
    repo: enochhz/{s}
    host: platform
''', ''))" && git commit -qam "undeploy: $SLUG" && git push
```

6. After the workflow run: service deleted, route dropped:
```bash
curl https://$SLUG.agents.turingplanet.ai/
```
→ `unknown agent — see https://agents.turingplanet.ai`
