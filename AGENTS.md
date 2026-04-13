# GWS-Meta-Workflows — Agent Context

## What This Project Is

Six production-ready meta-workflows that chain Google Workspace CLI (`gws`) commands into complex, end-to-end automations. Each workflow orchestrates multiple GWS services (Gmail, Calendar, Drive, Docs, Sheets, Model Armor) into a single agent-driven sequence. Human-in-the-loop approval is required before any destructive operations — emails are always drafted, never sent automatically.

**The six workflows:**
1. **inbox-triage-and-auto-reply** — Classify unread emails, draft routine replies, escalate action items
2. **pre-meeting-context-compiler** — Pull attendees, search threads, find docs, compile a briefing in Google Docs
3. **client-onboarding-automation** — Create Drive folders, copy templates, populate tracker, schedule kickoff, draft welcome email
4. **weekly-activity-digest-generator** — Compile sent emails, calendar events, and modified docs into a weekly summary
5. **email-prompt-injection-scanner** — Scan inbound emails for hidden agent instructions using Model Armor; quarantine flagged messages
6. **scheduling-conflict-resolver** — Extract proposed times from emails, check free/busy, draft reply with available slots

## Tech Stack

- **Skill definition**: Markdown with YAML frontmatter (SKILL.md) — Claude Code skill format
- **CLI dependency**: Google Workspace CLI (`@googleworkspace/cli`) — Rust binary via npm, requires Node.js 18+
- **Auth**: Google Cloud OAuth 2.0 via `gcloud` CLI or manual setup
- **Security**: Model Armor API (Google Cloud) for prompt injection detection (Workflow 5 only)
- **Runtime**: Claude Code (or any agent framework that reads SKILL.md files)
- **No executable code in this repo** — `gws` CLI is the execution engine

## Architecture

```
Agent (Claude Code)
      |
  SKILL.md (this file)
      |
  gws CLI commands → Gmail, Calendar, Drive, Docs, Sheets, Model Armor
```

| Workflow | GWS Services |
|----------|-------------|
| inbox-triage-and-auto-reply | Gmail, Calendar |
| pre-meeting-context-compiler | Calendar, Gmail, Drive, Docs |
| client-onboarding-automation | Drive, Sheets, Calendar, Gmail |
| weekly-activity-digest-generator | Calendar, Gmail, Drive, Docs |
| email-prompt-injection-scanner | Gmail, Model Armor |
| scheduling-conflict-resolver | Gmail, Calendar |

**Composable chains:**
- Morning routine: inbox-triage → meeting-prep → scheduling-conflict
- End of week: weekly-digest → email-scanner
- New client: client-onboarding → meeting-prep (on kickoff day)

## Directory Structure

```
GWS-Meta-Workflows/
├── CLAUDE.md          # Claude Code-specific instructions
├── AGENTS.md          # This file
├── README.md          # User-facing docs (install, usage, MCP setup)
└── SKILL.md           # All 6 workflows (722 lines) — the primary artifact
```

This is a single-file skill package. All workflow logic is in `SKILL.md` as natural-language instructions interpreted at runtime.

## Safety Defaults (All Workflows)

- **Drafts only** — never send emails automatically; always create drafts for human review
- **Dry-run first** — use `--dry-run` on mutating operations before executing
- **Field masks** — use `--params '{"fields": "..."}'` to limit response size
- **Sanitize untrusted input** — use `--sanitize <TEMPLATE>` when processing external email content

## Environment Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `GOOGLE_WORKSPACE_CLI_TOKEN` | Pre-authenticated access token | Optional (alternative auth) |
| `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` | Service account JSON path | Optional (alternative auth) |
| `GOOGLE_WORKSPACE_CLI_SANITIZE_TEMPLATE` | Default Model Armor template | Optional (Workflow 5 only) |
| `GOOGLE_WORKSPACE_CLI_SANITIZE_MODE` | `warn` or `block` | Optional (Workflow 5 only) |

Standard auth stores credentials at `~/.config/gws/client_secret.json` — no env vars needed for normal use.

## Installation & Prerequisites

```bash
# Install GWS CLI
npm install -g @googleworkspace/cli
gws --version   # expect 0.7.x or newer

# Authenticate
gws auth setup
gws auth login -s drive,gmail,calendar,docs,sheets
gws auth status  # confirm token_valid: true

# Install this skill
git clone https://github.com/grandamenium/gws-meta-workflows.git ~/.claude/skills/gws-meta-workflows
```

## Conventions for Editing SKILL.md

- YAML frontmatter at top with `name` and `description` fields
- Workflows are numbered sections: `## Workflow 1`, `## Workflow 2`, etc.
- Each workflow section includes: "What it does" summary → numbered steps with exact CLI commands → "GWS Skills Referenced" with `@see` links
- GWS helper commands use `+` prefix: `gws gmail +triage`, `gws calendar +insert`
- GWS raw API commands use dot-separated paths: `gws gmail users messages get`
- Placeholders use UPPER_SNAKE_CASE: `MESSAGE_ID`, `CLIENT_NAME`, `DOC_ID`
- Keep the Command Quick Reference table at the bottom of SKILL.md in sync with any new commands

## Optional: MCP Server Mode

```json
{
  "mcpServers": {
    "gws": {
      "command": "gws",
      "args": ["mcp", "-s", "drive,gmail,calendar", "-w", "-e"]
    }
  }
}
```

Use MCP mode for Claude Desktop or VS Code/Cursor where structured tool calling is preferred. CLI mode (SKILL.md) is the default for Claude Code sessions.

## What NOT to Touch

- **`~/.config/gws/client_secret.json`** — OAuth credentials; never commit
- **Service account JSON files** — always in `.gitignore`, never committed
- **`SKILL.md` `@see` references** — they point to skills from `googleworkspace/cli` repo; install those separately
- **Workflow 5 (email-prompt-injection-scanner)** — requires Model Armor API enabled on GCP; skip `--sanitize` if not configured

## Known Issues

- Model Armor requires `cloud-platform` scope — not available on all GCP projects
- Large Gmail messages or Drive listings can overflow agent context windows — always use `--fields`
- `Sheet1!A1` ranges in `--params` JSON cause shell history expansion — use just `"A1"` or `$'...'` quoting
- Gmail API has per-user rate limits — use `--page-delay 200` when paginating
- `@see` skill references are NOT bundled — install dependent skills separately from `googleworkspace/cli` repo

## Related

- See `CLAUDE.md` for Claude Code-specific configuration
