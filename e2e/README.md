# E2E walkthroughs — test the platform from a terminal

One markdown per scenario. Each is **copy-paste, top to bottom**, with expected
outputs and a teardown. They double as the platform's manual QA scripts — run
the relevant one after changing anything in its path.

| # | scenario | proves | needs | ~time |
|---|---|---|---|---|
| [01](01-try-demo.md) | Try a live agent | the demo + platform hosting chain | nothing (any machine with `claude`/`curl`) | 2 min |
| [02](02-scratch-to-fleet.md) | Scratch → fleet member | scaffold, gate, self-registration, `/review` | `gh` auth, copier, poetry | 15 min |
| [03](03-migrate-existing.md) | Migrate an existing project | `detect.sh`, `adopt_mode`, MIGRATE.md, gate-as-verifier | `gh` auth, copier | 15 min |
| [04](04-review-commands.md) | `/review` command family | all reply paths + quota accounting | a fleet-member repo | 10 min |
| [05](05-platform-hosting.md) | Platform hosting + kill switch | `deployments.yaml` → live subdomain → teardown | **admin** (registry write) | 15 min |

Conventions: one command per block · expected output quoted after "→" ·
teardown at the bottom of every file. Repo deletion needs the `delete_repo`
scope once: `gh auth refresh -h github.com -s delete_repo`.
