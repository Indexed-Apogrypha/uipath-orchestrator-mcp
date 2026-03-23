# UiPath Orchestrator MCP Server

Remote MCP server that connects [Claude](https://claude.ai) to [UiPath Orchestrator](https://www.uipath.com/product/orchestrator). Query jobs, queues, processes, robots, and logs directly from Claude conversations via OAuth 2.0.

Built for submission to [Anthropic's Connectors Directory](https://claude.ai/connectors).

> **Status:** Phase 1 — Proof of Concept

---

## Why This Exists

Search "UiPath" in Claude's Connectors panel today and you get zero results. The only existing options require developer tooling (local stdio servers with terminal setup) or route through third-party middleware. UiPath's own MCP infrastructure is built for UiPath agents consuming external tools — not for Claude consuming UiPath.

This project fills that gap with a production-grade, OAuth-authenticated, remotely hosted MCP server that meets Anthropic's directory submission requirements.

---

## What It Does

Once listed in the Connectors Directory, any Claude user with a UiPath Cloud account can:

1. Search "UiPath" in the Connectors panel and connect
2. Authenticate via OAuth 2.0 against their UiPath Cloud org
3. Query Orchestrator data using natural language in any Claude conversation

### v1 Tools (Read-Only)

| Tool | Description |
|------|-------------|
| `list_processes` | List available processes in a folder |
| `list_jobs` | List recent jobs with status filtering |
| `get_job_status` | Get status and details of a specific job |
| `list_queues` | List queue definitions in a folder |
| `get_queue_items` | Query queue items with status/date filters |
| `list_robots` | List robots and their availability |
| `list_machines` | List registered machines |
| `list_folders` | List Orchestrator folders |
| `get_assets` | Read asset values from a folder |
| `get_logs` | Retrieve execution logs with filtering |

### v1.1 Tools (Write Operations — Fast Follow)

| Tool | Description |
|------|-------------|
| `start_job` | Start a process as an unattended/attended job |
| `stop_job` | Stop or kill a running job |
| `add_queue_item` | Add a new transaction item to a queue |

---

## Examples

### Check Automation Health

**Prompt:** "Show me the status of all jobs that ran in the last 24 hours in my Production folder."

The connector resolves the Production folder via `list_folders`, then calls `list_jobs` with a date filter and returns a summary of completed, failed, and running jobs.

### Diagnose Queue Failures

**Prompt:** "How many failed items are in my Claims Processing queue? Show me the error details for the last 5 failures."

The connector calls `list_queues` to resolve the queue, then `get_queue_items` with a Failed status filter. Claude summarizes the error messages and suggests potential root causes.

### Infrastructure Check

**Prompt:** "Which robots are currently available and what machines are they connected to?"

The connector calls `list_robots` and `list_machines` to return availability status. Claude cross-references the results to show which machines have idle capacity.

---

## Architecture

```
Claude User → claude.ai / Desktop / Mobile
  → Anthropic MCP Client
    → UiPath Connector (Remote MCP Server)
      → UiPath Cloud Orchestrator API
```

- **Transport:** Streamable HTTP (with SSE fallback)
- **Auth:** OAuth 2.0 Authorization Code flow against UiPath Cloud Identity Server
- **Safety:** MCP tool annotations on every tool (readOnlyHint, destructiveHint, idempotentHint)

---

## Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Proof of Concept — validate OAuth flow, resolve org-scoping, minimal MCP server | 🔄 In Progress |
| **Phase 2** | MVP Build — all 10 read-only tools, tests, CI/CD | ⬜ Not Started |
| **Phase 3** | Submission Prep — privacy policy, README examples, test account, submit to Anthropic | ⬜ Not Started |
| **Phase 4** | Post-Submission — revision requests, v1.1 write tools, monitoring | ⬜ Not Started |

---

## Development

### Prerequisites

- UiPath Cloud account ([Community Edition](https://cloud.uipath.com/) works)
- Node.js 18+ or Python 3.11+ (language TBD in Phase 1)

### Setup

```bash
git clone https://github.com/Indexed-Apogrypha/uipath-orchestrator-mcp.git
cd uipath-orchestrator-mcp
# Setup instructions will be added as Phase 1 progresses
```

### Testing with Claude

You can test the server as a custom connector before it's listed in the directory:

1. Deploy the MCP server (or use a tunnel for local dev)
2. In Claude, go to **Settings → Connectors → Add custom connector**
3. Enter the server URL
4. Authenticate with your UiPath Cloud credentials

---

## Privacy Policy

Privacy policy will be published at [TODO: URL] before directory submission.

---

## Contributing

This project is in early development. Issues and discussions are welcome. If you work with UiPath and want to help shape the tool surface or test against your Orchestrator environment, open an issue.

---

## License

[MIT](LICENSE)

---

## Acknowledgments

- [Model Context Protocol](https://modelcontextprotocol.io/) — the open standard this server implements
- [UiPath Orchestrator API](https://docs.uipath.com/orchestrator/automation-cloud/latest/api-guide/) — the API surface this server wraps
- [Anthropic Connectors Directory](https://claude.ai/connectors) — the distribution channel this project targets
