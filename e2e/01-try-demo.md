# E2E 01 — Try a live agent (install nothing)

Proves: the demo agent, wildcard domain, router, and MCP-over-HTTP all work.

## Steps

1. Plain HTTP:
```bash
curl https://agent-demo.turingplanet.ai/api/health
```
→ `{"ok":true,...}`

2. Connect your Claude to the demo agent:
```bash
claude mcp add --transport http legion-demo https://agent-demo.turingplanet.ai/mcp
```

3. In a new Claude session, ask: *"use legion-demo to say hi"* → reply carries a
**UTC** timestamp (proof it's the remote server), and *"ask legion-demo how to
join the platform"* → docs-grounded answer from `tool_how_to_join`.

4. A platform-hosted member, through the fleet router:
```bash
curl https://hello-fleet.agents.turingplanet.ai/api/health
```
→ `{"ok":true,"agent":"hello-fleet"}`

5. Unknown subdomains 404 politely:
```bash
curl https://nonexistent.agents.turingplanet.ai/
```
→ `unknown agent — see https://agents.turingplanet.ai`

## Teardown
```bash
claude mcp remove legion-demo
```
