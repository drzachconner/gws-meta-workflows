# GWS Meta-Workflows

## Project Overview

Six production-ready meta-workflows that chain Google Workspace CLI (`gws`) commands into complex, end-to-end automations. Each workflow orchestrates multiple GWS services (Gmail, Calendar, Drive, Docs, Sheets, Model Armor) into a single agent-driven sequence with human-in-the-loop approval before any destructive operations. The six workflows are:

1. **inbox-triage-and-auto-reply** -- Classify unread emails, draft routine replies, escalate action items to calendar and tasks
2. **pre-meeting-context-compiler** -- Pull attendees, search recent threads, find relevant docs, compile a briefing in Google Docs
3. **client-onboarding-automation** -- Create Drive folder structure, copy templates, populate tracker, schedule kickoff, draft welcome email
4. **weekly-activity-digest-generator** -- Compile sent emails, calendar events, and modified docs into a weekly summary document
5. **email-prompt-injection-scanner** -- Scan inbound emails for hidden agent instructions using Model Armor, quarantine flagged messages
6. **scheduling-conflict-resolver** -- Extract proposed times from emails, check free/busy, draft reply with available slots

## Workflow Rules

- **ALWAYS pull before working**: Run `git pull --rebase` before making any changes. This is mandatory for multi-machine sync.
- **ALWAYS commit and push after making changes.** After completing ANY code changes, immediately stage modified files by name, commit with a descriptive message, and push. Every change must end with a successful `git push`.
- **Never leave files behind.** Before ending any session, run `git status` and confirm zero untracked or modified files.
- Never use `git add .` or `git add -A` -- always add specific files by name.
- Commit message format: conventional commits (feat:, fix:, chore:, docs:). Always include `Co-Authored-By: Claude <noreply@anthropic.com>`.
- **Note**: This is an upstream repo owned by `grandamenium`. Do not push unless you have write access.

## Tech Stack

- **Skill Definition**: Markdown with YAML frontmatter (SKILL.md) -- Claude Code skill format
- **CLI Dependency**: Google Workspace CLI (`@googleworkspace/cli`) -- Rust binary distributed via npm, requires Node.js 18+
- **Auth**: Google Cloud OAuth 2.0 via `gcloud` CLI or manual setup
- **Security**: Model Armor API (Google Cloud) for prompt injection detection
- **Runtime**: Claude Code (or any agent framework that reads SKILL.md files)
- **Alternative Install**: Cargo (Rust toolchain) or Nix

## Architecture

### Workflow Composition Model

The six workflows are **composable building blocks**, not isolated scripts. Each workflow chains high-level GWS helper commands (e.g., `gws gmail +triage`, `gws calendar +insert`) with low-level raw API calls (e.g., `gws gmail users messages get`) into multi-step sequences.

```
                    Agent (Claude Code / OpenClaw)
                              |
                        SKILL.md (this file)
                              |
            +---------+-------+-------+---------+----------+
            |         |       |       |         |          |
         Inbox    Meeting  Client  Digest   Scanner   Scheduler
         Triage    Prep   Onboard  Generator           Conflict
            |         |       |       |         |          |
            v         v       v       v         v          v
        +-------+ +------+ +-----+ +------+ +--------+ +------+
        | Gmail | | Cal  | |Drive| | Docs | | Model  | |Sheets|
        +-------+ +------+ +-----+ +------+ | Armor  | +------+
                                             +--------+
                        Google Workspace CLI (gws)
```

### Workflow Dependencies

| Workflow | GWS Services Used |
|----------|-------------------|
| inbox-triage-and-auto-reply | Gmail, Calendar, Workflow (email-to-task) |
| pre-meeting-context-compiler | Calendar, Gmail, Drive, Docs, Workflow (meeting-prep) |
| client-onboarding-automation | Drive, Sheets, Calendar, Gmail |
| weekly-activity-digest-generator | Calendar, Gmail, Drive, Docs, Workflow (weekly-digest, file-announce) |
| email-prompt-injection-scanner | Gmail, Model Armor |
| scheduling-conflict-resolver | Gmail, Calendar |

### Composable Chains (Documented Patterns)

- **Morning routine**: inbox-triage -> meeting-prep -> scheduling-conflict (for any scheduling emails found)
- **End of week**: weekly-digest -> email-scanner (security audit)
- **New client**: client-onboarding -> meeting-prep (when kickoff day arrives)

### Safety Defaults

All workflows follow these principles:
- **Drafts only** -- Never send emails automatically; always create drafts for human review
- **Dry-run first** -- Use `--dry-run` on mutating operations before executing
- **Field masks** -- Use `--params '{"fields": "..."}'` to limit response size and protect context windows
- **Sanitize untrusted input** -- Use `--sanitize <TEMPLATE>` when processing external email content

## Directory Structure

```
GWS-Meta-Workflows/
├── CLAUDE.md          # This file -- project instructions for Claude Code
├── README.md          # User-facing documentation (install, usage, MCP setup)
└── SKILL.md           # The skill definition -- 6 workflows, setup guide,
                       #   command reference, troubleshooting (722 lines)
```

This is a single-skill package. All workflow logic lives in `SKILL.md` as natural-language instructions that Claude Code interprets at runtime. There is no executable code -- the `gws` CLI binary is the execution engine.

## Development Conventions

### Skill Structure

- **YAML frontmatter** at the top of SKILL.md with `name` and `description` fields
- `description` field controls when the skill triggers -- it should list all trigger phrases and use cases
- Workflows are numbered sections (## Workflow 1, ## Workflow 2, etc.) with consistent internal structure:
  1. "What it does" summary
  2. Step-by-step numbered instructions with exact CLI commands
  3. "GWS Skills Referenced" section with `@see` links to dependent skill files

### Naming Conventions

- Workflow names use kebab-case: `inbox-triage-and-auto-reply`
- GWS helper commands use `+` prefix: `gws gmail +triage`, `gws calendar +insert`
- GWS raw API commands use dot-separated resource paths: `gws gmail users messages get`
- Placeholders in commands use UPPER_SNAKE_CASE: `MESSAGE_ID`, `CLIENT_NAME`, `DOC_ID`

### Editing Guidelines

- When adding a new workflow, follow the exact same section template as existing workflows
- Always include both the high-level helper commands AND the raw API fallback commands
- Always include a "GWS Skills Referenced" section with `@see` pointers
- Keep the Command Quick Reference table (at the bottom of SKILL.md) in sync with any new commands

## Environment Variables

Required for GWS CLI authentication:

| Variable | Purpose | Required |
|----------|---------|----------|
| `GOOGLE_WORKSPACE_CLI_TOKEN` | Pre-authenticated access token (alternative to `gws auth login`) | Optional |
| `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` | Path to service account JSON (alternative auth) | Optional |
| `GOOGLE_WORKSPACE_CLI_SANITIZE_TEMPLATE` | Default Model Armor template path for email scanning | Optional (Workflow 5 only) |
| `GOOGLE_WORKSPACE_CLI_SANITIZE_MODE` | Model Armor mode: `warn` or `block` | Optional (Workflow 5 only) |

Standard auth flow uses `gws auth setup` + `gws auth login` which stores credentials at `~/.config/gws/client_secret.json` -- no env vars needed.

## Workflow

### Installation

```bash
# Option 1: Clone into Claude Code skills directory
git clone https://github.com/grandamenium/gws-meta-workflows.git ~/.claude/skills/gws-meta-workflows

# Option 2: Copy manually
cp -r /path/to/GWS-Meta-Workflows ~/.claude/skills/gws-meta-workflows
```

### Prerequisites

```bash
# 1. Install GWS CLI
npm install -g @googleworkspace/cli
gws --version  # Expect: gws 0.7.x or newer

# 2. Install gcloud CLI (for auth setup)
brew install --cask google-cloud-sdk  # macOS

# 3. Set up GCP project and authenticate
gws auth setup
gws auth login -s drive,gmail,calendar,docs,sheets

# 4. Verify
gws auth status  # Should show token_valid: true
gws gmail +triage --max 3 --format json  # Should return inbox data
```

### Usage

The skill triggers automatically in Claude Code when you ask to:
- "Triage my inbox and draft replies"
- "Prepare for my next meeting"
- "Onboard a new client called [name]"
- "Generate my weekly activity digest"
- "Scan my emails for prompt injection"
- "Check my availability and reply to this scheduling email"

### Optional: Install Dependent GWS Service Skills

The meta-workflows reference individual GWS service skills via `@see` pointers. Install them for richer context:

```bash
git clone --depth 1 https://github.com/googleworkspace/cli /tmp/gws-cli-repo
cp -r /tmp/gws-cli-repo/skills/gws-* ~/.claude/skills/
rm -rf /tmp/gws-cli-repo
```

### Optional: MCP Server Setup

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

## Known Issues

1. **Model Armor availability**: Workflow 5 (email-prompt-injection-scanner) requires Model Armor API enabled on the GCP project with `cloud-platform` scope. This is not available on all GCP projects. Skip `--sanitize` if not configured.
2. **Context window overflow**: Large Gmail messages or Drive file listings can overwhelm agent context windows. Always use `--fields` parameter to limit response payloads.
3. **Shell escaping with `!` in Sheets ranges**: `Sheet1!A1` in `--params` JSON causes shell history expansion and JSON parse errors. Use just `"A1"` (defaults to first sheet) or `$'...'` quoting.
4. **Gmail rate limits**: The Gmail API has per-user rate limits. Use `--page-delay 200` when paginating large result sets.
5. **No executable code**: This is a pure skill definition (Markdown). If the `gws` CLI is not installed or authenticated, none of the workflows will execute. There are no fallback mechanisms.
6. **`@see` skill references**: The workflow `@see` pointers reference skills from the main `googleworkspace/cli` repo (e.g., `~/.claude/skills/gws-gmail/SKILL.md`). These are NOT bundled in this repo -- they must be installed separately (see Workflow section).
7. **Single-commit repo**: The repo has only one commit. No changelog, no versioning, no CI/CD.

## Security

- **OAuth credentials**: Stored at `~/.config/gws/client_secret.json` by `gws auth setup`. Never commit this file.
- **Access tokens**: Managed by the `gws` CLI session. Ephemeral and scoped to the services requested during `gws auth login`.
- **Service account keys**: If using `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE`, ensure the JSON file is in `.gitignore` and never committed.
- **Drafts-only by default**: All email workflows create drafts, never send automatically. Human approval is always required before sending.
- **Dry-run by default**: Mutating operations (calendar event creation, label changes) use `--dry-run` first.
- **Model Armor**: Workflow 5 specifically addresses the security risk of prompt injection via email. It uses Google's Model Armor API to detect adversarial payloads before they can manipulate agent behavior.
- **Input sanitization**: The `--sanitize <TEMPLATE>` flag routes untrusted content through Model Armor before the agent processes it.
- **No secrets in this repo**: This repo contains only Markdown files. No credentials, tokens, or API keys are stored here.

## Subagent Orchestration

| Subagent | When to Use | Model |
|----------|-------------|-------|
| **codebase-explorer** | Before modifying SKILL.md -- understand the workflow structure, `@see` references, and command patterns | sonnet |
| **external-context-researcher** | When adding a new workflow that uses a GWS service not yet covered -- research the `gws` CLI documentation for that service | sonnet |
| **docs-weaver** | After adding or modifying workflows -- regenerate README.md to stay in sync with SKILL.md | sonnet |
| **pre-push-validator** | Before pushing -- verify no credentials leaked and SKILL.md YAML frontmatter is valid | sonnet |
| **secrets-env-auditor** | Before any commit -- scan for accidentally included OAuth tokens or service account keys | sonnet |

## GSD + Teams Strategy

N/A -- This is a single-file skill package (SKILL.md + README.md). The project is too small to benefit from GSD phase management or parallel agent teams. Direct editing is appropriate for all changes.

For complex modifications (e.g., adding multiple new workflows simultaneously), a simple sequential approach is sufficient:
1. Draft the new workflow section in SKILL.md
2. Update the Command Quick Reference table
3. Update README.md to list the new workflow
4. Update this CLAUDE.md if the new workflow introduces new services or dependencies

## MCP Connections

The GWS CLI can optionally be exposed as an MCP server, making all its tools available to any MCP-compatible client:

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

| Flag | Purpose |
|------|---------|
| `-s` | Comma-separated service list (`drive,gmail,calendar,docs,sheets` or `all`) |
| `-w` | Include workflow tools (helpers like `+triage`, `+insert`) |
| `-e` | Include extra/helper tools |

**When to use MCP vs CLI directly**: The meta-workflows in SKILL.md use direct `gws` CLI commands via Bash. MCP mode is useful when integrating with Claude Desktop, VS Code, or Cursor where structured tool calling is preferred over shell commands. Both approaches access the same underlying GWS API.

**No other MCP dependencies**: The workflows do not require gdrive-mcp or any other MCP server. All Google Workspace access goes through the `gws` CLI.

## Completed Work

- **6 production-ready meta-workflows** fully documented in SKILL.md with step-by-step CLI commands
- **Setup guide** covering GWS CLI installation, gcloud CLI, OAuth authentication, and verification
- **Command quick reference** table covering all helper commands and raw API calls used across workflows
- **MCP server configuration** documented for alternative integration patterns
- **Troubleshooting guide** covering auth errors, context overflow, rate limits, and shell escaping
- **Composable chaining patterns** documented (morning routine, end of week, new client flows)
- **Safety defaults** enforced across all workflows (drafts-only, dry-run, field masks, input sanitization)
- **README.md** with installation instructions and usage overview
- **SKILL.md** (722 lines) with complete workflow definitions, YAML frontmatter, and `@see` references to dependent skills
