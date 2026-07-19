# GitHub Actions Workflows

What each workflow in this directory does, when it runs, and why.

## docs.yml

**Triggers:** `pull_request`, paths `**/*.md`.

- **markdown-lint** — checks out the PR branch and runs
  `DavidAnson/markdownlint-cli2-action` against every markdown file in the
  repo, using the ruleset in `../../.markdownlint-cli2.jsonc` (default
  rules, with `MD013` line-length exempted for tables — this repo's prose
  is hand-wrapped to ~80 columns, but table rows can't be wrapped without
  breaking table syntax).

This repo is nothing but documentation, so this one workflow is the whole
CI story — no path-filtering beyond `**/*.md` since there's nothing else
here to scope around, unlike a mixed app/infra repo where `docs.yml` and a
code-specific `checks.yml` split the surface between them.

**Not a hard gate:** this repo is private on GitHub's free plan, which
doesn't support required status checks (branch protection / rulesets both
return "Upgrade to GitHub Pro or make this repository public" from the
API). The check still runs and reports pass/fail on every PR — merging
without waiting for it to pass is possible but against the convention this
repo itself documents (see `CONVENTIONS.md`): treat it as a real gate by
discipline, the same model already used on the `montage-a-trois-infra`
repo's CI for the same reason.
