# Codebase Readiness

A Claude Code skill for assessing a codebase's readiness for AI-native development.

Implements the seven-dimension readiness model from [framework.ai-native-transformation.com/codebase-readiness](https://framework.ai-native-transformation.com/codebase-readiness). Given a codebase, it produces a scorecard (1–5 per dimension), a readiness level (set by the lowest score, not the average), and a prioritized remediation plan.

---

## What it measures

| # | Dimension | Question it answers |
|:-:|---|---|
| 1 | Test coverage and feedback latency | Can the agent get fast, useful signal on whether changes break things? |
| 2 | Type strictness | Can the agent reason about inputs and outputs without reading everything? |
| 3 | File size and context legibility | Can the agent understand one file independently? |
| 4 | Module boundary clarity | Can the agent modify one thing without breaking another? |
| 5 | API directness | Are API calls visible at the call site, or hidden behind abstractions? |
| 6 | Documented intent | Can the agent distinguish intentional behavior from historical accident? |
| 7 | Observability and operational legibility | Can failures be localized and reproduced? Is dev-to-deploy frictionless? |

The framework page linked above explains each dimension, the scoring rubric, and why the dimensions together predict whether AI agents can produce reliable work on a codebase.

---

## Why this exists

AI coding agents amplify whatever structure already exists in a codebase. In brownfield codebases, the agent's output quality is constrained by the infrastructure surrounding it, not the model's capability. This is the [DORA 2025 "amplifier" finding](https://dora.dev/dora-report-2025/) and [Fowler's Harness Engineering](https://martinfowler.com/articles/harness-engineering.html) conclusion: **Agent = Model + Harness**.

Readiness tools exist for individual dimensions — coverage reporters, type checkers, dependency graphers, telemetry stacks. None integrate them into a single AI-native readiness assessment. This skill does.

---

## Installation

```bash
git clone https://github.com/zoyth/codebase-readiness ~/dev/codebase-readiness
mkdir -p ~/.claude/skills
ln -s ~/dev/codebase-readiness/.claude/skills/assess-codebase-readiness.md ~/.claude/skills/
```

The symlink makes `/assess-codebase-readiness` available in any Claude Code session.

Alternative: drop the skill directly into any project's `.claude/skills/` directory to scope it to that project.

---

## Usage

From inside the codebase you want to assess:

```
/assess-codebase-readiness
```

The skill will:

1. Detect the repo's language(s) and framework(s).
2. Collect signals per dimension (coverage reports, type configs, file size distribution, dependency graph, telemetry libraries, etc.).
3. Read representative files where qualitative judgment is needed (especially dimensions 5 and 6).
4. Score each dimension 1–5 conservatively with cited evidence.
5. Output a structured report with a readiness level, scorecard, evidence per dimension, and prioritized remediation plan.
6. Recommend a brownfield strategy (remediate in place / strangler-fig / rebuild / isolate and bypass) based on the scorecard.

---

## Example output

```
# Codebase readiness assessment

Repo: acme-platform
Primary language: TypeScript
Framework(s): Next.js 15, React 19
Source files (approx): 287
Date: 2026-04-19

## Readiness level: Level 1 — Instrumented

Ceiling set by dimension 6 (Documented intent), scoring 1.

## Scorecard

| # | Dimension                            | Score | Evidence                                                      |
|:-:|--------------------------------------|:-----:|---------------------------------------------------------------|
| 1 | Test coverage and feedback latency   |   3   | Coverage: 64%. CI p50: 8 min.                                 |
| 2 | Type strictness                      |   4   | tsconfig strict: on. `any` count: 12 (0.3%).                  |
| 3 | File size and context legibility     |   4   | p50: 142 lines. Largest: 847 (src/admin/dashboard.tsx).       |
| 4 | Module boundary clarity              |   3   | 12 top-level modules; 37 boundary violations detected.        |
| 5 | API directness                       |   2   | Factory abstraction in 89% of API call sites.                 |
| 6 | Documented intent                    |   1   | No ADRs. 4 stale READMEs. No CLAUDE.md.                       |
| 7 | Observability and operational legibility | 3   | Structured logs. Dev setup: 3 cmds. Deploy: 1 cmd.         |

## Prioritized remediation
1. Document intent for top 5 critical modules — establish ADR process, add CLAUDE.md.
2. Refactor Factory abstraction in 3 high-traffic modules to direct API calls.
3. Enforce module boundaries via lint rules; resolve the 37 existing violations.

## Recommended mode: Remediate in place

The architecture is fundamentally sound (types and tests are decent). The gap is in intent
capture and API legibility — both remediable in place without structural rebuild.
```

---

## Design notes

**Why Claude Code skill (not CLI, not CI tool).** This is an AI-assisted assessment. Dimensions like "documented intent" can't be scored by static analysis alone — they require an LLM reading modules and judging whether intent is legible. Making this a skill puts the assessment where the agent already is, rather than fighting to reinvent the runtime.

**Why the lowest score sets the ceiling.** Agents fail at the weakest link. A codebase with six 5s and one 1 cannot support Rung 5 working mode — the one gap is where the agent produces confident wrong output. Averaging hides this.

**Why signal-then-judge, not pure static.** Static-only tools are reproducible but miss the hardest dimensions. LLM-only tools are deep but non-deterministic. The skill's structure — static signals as evidence, LLM interpretation for scoring — balances both.

---

## Roadmap

- **v1 (this release):** scorecard + remediation plan + mode recommendation.
- **v2:** `/remediate-readiness <dimension>` — takes a dimension, generates concrete PRs (add ADR for module X, refactor Factory at file:line, etc.).
- **v3:** `--save` flag to track scores over time in `.claude/readiness-history.json`; trend graphs.
- **v4:** headless CI mode; JSON output for dashboards.

---

## License

MIT. See [LICENSE](./LICENSE).

---

## Credits

Based on the codebase readiness model at [framework.ai-native-transformation.com](https://framework.ai-native-transformation.com/codebase-readiness), which synthesizes research from:

- [AI Codebase Maturity Model (ACMM)](https://arxiv.org/abs/2604.09388)
- [Harness Engineering for Coding Agent Users (Fowler)](https://martinfowler.com/articles/harness-engineering.html)
- [AI Development Patterns (Paul Duvall)](https://github.com/PaulDuvall/ai-development-patterns)
- [DORA 2025 State of AI-assisted Software Development](https://dora.dev/dora-report-2025/)
- [Research, Review, Rebuild (Fowler / EPAM)](https://martinfowler.com/articles/research-review-rebuild.html)
