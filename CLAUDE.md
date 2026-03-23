# UiPath Orchestrator MCP Server

Remote MCP server connecting Claude to UiPath Orchestrator APIs via OAuth 2.0, built on Cloudflare Workers. Targeting submission to Anthropic's Connectors Directory.

**Current phase:** Phase 1 (POC) — validating OAuth architecture. GitHub OAuth is the current auth provider as a scaffold; UiPath Cloud identity will replace it.

## Tech Stack

- **Runtime:** Cloudflare Workers (edge)
- **Language:** TypeScript (strict mode)
- **Framework:** Hono (HTTP routing)
- **MCP:** `@modelcontextprotocol/sdk` + `agents/mcp` (McpAgent with Durable Objects)
- **Auth:** `@cloudflare/workers-oauth-provider` (OAuth 2.0/2.1)
- **Validation:** Zod
- **Storage:** Workers KV (OAuth tokens), Durable Objects SQLite (MCP state)

## Project Structure

```
uipath-orchestrator-mcp/     # Main application
  src/
    index.ts                  # MCP server class (MyMCP) + OAuthProvider export
    github-handler.ts         # OAuth flow HTTP handler (Hono routes)
    utils.ts                  # OAuth URL/token helpers
    workers-oauth-utils.ts    # CSRF, state tokens, approval tracking
  wrangler.jsonc              # Workers config (bindings, dev port)
  .dev.vars                   # Local secrets (not committed)
  AGENTS.md                   # Cloudflare Workers reference docs
```

## Commands

All commands run from `uipath-orchestrator-mcp/`:

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start local dev server (port 8788) |
| `npm run deploy` | Deploy to Cloudflare Workers |
| `npm run type-check` | TypeScript type checking (`tsc --noEmit`) |
| `npm run cf-typegen` | Regenerate types after changing wrangler.jsonc bindings |

## Local Development

1. Copy `.dev.vars.example` to `.dev.vars` and fill in GitHub OAuth credentials
2. `npm run dev` starts the server at `http://127.0.0.1:8788`
3. Test with: `npx mcp-remote http://127.0.0.1:8788/mcp`
4. MCP Inspector v0.21.1 "Via Proxy" mode has an OAuth registration bug — use `mcp-remote` or Claude Desktop instead

## Architecture

```
Claude → MCP Client → OAuthProvider (token gate)
                         ↓
                      MyMCP (Durable Object, /mcp route)
                         ↓
                      UiPath Cloud Orchestrator API
```

- `OAuthProvider` wraps the entire Worker, handling `/register`, `/authorize`, `/token`, and proxying `/mcp` to `MyMCP.serve()`
- `MyMCP` extends `McpAgent<Env, State, Props>` — tools are registered in `init()`
- Auth props (login, name, email, accessToken) are encrypted in the auth token and passed as `this.props`
- Tool input schemas use Zod

## Conventions

- Tool definitions go in `MyMCP.init()` in `src/index.ts`
- Each tool uses Zod schemas for input validation
- Tools return `{ content: [{ text: string, type: "text" }] }` format
- OAuth/HTTP routing uses Hono in `github-handler.ts`
- Run `npm run cf-typegen` after any changes to bindings in `wrangler.jsonc`

## Important Notes

- No test framework configured yet (planned for Phase 2)
- The current demo tools (`add`, `userInfoOctokit`, `generateImage`) are scaffolding — they'll be replaced with UiPath Orchestrator tools
- The `as any` cast on `GitHubHandler` in `index.ts` is intentional (Cloudflare/Hono type mismatch)
- See `AGENTS.md` in the subdirectory for Cloudflare Workers-specific docs and limits
