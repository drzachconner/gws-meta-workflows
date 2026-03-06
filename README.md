# GWS Meta-Workflows

Six production-ready meta-workflows that chain Google Workspace CLI (`gws`) commands into complex, end-to-end automations. Each workflow orchestrates multiple GWS services into a single agent-driven sequence.

## What's Inside

| Workflow | What It Does |
|----------|-------------|
| **inbox-triage-and-auto-reply** | Classify unread emails, draft routine replies, escalate action items to calendar and tasks |
| **pre-meeting-context-compiler** | Pull attendees, search recent threads, find relevant docs, compile a briefing in Google Docs |
| **client-onboarding-automation** | Create Drive folder structure, copy templates, populate tracker, schedule kickoff, draft welcome email |
| **weekly-activity-digest-generator** | Compile sent emails, calendar events, and modified docs into a weekly summary document |
| **email-prompt-injection-scanner** | Scan inbound emails for hidden agent instructions using Model Armor, quarantine flagged messages |
| **scheduling-conflict-resolver** | Extract proposed times from emails, check free/busy, draft reply with available slots |

## Requirements

- Google Workspace CLI: `npm install -g @googleworkspace/cli`
- Google Cloud project with Workspace APIs enabled
- Authenticated session: `gws auth setup && gws auth login`
- For email scanning: Model Armor API enabled on your GCP project

## Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/YOUR_USERNAME/gws-meta-workflows.git ~/.claude/skills/gws-meta-workflows
```

Or copy directly:
```bash
cp -r gws-meta-workflows ~/.claude/skills/
```

## Usage

The skill triggers when you ask Claude Code to:
- Triage your inbox and draft replies
- Prepare for an upcoming meeting
- Onboard a new client
- Generate a weekly activity digest
- Scan emails for prompt injection
- Resolve scheduling conflicts from an email

Each workflow chains real `gws` CLI commands - helpers like `gws gmail +triage` and raw API calls like `gws gmail users messages get` - into a complete automation with human-in-the-loop approval before any sends or deletes.

## MCP Server Setup

To use these workflows through any MCP-compatible client:

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

## License

MIT
