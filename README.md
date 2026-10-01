# Agent Pack — SDLC Pipeline for Claude Code

Seven [Claude Code subagent](https://docs.claude.com/en/docs/claude-code/sub-agents) definitions that
split a software change across the roles of a delivery team: product, architecture, code, tests,
deployability, and an independent quality gate — plus an `orchestrator` that drives them end to
end as a state machine. A separate, standalone [`arquetype`](arquetype.md) agent covers working
inside existing and legacy codebases.

Each agent is a single Markdown file with YAML frontmatter (`name`, `description`, `tools`,
`model`) and a body that defines its **Description**, **Responsibilities**, **Constraints** and
**Output** shape. They are deliberately opinionated — they are starting points to edit, not
settled law.

---

## The agents

| Agent | Owns | Key constraint |
|---|---|---|
| [`product-owner`](product-owner.md) | The **what** and **why**: scope, user story, acceptance criteria | No technical design, no invented requirements |
| [`architect`](architect.md) | The **how**: impact map, contracts, ordered implementation plan | Verifies the repo before designing; writes no production code |
| [`developer`](developer.md) | Clean production code, one plan step at a time, plus branch & PR | Never merges or approves its own PR |
| [`testing`](testing.md) | Automated verification: unit, integration, contract tests | Never weakens a test to go green; signs off OK / NOT OK |
| [`implementation`](implementation.md) | Deployability: wiring, config, migrations, flags, rollout | No irreversible migrations; signs off OK / NOT OK |
| [`qa`](qa.md) | Independent verification against the original criteria | Reports GO / NO-GO; never fixes the code, never merges |
| [`orchestrator`](orchestrator.md) | Routing an intake through all six agents | Makes no decisions; follows a transition table and escalates to the human |

### Why Developer and Implementation are separate

`developer` owns business logic. `implementation` owns everything that makes the code *runnable* —
dependency wiring, config keys, DB migrations, topics and queues, feature flags, CI steps, rollout
and rollback. Splitting them keeps "it compiles" apart from "it ships safely".

### Why Testing and QA are separate

`testing` **writes** the automated suite. `qa` **independently verifies** the delivered change
against the original acceptance criteria, reviews the diff for defects, and assumes nothing works
until shown. One builds the safety net; the other doesn't trust it.

---

## The flow

```
product-owner  ──>  architect  ──>  developer  ──>  testing
   (scope,            (design,        (code,          (suite green +
    criteria)          plan)           branch, PR)     criteria covered)
                                           │                 │
                                           │                 v
                                           │          implementation
                                           │           (config, migrations,
                                           │            rollout ready)
                                           │                 │
                                           │                 v
                                           │                qa
                                           │          (GO / NO-GO)
                                           │                 │
                                           └─────────────────┘
                                             defects go back
```

**The merge gate.** `developer` opens its pull request as a **draft** and keeps it there until all
three sign-offs are in:

1. `testing` → **OK** (suite green *and* every acceptance criterion covered by an asserting test)
2. `implementation` → **OK** (config documented, migrations reversible, rollout + rollback defined)
3. `qa` → **GO**

No agent approves or merges the PR. The verdicts are reported; a human decides.

---

## The orchestrator

`orchestrator` takes an intake (one ticket, a list of tickets, a project brief, a spec) and runs
every resulting work item through the flow above as a **state machine**:

```
INTAKE -> SCOPING -> [GATE_SCOPE] -> DESIGN -> [GATE_PLAN] -> BUILD (step 1..n) -> TESTING
       -> DEPLOYABILITY -> QA -> [GATE_MERGE] -> next work item | DONE

NOT OK / NO-GO  -> BUILD (rework) -> TESTING -> DEPLOYABILITY -> QA   (max 3 cycles)
[GATE_*]        =  human decides, via a question in the session
anything unclear -> BLOCKED -> human picks where to resume
```

- **No agent decides on its own.** Transitions fire only on explicit signals from the agents'
  output contracts (`Sign-off: OK / NOT OK`, `Verdict: GO / NO-GO`, blocking open questions,
  build result). Scope, plan, waivers, outward git actions and the merge all go to a human gate.
- **Bounded loops.** Scoping, design and rework loops are capped at 3 rounds, then `BLOCKED`.
- **Run ledger.** Every transition, agent output and human answer is written to
  `.claude/orchestrator/runs/<id>.md`, so a run can be audited or resumed.

Subagents can't spawn subagents, so the orchestrator must run as the **main session agent**:

```bash
claude --agent orchestrator
> Here are this sprint's tickets: US-17327, US-17410 (descriptions below) ...
```

Its `tools:` line uses `Agent(product-owner, architect, ...)` so it can only call the pack's own
agents. Rename an agent and you must update that line too.

---

## Standalone: `arquetype`

[`arquetype`](arquetype.md) is **not** part of the pipeline above, and the orchestrator never
calls it. It is a single agent that works more like a skill. It is most useful when you drop it
into a project that already exists, especially a legacy codebase with little documentation.

It has two modes:

**1. Analyze.** Scans the project and writes **`arquetype-memory.md`** at the repo root: stack and
versions, build/run/test commands, directory map, architecture and layering, modules and
packages, domain model, entry points (endpoints, consumers, jobs), integrations, config key
names (never values), coding and test conventions, legacy hotspots, and a glossary. Every entry
cites a real file. Later runs refresh only what changed since the recorded commit, and sections
marked `<!-- manual -->` are left alone. Commit this file so the whole team shares the same
reference.

**2. Intake.** Takes a feature prompt, a ticket, a bug report, or any request about the project:

1. Reads `arquetype-memory.md` (and runs Analyze first if it's missing).
2. Finds where the change belongs and traces its impact on callers, contracts, DB, config and
   tests. For bugs, it locates the root cause.
3. **Stops and asks you** to approve, adjust or cancel. It shows the affected structure, the
   list of files to create, modify or delete, the tests to add or update, and an impact and
   risk analysis.
4. Implements the change and shows what changed in each file as it goes. Small changes appear
   as diffs. Large ones show the file name and 2–3 key points on what changed and why.
5. Writes new unit tests or updates existing ones (including a regression test for bugs),
   runs them, and reports the real result.
6. Updates `arquetype-memory.md` if the change added modules, dependencies, entry points or
   config.

It never edits files outside the approved list without asking again, and it never commits or
pushes unless you ask.

Because it stops to ask you questions, run it as the main session agent:

```bash
claude --agent arquetype
> analyze this project
> intake: add a `status` filter to the orders list endpoint
> intake: BUG-412 — invoices show the wrong currency for EU tenants
```

---

## Branch conventions

`developer` branches off the integration branch and names by change type:

| Change type | Prefix | Example |
|---|---|---|
| New functionality | `feature/` | `feature/US-17327-warehouse-lookup` |
| Defect in released behaviour | `bugfix/` | `bugfix/US-17410-null-tenant-header` |
| Small corrective / tech change | `fix/` | `fix/retry-backoff-config` |
| Urgent production defect | `hotfix/` | `hotfix/INC-204-payment-timeout` |

Target-branch flow: work merges into `dev`; `dev` → `test` is a promotion PR; `test` → `prod` is a
release PR; a `hotfix/` may target `prod` with a back-merge PR into `dev`.

---

## Install

**Per project** — available only in that repo, and shareable with the team through it:

```bash
mkdir -p <your-repo>/.claude/agents
cp *.md <your-repo>/.claude/agents/
```

**Per user** — available in every project on your machine:

```bash
# macOS / Linux
cp *.md ~/.claude/agents/

# Windows (PowerShell)
Copy-Item *.md $env:USERPROFILE\.claude\agents\
```

Project-level agents win over user-level ones when names collide. Restart or `/agents` to confirm
Claude Code picked them up.

## Usage

Claude Code selects an agent automatically when a task matches its `description`, or you can name
it directly:

```
> use the product-owner agent on ticket US-17327
> have the architect plan this, then the developer implement step 1
> run qa on the current diff
```

Or hand the whole thing to the orchestrator with `claude --agent orchestrator` (see above).

---

## Adapting the pack

Things worth changing before you rely on it:

- **Branch names.** The `dev` / `test` / `prod` flow and the prefix table are conventions —
  edit `developer.md` to match your real branches and ticket format.
- **`tools:`** in the frontmatter. Tighten or widen per agent; omit the line entirely to inherit
  every tool the session has. Note `qa` and `architect` are intentionally read-only (no `Write`/`Edit`).
- **`model:`** per agent. `opus` for the reasoning-heavy roles, `sonnet` for the mechanical ones;
  set `inherit` to just use the session's model.
- **Stack specifics.** The bodies avoid naming a language or framework. If you only ever work in one
  stack, pinning the real conventions (test framework, migration tool, layering rules) makes the
  agents noticeably sharper.
- **The gate itself.** Cut the three-way sign-off to a single reviewer if the ceremony is too heavy
  for your team's size.

## License

MIT — do what you like with it.
