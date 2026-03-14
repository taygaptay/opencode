# OpenCode Project Context

## Project Overview

OpenCode is an open-source AI-powered development tool that provides an intelligent coding assistant for developers. It offers multiple interfaces including a Terminal UI (TUI), web interface, and desktop application. The project is built with a client/server architecture that supports multiple AI providers (Claude, OpenAI, Google, local models, etc.) and includes built-in LSP support.

### Key Features
- **Multiple Agents**: Build (full-access), Plan (read-only), and General (subagent for complex tasks)
- **Multi-platform**: TUI, web app, and desktop app (Tauri-based)
- **Provider-agnostic**: Works with Claude, OpenAI, Google, Groq, and many other AI providers
- **LSP Support**: Out-of-the-box language server protocol support
- **Plugin System**: Extensible via `@opencode-ai/plugin`

## Tech Stack

- **Runtime**: Bun 1.3+
- **Language**: TypeScript
- **Frontend**: SolidJS with Tailwind CSS 4
- **Desktop**: Tauri (Rust)
- **Backend**: Hono (web framework), Drizzle ORM (SQLite)
- **AI SDK**: Vercel AI SDK (`ai` package)
- **Build Tools**: Turbo, TypeScript Go (`tsgo`)
- **Package Manager**: Bun (with workspaces)

## Repository Structure

```
opencode/
├── packages/
│   ├── opencode/          # Core business logic, server, and TUI
│   ├── app/               # Shared web UI components
│   ├── desktop/           # Tauri desktop wrapper
│   ├── desktop-electron/  # Electron desktop variant
│   ├── sdk/               # JavaScript/TypeScript SDK
│   │   └── js/            # SDK source and generated clients
│   ├── plugin/            # Plugin system
│   ├── script/            # Build and utility scripts
│   ├── util/              # Shared utilities
│   ├── ui/                # UI components library
│   ├── web/               # Web frontend
│   ├── console/           # Console application
│   ├── function/          # Function execution
│   ├── identity/          # Identity management
│   ├── enterprise/        # Enterprise features
│   ├── extensions/        # Editor extensions
│   ├── containers/        # Container support
│   └── slack/             # Slack integration
├── script/                # Root-level scripts
├── specs/                 # API specifications
├── infra/                 # Infrastructure configuration
├── nix/                   # Nix package definitions
└── patches/               # npm package patches
```

## Building and Running

### Prerequisites
- Bun 1.3+ (`bun --version` to check)
- For desktop: Tauri prerequisites (Rust toolchain)

### Installation
```bash
bun install
```

### Development Commands

**From repository root:**
```bash
# Start TUI dev server (runs in packages/opencode by default)
bun dev

# Run against a specific directory
bun dev <directory>

# Start headless API server (port 4096)
bun dev serve
bun dev serve --port 8080

# Type check all packages
bun typecheck

# Run tests (must be run from package directories, not root)
```

**Web UI development:**
```bash
# First start the server
bun dev serve

# Then run web app in separate terminal
bun run --cwd packages/app dev
```

**Desktop app development:**
```bash
# Native desktop (Tauri)
bun run --cwd packages/desktop tauri dev

# Web dev server only (no native shell)
bun run --cwd packages/desktop dev

# Production build
bun run --cwd packages/desktop tauri build
```

**Building standalone executable:**
```bash
./packages/opencode/script/build.ts --single
./packages/opencode/dist/opencode-<platform>/bin/opencode
```

### SDK Regeneration
After API/SDK changes, regenerate the JavaScript SDK:
```bash
./packages/sdk/js/script/build.ts
# or from SDK directory
bun run build
```

## Development Conventions

### Code Style (from AGENTS.md)

**General Principles:**
- Keep logic in one function unless composable/reusable
- Avoid `try/catch` where possible (prefer `.catch()`)
- Avoid `any` type; use precise types
- Prefer single-word variable names
- Use Bun APIs (`Bun.file()`, etc.)
- Rely on type inference; avoid explicit annotations unless necessary
- Use functional array methods (`flatMap`, `filter`, `map`) with type guards

**Naming:**
- Single-word names by default: `pid`, `cfg`, `err`, `opts`, `dir`, `root`, `child`, `state`, `timeout`
- Avoid: `inputPID`, `existingClient`, `connectTimeout`, `workerPath`
- Inline single-use values to reduce variable count

**Destructuring:**
```ts
// Good
obj.a
obj.b

// Bad
const { a, b } = obj
```

**Control Flow:**
```ts
// Good - early returns, no else
function foo() {
  if (condition) return 1
  return 2
}
```

**Variables:**
- Prefer `const` over `let`
- Use ternaries instead of reassignment

**Drizzle Schemas:**
Use snake_case for field names:
```ts
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})
```

### Testing
- Avoid mocks where possible
- Test actual implementation, don't duplicate logic
- **Never run tests from repo root** - run from package directories:
  ```bash
  cd packages/opencode && bun test
  ```

### Type Checking
```bash
# From package directories (not root)
cd packages/opencode && bun typecheck
```
Uses TypeScript Go (`tsgo`), not `tsc` directly.

### Git Workflow
- Default branch: `dev`
- Use `dev` or `origin/dev` for diffs (local `main` may not exist)
- PRs must reference an issue
- Use conventional commit titles: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`

## Key Configuration Files

- `package.json` - Root workspace configuration with Bun catalogs
- `turbo.json` - Turborepo task definitions with caching
- `bunfig.toml` - Bun configuration (build optimizations, test root guard)
- `tsconfig.json` - TypeScript config (extends `@tsconfig/bun`, incremental builds)
- `AGENTS.md` - Development guidelines for AI agents
- `CONTRIBUTING.md` - Contribution guidelines and policies

## Performance Optimizations

### Build Optimizations
- **Parallel builds**: Multi-target builds run in parallel via `Promise.all`
- **Content-based caching**: Models snapshot and migrations cached via SHA-256 hashes
- **Incremental TypeScript**: `tsBuildInfoFile` enabled for faster rebuilds
- **Tree shaking**: Enabled in `bunfig.toml` to reduce bundle size

### Runtime Optimizations
- **Lazy provider loading**: AI providers loaded on-demand via dynamic imports
- **Config memoization**: Config cached with 5-second TTL to avoid repeated disk reads
- **Async file operations**: Replaced `readFileSync`/`readdirSync` with async Bun APIs
- **Database indexes**: Added composite indexes on `session` table for common queries
- **Rate limiting**: Processor loop includes backoff for rapid tool calls
- **Doom loop detection**: Session processor exits after 3 failed attempts

### Database Indexes Added
```sql
-- Session table indexes for common query patterns
CREATE INDEX session_time_updated_idx ON session(time_updated, id);
CREATE INDEX session_project_time_updated_idx ON session(project_id, time_updated, id);
CREATE INDEX session_directory_idx ON session(directory);
CREATE INDEX session_archived_idx ON session(time_archived);
```

## Important Notes

1. **Branch**: Default is `dev`, not `main`
2. **Tests**: Cannot run from repo root (guard in `bunfig.toml`)
3. **SDK Changes**: Run `./packages/sdk/js/script/build.ts` after API changes
4. **Parallel Tools**: Use parallel tool calls when applicable
5. **Automation**: Prefer automation over confirmation prompts

## Common Workflows

### Debugging
```bash
# Debug server separately
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096

# Then attach TUI
opencode attach http://localhost:4096
```

### Database Migrations
```bash
cd packages/opencode && bun db
```

### Clean Build
```bash
cd packages/opencode && bun clean && bun install && bun build
```

### Build with Parallel Targets
```bash
./packages/opencode/script/build.ts --parallel
```
