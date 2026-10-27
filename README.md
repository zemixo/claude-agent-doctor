# claude-agent-doctor

<img src="https://raw.githubusercontent.com/zemixo/claude-agent-doctor/main/quiet-orbit-banner.svg" alt="Agent Doctor: Find the pattern. Review the fix. Zemixo pixel robot workshop." width="100%">

Local diagnostics for Claude Code session logs. Inspect cost, loop, and
tool-use patterns with twelve heuristic rules, then review the suggested fixes
yourself. Zero analysis tokens. No network requests.

[![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![CI](https://github.com/zemixo/claude-agent-doctor/actions/workflows/ci.yml/badge.svg)](https://github.com/zemixo/claude-agent-doctor/actions/workflows/ci.yml)

[Quick start](#30-seconds-from-install-to-first-finding) · [Commands](#all-the-commands) · [Catalog](#the-catalog) · [Boundaries](#what-it-doesnt-do)

---

## When you'd want this

- Your Claude bill or Max usage feels higher than last week and you can't tell why.
- You opened a session this morning, came back after lunch, and it's still going.
- You run multiple subagents and couldn't tell me which one eats the most.
- `/cost` shows the current session. You want to know the last 7 days.

If none of those sound like you, you probably don't need it yet.

## 30 seconds from install to first finding

Install the current source with [Bun](https://bun.sh/) and Node.js 18 or newer.
The npm package endpoint is not currently available; use the source checkout:

```bash
git clone https://github.com/zemixo/claude-agent-doctor.git
cd claude-agent-doctor
bun install --frozen-lockfile
bun run build
bun run link
doctor check
```

That's it. It reads the JSONL files Claude Code already writes locally,
runs them through 12 rules (full catalog: [PATHOLOGY.md](PATHOLOGY.md)),
and for each rule that fires prints:

- **which session / which agent** triggered it
- **evidence from your own data** — real numbers, not guesses
- **a one-paragraph fix** you can copy-paste

No API calls. Nothing uploaded. Nothing on your machine changes — doctor
just reads and prints.

## All the commands

```bash
doctor check                  # scan the last 7 days
doctor check --days 30        # scan the last month
doctor check <session-id>     # deep-scan one session
doctor explain                # browse the catalog (12 pathologies)
doctor explain BASH_STORM     # one pathology, in detail
doctor suggest-routing        # per-subagent model suggestions
```

Output is colorized in a TTY and plain when piped. JSON and Markdown are
available with `--format json` / `--format md`.

`--days` applies an inclusive cutoff to each assistant turn, not to the session
as a whole. Turns without a valid timestamp are omitted from windowed reports.
Invalid options, categories, formats and day counts return exit code `2` before
scanning logs. Day counts must be whole numbers from `1` to `36500`.

Routing suggestions use observed Agent/Task invocation counts in the selected
window. Heavily used agents receive more conservative suggestions. Changes are
proposed per agent, so protected roles retain their own model settings.

## Example output

Illustrative report format with an example project path. These shadow-cost
figures are not invoice totals, a benchmark, or a promise of savings:

```
Doctor report  ·  54 sessions  ·  $20,027 total shadow spend

  15 high · 25 med · 1 low
  potential weekly savings: $2,889

│ RUNAWAY_SESSION  [cost · high]
│ session example01  ~/Code/example-project
│
│ A single session consumed > 25% of weekly spend.
│
│ Detected:
│   session cost: $6,247
│   share of weekly total: 31.2%  (threshold: 25%)
│   turns: 5052
│
│ Fix:
│   Inspect the transcript:
│     agent-ledger explain example01
│   Add a guardrail to your agent config:
│     max_turns: 80
│     max_cost_usd: 50
│
│ doctor explain RUNAWAY_SESSION  # mechanism & full case study
```

Every finding has four parts: a rule, evidence from *your* data, a
copy-pasteable fix, and a link to the rule's mechanism. If you don't agree
with the rule, skip it — `doctor` doesn't change anything on your machine.

## The catalog

Twelve rules ship in v0.3, in three categories. Full mechanisms and case
studies in [PATHOLOGY.md](PATHOLOGY.md).

| Code | Category | Severity | Summary |
|------|----------|----------|---------|
| `MODEL_MONOCULTURE` | cost  | high | One model dominates > 95% of spend |
| `HAIKU_NEGLECT`     | cost  | med  | Haiku used in < 5% of sessions |
| `RUNAWAY_SESSION`   | cost  | high | A single session consumed > 25% of weekly spend |
| `CACHE_TTL_MISMATCH`| cost  | med  | Short session, heavy 1h cache writes |
| `BASH_STORM`        | loops | high | > 500 Bash calls in one session |
| `SUBAGENT_SPRAWL`   | loops | med  | > 8 distinct subagents per session |
| `LOOP_DEATH`        | loops | high | Same tool fired 8+ consecutive turns (**v0.3**) |
| `RETRY_THRASH`      | loops | med  | One file both heavily read and heavily edited (**v0.3**) |
| `EDIT_THRASH`       | tools | med  | Same file edited > 10 times in one session |
| `NO_TOOL_BURN`      | tools | low  | > 40% of spend in no-tool turns |
| `CONTEXT_BLOAT`     | tools | med  | Peak turn input > 150K tokens (**v0.3**) |
| `TOOL_CALL_STORM`   | tools | low  | Single turn fires 12+ tool_use blocks (**v0.3**) |

## `suggest-routing` — config advisor (v0.2)

`doctor suggest-routing` reads every subagent definition under
`~/.claude/agents/` (and `<project>/.claude/agents/`), infers intended role
from the frontmatter, and proposes per-agent model changes.

Two things make this different from a runtime router like
[musistudio/claude-code-router](https://github.com/musistudio/claude-code-router):

1. **No proxy.** We don't sit in front of the API. You apply suggestions
   yourself; nothing in your request path changes.
2. **Guarded by default.** Orchestrators, payment / subscription agents,
   safety / compliance auditors, and reality checkers are never auto-
   downgraded. The blast radius of a mis-route on those is too large.

Illustrative routing plan; evaluate model quality before applying a change:

```text
Routing suggestions

  1 change suggested:

  docs-reader               sonnet → haiku  high conf  [user]
    ~/.claude/agents/docs-reader.md
    read-only work signals (search, fetch, log) — candidate for a cheaper model; validate its output quality

  15 agents left unchanged (guarded roles or low-signal descriptions).

Safer first experiment
  Review the agent-specific suggestions above.
  Keep orchestrator, payment, and safety roles at their current tier;
  a global model override would bypass those guards.
```

Export as a unified diff and apply yourself:

```bash
doctor suggest-routing --export-patch > routing.patch
patch -p0 < routing.patch
```

## How it relates to `/cost` and `agent-ledger`

- **`/cost`** (built-in) tells you the current session's cost. That's it.
- **[agent-ledger](https://github.com/zemixo/claude-agent-ledger)** aggregates past sessions into a ledger (per subagent, per model, per day).
- **`doctor`** reads the same files and looks for *patterns worth fixing*.

Think of it as: ledger is `top(1)`, doctor is `pylint`. They don't overlap;
they answer different questions.

You don't need ledger installed to use doctor — doctor reads JSONL directly.
But if you do have ledger, `doctor`'s session-level prescriptions will
reference `agent-ledger explain <id>` for deeper forensics.

## What it doesn't do

- **Doesn't call the API.** Ever. Not for analysis, not for LLM-assisted
  diagnosis. Zero tokens.
- **Doesn't change your config.** Prescriptions are suggestions. You apply
  them by hand.
- **Doesn't upload anything.** Not even anonymized telemetry. If you want
  to share a finding, `doctor check --format md > report.md` and you
  decide what to do with it.
- **Doesn't replace code review or good judgment.** A pathology is a
  hypothesis about your workflow. Your context may make the "fix" wrong.

## Development

```bash
git clone https://github.com/zemixo/claude-agent-doctor
cd claude-agent-doctor
bun install
bun test
bun run build
bun run link
doctor check
```

## Contributing a new pathology

See [CONTRIBUTING.md](CONTRIBUTING.md). A good pathology proposal has: a
deterministic detection rule, a mechanism explanation, at least one real
case study, and a copy-pasteable fix. That's the bar.

## Status

The source manifest is `0.3.0`. The catalog includes twelve rules, and current
source also validates CLI inputs, filters usage by turn timestamp, and uses
observed subagent invocations for routing suggestions.

Cost estimates use the repository's pricing snapshot; they are not a verified
invoice reconciliation. Model prices and log formats can change.

Planned work includes per-project thresholds and additional agent log adapters.
These are roadmap items, not current features.

## Author

[Zemixo](https://github.com/zemixo). Also maintains
[agent-ledger](https://github.com/zemixo/claude-agent-ledger) (cost
attribution for Claude Code) and
[apple-presubmit-audit](https://github.com/zemixo/apple-presubmit-audit)
(evidence-labeled App Store submission checks).

## License

MIT.
