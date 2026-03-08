# Noto Agent MVP — Implementation Plan

**Goal:** A working personal agent that can read/send email, manage calendar, and send Slack messages — modular enough to bolt on memory, more integrations, and multi-agent collaboration post-MVP.

**Core constraint:** Every component is a plug-in. Nothing is hardwired. Adding a new capability (memory, a new MCP server, a new skill) should never require touching existing components.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────┐
│                  Noto Platform                    │
│                                                   │
│  ┌───────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  Router    │  │  Context   │  │  Output     │  │
│  │  (intent → │  │  Manager   │  │  Formatter  │  │
│  │   skill)   │  │  (what the │  │  (structured│  │
│  │            │  │  agent sees)│  │   results)  │  │
│  └─────┬──────┘  └─────┬──────┘  └──────┬──────┘  │
│        └───────────┬───┘                 │         │
│                    ▼                     │         │
│  ┌─────────────────────────────────────┐ │         │
│  │         Pi Mono SDK (Agent Core)    │ │         │
│  │  ┌──────────┐  ┌────────────────┐   │ │         │
│  │  │ LLM API  │  │ Tool Registry  │   │◄┘         │
│  │  │ (Claude)  │  │ (MCP + custom) │   │           │
│  │  └──────────┘  └───────┬────────┘   │           │
│  └────────────────────────┼────────────┘           │
│                           │                         │
│  ┌────────────────────────▼────────────────────┐   │
│  │              MCP Server Manager              │   │
│  │  ┌────────┐ ┌──────────┐ ┌───────┐          │   │
│  │  │ Gmail  │ │ Calendar │ │ Slack │ ← add    │   │
│  │  │ MCP    │ │ MCP      │ │ MCP   │   more   │   │
│  │  └───┬────┘ └────┬─────┘ └──┬────┘   later  │   │
│  └──────┼───────────┼──────────┼────────────────┘   │
│         └───────────┼──────────┘                     │
│                     ▼                                │
│  ┌─────────────────────────────────┐                │
│  │     Nango (OAuth Token Mgmt)    │                │
│  └─────────────────────────────────┘                │
└──────────────────────────────────────────────────┘
```

---

## Phase 1: Foundation (Week 1)

### 1.1 Project Scaffold

```
noto/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                  # Entry point
│   ├── config/
│   │   └── noto.config.ts        # Central config (env vars, feature flags)
│   ├── core/
│   │   ├── agent.ts              # Pi Mono session wrapper
│   │   ├── router.ts             # Intent → skill routing
│   │   ├── context.ts            # Context assembly for agent calls
│   │   └── output.ts             # Response formatting
│   ├── integrations/
│   │   ├── mcp-manager.ts        # Lifecycle manager for MCP server processes
│   │   ├── nango.ts              # Nango client wrapper (token fetching)
│   │   └── servers/              # Per-server config and startup
│   │       ├── gmail.ts
│   │       ├── calendar.ts
│   │       └── slack.ts
│   ├── skills/                   # SKILL.md files (loaded by Pi Mono)
│   │   ├── email.skill.md
│   │   ├── calendar.skill.md
│   │   └── slack.skill.md
│   └── types/
│       └── index.ts              # Shared types
├── .env.example
└── README.md
```

**Key decisions:**
- **TypeScript** — Pi Mono SDK is TS-native, MCP SDK is TS-native, Nango client is TS-native. No polyglot tax.
- **Monorepo-ready but single package for MVP** — don't split into packages yet, but keep directory boundaries clean so extraction is trivial.
- **Config-driven MCP servers** — each server in `integrations/servers/` exports a config object (command, args, env). The MCP Manager starts/stops them generically.

### 1.2 Pi Mono Integration

- Install Pi Mono SDK (`@anthropic/pi-mono` or current package name)
- Create `core/agent.ts`: thin wrapper around `createAgentSession()`
- Wire Claude as primary LLM via Pi's Unified LLM API
- Implement a `tools` registry interface that accepts both:
  - MCP-discovered tools (auto-registered from running servers)
  - Custom tools (for Noto-specific actions like `search_notes` later)

**Plugin contract for tools:**
```ts
interface NotoTool {
  name: string;
  description: string;
  parameters: JSONSchema;
  execute: (params: Record<string, unknown>) => Promise<unknown>;
  source: 'mcp' | 'custom' | 'skill';
}
```

All tools — MCP, custom, skill-derived — conform to the same interface. The agent doesn't know or care where a tool came from.

### 1.3 Nango Setup

- Self-host Nango (Docker) or use Nango Cloud free tier for MVP
- Configure OAuth providers: Google (Gmail + Calendar scopes), Slack
- Create `integrations/nango.ts`:
  - `getToken(provider: string, userId: string): Promise<string>`
  - Handles token fetch, surfaces clear errors on auth failure
- Each MCP server config references Nango for its token instead of managing OAuth itself

**Why this matters for modularity:** Adding a new integration (e.g., Outlook) means:
1. Add OAuth config in Nango dashboard
2. Add a server config file in `integrations/servers/outlook.ts`
3. Add a skill file in `skills/outlook.skill.md`
4. Done. No changes to core.

---

## Phase 2: MCP Integrations (Week 2)

### 2.1 MCP Server Manager

`integrations/mcp-manager.ts` handles:
- **Startup**: reads server configs, spawns each as a child process (STDIO transport)
- **Tool discovery**: on startup, calls `tools/list` on each server, registers tools in the agent's tool registry
- **Health**: restarts crashed servers, logs errors
- **Shutdown**: clean teardown on process exit

**Server config shape:**
```ts
interface MCPServerConfig {
  id: string;                    // e.g. 'gmail'
  command: string;               // e.g. 'npx'
  args: string[];                // e.g. ['google-workspace-mcp']
  env: Record<string, string>;   // injected at runtime (tokens from Nango)
  enabled: boolean;              // toggle without removing config
}
```

### 2.2 Gmail + Google Calendar

- Use `taylorwilsdon/google_workspace_mcp` (covers both Gmail and Calendar in one server)
- Nango provides OAuth tokens with scopes: `gmail.modify`, `calendar.events`
- Write `skills/email.skill.md` and `skills/calendar.skill.md` — these tell the agent *when* and *how* to use the Gmail/Calendar tools (prompt engineering, not code)

### 2.3 Slack

- Use `korotovsky/slack-mcp-server` (battle-tested, 9k+ users)
- Nango provides Slack OAuth token
- Write `skills/slack.skill.md`

### 2.4 Integration Testing

- Test each MCP server independently (can it list tools? can it execute a simple action?)
- Test through the agent (can it route "send an email to X" → Gmail MCP → successful send?)
- Test Nango token refresh (expire a token, verify auto-refresh works)

---

## Phase 3: Routing & Context (Week 3)

### 3.1 Router

`core/router.ts` — for MVP, this is simple:
- The LLM itself routes. Pi Mono's tool-calling handles intent → tool selection natively.
- The Router's job is just to assemble the right system prompt + available tools for a given session.
- No fancy classifier needed yet. The LLM sees all available tools and picks the right one.

**Post-MVP hook:** Router becomes the place to add skill selection logic, multi-agent spawning, and context budgeting. For now, it's a pass-through with a good interface.

### 3.2 Context Manager

`core/context.ts` — controls what the agent sees:
- **System prompt**: Noto's personality + user preferences
- **Available tools**: filtered by what's enabled and authenticated
- **User context**: for MVP, just the current conversation. Post-MVP, this is where memory plugs in.

**Plugin contract for context sources:**
```ts
interface ContextSource {
  id: string;
  priority: number;              // higher = included first if context is tight
  getContext(userId: string): Promise<string>;
}
```

MVP has one source: conversation history. Post-MVP, you add `MemoryContextSource`, `NotesContextSource`, etc. — each implements the same interface, gets registered, and the Context Manager merges them by priority.

### 3.3 Output Formatter

`core/output.ts` — normalizes agent responses:
- Extracts structured data when the agent returns it (e.g., "email sent" → `{ action: 'email_sent', to: '...', subject: '...' }`)
- Surfaces errors clearly
- Post-MVP: feeds structured outputs into memory, action logs, etc.

---

## Phase 4: Integration Surface (Week 4)

### 4.1 API Layer

Expose Noto as a simple API so any frontend or app can talk to it:

```
POST /api/chat         — send a message, get a response
POST /api/auth/connect — initiate OAuth for a provider (redirects to Nango)
GET  /api/auth/status  — which providers are connected
GET  /api/tools        — list available tools and their status
```

**Why an API, not a CLI:** Noto needs to integrate with a frontend app and potentially other agents. An HTTP API is the universal integration surface.

### 4.2 Session Management

- Sessions are stateless on the server (externalized to Redis or even just in-memory for MVP)
- Each session holds: conversation history, active MCP connections, user ID
- Session state is serializable — this is critical for horizontal scaling post-MVP

### 4.3 Auth Flow (User-Facing)

1. User hits `/api/auth/connect?provider=google`
2. Noto redirects to Nango's OAuth flow
3. Nango handles consent, token exchange, storage
4. Nango redirects back to Noto with success/failure
5. Noto enables the corresponding MCP server for that user

---

## Modularity Contracts (The Important Part)

These are the interfaces that make post-MVP extension painless:

### Adding a new integration
1. Add OAuth config in Nango
2. Create `integrations/servers/{name}.ts` (MCPServerConfig)
3. Create `skills/{name}.skill.md`
4. Register in config → MCP Manager auto-discovers tools

### Adding memory (post-MVP)
1. Implement `ContextSource` interface in `core/memory.ts`
2. Register it with Context Manager
3. Agent automatically gets memory context injected — no changes to agent code

### Adding multi-agent (post-MVP)
1. Router evolves from pass-through to skill selector
2. Each "skill-agent" is a Pi Mono session with a filtered tool set
3. Noto orchestrator spawns skill-agents and merges results
4. Same MCP servers, same Nango tokens — agents share infrastructure

### Adding a new LLM
1. Configure in Pi Mono's Unified LLM API (supports 18+ providers)
2. Update Router with model selection logic (e.g., Sonnet for simple, Opus for complex)
3. No changes to tools, integrations, or skills

---

## What's Explicitly Out of Scope for MVP

- Memory / persistent context (Phase 2 feature, but the `ContextSource` interface is ready)
- Multi-agent collaboration (Phase 3 feature, but Router is designed for it)
- ClawHub skill marketplace browsing (nice-to-have, not needed for core functionality)
- Mobile / native app (API-first means any client can connect later)
- Multi-tenant / multi-user (MVP is single-user; session externalization prep makes this easy later)
- Composio or any paid integration platform

---

## Success Criteria for MVP

The MVP is done when you can have this conversation:

> **You:** "Send an email to alex@example.com about the meeting tomorrow"
> **Noto:** *(uses Gmail MCP)* Done. Sent "Meeting Tomorrow" to alex@example.com.
>
> **You:** "What's on my calendar this week?"
> **Noto:** *(uses Calendar MCP)* You have 3 events: ...
>
> **You:** "Tell the team in #engineering on Slack that the release is delayed"
> **Noto:** *(uses Slack MCP)* Posted to #engineering: "The release is delayed."

And critically: adding a 4th integration (e.g., Notion, Discord, Outlook) requires **zero changes to core code** — just a new server config + skill file + Nango OAuth setup.

---

## Tech Stack Summary

| Component | Choice | Why |
|-----------|--------|-----|
| Language | TypeScript | Everything else is TS-native |
| Agent Core | Pi Mono SDK | Lightweight, embeddable, SKILL.md native |
| Primary LLM | Claude (Sonnet for MVP) | Best tool-calling, cost-efficient |
| Tool Protocol | MCP (STDIO transport) | Standard, huge ecosystem |
| OAuth | Nango (self-hosted or cloud) | Manages token lifecycle, open-source |
| Gmail + Cal | google_workspace_mcp | One server covers both |
| Slack | slack-mcp-server | Battle-tested, 9k+ users |
| API Framework | Hono or Fastify | Lightweight, TS-native |
| Session Store | In-memory (MVP) → Redis (scale) | Start simple, externalize later |
