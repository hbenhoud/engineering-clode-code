# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Claude Code plugin (`engineering-claude-code-plugin`) that automates Jira-to-production engineering workflow for Go teams. It exposes a `/ticket` skill that orchestrates a 6-phase pipeline: ticket loading → architecture → implementation → parallel analysis (QA + review + security) → consolidated report → documentation.

## Repo Layout

```
.claude-plugin/marketplace.json          # Marketplace registration (SHA must be kept in sync)
engineering-claude-code-plugin/
└── 0.1.0/
    ├── .claude-plugin/plugin.json       # Plugin manifest (declares skills)
    ├── skills/ticket/SKILL.md           # Core orchestration skill — the main entry point
    ├── agents/                          # Agents invoked in Phase 4 (parallel)
    │   ├── qa.md                        # Writes table-driven tests, runs go test
    │   ├── review.md                    # Code review + golangci-lint
    │   └── secu.md                      # Security audit + gosec
    └── rules/go-guidelines.md           # Auto-loaded for *.go files
```

## Release Process

After every push to `main`, update the SHA in `.claude-plugin/marketplace.json`:

```bash
git rev-parse HEAD
# paste SHA into .claude-plugin/marketplace.json → plugins[0].source.sha
git add .claude-plugin/marketplace.json
git commit -m "release: bump sha to <short-sha>"
git push
```

Without this step, users installing from the marketplace get the old version.

## Plugin Architecture

**`skills/ticket/SKILL.md`** is the orchestrator. It runs sequentially through phases, then fans out to three agents in parallel during Phase 4 using Claude Code's `@agent-*` syntax.

**Agents** (`agents/*.md`) are self-contained instruction sets. Each reads the context produced by earlier phases (from `.claude/<TICKET-ID>/`), performs its analysis, and writes its own output artifact.

**Rules** (`rules/go-guidelines.md`) are loaded automatically by Claude Code for all `*.go` files and constrain how Go code is generated across all phases. Key rules enforced:
- Errors: sentinel errors prefixed `Err`, wrap with `%w`, no log + return on same error
- Logging: `zap.L()` only, static messages, typed fields (`zap.String`, `zap.Error`, etc.)
- Tracing: `snake_case` span names, `span.RecordError` only where the error is handled, no sensitive attributes

## Artifacts

The skill writes working documents to `.claude/<TICKET-ID>/` (changed from `output/` in recent commit):

| File | Phase | Content |
|------|-------|---------|
| `context.md` | 1 | Ticket summary + impacted files |
| `arch.md` | 2 | Architecture decisions + interfaces |
| `qa.md` | 4 | Test results |
| `review.md` | 4 | Lint + review findings |
| `secu.md` | 4 | Security findings |
| `report.md` | 5 | Consolidated report (human validates here) |
| `doc.md` | 6 | Delivery documentation + godoc |

## Local Development / Testing the Plugin

Install the plugin locally by referencing the repo path in a target project's `.claude/settings.json`:

```json
{
  "plugins": ["/path/to/engineering-claude-code-plugin"]
}
```

Then invoke from the target Go project root (on a dedicated branch):

```
/ticket PROJ-42
```

The plugin requires the `jira_get_ticket` MCP tool to be configured in Claude Code.
