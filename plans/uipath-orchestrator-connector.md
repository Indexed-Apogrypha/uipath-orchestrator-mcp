# Plan: UiPath Orchestrator Connector for Claude

> Source PRD: [#30](https://github.com/Indexed-Apogrypha/uipath-orchestrator-mcp/issues/30)

## Architectural decisions

Durable decisions that apply across all phases:

- **Routes**: `/mcp` (MCP protocol, proxied by OAuthProvider), `/authorize` (consent screen), `/callback` (OAuth redirect), `/register` + `/token` (handled by OAuthProvider automatically)
- **Auth**: OAuth 2.0 Authorization Code flow against UiPath Cloud Identity Server. Authorize: `https://cloud.uipath.com/{org}/identity_/connect/authorize`. Token: `https://cloud.uipath.com/{org}/identity_/connect/token`. Scopes: `OR.Queues OR.Jobs OR.Robots OR.Execution OR.Assets OR.Folders OR.Machines`
- **Transport**: Streamable HTTP with SSE fallback, via McpAgent Durable Object on Cloudflare Workers
- **Storage**: Workers KV (OAuth state/tokens), Durable Objects SQLite (MCP session state)
- **API client pattern**: Typed UiPath API client handles OData query building, pagination, auth header injection, folder-scoping via `X-UIPATH-OrganizationUnitId` header
- **Tool pattern**: All tools registered in `MyMCP.init()` with Zod input schemas, safety annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`), returning `{ content: [{ type: "text", text }] }`
- **Response format**: All UiPath API responses normalized and formatted within Anthropic's 25,000 token limit

---

## Phase 1: UiPath OAuth - Single Org Auth Flow

**User stories**: 2, 3, 6, 22

### What to build

Replace the GitHub OAuth scaffold with UiPath Cloud Identity Server. A user authenticates against a single hardcoded org (for now), receives an access token, and the connector silently refreshes it when expired. The full path works: Claude initiates OAuth, user completes UiPath login, token is stored, and subsequent MCP tool calls carry a valid Bearer token.

### Acceptance criteria

- [ ] OAuth authorize redirects to UiPath Cloud Identity Server (not GitHub)
- [ ] Token exchange completes successfully and stores access + refresh tokens in KV
- [ ] Token refresh works automatically when access token expires (1-hour lifetime)
- [ ] `this.props` carries UiPath user context (org, tenant, access token)
- [ ] Error handling for invalid credentials, revoked access, and network failures
- [ ] Works end-to-end via `mcp-remote` or Claude Desktop

---

## Phase 2: Org-Scoping & Multi-Tenant

**User stories**: 6, 7

### What to build

Resolve the org-scoping deferred decision. Since UiPath's OAuth URLs embed the org name, the connector must know the org before starting the auth flow. Implement the chosen mechanism (pre-auth landing page where user enters org name, post-auth discovery call, or server-side configuration). Support users with multiple orgs selecting which one to target.

### Acceptance criteria

- [ ] User's org is resolved before or during the OAuth flow
- [ ] Single-org users have a seamless experience (no unnecessary prompts)
- [ ] Multi-org users can select their target org/tenant
- [ ] Org context is persisted in the session for subsequent tool calls
- [ ] The mechanism works within Anthropic's connector auth flow constraints

---

## Phase 3: First Tool - list_folders

**User stories**: 8

### What to build

The first real UiPath tool, proving the complete integration path. Implement the UiPath API client with auth header injection and OData response parsing. Register `list_folders` as an MCP tool with Zod schema and safety annotations. A user asks Claude about their folders and gets back a formatted list from their Orchestrator.

### Acceptance criteria

- [ ] `list_folders` tool registered with Zod input schema and safety annotations (`readOnlyHint: true`)
- [ ] UiPath API client makes authenticated GET to `/odata/Folders`
- [ ] Response is normalized and formatted as readable text within token limits
- [ ] Tool is visible in Claude's tool list after connecting
- [ ] End-to-end demoable: Claude asks list_folders, user sees their real Orchestrator folders

---

## Phase 4: Process & Job Tools

**User stories**: 9, 10, 11, 12, 13

### What to build

Three tools for process and job visibility: `list_processes`, `list_jobs`, `get_job_status`. These introduce OData filter construction (by status, date range, process name) and pagination for potentially large result sets. Users can ask Claude about their deployed processes, recent job history, and specific job details.

### Acceptance criteria

- [ ] `list_processes` returns processes in a given folder with safety annotations
- [ ] `list_jobs` supports filtering by process name, status, and date range via OData `$filter`
- [ ] `get_job_status` returns detailed status for a specific job ID
- [ ] Pagination works for result sets exceeding the default page size
- [ ] Responses stay within Anthropic's 25,000 token limit
- [ ] All three tools are demoable with real Orchestrator data

---

## Phase 5: Queue Tools

**User stories**: 14, 15, 16

### What to build

Two tools for queue analysis: `list_queues` and `get_queue_items`. Queue items support filtering by status (New, InProgress, Successful, Failed) and include error messages for failed items. Users can ask Claude to summarize queue health and diagnose processing issues.

### Acceptance criteria

- [ ] `list_queues` returns queues in a given folder with item count summaries
- [ ] `get_queue_items` supports filtering by status and returns error details for failed items
- [ ] Both tools have safety annotations (`readOnlyHint: true`)
- [ ] Responses format queue data in a way that enables Claude to reason about throughput and failures

---

## Phase 6: Infrastructure Tools

**User stories**: 17, 18, 19, 20

### What to build

Four tools for infrastructure monitoring: `list_robots`, `list_machines`, `get_assets`, `get_logs`. The log tool is the most complex, supporting filtering by level (Info, Warn, Error), process name, and time range. Users can check robot availability, machine status, configuration values, and trace execution issues through logs.

### Acceptance criteria

- [ ] `list_robots` returns robots with availability status
- [ ] `list_machines` returns machines with connection status
- [ ] `get_assets` returns asset values scoped to a folder
- [ ] `get_logs` supports filtering by level, process, and time range
- [ ] All four tools have safety annotations
- [ ] Log responses are concise enough to stay within token limits even for verbose log data

---

## Phase 7: Error Handling, Rate Limiting & Response Formatting

**User stories**: 21, 23, 24

### What to build

Harden all tools with production-grade error handling: user-friendly error messages (not raw stack traces), rate limit detection and retry/backoff for UiPath API 429s, and a response formatter that enforces the 25k token budget across all tools. Ensure all safety annotations are accurate and consistent. Validate the read-only guarantee is enforced.

### Acceptance criteria

- [ ] All UiPath API errors (403, 404, 429, 500) produce clear, actionable user-facing messages
- [ ] Rate limiting (429) triggers appropriate backoff/retry behavior
- [ ] Response formatter truncates or summarizes results that would exceed 25,000 tokens
- [ ] Every tool has complete and accurate safety annotations
- [ ] No tool can trigger a write operation against Orchestrator

---

## Phase 8: Test Suite & CI/CD

**User stories**: (cross-cutting quality)

### What to build

Vitest test suite covering unit tests (API client, OData filter construction, response formatting, token refresh logic) and integration tests (against UiPath Community Edition org with seeded data). GitHub Actions pipeline running unit tests on every PR and integration tests on merge to main. MCP Inspector validation of the full tool lifecycle.

### Acceptance criteria

- [ ] Vitest configured with unit and integration test separation
- [ ] Unit tests cover: OData filter construction, response formatting/truncation, token refresh logic, error normalization
- [ ] Integration tests cover: each of the 10 tools against real Orchestrator data
- [ ] GitHub Actions workflow runs unit tests on PR, integration tests on merge to main
- [ ] MCP Inspector validates: connect, authenticate, list tools, invoke each tool

---

## Phase 9: Submission Prep

**User stories**: 1, 4, 5

### What to build

All artifacts required for Anthropic's Connectors Directory submission: privacy policy, README with 3+ working examples, seeded test account with representative data, and compliance confirmation. Final end-to-end QA via Claude custom connector in claude.ai. Submit via Anthropic's form.

### Acceptance criteria

- [ ] Privacy policy hosted and accessible (GitHub Pages or project domain)
- [ ] README includes setup instructions, 3+ working conversation examples, and tool reference
- [ ] UiPath Community Edition test account seeded with processes, queues, robots, machines, assets, and log data
- [ ] Test account credentials documented for Anthropic's QA team
- [ ] All 10 tools pass final QA via Claude custom connector on claude.ai
- [ ] Connector discoverable by searching "UiPath" in Connectors panel after listing
- [ ] User can revoke access from Claude settings or UiPath external apps panel
- [ ] Submission form completed and accepted into review pipeline
