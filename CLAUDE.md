# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a monorepo for Cloudflare MCP (Model Context Protocol) servers. MCP servers allow LLMs (via clients like Cursor, Claude, OpenAI) to interact with Cloudflare services through natural language. Each app is an independent MCP server deployed to Cloudflare Workers that exposes tools for specific Cloudflare functionalities.

**Tech Stack:** TurboRepo + pnpm workspaces, Cloudflare Workers (V8 isolates), `agents` library (McpAgent framework), Hono, Zod, TypeScript 5.5.4

## Commands

### Root-level commands
```bash
# Testing
pnpm test                  # Run all tests across monorepo
pnpm test:watch            # Run tests in watch mode
pnpm test:ci               # Run tests in CI mode
pnpm test -- tests/path/to/test.test.ts  # Run specific test file

# Type checking & linting
pnpm types                 # Type check all packages
pnpm check:turbo           # Run all checks (types + lint)
pnpm check:format          # Check formatting with Prettier
pnpm check:deps            # Check dependency version consistency

# Fixing
pnpm fix:format            # Auto-fix formatting issues
pnpm fix:deps              # Fix dependency mismatches
pnpm update-deps           # Update dependencies
pnpm changeset:new         # Create a changeset
```

### App-specific commands (run in `/apps/{server-name}`)
```bash
pnpm dev                   # Start dev server (wrangler dev)
pnpm deploy                # Deploy to Cloudflare
pnpm types                 # Generate wrangler types
pnpm check:lint            # Run ESLint
pnpm check:types           # TypeScript type check
pnpm test                  # Run Vitest with 15s timeout
```

## Monorepo Structure

```
/apps/*                    # 17+ MCP server applications (each deployed to Workers)
/packages/*                # 6 shared packages
  - eslint-config          # Shared ESLint configuration
  - typescript-config      # Shared TypeScript configs (base, tools, workers, workers-lib)
  - mcp-common             # Core MCP utilities, API clients, tools, Durable Objects
  - mcp-observability      # Metrics tracking via Analytics Engine
  - tools                  # CLI tools (run-tsc, run-vitest, run-wrangler-deploy, etc.)
  - eval-tools             # Evaluation and testing utilities
/implementation-guides     # Detailed guides for tools and type validators
```

## Core Architecture

### MCP Agent Pattern
Each app extends `McpAgent<Env, State, Props>` from the `agents` library:

```typescript
export class ObservabilityMCP extends McpAgent<Env, State, Props> {
  _server: CloudflareMCPServer | undefined

  async init() {
    this.server = new CloudflareMCPServer({ ... })
    registerAccountTools(this)      // Common tools
    registerWorkersTools(this)      // Common tools
    registerObservabilityTools(this) // App-specific tools
    registerPrompts(this)
  }

  async getActiveAccountId() { ... }  // From UserDetails DO
  async setActiveAccountId(id: string) { ... }
}

export default {
  fetch(request: Request, env: Env, ctx: ExecutionContext) {
    return ObservabilityMCP.fetch(request, env, ctx)
  }
}
```

### App Entry Point Structure
```
apps/{server-name}/
├── src/
│   ├── {server-name}.app.ts      # Main McpAgent class
│   ├── {server-name}.context.ts  # Env type definition
│   └── tools/
│       └── {server-name}.tools.ts # Tool registrations
├── wrangler.jsonc                # Cloudflare Workers config
├── package.json
├── vitest.config.ts
└── README.md
```

### Tool Registration Pattern
```typescript
export function registerObservabilityTools(agent: ObservabilityMCP) {
  agent.server.tool(
    'tool_name',                    // snake_case
    'Description for LLM...',       // CRITICAL - LLM's only understanding
    {
      param1: ZodSchema,
      param2: ZodSchema.optional(),
    },
    async ({ param1, param2 }) => {
      const accountId = await agent.getActiveAccountId()
      // ... implementation
      return {
        content: [{ type: 'text', text: JSON.stringify(result) }]
      }
    }
  )
}
```

**Tool descriptions are critical** - the LLM only sees the description, not the implementation. See `implementation-guides/tools.md` for detailed patterns.

### Durable Objects
State persistence uses Durable Objects (SQLite-based):

```typescript
export class UserDetails extends DurableObject {
  private readonly kv: DurableKVStore<UserDetailsKeys>
  public async getActiveAccountId() { ... }
  public async setActiveAccountId(id: string) { ... }
}

// Usage
export function getUserDetails(env, userId) {
  const id = env.USER_DETAILS.idFromName(userId)
  return env.USER_DETAILS.get(id)
}
```

### Authentication
Dual authentication modes via `@repo/mcp-common`:
- **OAuth flow**: Via `@cloudflare/workers-oauth-provider`
- **API token mode**: Direct API token authentication

Auth context accessed via `getProps(agent)` - returns `AuthProps` with `type: 'user_token' | 'account_token'`.

### Cloudflare API Integration
- Client wrapper: `packages/mcp-common/src/cloudflare-api.ts`
- Service-specific APIs in `packages/mcp-common/src/api/` (workers.api.ts, account.api.ts, etc.)
- Type-safe calls with Zod validation

## Code Style (from AGENTS.md)

**Formatting:**
- Indentation: Tabs (2 spaces/tab width)
- Quotes: Single quotes, no semicolons
- Line width: 100 characters, LF endings
- Trailing commas: ES5 style

**Import order** (enforced by prettier-plugin-sort-imports):
1. Node.js built-ins (`<BUILTIN_MODULES>`)
2. Third-party modules (`<THIRD_PARTY_MODULES>`)
3. Workspace imports (`@repo/*`)
4. Local relative imports (sorted: `..`, `../`, `./foo`, `.`, `./index`)
5. Type imports (same order with `<TYPES>` prefix)

**Naming conventions:**
- Files: `kebab-case` (e.g., `auditlogs.tools.ts`)
- Classes: `PascalCase` (e.g., `AuditlogMCP`)
- Functions: `camelCase` (e.g., `handleGetAuditLogs`)
- Schemas: `PascalCase` (e.g., `auditLogsQuerySchema`)
- Tool names: `snake_case` (e.g., `auditlogs_by_account_id`)

**TypeScript:**
- Use `type` imports: `import type { X }`
- Array notation: `string[]` not `Array<string>`
- Unused variables: Prefix with `_`
- `@ts-ignore` allowed when necessary

## Zod Type Validation

All tool parameters use Zod schemas. Key patterns from `implementation-guides/type-validators.md`:

1. **Link to SDK types**: Use `z.ZodType<SDKType>` for compile-time dependency on Cloudflare SDK types
2. **Individual field schemas**: Define separate named schemas per field (not object schemas for tool params)
3. **Always `.describe()`**: Add descriptions to every schema for LLM context
4. **Naming**: `ServiceNameFieldNameSchema` (e.g., `HyperdriveConfigIdSchema`)

Place validators in `packages/mcp-common/src/types/` (e.g., `hyperdrive.types.ts`, `kv.ts`).

## Testing

- Framework: Vitest with `@cloudflare/vitest-pool-workers`
- Mocking: `fetchMock` for API mocking
- Test timeout: 15 seconds
- Run single test: `pnpm test -- tests/tools/queues.test.ts`
- Watch mode: `pnpm test:watch`

## Key Implementation Guides

- `AGENTS.md` - Comprehensive coding guidelines for AI agents
- `implementation-guides/tools.md` - How to implement MCP tools
- `implementation-guides/type-validators.md` - Zod schema patterns
- `implementation-guides/evals.md` - Evaluation guidelines

## Workspace Dependencies

Internal packages use `workspace:*` protocol. Use `syncpack` to check consistency (`pnpm check:deps`).

## Deployment

Each app deploys to Cloudflare Workers via `wrangler`. Custom domains follow pattern `{service}.mcp.cloudflare.com`. Environments: dev, staging, production.

## Changesets

Use Changesets for version management: `pnpm changeset:new` to create changesets for releases.
