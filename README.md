# engineering-claude-code-plugin

A Claude Code orchestration plugin for Go engineering teams. Executes a Jira ticket end-to-end — from reading the spec to implementing in the codebase, including QA analysis, code review, and security audit.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/claude-code) installed
- Jira MCP configured in Claude Code with the `jira_get_ticket` tool available
- Go codebase with standard structure (`cmd/`, `internal/`, `pkg/`)
- Always work on a dedicated Git branch before invoking the skill

## Installation

### From GitHub (recommended)

Claude Code does not install plugins directly from a URL. The repo must first be registered as a marketplace source, then the plugin is installed from it.

**Step 1 — Add the GitHub repo as a marketplace:**

```
/plugin marketplace add <owner>/engineering-claude-code-plugin
```

**Step 2 — Install the plugin:**

```
/plugin install engineering-claude-code-plugin@engineering-claude-code-plugin
```

The marketplace name (after `@`) matches the repo name by default.

### Locally (for development or self-hosting)

Clone the repository and reference the local path in your project's `.claude/settings.json`:

```bash
git clone https://github.com/<owner>/engineering-claude-code-plugin
```

```json
{
  "plugins": [
    "/path/to/engineering-claude-code-plugin"
  ]
}
```

## Usage

From the root of the Go codebase, on a dedicated branch:

```
/ticket PROJ-42
```

Claude fetches the Jira ticket, explores the codebase, and orchestrates the full pipeline.

## Pipeline

```
Phase 1 — Loading
  └─ Fetch ticket via jira_get_ticket
  └─ Explore codebase
  └─ output/<TICKET-ID>/context.md

Phase 2 — Architecture
  └─ output/<TICKET-ID>/arch.md

Phase 3 — Development
  └─ Direct implementation in the codebase (cmd/, internal/, pkg/...)

Phase 4 — Parallel Analysis
  ├─ @agent-qa     → unit tests + output/<TICKET-ID>/qa.md
  ├─ @agent-review → code review + output/<TICKET-ID>/review.md
  └─ @agent-secu   → security audit + output/<TICKET-ID>/secu.md

Phase 5 — Consolidated Report
  └─ output/<TICKET-ID>/report.md
  └─ Terminal display + human validation

Phase 6 — Documentation (if validated)
  └─ output/<TICKET-ID>/doc.md
  └─ godoc comments in implemented files
```

## Artifacts Produced

```
output/<TICKET-ID>/
├── context.md   — ticket summary and impacted files
├── arch.md      — architecture decisions and interfaces
├── report.md    — consolidated QA + review + security
└── doc.md       — delivery documentation
```

The Go implementation is written directly into the codebase. Files under `output/` are working artifacts — they are not part of the deliverable.

## Best Practices

- Always work on a dedicated Git branch (`git checkout -b feat/PROJ-42`)
- Review the report before validating phase 6
- If blocking findings are present, answer "yes" to the correction prompt to re-run phase 3
- Read `output/<TICKET-ID>/arch.md` to understand the decisions made

## Go Rules Loaded Automatically

The plugin loads `rules/go-guidelines.md` for all `*.go` files and enforces:

- **Error handling**: sentinel errors, `%w` wrapping, no double handling
- **Logging**: `zap.L()` exclusively, static messages, typed fields
- **Tracing**: `snake_case` names, `RecordError` at the right level, no sensitive data in attributes

## License

MIT
