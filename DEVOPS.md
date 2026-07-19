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

## Availability & cost-effective resilience

- **Actively look for the rare case where the cheaper option is also the
  more available one, instead of assuming resilience costs more.**
  Migrating the site off a self-managed VPS onto Cloudflare Pages +
  Workers didn't just retire an accepted-risk "single VPS, no
  redundancy/failover" line by *adding* failover engineering — it retired
  it by moving onto a host whose edge network already provides that
  redundancy, while cutting the ongoing patch/harden/monitor burden of
  owning a box to roughly zero in the same move. Look for that move
  before reaching for custom HA machinery.
- **Let the deploy platform own atomic deploy and rollback once it
  genuinely can, instead of hand-rolling it.** A custom
  `releases/<timestamp>/` + `current`-symlink swap with a post-deploy
  health check and auto-rollback was real, working infrastructure — and
  was still correctly retired in favor of Cloudflare Pages' own
  deployment history and instant rollback the moment the platform made it
  redundant, "no custom scripting needed." Keep a cheap post-deploy health
  check (a `curl` for `200` against the live URL) as a pipeline gate
  regardless of platform — that part earns its keep even when you trust
  the platform's own rollback.
- **A hosting/DNS provider's bundled CDN/WAF is a legitimate way to get
  both for free, not a corner cut.** Cloudflare's proxy sitting in front
  of both the Pages site and the Worker retired a separate "no CDN/WAF in
  front of the origin" accepted-risk line entirely, with no separate
  service to pay for or operate.
- **Monitor externally and cheaply, and monitor what users actually
  hit** — an outside-the-perimeter uptime check (Better Stack watching
  the live HTTPS endpoint) catches exactly the class of failure internal
  automation can't: the case where the automation itself is what's down.
- **Name availability tradeoffs as an explicit, revisitable accepted
  risk — never a silent assumption of "that's just how it is at this
  scale."** Track it the way `ROADMAP.md`'s accepted-risk phase already
  asks for, and treat a provider migration that happens to resolve one as
  a real resolution worth recording, not a fact to mention only in
  passing.

## Firewall and other security policies: least privilege

- **Separate the human/admin identity from the CI/automation identity,
  even when both reach the same box.** A dedicated, fully-privileged
  admin account plus a separate CI-authenticated account with a sudoers
  rule scoped to exactly one command (e.g. `systemctl reload nginx`, and
  nothing broader) means a leaked CI secret's blast radius is capped at
  that one command, no matter how privileged the human account is
  allowed to be.
- **Disable root login and password auth; key-only access under a named
  user, for the audit trail as much as the access control.** The real
  security win of disabling root SSH login isn't reducing what the admin
  account can do once authenticated (it can still do everything via
  sudo) — it's removing the single most common target of internet-wide
  SSH brute-force scanning, plus attaching every privileged action to a
  named identity instead of an anonymous "root."
- **Bind internal-only services to loopback, not a public interface, and
  reverse-proxy them.** An internal service bound to `127.0.0.1` and
  reverse-proxied by the edge web server is unreachable from outside
  regardless of firewall state — a second, independent layer that holds
  even if a firewall rule is ever misconfigured or simply never turned
  on (see the honesty point below).
- **Default-deny inbound at the firewall, and enumerate exactly which
  ports are meant to be reachable and why** — a periodic services audit
  (what's bound to `0.0.0.0`, what's loopback-only, what has no listener
  at all) is what that enumeration actually looks like in practice.
  **Be honest that a written firewall plan and an enabled firewall are
  two different states** — a `ufw` policy drafted but never `enable`d is
  a known gap to name explicitly, not a control to claim credit for.
- **Scope cloud/API credentials to the minimum capability the job
  actually needs — a read-only credential for anything that only
  reads.** A CI job that only runs `plan` should authenticate with a
  distinct read-only token, never the write-scoped one used for a real
  `apply`/deploy, so that even a fully compromised run can see a diff but
  has no credential capable of acting on it. Give the read-only and
  write-scoped tokens clearly distinct variable names, not just distinct
  values — wiring the read-only token into a slot meant for the
  write-scoped one is a realistic way to silently defeat the whole
  point, and a naming collision is what lets that happen unnoticed.
- **Least privilege has to be paired with correctly identifying the
  actual minimum — defaulting to zero permissions isn't automatically
  safer if it's wrong.** Explicitly declaring a `permissions:` block per
  CI job (rather than relying on a repo's default) is the right instinct,
  but under-scoping it is a real outage, not just a hardening nicety: a
  job missing `pull-requests: read` broke every non-PR-triggered deploy
  silently at startup. Scope down deliberately, then actually exercise
  every trigger path before trusting it.
- **Pin third-party CI dependencies to immutable commit SHAs, not
  mutable version tags**, and let an automated dependency bot keep the
  pin current alongside its version comment — supply-chain hardening
  that costs nothing ongoing once set up.
- **Scan for secrets and known vulnerabilities on every change, not just
  at initial setup.** A secret scanner (e.g. gitleaks) and a
  vulnerability/IaC scanner (e.g. Trivy) on every PR catch what code
  review alone won't.
- **Suppress version fingerprints and set baseline security response
  headers on anything public-facing** — turning off a web server's
  version string, plus `X-Content-Type-Options`/`X-Frame-Options` at
  minimum (and, honestly flagged rather than silently skipped,
  `Strict-Transport-Security`/`Referrer-Policy` as well). Small, cheap,
  and easy to forget precisely because nothing breaks when it's missing.
- **Give secrets on disk the tightest permission that still works, not
  the most convenient one.** A secrets file owned `600 root:root`,
  readable only before a service drops privilege to its own unprivileged
  user, means the running service itself never needs read access to the
  credential it was configured with in the first place.

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
| An availability/cost tradeoff (redundancy, CDN/WAF, monitoring) | `ROADMAP.md`'s accepted-risk phase if deliberate at current scale; struck through into `HISTORY.md` once a later change (e.g. a platform migration) actually resolves it |
| Firewall rules, port exposure, service account/sudo scoping | `README.infra.md`'s **One-time setup** (what's opened and why) and **Known Gaps** (anything drafted but not yet enforced — e.g. a firewall plan written but not enabled) |
| Credential scoping (read-only vs. write tokens, secret file permissions) | README `Dependencies`/provider table, with the *why* stated inline the same as any other non-default choice |
| A least-privilege change that itself caused an outage (under-scoping, not over-scoping) | Written up in full in `.github/workflows/README.md` or `ROADMAP.md`, exactly like any other incident — the lesson is "find the actual minimum," not "grant less" |

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
5. Before reaching for custom redundancy/failover engineering, check
   whether a managed platform already provides it as a side effect of
   being the cheaper choice — that's a strictly better outcome than
   either optimizing for cost or for availability alone, when it's
   available.
6. Default every service and credential to the minimum exposure that
   still works: loopback-bind what doesn't need to be public, a
   default-deny firewall with an enumerated allow-list, a read-only
   credential for anything that only reads, a scoped sudo rule instead of
   a shared privileged account. Then actually verify the minimum is
   correct (exercise every trigger path, every legitimate use) before
   trusting it — an unverified restriction can break things as silently
   as an unverified grant.
7. If a security control is planned but not yet enforced (firewall
   written but not enabled, a header not yet added), say so explicitly in
   `Known Gaps` — a drafted plan is not a control, and the gap between
   the two is exactly the kind of thing that should never be discovered
   by surprise.
