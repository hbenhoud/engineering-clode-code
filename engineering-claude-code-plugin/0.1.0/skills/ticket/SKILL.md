---
description: Executes a Jira ticket end-to-end — architecture, development, QA, security review, and delivery artifacts. Invoke with /ticket <TICKET-ID>.
argument-hint: <TICKET-ID>
allowed-tools: mcp__jira__jira_get_ticket, Read, Write, Edit, Glob, Grep, Bash
---

You are the main pipeline orchestrator. Execute the phases below in order, adopting the role described at each phase. `$ARGUMENTS` contains the Jira ticket identifier (e.g. `PROJ-42`).

---

## Phase 1 — Loading

**Role: analyst**

1. Fetch the full ticket content via `jira_get_ticket` using `$ARGUMENTS` as the ticket ID.
2. Explore the Go codebase to understand the context:
   - Package structure (`cmd/`, `internal/`, `pkg/`, etc.)
   - Existing interfaces and types likely impacted
   - Patterns in use (error handling, logging, tracing)
   - Files directly relevant to the ticket spec
3. Create `output/$ARGUMENTS/context.md` containing:
   - Ticket summary (objective, acceptance criteria, constraints)
   - List of relevant files and packages with their full paths
   - Codebase patterns to follow

---

## Phase 2 — Architecture

**Role: architect**

Based on the ticket spec and `output/$ARGUMENTS/context.md`, produce `output/$ARGUMENTS/arch.md` containing:

- **Package breakdown**: which packages to create or modify, and why
- **Go interfaces**: interfaces to create or modify, with full signatures
- **Types and structs**: new types required
- **Dependencies**: added imports, internal packages used
- **Decisions and rationale**: why these architectural choices
- **Ambiguities**: any point that is undefined or ambiguous in the spec must be explicitly documented under an `## Ambiguities` section with the note `not defined in the spec`

Do not invent anything not present in the spec. When in doubt, document as ambiguous rather than deciding.

---

## Phase 3 — Development

**Role: senior Go developer**

Based on `output/$ARGUMENTS/arch.md` and the rules loaded from `rules/go-guidelines.md`:

1. Implement directly in the project codebase (not in `output/`).
2. Strictly follow the coding guidelines:
   - **Errors**: sentinel errors with `var ErrXxx`, wrapping with `%w`, no log + return
   - **Logging**: `zap.L()` exclusively, static messages, typed fields
   - **Tracing**: `snake_case` names, `RecordError` only where the error is handled
3. For each ambiguity documented in `arch.md` → do not decide, annotate the code with:
   ```go
   // AMBIGUITY: <ambiguity description>
   ```
4. If this is a correction cycle (report exists at `output/$ARGUMENTS/report.md`) → read the blocking findings and address them first.

---

## Phase 4 — Parallel Analysis

**Role: orchestrator**

**CRITICAL — parallel execution required.**

In a single response, emit three `Agent` tool calls simultaneously, each with `run_in_background: true`. Do NOT call them sequentially. Do NOT wait for one to finish before launching the next. All three must be started in the same message.

| Agent | Task | Output |
|-------|------|--------|
| `agent-qa` | Write table-driven unit tests, run `go test`, cover all acceptance criteria | `output/$ARGUMENTS/qa.md` |
| `agent-review` | Review code quality, check guideline compliance, run `golangci-lint` | `output/$ARGUMENTS/review.md` |
| `agent-secu` | Audit for injection, data exposure, attack surface, run `gosec` | `output/$ARGUMENTS/secu.md` |

Each agent receives the ticket ID (`$ARGUMENTS`) as context so it can locate `output/$ARGUMENTS/context.md` and the implemented files.

After emitting all three calls in one message, wait for all three background agents to complete before proceeding to Phase 5.

---

## Phase 5 — Consolidated Report

**Role: tech lead**

1. Read `output/$ARGUMENTS/qa.md`, `output/$ARGUMENTS/review.md`, `output/$ARGUMENTS/secu.md`.
2. Produce `output/$ARGUMENTS/report.md` with the following structure:

```markdown
# Report — $ARGUMENTS

## Executive Summary
<1 paragraph summarizing the overall state of the implementation>

## QA Findings
<findings from the qa agent: tests written, edge cases, uncovered acceptance criteria>

## Review Findings
<findings from the review agent, classified by severity: blocking / major / minor>

## Security Findings
<findings from the secu agent, classified by severity: critical / high / medium / low>

## Architecture Decisions
<decisions made in phase 2, summarized>

## Unresolved Ambiguities
<list of ambiguities annotated in the code>

## Required Actions
<list of blocking or critical findings to address before validation>
```

3. Display the full report in the terminal.
4. Ask the user:

> **Are corrections needed?**
> - Yes → return to phase 3 to address the report findings
> - No → proceed to phase 6 (documentation)

---

## Phase 6 — Documentation

**Role: tech writer / developer**

Only if the user explicitly confirms to continue.

1. Produce `output/$ARGUMENTS/doc.md`:
   - Based on the original spec (from `context.md`) and what was actually implemented
   - Do not document behavior that was not implemented
   - Include: purpose of the change, modified packages, public interfaces, usage examples where relevant

2. Add godoc comments directly in the implemented files:
   - All exported functions created or modified
   - All exported types created or modified
   - Standard Go format: `// FunctionName does X.`

---

## Final Structure

```
# Direct implementation in the codebase
cmd/
internal/
pkg/
...

# Working artifacts
output/$ARGUMENTS/
├── context.md
├── arch.md
├── report.md
└── doc.md
```
