# Agent Guide for Cloudflare MCP Server Monorepo

This repository contains Cloudflare MCP servers using TurboRepo + pnpm for monorepo management.

## Build/Lint/Test Commands

### Testing
- `pnpm test` - Run all tests across monorepo
- `pnpm test -- tests/tools/queues.test.ts` - Run single test file
- `pnpm test:watch` - Run tests in watch mode (development)
- `pnpm test:ci` - Run tests in CI mode

### Type Checking & Linting
- `pnpm types` - Type check all packages
- `pnpm check:lint` - Run ESLint
- `pnpm check:turbo` - Run all checks (types + lint)
- `pnpm check:format` - Check formatting with Prettier
- `pnpm check:deps` - Check dependency version consistency

### Fixing & Building
- `pnpm fix:format` - Auto-fix formatting issues
- `pnpm fix:deps` - Fix dependency mismatches
- `pnpm update-deps` - Update dependencies
- Individual apps: `pnpm dev` or `wrangler dev` in app directory

### App-Specific Commands (in app directories)
- `pnpm check:lint` - Run `run-eslint-workers`
- `pnpm check:types` - Run `run-tsc` (TypeScript type check)
- `pnpm types` - Run `wrangler types --include-env=false`
- `pnpm test` - Run Vitest with 15s timeout

## Code Style Guidelines

### Formatting
- **Indentation**: Tabs (2 spaces/tab width)
- **Quotes**: Single quotes
- **Semicolons**: No semicolons
- **Line width**: 100 characters
- **Line endings**: LF
- **Trailing commas**: ES5 style
- **Final newline**: Required
- **YAML/MDX**: 2 spaces indentation

### Imports (ordered by prettier-plugin-sort-imports)
1. `<BUILTIN_MODULES>` - Node.js built-ins
2. `<THIRD_PARTY_MODULES>` - npm packages
3. Empty line
4. Workspace imports: `^(@repo)(/.*)$`
5. Empty line
6. Local relative imports (sorted: `..`, `../`, `./foo`, `.`, `./index`)
7. Empty line
8. Type imports: Same order with `<TYPES>` prefix

### TypeScript
- Use `type` imports: `import type { X } from '...'`
- Explicit function return types: Optional
- Array notation: `string[]` not `Array<string>`
- Unused variables: Prefix with `_` to ignore warnings
- `@ts-ignore` allowed when necessary
- Type errors: Throw Error objects

### Naming Conventions
- **Files**: kebab-case (e.g., `auditlogs.tools.ts`)
- **Classes**: PascalCase (e.g., `AuditlogMCP`)
- **Functions**: camelCase (e.g., `handleGetAuditLogs`)
- **Constants**: camelCase or UPPER_SNAKE_CASE
- **Schemas**: PascalCase (e.g., `auditLogsQuerySchema`)
- **Interfaces**: PascalCase (e.g., `AuditLogOptions`, `Env`)

### Error Handling
- Use try-catch for async operations
- Wrap errors with context: `Error message: ${error instanceof Error && error.message}`
- Record errors: `this.server.recordError(e)` in MCP agents
- Return user-friendly error responses in tools

### Zod Schemas
- Use Zod for all input validation
- Describe each field: `.describe('...')`
- Infer types: `z.infer<typeof schema>`
- Enum patterns: `z.enum(['val1', 'val2'])`
- Optional fields: `.optional()` with sensible defaults

### MCP Tool Registration
- Function name: `register{Feature}Tools(agent: Agent)`
- Tool names: snake_case (e.g., `auditlogs_by_account_id`)
- Schema shape: Pass `.shape` for parameter schema
- Return format: `{ content: [{ type: 'text', text: JSON.stringify(result) }] }`

### Durable Objects
- Extend `McpAgent<Env, State, Props>`
- Implement `init()` for setup
- Use `getProps(agent)` to access auth context
- Export `State` type for typing
- Fetch DOs: `getUserDetails(env, userId)`

### Testing
- Use Vitest with `@cloudflare/vitest-pool-workers`
- Mock APIs with `fetchMock`
- Test timeout: 15 seconds
- Group tests: `describe()` suites
- Assertions: `expect().toBe()`, `expect().toThrow()`
