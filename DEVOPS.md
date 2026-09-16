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
- **Codify anything that was done manually, the moment it's practical to
  — don't just avoid *new* manual work, go back for the old kind.** A
  manual action leaves nothing a future `plan` can see: a DNS record
  pasted into a provider's dashboard, an account setting flipped once
  during setup. The email DNS records for the site (domain-verification
  TXT, MX, three DKIM/ARC CNAMEs, SPF, DMARC) were pulled directly from
  the provider's own account-specific setup page — not guessed — and
  written into `migadu-email.tf` as `cloudflare_dns_record` resources
  instead of being pasted into Cloudflare's dashboard by hand, then
  verified resolving via `dig` immediately after `apply`, with zero drift
  on the next `plan`. Treat a piece of still-manual working infrastructure
  as an open item, not a closed one, until it's been backfilled into code
  — imported where the provider's tooling supports it, recreated as a new
  resource where it doesn't. If something genuinely can't be codified (no
  API exists — classic GitHub OAuth Apps have no creation API, so this
  project's Decap CMS proxy OAuth App stays hand-created), that's a
  `Known Gaps` entry naming the specific limitation, not a reason to stop
  looking at everything else. Codifying only runs one direction, too:
  removing a resource from IaC state doesn't undo its real-world
  counterpart — dropping the old VPS from Terraform didn't stop it from
  running (and billing) for two more days until someone actually checked
  and shut it down by hand.
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
- **Never truncate a command's output when the result will inform a
  decision — read the whole thing.** Piping to `head`/`tail -N`, passing
  a small `limit`, or otherwise capping what comes back trades a false
  sense of speed for a real risk: the exact line that would have changed
  the decision is the one most likely to be the one that got cut. This
  isn't hypothetical — an early `git status --short --branch | head -1`
  check on a repo reported it as clean by only showing the first line,
  silently hiding real uncommitted work-in-progress (an in-flight
  landing-page redesign) that only surfaced several steps later when it
  became impossible to miss. Prefer redirecting full output to a
  scratch file and reading it whole, or raising the read limit, over any
  command-level truncation — a slightly longer read costs nothing next
  to acting on an incomplete picture. This is the same instinct as
  reading a full `tofu plan` before applying, generalized to every
  other command whose output gets trusted.

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
- **Write down one explicit reliability target per live service, even
  informally — and let it actually govern the pace of risk-taking.**
  Google's [SRE book](https://sre.google/sre-book/embracing-risk/) calls
  this an SLO (Service Level Objective): a stated target (e.g. "99.9% of
  requests succeed this month"), measured against reality, with the gap
  between the two — the error budget — as what decides whether to ship
  the next risky change now or slow down first. This doesn't need the
  full apparatus (per-service on-call rotations, a formal error-budget
  policy document) to be worth doing at solo-operator scale — even one
  number, checked before a launch or a risky migration, replaces a
  gut-feel judgment call with an actual answer to "are we being too
  aggressive right now." Track it the same place an accepted-risk line
  already lives (`ROADMAP.md`), not as a separate process.

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
- **SHA-pinning covers what a build consumes — a build's own output
  needs the same integrity story when another repo consumes it.**
  [SLSA](https://slsa.dev/spec/v1.0/levels) (Supply-chain Levels for
  Software Artifacts) frames this in levels: Level 1 is just a
  provenance record of how an artifact was actually built (source
  commit, build command, build platform) — cheap, no new
  infrastructure; Level 2 adds a hosted build platform signing that
  provenance so it's tamper-evident, natively supported on GitHub
  Actions via `actions/attest-build-provenance` with no separate
  key-management to run. Anything one repo in this account consumes as
  a built artifact from another (e.g. a `dist/` published as a
  `github:owner/repo#main` git dependency) should generate that
  provenance at build time, not rely on the consuming repo trusting the
  committed output on faith.
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

## Logging & observability

Sourced differently from the rest of this document: the other sections are
extracted purely from real decisions in this account's own repos; these
principles are grounded primarily in established external references —
[the Twelve-Factor App's Logs factor](https://12factor.net/logs),
[OWASP's Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html),
and [Cloudflare's own Workers observability guidance](https://developers.cloudflare.com/workers/observability/)
— applied to this account's actual, mostly-Cloudflare-Workers stack, with
real internal precedent cited wherever it already exists.

- **Treat logs as an event stream, not a file the application manages.**
  The Twelve-Factor App's Logs factor: a process writes its events to
  `stdout`/`stderr`, unbuffered, and never concerns itself with routing,
  rotating, or storing that stream — that's the execution environment's
  job, not the app's. On this account's stack that maps directly onto
  Cloudflare Workers' `console.log`/`console.error`: don't build custom
  log-shipping infrastructure. `montage-a-trois-infra`'s own `HISTORY.md`
  already records this lesson from the other direction — a "centralized
  log shipping" item was correctly retired once the VPS it was written
  for was gone, noting that "Cloudflare's own dashboard/Logpush would be
  the equivalent tool for the current architecture" — use the platform's
  own tool rather than re-inventing one.
- **Log in structured JSON, not string concatenation.**
  `console.log(JSON.stringify({...}))`, not
  `console.log("Got a request to " + path)` — Cloudflare's own Workers
  best practices flag unstructured string logs as an anti-pattern
  specifically because they can't be queried or filtered later. Use
  `console.error` for anything that should surface at error severity in
  the dashboard, not `console.log` with a hand-rolled `"ERROR:"` prefix.
- **Enable the platform's native observability before production, not
  after the first incident.** Every Worker should ship with
  `observability.enabled: true` in its config, with `head_sampling_rate`
  set deliberately rather than left at whatever the scaffold defaulted
  to — sampling rate is the actual cost/volume lever, so tune it instead
  of either drowning in log volume or flying blind during an incident.
  Same instinct as this document's cost-effective-availability
  section: reach for the platform's own tool before building one.
- **Log the meaningful steps of a multi-step or asynchronous process —
  not just its final success or failure.** A request or job that fails
  partway through a pipeline is far easier to debug when each stage logs
  its own start/complete than when only the terminal outcome is visible;
  the alternative is reconstructing what happened from database state
  after the fact. This doesn't mean logging everything — it means
  logging *transitions*, the specific points where a bug would otherwise
  stay invisible until someone goes looking for it by hand. Real
  precedent, not hypothetical: `scenestealer-app`'s job runner
  (`apps/worker/src/analyze.ts`) added a `logStep()` helper logging
  elapsed time plus RSS/heap at each pipeline checkpoint specifically
  *after* an OOM killed the process with no way to tell which step was
  responsible — the code's own comment frames this as "observability
  generally, not just crash forensics." Separately, `apps/api/src/routes/videos.ts`
  keeps a deliberate `console.log` in its queue-consumer handler that its
  own comment explicitly calls out as "not a temporary debug leftover":
  Cloudflare's own queue-execution log only shows "Queue ... - Ok," which
  reveals nothing about whether the actual DB write landed — exactly the
  gap that turned a real client-saw-failure-but-server-logs-showed-success
  incident into a multi-round diagnosis before this log line existed.
- **Carry a correlation/request ID across service boundaries, once a
  request crosses more than one.** A request that flows through
  multiple services (an API, a queue consumer, a separate worker
  process) is far harder to trace without a shared ID tying its log
  lines together across all of them — `scenestealer-app`'s own api →
  Fly-worker → database flow has no such ID today, an admitted gap in
  the code's own comments, not a hypothetical risk. Add one before the
  second real cross-service debugging session, not after several.
- **Uptime monitoring and application error tracking are different
  observability layers — one doesn't substitute for the other.** An
  external check (Better Stack) answers "is it reachable"; it says
  nothing about whether requests that got a `200` actually did the
  right thing. `montage-a-trois-infra` already draws this line
  explicitly in its own `ROADMAP.md`: Better Stack is live
  (`opentofu/monitoring.tf`), while a separate, still-unbuilt item for
  real exception tracking (e.g. Sentry) is named specifically because it
  "catches actual application exceptions rather than just 'is it
  reachable.'" Track them as two separate line items, not one.
- **Naming a swallowed error as a deliberate tradeoff is the right
  instinct — but it's still a real gap, and the write-up should say
  so.** `queenjupiter-site`'s booking-confirmation email path
  deliberately swallows a send failure with a comment explaining exactly
  why (the operator otherwise never finds out about a submission whose
  own confirmation email failed), which is the correct way to make that
  call visible instead of silent. Its `ROADMAP.md` then names the
  resulting blind spot explicitly under its own "Observability —
  suggested, not built" heading, with a concrete cheap fix proposed
  (a Worker analytics event or a webhook) rather than left as vague
  future work — that's the model to copy: comment the tradeoff at the
  code, then track the gap it leaves at the doc level with an actual
  next step attached, not just "TODO: add monitoring."
- **Never log a secret or sensitive-PII value in plaintext, even at
  debug level.** Per OWASP's Logging Cheat Sheet: session tokens, access
  tokens, passwords, encryption keys, and payment/health/government-ID
  data must never appear in a log line — redacted or hashed only, if
  referencing them is unavoidable at all. This isn't hypothetical for
  this account: a project whose booking form encrypts identity-revealing
  fields at rest and GPG-encrypts the operator notification does that
  specifically so the data stays protected end to end — a stray debug
  log of the raw submission anywhere in that pipeline would undo all of
  it in one line. Decide and write down what's safe to log *before*
  adding the first log line to code that touches sensitive data, not
  after.
- **Always log security-relevant events, specifically.** OWASP's
  non-negotiable list: authentication successes and failures,
  authorization/access-control failures, input-validation failures, and
  session-management failures. A rejected attempt against an
  Access-gated admin route, or a failed step in an OAuth proxy flow, is
  exactly the kind of event that's cheap to log now and expensive to
  have missed later.
- **Sanitize event data before it reaches a log line.** OWASP's
  log-injection guidance: strip or escape carriage-return/line-feed and
  delimiter characters from anything user-supplied before logging it,
  the same way it would be sanitized before being rendered — an
  unsanitized log line is an injection surface into whatever reads the
  log next (a dashboard, a script parsing it later, a future SIEM).
- **A log line should answer when, where, who, and what.** OWASP's
  structure for a useful event: a timestamp, the service/component and
  code location, the actor (user ID or source IP, if known), and the
  event type plus outcome. A bare `console.log("done")` fails this on
  every axis — the fix costs one more object key, not a new logging
  framework.

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
| Manual state that still needs backfilling into code | A `ROADMAP.md` item until it's done (not a `Known Gaps` entry — that's reserved for the genuinely-impossible case); once codified, mark it done in place, the same as any other roadmap item |
| Something confirmed genuinely impossible to codify (no API exists) | `README.infra.md`'s **Known Gaps**, naming the specific limitation — not silence, and not lumped in with items that are simply not done yet |
| Observability config for a new service (Worker, Function, background job) | README `Dependencies`/one-time setup — state that `observability` is enabled and why `head_sampling_rate` is set where it is |
| A logging/PII decision (what's redacted, what's never logged) | Written down next to the code it protects, the same "explain why" rule as any other non-default choice — not left implicit |
| A gap in step-level logging for a multi-stage pipeline | `Known Gaps` or a `ROADMAP.md` item, not silence — same treatment as any other incomplete piece |
| A service's reliability target (SLO) | `ROADMAP.md`, the same place an accepted-risk line already lives — stated as a number, not a vibe |
| Build-provenance/attestation status for a published artifact | README `Dependencies`, next to the dependency direction it affects; a `Known Gaps` entry if not yet generated |

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
8. Periodically look for infrastructure that exists only because someone
   clicked a button once — a DNS record, an account setting, an API
   credential created through a provider's UI — and pull its exact values
   from the provider's own setup page (never guessed) to write into code.
   Verify it still works the same way after codifying (e.g. `dig` for a
   DNS record) before trusting that the next `plan` showing zero drift
   actually means zero drift. Only stop and file a `Known Gaps` entry
   instead once you've confirmed the provider genuinely has no API for
   it — not on the first sign that codifying it would take longer than
   doing it by hand again.
9. Before shipping a new Worker/Function/service to production, confirm
   three things together, as part of "done" rather than a follow-up
   task: `observability` is enabled with a deliberately-chosen
   `head_sampling_rate`; nothing it logs could be a secret or sensitive
   PII in plaintext; and its meaningful step transitions — not just
   final success/failure — are visible in the log stream.
10. Write down one reliability number per live service — even a rough
    one — before assuming "we'll know if it's a problem." Use it as the
    actual answer to "can we ship this risky thing now," not just a
    number that sits unread in `ROADMAP.md`.
11. If a repo's build output is consumed by another repo as a
    dependency (a committed `dist/`, a published package), generate
    build provenance for it — not just SHA-pin the actions that produce
    it. If that's not yet in place, it's a `Known Gaps` entry, not a
    silent trust assumption.
12. Before concluding anything from a command's output — a status
    check, a log tail, a search result — make sure it's the full
    output, not a piped/limited slice. If a result seems to say
    "nothing here," confirm that's actually true rather than an
    artifact of truncation before acting on it.
