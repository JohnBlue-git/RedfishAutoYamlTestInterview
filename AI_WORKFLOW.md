<style>
h1, h2, h3, h4, h5, h6 {
  color: #1976d2 !important;
}
</style>

# Working with an AI on a design this size

How the engine in [`DESIGN.md`](DESIGN.md) was built with an AI — the
*process*, not the design. Written down because the method transferred
better than any single output: the same loop built the engine, migrated the
old tests, and produced these docs.

## Contents

| § | Section |
|---|---|
| 1 | [Three loops, not a line](#1-three-loops-not-a-line) |
| 2 | [The AI setup](#2-the-ai-setup) |
| 3 | [Grill the design before any code](#3-grill-the-design-before-any-code) |
| 4 | [Spec, then vertical-slice tickets](#4-spec-then-vertical-slice-tickets) |
| 5 | [Three gates that keep it safe](#5-three-gates-that-keep-it-safe) |
| 6 | [What this caught](#6-what-this-caught) |
| 7 | [Records](#7-records) |
| 8 | [Open problems](#8-open-problems) |
| 9 | [Credits](#9-credits) |

## 1. Three loops, not a line

```
Feature loop      idea → /grill-me → /to-spec → /to-tickets → tickets on disk
Ticket loop       a ticket → /implement → verify live → /code-review → outcome
Improvement loop  a mistake or a correction → a rules line, a skill edit, or memory
```

Each ticket's outcome feeds back somewhere:

```
  ├─ done as planned        → update ticket status
  ├─ found something new    → a new ticket
  ├─ ticket premise wrong   → re-grill that ticket, update spec / tickets
  ├─ AI repeated a mistake  → a rules line, a skill edit, or memory
  └─ whole direction wrong  → back to /grill-me
```

**Every stage produces a durable file on disk.** Nothing depends on
conversation memory surviving, which matters when the work spans many
sessions and context windows.

**What each workflow skill does.** Each one picks up from what the one
before it left on disk:

| Skill | What it does | What it leaves behind |
|---|---|---|
| `/grill-me` | Asks about a plan one question at a time, each with its own recommended answer, until every decision is made. Looks facts up itself and asks you only for decisions. | Only the conversation, which is why `/to-spec` runs straight after it |
| `/to-spec` | Writes up what the grilling settled: the problem, the solution, user stories, implementation decisions, and the test seams. | a spec file |
| `/to-tickets` | Splits the spec into thin end-to-end slices, each verifiable on its own, sized to one session, naming the tickets that block it. | one file per ticket |
| `/implement` | Does one ticket: reads it, makes the change, runs tests along the way, then calls `/code-review`. | the change, in the working tree |
| `/tdd` | Test first: one failing test at an agreed seam, then the code to pass it, then the next. | tests next to the change |
| `/code-review` | Reviews the diff with two sub-agents in parallel: **Standards** (against the repo's written rules) and **Spec** (against the ticket or requirement). | a findings report; it changes no code |

## 2. The AI setup

The AI is Claude Code. What it knows about the project comes from Markdown
files it reads automatically, in layers:

```
 personal (your machine)          in git (every teammate)
 ├─ global rules                  ├─ project rules (root CLAUDE.md)
 ├─ global workflow skills        ├─ per-folder rules (<folder>/CLAUDE.md)
 └─ memory                        ├─ project skills (.claude/skills/)
                                  └─ skill adapters (.claude/agents/)

        all of it feeds → Claude Code (the main agent), which
           ├─ spawns sub-agents   fresh AIs for parallel search or review
           ├─ reads and writes    the feature's spec, tickets and ledger on disk
           └─ talks to            the live BMC (curl, pytest)
```

- **Per-folder rules** record each area's conventions and, more valuably,
  the traps already paid for — so no session rediscovers them.
- **Project skills** encode repeatable jobs with verification built in:
  writing a test case from a plain-language description or a curl command,
  onboarding a new platform, refreshing an existing platform's inventory
  from its live rig (read-only, each change adopted deliberately),
  regenerating compliance data from the spec.
- **Skill adapters** are short notes telling the global workflow skills where
  this repo keeps its tickets and docs, so they write to the right place.
- **Rule of thumb:** if you explain the same thing a third time, write it
  down where the AI will read it automatically.

**Getting the same setup.** Cloning gives a teammate every in-git layer. The
personal layers they add themselves: copy the workflow skill set into their
own skills folder (the adapters are already in git), write their own global
rules — the author's has four: think before coding, simplicity first,
surgical changes, and goal-driven execution with a checkable definition of
done — and let memory build up as they correct the AI.

### What goes in, what comes out

Everything the agent knows comes in as files or messages, and everything it
does goes out as files or messages. It keeps nothing between sessions except
what it writes to a file.

```
        INPUT                                            OUTPUT
┌────────────────────────┐                         ┌────────────────────────┐
│ Rules                  │─┐                    ┌─►│ Work                   │
│ root + per-folder      │ │                    │  │ code · test YAML       │
│ CLAUDE.md              │ │                    │  │ compliance data · docs │
└────────────────────────┘ │                    │  └────────────────────────┘
┌────────────────────────┐ │                    │  ┌────────────────────────┐
│ Adapters               │─┤                    ├─►│ Ticket state           │
│ where tickets, labels  │ │                    │  │ status · checkboxes    │
│ and docs live          │ │                    │  │ comments · ledger      │
└────────────────────────┘ │                    │  └────────────────────────┘
┌────────────────────────┐ │                    │  ┌────────────────────────┐
│ Skills                 │─┤                    ├─►│ To you                 │
│ project + global       │ │  ╔══════════════╗  │  │ questions · proof      │
│                        │ │  ║              ║  │  │ gates 1 / 2 / 3        │
└────────────────────────┘ │  ║   AI AGENT   ║  │  └────────────────────────┘
┌────────────────────────┐ ├─►║              ║──┤  ┌────────────────────────┐
│ Tickets & spec         │─┤  ║ Claude Code  ║  ├─►│ To the BMC             │
│ one folder per feature │ │  ║              ║  │  │ curl · pytest          │
│                        │ │  ╚══════════════╝  │  │ after you approve      │
└────────────────────────┘ │                    │  └────────────────────────┘
┌────────────────────────┐ │                    │  ┌────────────────────────┐
│ Memory                 │─┤                    ├─►│ Lessons                │
│ personal               │ │                    │  │ memory (just you)      │
└────────────────────────┘ │                    │  │ CLAUDE.md · skills     │
┌────────────────────────┐ │                    │  └────────────────────────┘
│ You & the rig          │─┘                    │  ┌────────────────────────┐
│ requests · answers     │                      └─►│ Sub-agents             │
│ BMC responses          │                         │ search · review        │
└────────────────────────┘                         │ → one report back      │
                                                   └────────────────────────┘
```

Read it in two directions:

- **Left to right, one session.** The agent reads the left side, does the
  work, and produces the right side.
- **Right back to left, across sessions.** The right side becomes the next
  session's left side: an updated ticket, a new rules line, a saved memory.
  That loop is the only way the agent "remembers". If it skips writing
  ticket state, the next session believes the ticket is still open.

Memory is the one personal piece. A correction saved only there changes only
your agent; if a teammate's agent should follow it too, it goes in a
`CLAUDE.md` line in git.

## 3. Grill the design before any code

The single highest-leverage step. Before any code, the design was
interrogated one question at a time — execution model, config format, file
layout, platform hierarchy, risk gating, reporting, migration — each with a
recommended answer, waiting for a decision before moving on.

**Three core structural decisions went against the AI's recommendation, and
they were right.** Had the AI simply proposed a design and started building,
all three would have gone the other way, and each would have been expensive
to reverse later.

- **One question at a time.** A batch of five gets one skimmed answer.
- **Ask for a recommendation with each question.** It surfaces reasoning you
  can push back on.
- **Facts are looked up, decisions are asked.** Anything discoverable from
  the code or the rig shouldn't become a question.

## 4. Spec, then vertical-slice tickets

**The spec** freezes the grilling outcomes as *Implementation Decisions*,
plus an *Out of Scope* section so later sessions don't quietly expand the
work.

**The tickets** are vertical slices — each cuts through the whole stack and
is demoable on its own. The first ticket: *read one YAML test case, turn it
into a real pytest Item, run it against the live BMC, and show it as a
report row.* The whole stack, for one read operation.

- **Size each ticket to one context window.** Too big → split it.
- **Explicit blockers.** Any session can pick up a ticket whose blockers are
  all done, without re-deriving the order.

## 5. Three gates that keep it safe

There is no mock BMC — every test hits real, shared hardware. Four failure
modes came up when implementing with an AI:

1. **The review misses something** — the change looks right and is wrong.
2. **"All pass", but it's failing** — a smaller run than claimed, an old
   report, skipped tests counted as passes, the wrong rig.
3. **A conversion silently drops a step**, or writes Python where a
   declarative field already does the job.
4. **A dangerous action** — e.g. locking out the account the suite logs in
   with.

Knowing pytest and Redfish helps a person *notice* these; it doesn't
*prevent* them. A fixed procedure does:

**Gate 1 — before a run against a rig.** The user is the safeguard: nothing
in the tooling blocks a dangerous request, so read what the AI is about to
send and don't auto-approve commands that touch a rig. Risky tests carry a
risk category; confirm anything hard to reverse.

**Gate 2 — before claiming a pass: show the proof, not just "passed".** The
AI must show four things:

1. the command it ran
2. which rig it ran on
3. pytest's final summary line, copied exactly
4. the report file name

These **don't** count as a pass: a collection-only dry run (nothing is sent
to the BMC), a smaller run than claimed, and skipped tests. Four things can
be checked in seconds instead of taking the AI's word for it. Where it
matters, prove the check *can* fail — a check that can't fail passes forever.

**Gate 3 — before committing.** Run `/code-review`, passing in the
requirement when it lives outside the repo — without it the Spec reviewer has
nothing to check against, and a dropped step is exactly what nobody catches.
A person reviews the result and re-implements if needed. Engine changes need
the self-test suite green.

Rules that did the heavy lifting:

- **Surface disagreement once, then defer.**
- **Verify claims about the code against the code** — names, callers, counts.
- **Say what wasn't done.** Deferred work and unverified assumptions are
  stated, not omitted.
- **Let a firmware bug fail honestly** rather than loosening the assertion.

## 6. What this caught

| Finding | Caught by |
|---|---|
| Three structural design decisions | grilling — all three overrode the AI's recommendation |
| An old test whose core assumptions about an endpoint were all wrong | exploring the live rig before trusting existing code |
| A compliance file that validated **nothing** — garbage data passed clean | proving a check can fail before trusting that it passes |
| A write check reporting PASSED while the resource was unchanged | a manual curl replay, because the pass looked too easy |
| A ticket whose premise was wrong — following it would have broken the engine | verifying the premise instead of executing the ticket |
| A "skipped" request variant that was still sent when reached through a sequence — including host resets | the engine audit below, proven live |
| A generated compliance check that silently didn't run on some responses, leaving the row PASS | the engine audit below |

### A read-only audit of the engine

Once the engine had grown to about 4,800 lines across 14 modules, nobody
had read it end to end, and its worst past bugs had all been *silent*: a
check that verified nothing still reported PASS. So the same loop ran on the
engine itself:

1. **Grill** the audit's scope and ranking: silent false passes first, then
   docs that no longer match the code, then gaps in the self-tests.
2. **Audit read-only.** Each finding records a claim, a concrete scenario,
   the code path as evidence, and a proof (by reading, or a live run the
   maintainer approved). Nothing was changed or committed.
3. **The maintainer decides.** 59 findings: 55 accepted, 4 rejected.
4. **Accepted findings become tickets,** grouped by shared fix (15
   tickets), then go through the normal ticket loop with gates 1–3.

Result: 24 of the 26 tickets from the audit and its follow-ups are done, and the engine's self-test
suite went from about 215 to 523 tests, each one pinning down behaviour that
used to be unchecked.

## 7. Records

Every ticket is a file on disk with a status, so progress can be counted
rather than remembered. As of 2026-10-20:

| Feature (one grill → spec → tickets pass) | Done / total |
|---|---|
| The config-driven engine, including migration of every schema and the engine audit | 53 / 79 (12 more partly done) |
| Documentation passes (design, AI workflow, code guide, YAML guide) | 18 / 18 |
| Property-compliance redesign (generating compliance data from the specs) | 11 / 21 (4 more partly done) |
| Validator-comparison docs | 2 / 5 (3 more partly done) |
| Live-findings log (where firmware and spec disagree) | 0 / 9 |
| **Total** | **84 / 132** |

Other records:

- **Engine:** 14 modules, about 5,500 lines, 523 self-tests run in a
  synthetic sandbox.
- **Tests:** 244 test cases, 271 named request variants, and 36 sequences
  covering 72 Redfish schemas in 42 resource areas, on 12 platforms.
- **Project skills:** 4, each with verification built in.
- The live-findings tickets record findings rather than code work; several
  are waiting for a person to raise them with the firmware or spec owners.

## 8. Open problems

- **Several people, each with an AI, at once.** Two sessions can claim the
  same ticket or change the same shared rig under each other.
- **A visual editor for the YAML.** Postponed; an AI authoring skill fills
  the gap today.
- **Where live findings are recorded.** Moving from in-file comments to
  tracked issues.

## 9. Credits

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** — the
  agent. Per-directory rules and checked-in skills make the layered setup
  possible.
- **[AI Hero](https://www.aihero.dev/)** (Matt Pocock) — the workflow skills
  this process is built on (`grill-me`, `to-spec`, `to-tickets`,
  `implement`, `tdd`, `code-review`). The *grill-first* insight changed the
  most about how this project was built.
- Further reading: [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)

---

**The short version:** grill the design before writing code — and expect to
overrule the AI. Write it down (spec → tickets), do one slice at a time,
verify against reality, and feed what you learn back into the rules the AI
reads. The files on disk, not the conversation, carry the work between
sessions.
