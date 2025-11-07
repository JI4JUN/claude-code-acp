# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **claude-code-acp** adapter - an ACP (Agent Client Protocol) server implementation that enables Claude Code to be used from ACP-compatible clients like Zed, Emacs, Neovim, and marimo notebooks.

The adapter bridges the ACP protocol with the Claude Code SDK (TypeScript), translating between ACP requests/responses and Claude Code's internal message format.

## Common Commands

**Setup and Build:**
```bash
npm install              # Install dependencies
npm run build           # Compile TypeScript to JavaScript (outputs to dist/)
npm run dev             # Build and run the adapter
```

**Development:**
```bash
npm run lint            # Run ESLint on src directory
npm run lint:fix        # Auto-fix linting issues
npm run format          # Format code with Prettier
npm run format:check    # Check if code is properly formatted
npm run check           # Run lint + format:check (use before commits)
```

**Testing:**
```bash
npm run test                    # Run Vitest in watch mode
npm run test:run               # Run all tests once
npm test -- src/tests/acp-agent.test.ts  # Run a specific test file
npm run test:integration       # Run integration tests (RUN_INTEGRATION_TESTS=true)
npm run test:coverage          # Generate coverage report
```

## Architecture Overview

The adapter uses a multi-layer architecture:

### Layer 1: ACP Agent (`src/acp-agent.ts`)
- Implements the `Agent` interface from `@agentclientprotocol/sdk`
- `ClaudeAcpAgent` is the main class that handles ACP protocol methods: `initialize()`, `newSession()`, `prompt()`, `cancel()`, `setSessionModel()`, `setSessionMode()`
- Maintains session state with `sessions` map storing `Query` objects from Claude Code SDK
- Converts between ACP format and Claude Code SDK format:
  - `promptToClaude()`: Converts ACP PromptRequest to SDK user messages (handles text, images, resources)
  - `toAcpNotifications()`: Converts SDK messages to ACP SessionNotifications
  - `streamEventToAcpNotifications()`: Handles streaming events from SDK

### Layer 2: MCP Server (`src/mcp-server.ts`)
- Creates two MCP servers that expose tools to Claude Code:
  - **Main MCP Server**: Exposes ACP-specific tools (Read, Edit, Write, Bash, BashOutput, KillShell) with prefixed names (`mcp__acp__Read`, etc.)
  - **Permission MCP Server**: Handles permission prompts for tools, runs on HTTP server at dynamic port
- Delegates file operations to the ACP client via `ClaudeAcpAgent` methods
- Handles tool input/output conversion for proper ACP representation

### Layer 3: Tool Translation (`src/tools.ts`)
- `toolInfoFromToolUse()`: Converts tool uses to ACP ToolCallContent (title, kind, content, locations)
- `toolUpdateFromToolResult()`: Converts tool results for ACP representation
- `planEntries()`: Converts TodoWrite tool data to ACP plan entries

### Layer 4: Utilities (`src/utils.ts`)
- `Pushable<T>`: Async iterable queue for bridging push-based and async-iterator-based code
- Stream converters: `nodeToWebReadable()`, `nodeToWebWritable()` - bridge Node.js and Web Streams
- File utilities: `extractLinesWithByteLimit()`, `loadManagedSettings()`, `applyEnvironmentSettings()`

### Entry Points
- **CLI**: `src/index.ts` - Redirects console to stderr, initializes MCP server, and keeps process alive
- **Library**: `src/lib.ts` - Exports main classes and utilities for programmatic use in other Node.js apps

## Session and State Management

Each ACP session maps to a Claude Code SDK `Query` object:
```
ACP Session (sessionId) → ClaudeAcpAgent.sessions[sessionId] → {
  query: Query,           // Core SDK conversation state
  input: Pushable,        // User message input stream
  cancelled: boolean,     // Track cancellation
  permissionMode: string  // "default" | "acceptEdits" | "bypassPermissions" | "plan"
}
```

**Key Flow:**
1. Client calls `newSession()` → Creates session with Query object
2. Client calls `prompt()` → Pushes user message to Pushable input
3. Query yields messages → Adapter translates to ACP notifications
4. Adapter caches tool uses in `toolUseCache` for result tracking
5. Tool results trigger updates via `sessionUpdate()` back to client

## Permission and Tool Handling

**Permission Modes:**
- `default`: Prompts for permission on first tool use
- `acceptEdits`: Auto-accepts file edit permissions
- `bypassPermissions`: Skips all prompts (disabled for root/sudo)
- `plan`: Read-only mode, no tool execution

**Tool Mapping:**
ACP clients can provide native implementations of tools (Read, Write, Bash, etc.) via `clientCapabilities`. The adapter routes calls accordingly:
- If client has capability → Use ACP client's tool
- Otherwise → Use MCP server version with appropriate permissions

## Important Implementation Details

### Streaming Architecture
- SDK provides partial messages with streaming enabled
- Adapter buffers and translates stream events to ACP notifications in real-time
- File content is cached after reads to avoid duplicate fetches

### File Size Limits
- Default max file size: 50KB (enforced in `extractLinesWithByteLimit()`)
- Default lines to read: 2000
- Reads can be paginated with offset/limit parameters

### Command Support
- Slash commands come from SDK and are filtered (unsupported: `context`, `cost`, `login`, `logout`, `output-style:new`, `release-notes`, `todos`)
- MCP commands appear as `mcp:server:command` in input, converted to `/server:command (MCP)` format

### Root User Handling
- `bypassPermissions` mode is disabled when running as root/sudo (checked via `process.geteuid/getuid`)
- Prevents Claude Code from failing to start in restricted environments

## Testing

Tests use Vitest and are in `src/tests/`:
- `acp-agent.test.ts`: Core session and protocol handling
- `extract-lines.test.ts`: File reading byte limit logic
- `replace-and-calculate-location.test.ts`: Edit location tracking

Run tests frequently during development to catch regressions.

## Dependencies

**Key Production Dependencies:**
- `@agentclientprotocol/sdk`: ACP protocol implementation
- `@anthropic-ai/claude-agent-sdk`: Claude Code SDK with Query interface
- `@anthropic-ai/claude-code`: Model/command/permission definitions
- `@modelcontextprotocol/sdk`: MCP server for tool exposure
- `express`: HTTP server for permission endpoints
- `uuid`: Session ID generation

**Build/Test Dependencies:**
- `typescript`: Strict compilation to ES2020
- `eslint`, `prettier`: Linting and formatting
- `vitest`: Test runner with coverage support

## Debugging

- All console output (except stdout) goes to stderr to avoid protocol interference
- Hook responses are partially implemented (commented with TODO)
- Tool use and file content are cached for efficient result tracking
- Integration tests require `RUN_INTEGRATION_TESTS=true` environment variable
