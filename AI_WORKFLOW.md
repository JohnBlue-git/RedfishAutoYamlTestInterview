<style>
h1, h2, h3, h4, h5, h6 {
  color: #1976d2 !important;
}
</style>

# Working with an AI on a design this size

How the config-driven engine described in [`DESIGN.md`](DESIGN.md) was
actually built — the *process*, not the design. `DESIGN.md` says what was
built and why; this says *how the work was organised* so an AI could do most
of it without the result drifting from what was wanted.

Written down because the method transferred better than any individual
output did. The same loop produced the engine itself, dozens of schema
migrations, and the documentation set — and it caught several mistakes that
a write-code-and-hope approach would have shipped.

## Contents

| § | Section |
|---|---|
| 1 | [The shape of the loop](#1-the-shape-of-the-loop) |
| 2 | [Phase 1 — grill the design before any code](#2-phase-1--grill-the-design-before-any-code) |
| 3 | [Phase 2 — spec, then tickets](#3-phase-2--spec-then-tickets) |
| 4 | [Phase 3 — implement one ticket at a time](#4-phase-3--implement-one-ticket-at-a-time) |
| 5 | [Phase 4 — record what was learned, not just what was done](#5-phase-4--record-what-was-learned-not-just-what-was-done) |
| 6 | [Durable context: project memory and skills](#6-durable-context-project-memory-and-skills) |
| 7 | [Rules that did the heavy lifting](#7-rules-that-did-the-heavy-lifting) |
| 8 | [What this caught](#8-what-this-caught) |
| 9 | [Tooling and credits](#9-tooling-and-credits) |

---

## 1. The shape of the loop

```
  idea
    │
    ▼
  GRILL      one question at a time, until the design is pinned down
    │        → decisions, including the ones that overrode the recommendation
    ▼
  SPEC       problem statement, user stories, decisions frozen in writing
    │
    ▼
  TICKETS    vertical slices, each with acceptance criteria and its blockers
    │
    ▼
  IMPLEMENT  one ticket per session · verify live · commit
    │
    ▼
  RECORD     status ledger entry + a design-history paragraph,
    │        written in the same commit as the work
    ▼
  next ticket
```

The important property: **each stage produces a durable artifact on disk.**
No stage depends on conversation memory surviving, which matters because a
design this size spans many sessions and context windows.

## 2. Phase 1 — grill the design before any code

The single highest-leverage step, and the least obvious one.

Before any code, the design was interrogated question by question —
execution model, config format, setup/teardown scope, verification design,
id grammar, file layout, step data passing, platform hierarchy, risk
guarding, compliance tiering, reporting, migration strategy. One question at
a time, each with a recommended answer, waiting for a decision before moving
on.

The resulting spec's closing note records the outcome:

> *"This spec is the synthesis of an extended grilling session … every
> decision point above reflects an explicit choice made during that session,
> including several where the chosen option was not the initially
> recommended one — e.g. one-file-per-schema config layout, sequences living
> inside the same file as the tests they sequence rather than a separate
> top-level directory, and the exact order of the platform-tier hierarchy."*

That parenthetical is the point. **Three core structural decisions went
against the AI's recommendation** — and they were right. Had the AI simply
proposed a design and started building, all three would have gone the other
way, and each would have been expensive to reverse once dozens of schemas
depended on it.

Practical notes:

- **One question at a time.** A batch of five questions gets one skimmed
  answer.
- **Insist on a recommendation with each question.** "What do you prefer?"
  wastes the AI's judgement; "here's my recommendation and why, do you
  agree?" surfaces reasoning you can push back on.
- **Facts are looked up, decisions are asked.** Anything discoverable from
  the filesystem or the live rig shouldn't become a question.

## 3. Phase 2 — spec, then tickets

**The spec** captures the problem statement, the solution, user stories, and
— most usefully — an **Implementation Decisions** section that freezes the
grilling outcomes, plus an **Out of Scope** section so later sessions don't
quietly expand the work.

**The tickets** break it into **vertical slices**: each one cuts a complete
path through the design and is demoable on its own, rather than being a
horizontal layer. The very first ticket is instructive — *"reads a schema's
test-definition file, finds a single test case, turns it into a real
collected pytest Item, executes it against the live BMC, and shows up as a
normal report row."* That's the whole stack, for exactly one read operation.

Each ticket carries:

```markdown
**What to build:** the end-to-end behaviour, from the user's perspective
**Blocked by:** other tickets, or "None — can start immediately"
**Status:** ready-for-agent

- [ ] Acceptance criterion 1
- [ ] Acceptance criterion 2
```

Two things that mattered in practice:

- **Sizing to one context window.** A ticket that doesn't fit gets split.
  The engine took roughly 48 tickets in total.
- **Explicit blockers.** So any session can pick up the frontier — any
  ticket whose blockers are all done — without re-deriving the order.

## 4. Phase 3 — implement one ticket at a time

One ticket per session, then commit. The verification discipline is what
makes the output trustworthy, and in this project it is unusually strict for
a specific reason: **there is no mock BMC.** Every test hits real, shared
hardware.

The rules that earned their place:

- **Explore the live rig before trusting the existing code.** Repeatedly, an
  older hand-written test's assumptions were simply wrong — see §8.
- **A dry-run collection check is not proof.** It catches structural/config
  mistakes, and nothing else. It cannot see a step referencing the wrong
  data, a typo'd placeholder, or a code path that doesn't exist.
- **Verify with the narrowest thing that proves it.** One test, then the
  file, then the suite. A full-suite run on flaky shared hardware produces
  noise that buries the signal.
- **Run the engine's own self-test suite before committing any change to
  the engine itself.** Roughly 200 tests against a synthetic sandbox — the
  only way to change the engine without a rig.
- **Confirm before anything hard to reverse.** Deleting a resource the test
  didn't create, a reboot/factory-reset, toggling a live service, pushing.

## 5. Phase 4 — record what was learned, not just what was done

Three artifacts, updated **in the same commit as the work** rather than
reconstructed later:

| Artifact | Holds |
|---|---|
| a living status ledger | one row per ticket: status **and what was actually found** — including where the ticket's own premise turned out wrong |
| a running design-history narrative | one paragraph per ticket: the *reasoning*, so a future reader knows why a mechanism exists |
| the commit message | the full account — what changed, what was measured, what was left undone and why |

The commit messages here are long on purpose. A one-line message optimises
for the writer; these optimise for whoever asks "why is this like this?" six
months on, which is the actual expensive question.

**Record findings, don't fix them into invisibility.** When the hardware is
genuinely wrong, the house style is to write the test as though it worked,
let it fail honestly, and comment the finding with enough detail to
reproduce it. A green report that hides a firmware bug is worse than a red
one that names it.

## 6. Durable context: project memory and skills

Two mechanisms stop each session from rediscovering the same things.

**Per-directory context notes.** Each area of the codebase carried its own
short, auto-loaded note recording that area's conventions and, more
valuably, **the traps already paid for** — the kind of thing that otherwise
gets rediscovered the hard way every few months.

**Checked-in project skills** — encoding verification discipline rather than
personal preference:

| Skill | Job |
|---|---|
| single-test-case authoring | add one test case (or one step) to an existing schema's coverage |
| whole-schema migration | port a whole schema off the old style: explore the live rig → build → verify → clean up |

The rule of thumb for what belongs in these: **if you find yourself
explaining the same thing a third time, write it down where the AI will read
it automatically.**

## 7. Rules that did the heavy lifting

- **Surface disagreement early, then defer.** If the AI thinks the approach
  is wrong, it says so once, clearly — then does what was asked. Repeated
  re-litigation wastes the session; silent compliance ships bad decisions.
- **Corrections mid-task are normal.** Several mechanisms here reached their
  final shape only after three or four rounds of "no, do it this way" — one
  retry/drain design for a collection that renumbers its members on delete
  being the clearest case. Rapid correction beat a defended first draft
  every time.
- **Verify claims about the code against the code.** Names, line counts,
  "who calls this" — all checkable, so check rather than assert. Several
  documentation errors were caught this way (a class name that had never
  existed; a function presented as dead that a live verification path
  actually depended on).
- **Say what wasn't done.** Deferred work, unverified assumptions, and
  environment problems get stated plainly, not omitted because the summary
  reads better without them.

## 8. What this caught

Concrete things the loop surfaced that a build-it-and-move-on approach would
have shipped. These are the return on the process:

| Finding | How the process caught it |
|---|---|
| Three structural design decisions | grilling — all three overrode the AI's recommendation |
| A hand-written suite whose core assumptions about a collection endpoint were all wrong (creation order, a required field its own payload omitted, ids renumbering on delete) | "explore the live rig before trusting the old test" |
| A compliance file that validated **nothing** — every property left unmarked, so garbage data passed clean | proving a check can fail before trusting that it passes |
| A write-verification check that reported PASSED while the resource was, in fact, unmodified | a manual replay of the same request, because the pass looked too easy |
| A whole class of verification checks silently dead for years on one write path | traced to its cause from the finding above |
| A ticket whose own premise was wrong — following it literally would have broken the engine's self-test suite | verifying the premise instead of executing the ticket |
| Reference data files asserting properties the hardware has never actually had | live comparison during conversion |
| A documented class name that never existed anywhere in the code | cross-checking every documented symbol against the source |

## 9. Tooling and credits

**Claude Code** ([docs](https://docs.claude.com/en/docs/claude-code)) — the
agent. Its automatic per-directory context loading and checked-in project
skills are what make the durable-context part of this workflow possible.

**AI Hero — <https://www.aihero.dev/>** (Matt Pocock) — the source of the
workflow skills this process is built on. The grill → spec → tickets →
implement loop in §§2–4 is his, and the *grill-first* insight is the one
that changed the most about how this project was built. His skills used
here:

| Skill | Used for |
|---|---|
| `grilling` / `grill-me` | Phase 1 — relentless one-question-at-a-time interrogation of a design |
| `to-spec` | Phase 2 — synthesising the conversation into a spec |
| `to-tickets` | Phase 2 — breaking a spec into vertical-slice tickets with blockers |
| `implement` | Phase 3 — work a spec/ticket with TDD at agreed seams |
| `code-review` | Phase 3 — two-axis review: repo standards, and spec conformance |
| `writing-great-skills` | Phase 4 — writing this project's own skills |

Related reading:

- [Anthropic — Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Agent skills documentation](https://docs.claude.com/en/docs/agents-and-tools/agent-skills)

---

## The short version

**Grill the design before writing code — and expect to overrule the AI.**
Then write it down (spec → tickets), do one slice at a time, verify against
reality rather than against your own expectations, and record what you
learned in the same commit. The artifacts on disk, not the conversation, are
what carry the work between sessions.
