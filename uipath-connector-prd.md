# Product Requirements Document

## UiPath Orchestrator Connector for Anthropic's Claude Connectors Directory

**Author:** Matthew Harper
**Date:** March 22, 2026
**Status:** Approved
**Version:** 0.2

---

## Problem Statement

Claude users who work with UiPath Orchestrator today have no native way to connect the two platforms. Searching "UiPath" in the Claude Connectors panel returns zero results. The only existing options require developer tooling (a local stdio-based Node.js MCP server from LobeHub that demands terminal setup, environment variables, and client credentials) or route through a third-party intermediary (Pipedream's hosted MCP, which interposes its own auth layer and infrastructure between the user and UiPath). UiPath's own MCP infrastructure is designed for UiPath agents consuming external tools - not for Claude consuming UiPath.

This means that automation engineers, operations teams, and RPA developers who use both Claude and UiPath cannot:

- Ask Claude to check job status, queue health, or robot availability against live Orchestrator data
- Trigger or monitor attended/unattended processes from a conversational interface
- Use Claude's reasoning capabilities to analyze execution logs, diagnose failed queue items, or summarize automation health - without manually copying data between systems

The gap is not technical capability - UiPath's Orchestrator API is well-documented and supports OAuth 2.0. The gap is that nobody has built and submitted a production-grade remote MCP server that meets Anthropic's directory requirements: OAuth authentication, tool safety annotations, privacy policy, and QA-ready test data.

## Solution

Build and publish a remote MCP server that connects Claude to UiPath Cloud Orchestrator, then submit it to Anthropic's Connectors Directory for public listing. Once listed, any Claude user with a UiPath Cloud account can:

- Search "UiPath" in the Connectors panel and find the connector
- Authenticate via OAuth 2.0 against their own UiPath Cloud org
- Use natural language in any Claude conversation to query Orchestrator data - processes, jobs, queues, robots, machines, assets, folders, and execution logs

The v1 connector is read-only. Users can query and analyze Orchestrator data but cannot trigger, stop, or modify anything. Write operations (start_job, stop_job, add_queue_item) will ship in a fast-follow v1.1 once the connector is listed and the read-only surface has been validated in production.

**Key design constraint:** UiPath's OAuth endpoints are org-scoped (the org name is embedded in the authorize and token URLs). The connector must resolve the user's org/tenant context before or during the OAuth flow. The exact mechanism will be determined during Phase 1 prototyping - candidates include a pre-auth landing page, a post-auth discovery call, or leveraging Anthropic's Dynamic Client Registration (DCR) support.

## User Stories

### Discovery and Setup

1. As a Claude Pro/Team/Enterprise user, I want to search "UiPath" in the Connectors panel and find a verified connector, so that I don't have to hunt for community MCP servers or set up developer tooling.
2. As a UiPath Cloud user, I want to authenticate with my existing UiPath credentials via OAuth, so that I don't need to create API keys, manage secrets, or configure environment variables.
3. As a user connecting for the first time, I want to complete the entire setup in under 2 minutes with no terminal or code, so that the barrier to entry is the same as connecting Slack or Google Drive to Claude.
4. As a user who has connected, I want to see the UiPath tools available in the conversation's tool list, so that I know what I can ask Claude to do before I start prompting.
5. As a user who wants to disconnect, I want to revoke the connector's access from Claude settings or from UiPath's external apps panel, so that I maintain full control over what Claude can access.

### Organizational Context

6. As a user with access to one UiPath org, I want to have the connector automatically resolve my org context, so that I don't have to manually enter org names or tenant IDs.
7. As a user with access to multiple UiPath orgs, I want to select which org/tenant I'm working with, so that I can target the right environment (dev, staging, production).
8. As a user working in a specific Orchestrator folder, I want to specify or switch folder context for my queries, so that I see data scoped to the right folder, not everything across the org.

### Process and Job Visibility

9. As an automation engineer, I want to ask Claude "What processes are available in my Production folder?", so that I can see what's deployed without opening Orchestrator.
10. As an operations team member, I want to ask Claude "Show me all jobs that ran in the last 24 hours", so that I get a quick health check of recent automation activity.
11. As a support engineer, I want to ask Claude "What's the status of job ID 12345?", so that I can check on a specific run without navigating the Orchestrator UI.
12. As a team lead, I want to ask Claude "How many jobs failed yesterday and what were the errors?", so that I can triage failures conversationally and get Claude's help analyzing root causes.
13. As an automation engineer, I want to ask Claude to filter jobs by process name, status, or date range, so that I can narrow down to exactly the runs I care about.

### Queue Analysis

14. As a process owner, I want to ask Claude "How many items are in my Claims queue by status?", so that I get a breakdown of New, In Progress, Successful, and Failed items.
15. As a support engineer, I want to ask Claude "Show me the last 10 failed queue items with their error messages", so that I can diagnose queue processing issues without manually filtering in Orchestrator.
16. As a business analyst, I want to ask Claude to analyze queue throughput patterns, so that I can identify bottlenecks and capacity planning needs from conversational analysis.

### Infrastructure Monitoring

17. As an IT administrator, I want to ask Claude "Which robots are currently available?", so that I can check capacity before triggering new work.
18. As an operations team member, I want to ask Claude "List all machines and their status", so that I can identify offline or disconnected machines.
19. As an automation engineer, I want to ask Claude to read asset values from a specific folder, so that I can check configuration values without opening Orchestrator.
20. As a developer debugging a process, I want to ask Claude to pull execution logs with filtering (by level, process, time range), so that I can trace issues through log data conversationally.

### Cross-Cutting Concerns

21. As a user working with a large Orchestrator environment, I want to have results paginated sensibly within Anthropic's 25,000 token limit, so that I don't hit errors or get truncated responses on high-volume queries.
22. As a user whose OAuth token has expired, I want to have the connector silently refresh the token, so that I don't get interrupted mid-conversation to re-authenticate.
23. As a cautious user, I want to know that the v1 connector is read-only and cannot modify my Orchestrator environment, so that I can explore the connector's capabilities without risk.
24. As a user who encounters an error, I want to see a clear, actionable error message (not a raw API stack trace), so that I can understand what went wrong and what to try next.

## Implementation Decisions

### Settled Decisions

- **Transport protocol:** Streamable HTTP (preferred) with SSE fallback. Anthropic has signaled SSE deprecation, so Streamable HTTP is the forward-looking choice.

- **Authentication:** OAuth 2.0 Authorization Code flow against UiPath Cloud Identity Server. Authorization endpoint: `https://cloud.uipath.com/{org}/identity_/connect/authorize`. Token endpoint: `https://cloud.uipath.com/{org}/identity_/connect/token`. Scopes: OR.Queues, OR.Jobs, OR.Robots, OR.Execution, OR.Assets, OR.Folders, OR.Machines (read-only for v1). Access tokens expire in 1 hour; refresh tokens are supported.

- **Hosting platform:** Cloudflare Workers (edge deployment, built-in OAuth helpers via `@cloudflare/workers-oauth-provider`, Durable Objects for MCP session state, Workers KV for token storage). *Validated in Phase 1 POC.*

- **Language/runtime:** TypeScript with `@modelcontextprotocol/sdk` and `agents/mcp` (McpAgent with Durable Objects). HTTP routing via Hono. Input validation via Zod. *Validated in Phase 1 POC.*

- **v1 access level:** Read-only. All 10 tools in v1 are read operations. Write operations (start_job, stop_job, add_queue_item) will ship in v1.1 as a fast-follow once the connector is listed and the read-only surface is validated. This simplifies Anthropic's security review and reduces the safety annotation surface.

- **v1 tool surface (10 read-only tools):**
  - Process Management: list_processes, get_job_status, list_jobs
  - Queue Management: list_queues, get_queue_items
  - Infrastructure: list_robots, list_machines, list_folders, get_assets, get_logs

- **v1.1 tool surface (3 write tools):** start_job, stop_job (destructive), add_queue_item. Each will carry appropriate MCP safety annotations with confirmation requirements for destructive operations.

- **Safety annotations:** Every tool will include MCP tool annotations: readOnlyHint (true for all v1 tools), destructiveHint, idempotentHint, and openWorldHint per Anthropic's mandatory directory policy.

- **Test environment:** Existing UiPath Community Edition org. Will be seeded with sample processes, populated queues, test robots, and representative log data for Anthropic's QA review.

- **Submission artifacts:** Privacy policy (hosted on project domain or GitHub Pages), README with 3+ working examples, test account credentials, compliance confirmation with MCP Directory Policy.

### Deferred Decisions (Resolve in Phase 1)

The following decisions depend on what is learned during Phase 1 prototyping. Each has candidate approaches but requires hands-on validation.

- **Org-scoping mechanism:** UiPath's OAuth endpoints embed the org name in the URL, which means the connector must know the org before initiating the auth flow. Candidates: (a) pre-auth landing page where user enters org name, (b) configuration step stored server-side, (c) post-auth discovery via UiPath API. Phase 1 will test each approach and determine which works within Anthropic's connector auth flow.

- **Multi-tenant handling:** Users with access to multiple UiPath orgs/tenants need a way to target the right one. Candidates: (a) one org per connector connection (user adds multiple connectors if needed), (b) org-switching tool within the connector. Depends on org-scoping mechanism decision.

- **Pagination strategy:** Anthropic enforces a 25,000 token limit per tool response. High-volume UiPath endpoints (jobs, queue items, logs) need a pagination approach. Candidates: (a) default small page sizes with explicit "get more" tool calls, (b) return summary counts + first page and let user ask for more, (c) auto-paginate with truncation. Phase 1 will test response sizes against real Orchestrator data.

- **Dynamic Client Registration (DCR):** Anthropic supports DCR, which could simplify the OAuth handshake. Phase 1 will validate whether UiPath's Identity Server supports DCR or whether static client registration per org is required.

## Module Architecture

The connector decomposes into the following modules, designed for testability and independent development:

### Module 1: Auth Module

**Responsibility:** Manage the full OAuth 2.0 lifecycle - authorization code exchange, token storage, token refresh, and session management. Encapsulate all UiPath Identity Server interaction behind a simple interface that returns a valid access token or throws.

**Interface:**
- `getAccessToken(sessionId)` -> string | throws AuthError
- `handleCallback(code, state)` -> session
- `refreshIfNeeded(sessionId)` -> void

**Key complexity:** Org-scoped endpoints, token refresh timing, session state persistence across serverless invocations.

### Module 2: UiPath API Client

**Responsibility:** Typed wrapper around UiPath Orchestrator's OData REST API. Handles request construction, authentication header injection, OData query parameter building, pagination, error normalization, and rate limiting.

**Interface:** Methods per resource: `listFolders(opts)`, `listProcesses(folderId, opts)`, `listJobs(folderId, filters)`, `getJob(jobId)`, `listQueues(folderId)`, `getQueueItems(queueId, filters)`, `listRobots(folderId)`, `listMachines(folderId)`, `getAssets(folderId)`, `getLogs(filters)`. Each returns normalized response objects, not raw OData.

**Key complexity:** OData filter syntax construction, pagination within Anthropic's 25k token limit, folder-scoped API calls (X-UIPATH-OrganizationUnitId header).

### Module 3: MCP Tool Definitions

**Responsibility:** Define each MCP tool's schema (name, description, input parameters, safety annotations) and map tool invocations to UiPath API Client calls. Each tool definition is a self-contained unit: JSON schema for inputs, handler function, and annotation metadata.

**Interface:** Standard MCP tool registration via the SDK's `server.setRequestHandler` pattern. Each tool exports its schema and handler independently.

**Key complexity:** Writing tool descriptions that are clear enough for Claude to invoke correctly. Designing input parameter schemas that are flexible (optional filters) but well-constrained. Ensuring safety annotations are accurate.

### Module 4: MCP Server (Transport Layer)

**Responsibility:** HTTP server that implements the MCP protocol over Streamable HTTP (with SSE fallback). Handles incoming tool calls, routes to the correct tool handler, manages connection lifecycle, and returns responses. This is the entry point that Anthropic's infrastructure connects to.

**Interface:** Single HTTPS endpoint (the connector URL registered with Anthropic). Implements MCP protocol handshake, tool listing, and tool invocation.

**Key complexity:** Cloudflare Workers request/response model, stateless execution with session lookup via Durable Objects.

### Module 5: Response Formatter

**Responsibility:** Transform raw UiPath API responses into Claude-friendly text. Handles token budget awareness (staying under 25k tokens), summarization of large result sets, and consistent formatting so Claude can reason effectively over the data.

**Interface:** `formatToolResponse(rawData, toolName, opts)` -> string. Accepts raw API output and returns a formatted string within token limits.

**Key complexity:** Balancing completeness vs. token budget. Deciding what to summarize vs. return in full. Making output structured enough for Claude to parse but readable enough for the user.

## Testing Decisions

### Testing Philosophy

Tests should validate external behavior, not implementation details. A good test for this project answers the question: "If a user asks Claude this question, does the connector return correct, well-formatted data from Orchestrator?" Tests should not assert on internal method calls, specific HTTP header ordering, or response formatting minutiae - they should verify that the right UiPath API is called with the right parameters and the response is within Anthropic's constraints.

### Modules to Test

- **Auth Module:** Test OAuth flow end-to-end against UiPath's Identity Server (integration test). Test token refresh logic with expired/valid/missing tokens (unit test with mocked responses). Test error handling for invalid credentials, revoked access, network failures.
- **UiPath API Client:** Test each API method against the Community Edition org (integration tests). Verify OData filter construction produces valid queries. Test pagination behavior with result sets that exceed page size. Test error normalization for common API errors (403, 404, 429, 500).
- **MCP Tool Definitions:** Validate that each tool's JSON schema is valid and complete. Test that tool handlers correctly translate MCP inputs to API Client calls. Verify safety annotations are present and accurate on every tool.
- **Response Formatter:** Test that formatted responses stay within 25,000 token estimates for large result sets. Test that empty result sets produce helpful messages (not empty strings). Test that error responses are user-readable.
- **End-to-End:** Use MCP Inspector to validate the full tool lifecycle: connect, authenticate, list tools, invoke each tool, verify responses. Final QA via Claude custom connector in claude.ai across all 10 tools.

### Test Infrastructure

- **Test framework:** Vitest
- **MCP protocol testing:** MCP Inspector (Anthropic's official debugging tool)
- **Integration test target:** UiPath Community Edition org with seeded test data
- **CI:** GitHub Actions running unit tests on every PR, integration tests on merge to main

## Out of Scope

The following are explicitly excluded from v1 and v1.1:

- **Write operations in v1.** start_job, stop_job, and add_queue_item are deferred to v1.1. v1 is read-only.
- **Full Orchestrator API coverage.** UiPath exposes 240+ API endpoints. The connector covers 10 (v1) to 13 (v1.1) of the most conversationally useful ones.
- **On-premise UiPath support.** Automation Suite and standalone Orchestrator installations are not supported. The connector targets UiPath Cloud only.
- **Real-time event streaming.** No webhooks, no push notifications from Orchestrator. All data is fetched on-demand when a tool is invoked.
- **Interactive UI widgets.** The connector will not render dashboards, charts, or interactive elements within Claude conversations. Responses are text-based.
- **Action Center, Document Understanding, AI Center, Test Manager.** These UiPath services are candidates for v2+ expansion but are not part of the initial connector.
- **Sibling connectors.** ABBYY Vantage, Camunda, M-Files, and other enterprise automation platforms are future projects that could use the same architecture pattern, but are not part of this PRD.

## Phased Delivery

### Phase 1 - Proof of Concept (1-2 weeks)

Validate OAuth flow end-to-end, resolve org-scoping, stand up minimal MCP server with 1-2 tools, test with Claude custom connector.

**Current status:** Architecture validated. Cloudflare Workers + TypeScript + McpAgent + Durable Objects running. GitHub OAuth scaffold working end-to-end with CSRF protection, state validation, and session binding. Demo tools (add, userInfoOctokit) functional. Next: replace GitHub OAuth with UiPath Cloud identity and resolve org-scoping.

**Exit criteria:** Claude can authenticate against UiPath Cloud and successfully call list_folders.

### Phase 2 - MVP Build (2-3 weeks)

Implement all 10 read-only tools, add safety annotations, implement token refresh, add rate limiting, write tests, set up CI/CD.

**Exit criteria:** All 10 tools functional and deployed.

### Phase 3 - Submission Prep (1 week)

Write privacy policy, create README with examples, seed test account, final QA, submit via Anthropic's form.

**Exit criteria:** Submission accepted into review pipeline.

### Phase 4 - Post-Submission (ongoing)

Respond to revision requests, ship v1.1 write tools, monitor reliability, gather feedback, write announcement content.

## Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| UiPath's org-scoped OAuth breaks Anthropic's connector auth flow | Medium | Phase 1 validates this first. Multiple fallback approaches identified. |
| An official UiPath connector ships before this project | Low-Medium | Portfolio value persists. An official connector would likely differ in scope. |
| Anthropic rejects the submission | Low | Requirements are well-documented. Common rejection causes addressed in Phase 3. |
| Hosting platform constraints | Low | Alternative platforms (Railway, Fly.io) as fallback. Phase 1 validates. |

## Open Questions

1. Does UiPath's Identity Server support Dynamic Client Registration (DCR), or will we need static client registration per org?
2. What is the optimal default page size for each tool to stay comfortably under Anthropic's 25,000 token limit?
3. Should the connector offer a "read-only mode" toggle in v1.1 to let cautious users keep write operations disabled even after they ship?
4. Is there a UiPath Partner Program path that could grant official endorsement or co-marketing?
5. How should the connector handle UiPath Cloud regions (EMEA, US, APAC) - does the org name implicitly resolve region, or does the user need to specify?

## Success Criteria

**Minimum:** Connector is listed in Anthropic's Connectors Directory, a UiPath Cloud user can authenticate and use all tools from claude.ai, and the project is open source on GitHub with clear documentation.

**Stretch:** Community adoption (stars, forks, feedback), referenced in job applications as a portfolio piece, featured in Anthropic's or UiPath's community channels, and foundation for a broader "Enterprise Automation" connector family.
