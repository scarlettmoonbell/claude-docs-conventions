# DevOps conventions

Extracted from the same `montage-a-trois` / `montage-a-trois-infra` build as
[`CONVENTIONS.md`](CONVENTIONS.md), but a different layer: that document
says how a project's decisions get *written up*; this one says how a
project should actually be *built and operated* in the first place. Written
down once so it's a deliberate default on the next project instead of being
re-decided (or silently skipped) each time.

## Principles

- **Prefer open-source tooling, by default, not by exception.** Pick OSS
  unless there's a concrete reason not to, and when a closed/hosted
  alternative is picked anyway, write the reasoning down next to the
  decision (see [Where this shows up](#where-this-shows-up-in-the-document-set)
  below) so it reads as a conscious tradeoff, not an oversight. A vendor's
  licensing move is a real trigger to migrate, not just a philosophical
  preference: Terraform's 2023 BSL relicense was the concrete reason the
  infra repo runs OpenTofu instead. The same instinct applies to individual
  modules, not just the core tool — `getstackhead/nginx` was evaluated and
  rejected for being unmaintained since 2020, in favor of a generically
  capable, actively maintained provider. It also applies to hosted services
  quietly acting as a dependency: when Netlify's shared OAuth proxy turned
  out to be broken for this setup, the fix was a small self-hosted proxy
  under our own control, not a workaround bolted onto someone else's
  black box.
- **Automate relentlessly, and let code stay the single source of truth.**
  All infrastructure and automation is defined in code (OpenTofu, GitHub
  Actions) — no manual click-ops change, even a "quick fix," that code
  doesn't know about. When a tool's own convenience feature would fight that
  (certbot's `--nginx` plugin mutates the same config file OpenTofu manages),
  resolve it architecturally — switch to `certonly --webroot` and let
  OpenTofu hand-author the server blocks — rather than accepting an
  untracked exception. If a resource type doesn't support `import`, it's
  still fine to let `apply` adopt already-correct live state fresh, as long
  as `plan` is read in full first to confirm it's non-destructive.
- **Contain and make the development environment portable.** A contributor
  (or an agent) should be able to get a working dev environment from a
  fresh checkout with minimal manual setup — pinned versions, no
  "works on my machine" institutional knowledge required. Where this isn't
  true yet, that's an honest, named gap (see below), not silence.
- **Deploy processes are documented clearly, for every distinct layer** —
  the dev environment, the application, and the provisioned resources each
  get their own clear write-up, in the doc that already owns that concern
  (see the mapping below) rather than one document trying to cover all
  three at once.
- **Real bugs that automation itself catches are the return on investment
  — write them up in full, the same way `ROADMAP.md` and
  `.github/workflows/README.md` already ask for.** The infra repo's first
  CI run caught a hardcoded `~/.ssh/...` path that had silently worked on
  one laptop for weeks and would have broken for literally any other
  machine — a portability bug, not a CI-specific one, that only running
  the automation on a second machine exposed. That's the case for the
  convention, not just a nice anecdote.

## Where this shows up in the document set

These principles aren't a new document type — they're captured by the
existing shape from `CONVENTIONS.md`:

| Principle | Lives in |
| --- | --- |
| Open-source vs. closed tooling choice, and why | README `Dependencies` section (state the choice **and** the reasoning if it's non-default); a `ROADMAP.md` entry if it was deliberated over multiple rounds |
| Portable, containerized dev environment | `AGENTS.md`'s **Development** section (the exact commands); README `Dependencies` → local/runtime tooling and version constraints |
| Infrastructure/automation defined as code | The whole shape of `README.infra.md` — repository structure, provider table, one-time setup, Known Gaps, CI/CD |
| Deploy process — dev environment | `AGENTS.md` |
| Deploy process — application | `.github/workflows/README.md` |
| Deploy process — provisioned resources | `README.infra.md`'s **One-time setup** and **CI/CD** sections |
| A gap where full automation genuinely isn't possible | `README.infra.md`'s **Known Gaps** — named and explained, never discovered by surprise |
| A bug automation itself caught | Written up in full in `ROADMAP.md` or `.github/workflows/README.md`, same rigor as any other incident |

## How to apply this to a new project

1. Default to OSS for the core tool and for individual modules/providers
   within it. If a closed or hosted alternative wins anyway, write the
   reasoning into the README `Dependencies` section at the time of the
   decision, not after the fact.
2. Before hand-editing any live resource — server config, DNS record,
   installed package — ask "can this go through the IaC instead?" If yes,
   do it there even when the manual edit would be faster. If truly no,
   that's a `Known Gaps` entry, written down before someone finds out the
   hard way mid-incident.
3. If the dev environment isn't yet reproducible/portable from a fresh
   checkout, say so plainly — a `Known Gaps` line or a `ROADMAP.md` phase
   item with a concrete trigger, not a silent omission.
4. When CI or automation catches a real bug, write it up with root cause
   and takeaway in the relevant doc — that write-up is the evidence the
   practice is earning its keep.
