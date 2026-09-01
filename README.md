# Documentation Conventions

A reusable reference for the documentation pattern developed in the
`montage-a-trois` / `montage-a-trois-infra` repos — written down once so it
can be applied directly to a new project instead of being reconstructed by
re-reading old repos each time.

Read [`CONVENTIONS.md`](CONVENTIONS.md) for the full writeup: the philosophy
behind the pattern, and a section-by-section breakdown of what belongs in
`AGENTS.md`/`CLAUDE.md`, `README.md` (both an app-repo and an infra-repo
flavor), `ROADMAP.md`, `HISTORY.md`, and `.github/workflows/README.md`.

Read [`DEVOPS.md`](DEVOPS.md) alongside it for the engineering-practice
conventions that pair with the documentation shape: preferring open-source
tooling, keeping the dev environment portable, defining infrastructure and
automation as code, documenting the deploy process for the dev environment,
the application, and provisioned resources as three distinct, clearly-owned
write-ups, favoring cost-effective availability (managed platforms over
self-managed redundancy where that's genuinely cheaper), least-privilege
security policy (firewalling, credential/service-account scoping, secret
handling), and codifying anything done manually — not just avoiding new
manual work, but backfilling the old kind into code too.

Read [`INTERFACE.md`](INTERFACE.md) for the third layer: how anything with
a user-facing screen should look and behave — defaulting to the target
platform's own official human-interface guidelines (Apple HIG, Material
Design, Fluent) rather than bespoke patterns, with accessibility treated as
inside that guidance rather than a separate pass.

## Using this repo

Copy the relevant file from [`templates/`](templates/) into a new project
and fill in the bracketed placeholders:

| Template | Copy to |
| --- | --- |
| `AGENTS.md.template` | `AGENTS.md` (then `ln -s AGENTS.md CLAUDE.md`) |
| `README.app.md.template` | `README.md`, for an application/product repo |
| `README.infra.md.template` | `README.md`, for an infra/IaC repo (or a nested tool-specific README) |
| `ROADMAP.md.template` | `ROADMAP.md`, once there's a real build/ops sequence to track |
| `HISTORY.md.template` | `HISTORY.md`, only once something is first retired |
| `workflows-README.md.template` | `.github/workflows/README.md`, the moment the first workflow file exists |
| `dependabot-gitsubmodule-block.yml.template` | Merge into `.github/dependabot.yml`, after adding this repo as a git submodule — see `CONVENTIONS.md`'s "Consuming this repo from another project" |

First applied to the four [SceneStealer](https://github.com/scarlettmoonbell/scenestealer-app)
repos — check those for a worked example of the pattern in a brand-new
project.
