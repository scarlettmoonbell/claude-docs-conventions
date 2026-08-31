# Documentation conventions

Extracted from the `montage-a-trois` / `montage-a-trois-infra` repos, where
this pattern developed in practice. Written down once here so it can be
applied to a new project directly instead of being reconstructed by reading
old repos each time.

This document covers the *shape* of the docs themselves. See
[`DEVOPS.md`](DEVOPS.md) for the paired engineering-practice conventions —
open-source tooling by default, a portable dev environment, infrastructure
and automation defined as code — that these docs exist to record. See
[`INTERFACE.md`](INTERFACE.md) for the paired UI conventions — deferring to
the target platform's own human-interface guidelines rather than bespoke
patterns — for any project with a user-facing screen.

## Philosophy

- **Explain why, not just what.** Every non-obvious decision gets its
  reasoning recorded next to it — alternatives considered, why they were
  rejected, what would trigger revisiting the choice. A reader should never
  have to guess "why is it built this way?"
- **Nothing gets silently deleted.** When something is superseded, retired,
  or turns out to be wrong, it gets struck through or moved to a history
  document with a note explaining what changed and why — not erased. Git
  history alone isn't enough; the *reasoning* trail matters as much as the
  *code* trail.
- **Be honest about what's manual, unfinished, or a known gap.** Docs say
  plainly what doesn't work yet, what a human still has to do by hand, and
  why something wasn't automated — instead of describing an aspirational
  end state as if it were already true.
- **Cross-reference relentlessly, but precisely.** Docs point at each other
  by relative link/anchor rather than duplicating content. Each document has
  one clear job; if a reader needs detail that belongs elsewhere, link to
  it instead of restating it.
- **Real incidents get written up in full**, including root cause, how it
  was found, and the concrete takeaway — not just "fixed a bug." These are
  often the most valuable paragraphs in the whole repo six months later.
- **Track wall-clock reality**, not just intentions — dates, durations, what
  was actually verified (not just "should work").

## The document set

Not every project needs all of these on day one — add `HISTORY.md` when
something first gets retired, add `.github/workflows/README.md` when the
first workflow exists. But the *shape* below is the target.

### `AGENTS.md` (symlinked as `CLAUDE.md`)

Short — a working-agreement doc for an AI agent (or a new human contributor)
picking up the repo cold, not a full README. Two sections, typically:

- **Development** — the one or two commands that matter to know before
  touching anything (e.g. "start the dev server in background mode:
  `<command> --background`, manage it with `<tool> status`/`stop`/`logs`").
- **Documentation** — links to the official docs for the primary
  framework/tool this repo is built on, with a short list of specific guide
  pages worth reading before working on related tasks (routing, components,
  styling, etc.) — not a link dump, a curated "read this before you touch
  that" list.

Keep `CLAUDE.md` a symlink to `AGENTS.md` (`ln -s AGENTS.md CLAUDE.md`), not
a separate file — one source of truth for any agent, named either way.

### `README.md`

The front door. Structure, in order:

1. **One-paragraph description** of what this repo actually is/does, and —
   critically — what it is *not* (e.g. "this repo does not contain the
   website itself — that's the sibling `X` repo").
2. **Operations** (or folded into the description) — where the
   deploy pipeline, infra, and roadmap actually live if they're not in this
   repo. Point at the sibling repo and its `ROADMAP.md`, and at
   `.github/workflows/README.md` for what this repo's own CI does.
3. **Repository structure** — an annotated directory tree, comments
   explaining the non-obvious pieces, for anything more than a couple of
   files.
4. **Dependencies** — exhaustive, end to end:
   - Local/runtime tooling and version constraints
   - Key packages/libraries and *why* (not just a list — one line each on
     what non-obvious role each plays)
   - External services this repo depends on, and what each one is for
   - Deploy target, and what provisions it if that's a different repo
   - The sibling-repo relationship, spelled out: what depends on what, and
     confirming there's no reverse dependency if that's true
5. **Whatever operational sections are specific to this repo** (e.g.
   "Deploying" with the exact commands, "DNS", "Monitoring" — one section
   per genuinely distinct concern, not a wall of text).
6. **"Where to go next"** — a short linked list at the very end pointing to
   every other doc in the project (ROADMAP, HISTORY, sibling repos, deep
   nested READMEs), so a reader always has a map back out.

For a tool-specific nested README (e.g. `opentofu/README.md`,
`docker/README.md`) inside a larger repo, the same shape applies plus:

- **"What this does NOT do"** — a pointer up front to the Known Gaps section,
  so nobody assumes a fresh setup gets them a fully working system from
  nothing.
- **One-time setup** — numbered, copy-pasteable steps, including exactly
  where to get each credential/token and what scopes it needs.
- **Known Gaps** — things automation genuinely cannot do (a manual registrar
  step, a platform API that doesn't exist, a resource that has to be created
  by hand) and why, so nobody discovers this mid-emergency.
- **CI/CD** — what runs in CI and, just as important, what was *evaluated
  and deliberately not automated*, with the actual reasoning (blast radius,
  who could be trusted with what credential, etc.) — not just silence about
  the gap.

### `ROADMAP.md`

A phased, living record of operational/build work — not a static backlog.

- **Phase 0, 1, 2… "Done"** — marked ✅, with the date and (if useful)
  wall-clock duration. Each item is a real, verifiable accomplishment, not
  an aspiration. Real incidents found and fixed during that phase get a full
  writeup inline (root cause, how found, the fix, the general takeaway) —
  this is often the most valuable content in the whole document.
- **A "revisit later" phase** (🗓) — items that are correct to defer, each
  with an explicit **"*Revisit when*: \<concrete trigger\>"** — never a bare
  "later" with no condition attached.
- **An "accepted risk" phase** (📌) — tradeoffs made deliberately at current
  scale, named explicitly so they're a conscious choice being revisited
  periodically, not an unnoticed blind spot.
- **A "preproduction" phase** (🚀), if relevant — the pre-launch checklist,
  distinct from "someday" items because it has a concrete triggering event
  (going live) rather than a condition to watch for.
- **A "How to use this document" section** at the end, stating the actual
  convention in force: when an item completes, mark it done *in place*
  within its original phase (don't relocate it to Phase 0); when an item is
  retired (no longer relevant, as opposed to completed), strike it through
  with a one-line pointer to `HISTORY.md`, where the full original context
  moves — never delete outright.

### `HISTORY.md`

Where retired/superseded material's *full original context* lives once it's
struck through in `ROADMAP.md` — decisions, lessons learned, rejected
approaches, provider bugs hit along the way, exact configuration that used
to matter. Framed explicitly at the top as "nothing here is actionable
against the current state; this is the record of why things were once built
the way they were." This is what keeps `ROADMAP.md` and `README.md` honest
and current without losing institutional memory to git-log archaeology.

### `.github/workflows/README.md`

One section per workflow file, each covering:

- **Triggers** — exactly which events, including any `workflow_call` reuse
  relationships between workflows.
- **What each job does**, plus *why* for anything non-obvious (a specific
  permission needed for a specific reason, a scan tool chosen over an
  alternative, a path filter's purpose).
- **Real bugs hit in this exact CI setup**, written up with the same rigor
  as a `ROADMAP.md` incident — these are exactly the kind of thing that
  silently bites the next person touching CI config if it isn't recorded.

**The first workflow in any new repo following this pattern is a
`docs.yml`** — a `pull_request`-triggered markdown-lint check scoped to
`**/*.md`, not path-filtered to any subdirectory. A repo whose only content
is documentation (or that hasn't written any application code yet) still
has something worth checking on every PR from commit one; don't wait for
"real" CI to exist first. If the repo is private on GitHub's free plan,
required status checks aren't available (the branch-protection and
rulesets APIs both return an upgrade-required error) — the check still
runs and reports pass/fail on the PR, so treat it as a real gate by
discipline (wait for it before merging) rather than one enforced by the
platform. Say so explicitly in `.github/workflows/README.md` rather than
letting a reader assume the green check is a hard gate when it isn't.

## How to apply this to a new project

1. Start every new repo with `AGENTS.md` (symlinked `CLAUDE.md`) and a
   `README.md` with at least the Dependencies section — don't wait until the
   repo feels "big enough."
2. Add `.github/workflows/README.md` the moment the first workflow file
   exists, not after several have accumulated. That first workflow should
   be `docs.yml` (markdown-lint on every PR touching `**/*.md`) — add it
   before any code-specific CI, even on a repo that's nothing but docs so
   far.
3. Add `ROADMAP.md` once there's a real, ongoing operational/build sequence
   to track — a brand-new scaffold can seed it directly from a build plan's
   phases.
4. Don't create `HISTORY.md` preemptively — add it the first time something
   actually gets retired, then follow the pattern from there.
5. When in doubt about whether something belongs in a given doc, ask "which
   of these would a new contributor, or an AI agent with zero prior context,
   need to read first to avoid repeating a mistake someone already made
   here?" — that's the document it belongs in.

See `templates/` in this repo for copy-pasteable skeletons of each document.
