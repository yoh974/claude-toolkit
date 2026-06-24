---
name: review-pr-domain
description: "Review code changes against DDD/CQRS/Event Sourcing architecture rules"
argument-hint: "[base-branch] e.g. develop (default) or HEAD~3"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Agent", "Task"]
---

# DDD Architecture PR Review

Review the current branch changes against the project's DDD/CQRS/Event Sourcing architecture rules.

**Base ref:** "$ARGUMENTS" (defaults to `develop` if empty)

## Workflow

### Step 1 — Determine diff scope

```bash
BASE="${ARGUMENTS:-develop}"
git diff "$BASE"...HEAD --name-only
git diff "$BASE"...HEAD --stat
```

If there are no changes, stop and say so.

### Step 2 — Categorize changed files

Classify each changed file by layer:
- **domain/** → domain layer (aggregates, events, value objects, exceptions, ports)
- **application/commands/** → commands & handlers
- **application/queries/** → queries & handlers
- **application/sagas/** or **infrastructure/sagas/** → sagas
- **infrastructure/repositories/** → repositories (command & query adapters)
- **infrastructure/projections/** → projection handlers
- **presentation/** → controllers, DTOs, interceptors
- **\*.spec.ts** → unit tests
- **\*.e2e-spec.ts** → E2E tests
- **prisma/** → schema changes
- **\*.module.ts** → module wiring

### Step 3 — Launch parallel review agents

Based on which layers are affected, launch up to 3 agents **in parallel**:

**Agent A — Domain & Application** (launch if domain/, application/, or *.module.ts files changed):
> Use the `ddd-domain-reviewer` agent. Pass it the full diff output for domain and application files.

**Agent B — Infrastructure** (launch if infrastructure/ or prisma/ files changed):
> Use the `ddd-infra-reviewer` agent. Pass it the full diff output for infrastructure and prisma files.

**Agent C — Presentation & Tests** (launch if presentation/, *.spec.ts, or *.e2e-spec.ts files changed):
> Use the `ddd-presentation-reviewer` agent. Pass it the full diff output for presentation and test files.

To get the diff for a specific group:
```bash
git diff "$BASE"...HEAD -- src/*/domain/ src/*/application/
git diff "$BASE"...HEAD -- src/*/infrastructure/ prisma/
git diff "$BASE"...HEAD -- src/*/presentation/ src/**/*.spec.ts test/
```

### Step 4 — Aggregate results

After all agents complete, compile a single structured report:

```markdown
# PR Review — DDD Architecture

## Summary
<1-2 sentences on what the PR changes and overall quality>

## Critical Issues 🔴 (must fix before merge)
<list from agents, with file:line references>

## Important Issues 🟡 (should fix)
<list from agents, with file:line references>

## Minor Issues 🔵 (suggestions)
<list, or "None">

## Strengths ✅
<what was done well architecturally>
```

Only include sections with actual content. If no issues are found, say so explicitly.

## Notes

- Focus exclusively on **architecture rule violations** (see `.claude/rules/`), not style (Prettier/ESLint handle that)
- Do NOT flag pre-existing code outside the diff
- Do NOT suggest adding more tests unless a critical path has zero coverage
- If the diff is large (>500 lines), focus on the most complex/risky files first
