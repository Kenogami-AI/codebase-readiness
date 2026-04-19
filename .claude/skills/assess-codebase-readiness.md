---
name: assess-codebase-readiness
description: Assess the current codebase's readiness for AI-native development against the 8-dimension model. Produces a codebase state (greenfield/brownfield/hybrid), a scorecard, a ceiling level, and a prioritized remediation plan or roadmap. Use when the user asks "is this codebase ready for agents?", "what would it take to AI-ify this?", or equivalents — or when the user runs /assess-codebase-readiness.
---

# Assess codebase readiness

You are assessing the current working directory's codebase against the eight-dimension readiness model defined at https://framework.ai-native-transformation.com/codebase-readiness.

Produce a structured report: the codebase's state (greenfield / brownfield / hybrid), a score 1-5 per dimension with evidence, an overall readiness level (set by the lowest score, not the average), and a prioritized remediation plan.

## Operating rules

1. **Actually run commands.** Use Bash to collect evidence. Don't guess at coverage or CI times — measure them.
2. **Cite evidence.** For every score, reference file paths, line numbers, or command output.
3. **Score conservatively.** If the signal is ambiguous, round down.
4. **The ceiling is the lowest score.** A codebase with seven 5s and one 1 is at the level of that 1. Agents fail at the weakest link.
5. **Apply deferral credit.** An intentional, documented deferral (in a spec, ADR, roadmap, or README) scores one level higher than the same gap undocumented. Never apply deferral credit based on a verbal claim — only on something in the repo.
6. **Every score below 4 must map to a concrete remediation item.** "Add tests" is not concrete. "Add test coverage for src/billing/invoice.ts (currently 0%)" is.
7. **Keep the output navigable.** Follow the output template exactly. No creative reformatting.

## Process

### Step 1 — Detect the codebase shape

Before scoring, establish:
- Primary language(s) and framework(s). Read `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `composer.json`, etc.
- Repo size: `find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.php" \) -not -path "*/node_modules/*" -not -path "*/dist/*" -not -path "*/build/*" -not -path "*/.next/*" | wc -l`
- Test infrastructure: presence of `tests/`, `__tests__/`, `spec/`, `*_test.go`, etc.
- CI config: check `.github/workflows/`, `.circleci/config.yml`, `.gitlab-ci.yml`, `azure-pipelines.yml`.

Report what you found in the opening section of the scorecard.

### Step 2 — Determine the codebase state

Classify the codebase as one of three states (see https://framework.ai-native-transformation.com/brownfield-strategy#first-is-your-codebase-actually-brownfield):

- **Greenfield** — new codebase, not yet in production or young enough that the architecture is still being actively designed. Low scores are usually scheduling decisions, not debt. Signal: short git history (<6 months of substantial commits), no legacy versions, pre-GA or MVP phase, explicit roadmap deferrals documented.
- **Brownfield** — production codebase with accumulated decisions, real users, years of patches. Gaps are debt. Signal: long git history, production for substantial time, multiple contributor generations.
- **Hybrid** — a greenfield service being built inside a brownfield org. The new service is greenfield; the integration layer to legacy is brownfield. Signal: clean modern stack with an awkward API boundary to a legacy system, or references to legacy services that can't be modernized.

State the classification and why.

### Step 3 — Score each dimension

Work through the eight dimensions in order. For each:
1. Run the signal-collection commands (see per-dimension sections below).
2. Sample representative files when the rubric calls for qualitative judgment.
3. Apply the scoring rubric.
4. Apply deferral credit if a documented deferral exists.
5. Record evidence with file paths and command output.

### Step 4 — Synthesize

- Compute the ceiling (lowest score).
- Map the ceiling score to a readiness level: 1 → Level 0 (Opaque), 2 → Level 1 (Instrumented), 3 → Level 2 (Validated), 4 → Level 3 (Legible) or Level 4 (Specified), 5 → Level 5 (Scenario-governed).
- Build the remediation plan: start with the lowest-scored dimensions. For each, produce concrete actions that would raise the score by one level.
- Produce a **path to Level 5** recommendation. The question is always: what is the shortest path from the current level to Level 5 for this codebase? Team capacity, strategic priorities, and resource allocation are explicitly out of scope — those are decisions the human makes after reading the recommendation, not inputs to it. Answer purely on the evidence.

  Four possible paths map to the four modes:
  - **Remediate in place** — Level 5 happens in this codebase via staged harness-building. Use when the architecture is sound and remediation cost is bounded.
  - **Strangler-fig** — Level 5 happens gradually; new pieces are built Level 5-ready and old pieces retired. Use when clean seams exist and continuity matters.
  - **Full rebuild** — Level 5 happens in a new version of this codebase. Use when the existing structure actively prevents the work.
  - **Isolate and bypass** — Level 5 does not happen in this codebase; it happens alongside in new Level 5-ready codebases. This codebase stays at its current level. Use when remediation cost exceeds the value remaining in the codebase.

  For greenfield state: the path is "continue development and close deferred gaps on project milestones." No brownfield mode applies yet.

  For hybrid state: the path has two components — the greenfield service continues development; each legacy boundary gets its own mode selection.

### Step 5 — Output

Produce the report exactly in the template below.

## Dimensions

### 1. Test coverage and feedback latency

**Signals to collect:**
- Coverage: run the test command with coverage (`npm test -- --coverage`, `pytest --cov`, `go test -cover`, `cargo tarpaulin`, `mvn test jacoco:report`, etc.) OR parse existing coverage report files if present (`lcov.info`, `coverage.xml`, `coverage-summary.json`).
- CI wall-clock time: inspect CI config for timeout budgets and parallelism, then check recent CI runs via `gh run list --limit 10 --json databaseId,status,conclusion,createdAt,updatedAt` if the repo is on GitHub.
- Test file count vs. source file count.
- Whether tests run on every commit/PR.

**Scoring rubric:**
- 1: No tests, or dead tests that fail.
- 2: Low coverage (<30%), slow or unreliable CI.
- 3: Meaningful coverage on hot paths, CI under 30 min.
- 4: Enforced coverage threshold in CI, CI under 5 min, failures localize.
- 5: Behavioral scenarios cover critical paths, CI fast and green as the deploy gate.

### 2. Type strictness

**Signals to collect:**
- TypeScript: read `tsconfig.json`, check `strict`, `noImplicitAny`, `strictNullChecks`. Count `any` occurrences: `grep -rn "\bany\b" --include="*.ts" --include="*.tsx" src/ | wc -l`. Count type-checker disable comments: `grep -rn "@ts-ignore\|@ts-expect-error" src/ | wc -l`.
- Python: look for `mypy.ini`, `pyrightconfig.json`, or `[tool.mypy]` in `pyproject.toml`. Check `strict = true`. Count `# type: ignore`: `grep -rn "# type: ignore" --include="*.py" | wc -l`.
- Go: types are always strict; score 4+ unless `interface{}` is pervasive.
- Untyped languages (JS, Ruby, raw Python): score 1 unless there's partial type annotation coverage via JSDoc/TypedDict/etc.

**Scoring rubric:**
- 1: Untyped language, or types disabled.
- 2: Types optional; `any` or equivalent is pervasive (>5% of type annotations).
- 3: Types enforced; some escape hatches.
- 4: Strict mode on; `any` rare and justified (<1%).
- 5: Strict mode enforced in CI; zero `any`; explicit contracts at every boundary.

### 3. File size and context legibility

**Signals to collect:**
- File size distribution: `find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.php" \) -not -path "*/node_modules/*" -not -path "*/dist/*" -not -path "*/build/*" -not -path "*/.next/*" -exec wc -l {} + | sort -n | tail -20`
- Compute p50, p95, max.

**Scoring rubric:**
- 1: God files common (2000+ lines), mixed responsibilities, p50 > 600.
- 2: Many 1000+ line files, responsibilities tangled.
- 3: p50 under 500 lines, some outliers over 1000.
- 4: p50 under 300 lines, outliers rare and justified.
- 5: Files sized to one responsibility each; largest files are intentional.

### 4. Module boundary clarity

**Signals to collect:**
- Directory structure: `find . -maxdepth 3 -type d -not -path "*/node_modules*" -not -path "*/.git*" -not -path "*/dist*" -not -path "*/build*" -not -path "*/.next*" | head -50`
- Dependency graph (if available): `npx madge --json src/` for JS/TS, `pydeps` for Python, or the language equivalent.
- Boundary violations: check for lint rules like `@typescript-eslint/no-restricted-imports`, Nx boundary rules, import-linter for Python.
- Sample cross-cutting concerns (auth, logging, data access): `grep -rn "import.*auth\|import.*logger\|import.*db" src/ | head -20`.

**Scoring rubric:**
- 1: No visible modular structure; everything reaches everything.
- 2: Structure on paper, violated in practice.
- 3: Clear high-level modules, occasional leaks.
- 4: Enforced boundaries (lint/check rules), stable contracts.
- 5: Module graph is a navigable map; architecture documented and current.

### 5. API directness

**Signals to collect:**
- Find data/API access patterns: `grep -rn "fetch\|axios\|\.request\|\.query\|\.exec\|HttpClient" src/ | head -30`.
- Detect Factory/Registry patterns: `grep -rn "Factory\|\.resolve\b\|\.dispatch\b\|\.invoke\b" src/ | head -20`.
- Sample 3 representative API-calling files and read them. Is the HTTP/SQL call visible at the call site, or hidden?

**Scoring rubric:**
- 1: All API access via opaque abstractions (Factory, dynamic dispatchers, ORM magic with invisible queries).
- 2: Significant abstraction; traceable with effort.
- 3: Mix of direct and abstracted; common patterns clear.
- 4: Mostly direct calls with typed responses; abstraction limited to cross-cutting concerns.
- 5: Call sites show what's happening; no hidden network or database operations.

### 6. Documented intent

**Signals to collect:**
- ADR/decision-log presence: `find . -type d -iname "adr*" -o -iname "decisions*" -o -iname "rfc*"`.
- Module-level docs: `find . -name "README.md" -not -path "*/node_modules/*"`.
- CLAUDE.md or AGENTS.md: `ls CLAUDE.md AGENTS.md 2>/dev/null`.
- Sample 5 non-trivial modules. Read the main file. Can you explain *why* it exists from what's documented, or only *what* it does from the code?

**Scoring rubric:**
- 1: No intent documentation. Code is the only truth.
- 2: Scattered docs, often stale. No ADR process.
- 3: Module-level READMEs, some ADRs.
- 4: Current ADR log, intent documented for critical modules, CLAUDE.md / AGENTS.md in place.
- 5: Every non-trivial decision has a spec or ADR; historical context preserved; documentation current enough to trust.

**Note:** this dimension is LLM-assisted by necessity. Static signals are weak. Read modules and judge.

### 7. Observability

**Signals to collect:**
- Telemetry libraries: `grep -r "opentelemetry\|datadog\|sentry\|newrelic\|honeycomb" package.json requirements.txt go.mod Gemfile composer.json 2>/dev/null`.
- Structured logging: `grep -r "winston\|pino\|structlog\|zap\|slog\|logrus\|log/slog" package.json requirements.txt go.mod 2>/dev/null`; sample a representative file and check for structured log output vs. `console.log` / `print`.
- Error tracking: Sentry, Bugsnag, Rollbar etc. presence.
- Metrics: Prometheus, StatsD, Datadog APM, OpenTelemetry instrumentation.

**Scoring rubric:**
- 1: No structured logs, no traces, no metrics, no error tracking.
- 2: Some logs, no tracing.
- 3: Structured logs and basic metrics; error tracking in place.
- 4: Full telemetry (logs + traces + metrics) with query access.
- 5: Production behavior fully observable; errors produce reproducible test cases; observability itself is part of the dev workflow.

### 8. Dev and deploy simplicity

**Signals to collect:**
- Dev setup: read `README.md` setup section or `Makefile`; count the commands required to go from fresh clone to running locally.
- Dev environment: check for `docker-compose.yml`, `devcontainer.json`, `Nix` config, or a setup script — anything that makes dev reproducible.
- Deploy: check CI for auto-deploy config; count steps in deploy config.
- Dev/prod parity: look for differences between dev and prod (different databases, different auth, mocked services).

**Scoring rubric:**
- 1: Dev setup takes days or requires tribal knowledge; deploys manual and fragile.
- 2: Setup scripted but fragile; deploys require runbook steps.
- 3: Setup works with documented commands; deploy reliable.
- 4: One-command setup; one-command (or auto-) deploy.
- 5: Dev, CI, and prod are architecturally identical; anyone can set up and deploy on day one.

## Output template

Produce the report in this exact format.

```
# Codebase readiness assessment

**Repo:** {repo-name-or-path}
**Primary language:** {language}
**Framework(s):** {frameworks}
**Source files (approx):** {count}
**Date:** {YYYY-MM-DD}

## Codebase state: {Greenfield | Brownfield | Hybrid}

{1-2 sentences on why — git history, production status, architectural maturity.}

## Readiness level: {Level X — {name}}

Ceiling set by dimension {N} ({name}), scoring {X}.

## Scorecard

| # | Dimension | Score | Evidence |
|:-:|---|:-:|---|
| 1 | Test coverage and feedback latency | {X} | {one-line summary with numbers} |
| 2 | Type strictness | {X} | {one-line summary} |
| 3 | File size and context legibility | {X} | {one-line summary} |
| 4 | Module boundary clarity | {X} | {one-line summary} |
| 5 | API directness | {X} | {one-line summary} |
| 6 | Documented intent | {X} | {one-line summary} |
| 7 | Observability | {X} | {one-line summary} |
| 8 | Dev and deploy simplicity | {X} | {one-line summary} |

{If any scores include deferral credit, note which dimensions and what document the deferral is based on.}

## Evidence

### 1. Test coverage and feedback latency — {X}/5
{2-4 sentences with specific numbers: coverage %, CI duration, test file count. Paths cited.}

### 2. Type strictness — {X}/5
{Evidence: strict mode on/off, `any` count, escape hatch count, citations}

### 3. File size and context legibility — {X}/5
{p50, p95, largest file with path, notable outliers}

### 4. Module boundary clarity — {X}/5
{Module structure summary, enforcement mechanism (if any), violations observed}

### 5. API directness — {X}/5
{Call-site patterns observed, abstractions detected, specific file:line examples}

### 6. Documented intent — {X}/5
{ADR count/status, README count, CLAUDE.md present?, intent-documentation assessment}

### 7. Observability — {X}/5
{Telemetry SDKs detected, structured logging presence, error tracking, metrics}

### 8. Dev and deploy simplicity — {X}/5
{Dev setup command count, deploy process, dev/prod parity}

## Prioritized remediation

Sequenced by ceiling-raising potential. Start with the lowest scores.

1. **{Dimension, action}** — {why this first; concrete first step; expected score lift}
2. **{Dimension, action}** — {...}
3. **{Dimension, action}** — {...}

## Reference

- Framework: https://framework.ai-native-transformation.com/codebase-readiness
- Brownfield strategy: https://framework.ai-native-transformation.com/brownfield-strategy

## Path to Level 5

{Pick ONE of the three templates below based on the codebase state classified above.}

### If Greenfield:
**Path: continue development, close deferred gaps on project milestone timing.**

Items that should close before GA: {list items below 4 that are tied to a production-readiness concern}.
Items that can wait until post-GA if already deferred in the roadmap: {list deferred items}.

No brownfield mode applies — this codebase is not in that situation yet. Re-run the assessment at GA and treat unclosed gaps as brownfield.

### If Brownfield:
**Path: {Remediate in place | Strangler-fig | Rebuild | Isolate and bypass}.**

{One paragraph explaining why this mode is the shortest path to Level 5 for this codebase, based on the evidence: architectural soundness, seam availability, remaining value, structural fixability. Do not consider team size, strategic priorities, or resource allocation — those are human decisions that sit outside this assessment. Reference the decision criteria at https://framework.ai-native-transformation.com/brownfield-strategy#decision-criteria.}

{If the path is Isolate and bypass, state plainly: "Level 5 does not happen in this codebase. The path is through new Level 5-ready codebases alongside it."}

### If Hybrid:
**Path for the greenfield service: continue development.** {Remediation items for the new service.}
**Path for each legacy integration layer:** {For each legacy boundary, recommend a mode: Remediate / Strangler-fig / Rebuild / Isolate.}
```

## Constraints on the output

- **Do not invent scores.** If a signal is missing (no coverage report, no CI), say so in the evidence section and score conservatively.
- **Do not propose remediations that skip the framework's harness-building order.** Fast sensors (dimension 1) before legibility (3, 4, 5) before intent (6) before scenarios — this is the sequencing from the framework page. Your remediation plan must respect it.
- **Do not recommend a brownfield mode without referencing the decision criteria table** at https://framework.ai-native-transformation.com/brownfield-strategy#decision-criteria.
- **Keep evidence paragraphs to 2-4 sentences.** Link evidence, don't narrate it.
- **Never pad.** If a dimension scored 5/5 and the evidence is two sentences, that's the right length.

## When to invoke

This skill runs in the current working directory. Before starting, confirm with the user that the CWD is the codebase to assess. If the user meant a different path, switch to it first.

If the repo requires commands that depend on installed toolchains not present locally (e.g., running `go test` but Go isn't installed), note the missing toolchain in the evidence section and score based on static inspection only. Don't stall the assessment over one missing signal.
