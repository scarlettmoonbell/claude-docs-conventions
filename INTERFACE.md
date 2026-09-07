# Interface conventions

A companion to [`CONVENTIONS.md`](CONVENTIONS.md) and [`DEVOPS.md`](DEVOPS.md)
covering a third layer: not how decisions get written up, and not how a
project gets built and operated, but how anything with a user-facing screen
should look and behave. Written down once so it's a deliberate default the
first time a project actually needs it, rather than being decided ad hoc
per screen.

*Unlike the other two documents, this one isn't yet extracted from a real
decision made in one of this account's projects — nothing built so far has
had enough interface surface to test it against. It's written proactively
so the first real UI work has something concrete to check against, and can
correct or extend it with a real example the way `DEVOPS.md` already does.*

## Principles

- **Default to the target platform's own official human-interface
  guidelines, not a custom pattern, unless there's a documented reason
  not to.** iOS/macOS/iPadOS/watchOS/tvOS → Apple's
  [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines).
  Android → Google's
  [Material Design](https://m3.material.io) guidelines. Windows →
  Microsoft's [Fluent Design System](https://fluent2.microsoft.design). The
  web has no single owning platform, but still has real conventions to
  defer to — don't override a native control (date picker, `<select>`,
  checkbox, focus ring) with a custom-styled equivalent unless the native
  one genuinely can't do the job; when it's replaced, match the native
  one's keyboard and screen-reader behavior, not just its look.
- **A deviation from the platform guideline is a decision, not a
  default** — write down *why* next to the decision (the same "explain
  why, not just what" rule `CONVENTIONS.md` already applies to everything
  else), what the guideline-compliant version would have looked like, and
  what would trigger reverting to it.
- **Accessibility guidance isn't separate from HIG guidance — it's
  inside it.** Apple's HIG, Material Design, and
  [WCAG 2.2](https://www.w3.org/TR/WCAG22/) (the current version —
  supersedes 2.1) all treat contrast, touch-target size,
  dynamic-type/text scaling, and screen-reader semantics as first-class
  requirements, not an add-on pass at the end. Treat a screen that
  hasn't been checked against them as unfinished, not as "done, needs an
  accessibility pass" — the `design:accessibility-review` skill is the
  mechanical way to do that check. Concrete, testable thresholds at
  Level AA, not just "check contrast": text contrast at least 4.5:1
  (3:1 for large text — 18pt+, or 14pt+ bold), non-text UI components
  at least 3:1 against their adjacent color (WCAG 1.4.3, 1.4.11), and
  touch targets at least 44×44 CSS px (WCAG 2.5.8).
- **Two WCAG 2.2 criteria worth checking by name, since they're new
  enough to be commonly missed:** *Focus Not Obscured* (2.4.11) — a
  sticky header, footer, or dialog must never hide the element that
  currently has keyboard focus, an easy bug to ship without ever
  noticing since it only shows up navigating by keyboard — and
  *Accessible Authentication* (3.3.8) — a login/signup flow can't
  require solving a puzzle or transcribing something from memory as the
  only path in; if a cognitive-function test is used at all (a CAPTCHA,
  a security question), there must be a way in that doesn't need it.
- **Target the current guideline, not whatever was memorized once.**
  Platform guidelines get revised — new platform capabilities, deprecated
  patterns, updated accessibility minimums. Re-check the live guideline
  rather than a cached mental model, especially when picking UI work back
  up after a long gap between sessions on the same project.
- **Native-feeling beats novel, by default.** Standard navigation
  patterns (tab bars, platform-standard back behavior, conventional
  iconography for common actions) reduce what a first-time user has to
  learn. A bespoke interaction pattern has to earn its place by being
  clearly better for this specific product, not just different — the
  burden of proof is on the deviation, not on the default.

## Where this shows up in the document set

Same shape as `DEVOPS.md` — no new document type needed:

| Principle | Lives in |
| --- | --- |
| Target platform(s) and which guideline governs the UI | README `Dependencies`/description, named up front the same as any other framework choice |
| A deliberate deviation from the platform guideline | Written down next to the decision — README, or a `ROADMAP.md` entry if it was deliberated over multiple rounds |
| An accessibility gap not yet addressed | `Known Gaps`, not silence |
| A guideline-driven redesign or correction | `ROADMAP.md`, with the same before/after-and-why rigor as any other incident |

## How to apply this to a new project

1. Name the target platform(s) up front — native iOS/macOS, Android,
   Windows, or web — and the specific guideline document that governs
   each, next to the other entries in the README `Dependencies` section.
2. Before introducing a custom control or interaction pattern, check
   whether the platform's own guideline already defines one for this
   exact case. Use the built-in one unless there's a specific, named
   reason not to — and record that reason where the decision lives.
3. Run an accessibility check (contrast ≥4.5:1 text / ≥3:1 large text
   and UI components, touch targets ≥44×44 CSS px, screen-reader
   labels, dynamic type, keyboard focus visibility) as part of
   finishing a screen, not as an optional follow-up.
4. When picking up UI work after a long gap, re-check against the
   current version of the guideline being followed — treat "the
   guideline changed since I last checked" as the normal case, not an
   edge case.
