# Noto Agent MVP — Implementation Plan (v2)

**Goal:** A working personal agent that can read/send email, manage calendar, and send Slack messages — modular enough to bolt on memory, more integrations, and multi-agent collaboration post-MVP.

**Core constraint:** Every component is a plug-in. Nothing is hardwired. Adding a new capability (memory, a new tool, a new skill) should never require touching existing components.

**Key architecture decision:** CLI tools over MCP. Each integration is a thin TypeScript function that calls the API directly — no protocol layer, no persistent server processes, no JSON-RPC overhead. LLMs are trained on CLI patterns and perform better with lightweight tool descriptions (~100 tokens) vs MCP schemas (~55,000 tokens). MCP can be wrapped around these same functions later if needed for enterprise/compliance use cases.

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
│  │  │ (Claude)  │  │ (CLI tools)    │   │           │
│  │  └──────────┘  └───────┬────────┘   │           │
│  └────────────────────────┼────────────┘           │
│                           │                         │
│  ┌────────────────────────▼────────────────────┐   │
│  │                CLI Tools                     │   │
│  │  ┌────────┐ ┌──────────┐ ┌───────┐          │   │
│  │  │ Gmail  │ │ Calendar │ │ Slack │ ← add    │   │
│  │  │ tool   │ │ tool     │ │ tool  │   more   │   │
│  │  └───┬────┘ └────┬─────┘ └──┬────┘   later  │   │
│  └──────┼───────────┼──────────┼────────────────┘   │
│         └───────────┼──────────┘                     │
│                     ▼                                │
│  ┌─────────────────────────────────┐                │
│  │     Nango (OAuth Token Mgmt)    │                │
│  └─────────────────────────────────┘                │
└──────────────────────────────────────────────────┘
```

**What changed from v1:** The MCP Server Manager and persistent MCP processes are gone. Each integration is now a direct function call: `tool function → Nango token → API call → result`. No child processes, no JSON-RPC, no protocol overhead.

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
│   ├── tools/                    # CLI tool functions (one per integration)
│   │   ├── registry.ts           # Auto-discovers and registers tools
│   │   ├── gmail.ts              # Gmail send/read/search
│   │   ├── calendar.ts           # Calendar list/create/update events
│   │   └── slack.ts              # Slack send/read messages
│   ├── auth/
│   │   └── nango.ts              # Nango client wrapper (token fetching)
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
- **TypeScript** — Pi Mono SDK, Nango client, and Google/Slack API clients are all TS-native. No polyglot tax.
- **Monorepo-ready but single package for MVP** — keep directory boundaries clean so extraction is trivial later.
- **Convention-based tool discovery** — every file in `tools/` that exports a `NotoTool` gets auto-registered. Drop in a file, it's available.

### 1.2 Pi Mono Integration

- Install Pi Mono SDK (`@anthropic/pi-mono` or current package name)
- Create `core/agent.ts`: thin wrapper around `createAgentSession()`
- Wire Claude as primary LLM via Pi's Unified LLM API
- Implement the tool registry (`tools/registry.ts`) that scans `tools/` and registers everything it finds

**Plugin contract for tools:**
```ts
interface NotoTool {
  name: string;                                         // e.g. 'gmail_send'
  description: string;                                  // short, for the LLM
  parameters: JSONSchema;                               // input shape
  execute: (params: Record<string, unknown>) => Promise<ToolResult>;
}

interface ToolResult {
  success: boolean;
  data?: unknown;
  error?: string;
}
```

Every tool — Gmail, Slack, Calendar, and future ones like memory or notes — conforms to this interface. The agent doesn't know or care how the tool works internally.

**Each tool file exports an array of NotoTools:**
```ts
// tools/gmail.ts
export const tools: NotoTool[] = [
  { name: 'gmail_send', description: 'Send an email', ... },
  { name: 'gmail_read', description: 'Read recent emails', ... },
  { name: 'gmail_search', description: 'Search emails', ... },
];
```

The registry reads all files in `tools/`, collects exports, and registers them with Pi Mono. Adding a new tool = adding a new file.

### 1.3 Nango Setup

- Self-host Nango (Docker) or use Nango Cloud free tier for MVP
- Configure OAuth providers: Google (Gmail + Calendar scopes), Slack
- Create `auth/nango.ts`:
  - `getToken(provider: string, userId: string): Promise<string>`
  - Handles token fetch, surfaces clear errors on auth failure
- Each tool function calls `getToken()` at execution time — tokens are never cached in the tool itself

**Why this matters for modularity:** Adding a new integration (e.g., Outlook) means:
1. Add OAuth config in Nango dashboard
2. Add a tool file `tools/outlook.ts` (exports `NotoTool[]`)
3. Add a skill file `skills/outlook.skill.md`
4. Done. No changes to core, no changes to existing tools.

---

## Phase 2: CLI Tool Integrations (Week 2)

### 2.1 Gmail Tool

`tools/gmail.ts` — thin functions over the Gmail REST API:

```ts
// Simplified — each function:
// 1. Gets token from Nango
// 2. Calls Gmail API directly (googleapis SDK or raw fetch)
// 3. Returns structured result

gmail_send({ to, subject, body })    → sends email, returns messageId
gmail_read({ count?, labels? })      → returns recent emails (subject, from, snippet)
gmail_search({ query })              → searches emails, returns matches
```

- Uses `googleapis` npm package (official Google SDK) or raw `fetch` — whichever is lighter
- Nango provides OAuth token with `gmail.modify` scope
- Each function is 20-40 lines. No abstraction needed.

### 2.2 Google Calendar Tool

`tools/calendar.ts` — same pattern:

```ts
calendar_list({ days? })             → returns upcoming events
calendar_create({ title, start, end, attendees? }) → creates event
calendar_update({ eventId, changes })  → updates existing event
```

- Same Google OAuth token as Gmail (shared Nango connection, different scopes)
- Uses `googleapis` SDK

### 2.3 Slack Tool

`tools/slack.ts`:

```ts
slack_send({ channel, message })     → posts message to channel
slack_read({ channel, count? })      → reads recent messages
slack_channels()                     → lists available channels
```

- Uses `@slack/web-api` npm package (official Slack SDK)
- Nango provides Slack OAuth token

### 2.4 Skill Files

Each SKILL.md tells the agent *when* and *how* to use the tools (prompt engineering, not code):

```markdown
<!-- skills/email.skill.md -->
# Email Skill

You can send, read, and search emails using these tools:
- gmail_send: Use when the user wants to compose and send an email
- gmail_read: Use when the user asks about recent emails
- gmail_search: Use when the user wants to find specific emails

Always confirm the recipient and subject before sending.
Never send an email without explicit user approval.
```

These are ~100 tokens each — compared to ~55,000 tokens for an MCP server's full schema dump.

### 2.5 Integration Testing

- Test each tool function independently (unit tests with mocked API responses)
- Test through the agent (can it route "send an email to X" → `gmail_send` → successful result?)
- Test Nango token refresh (expire a token, verify transparent re-auth)

---

## Phase 3: Routing & Context (Week 3)

### 3.1 Router

`core/router.ts` — for MVP, this is simple:
- The LLM itself routes. Pi Mono's tool-calling handles intent → tool selection natively.
- The Router's job is just to assemble the right system prompt + available tools + skill context for a given session.
- No fancy classifier needed yet. The LLM sees all available tools and picks the right one.
- Skills (SKILL.md files) are injected into the system prompt to guide tool selection — lightweight and token-efficient.

**Post-MVP hook:** Router becomes the place to add skill selection logic, multi-agent spawning, and context budgeting. For now, it's a pass-through with a good interface.

### 3.2 Context Manager

`core/context.ts` — controls what the agent sees:
- **System prompt**: Noto's personality + user preferences
- **Skills**: loaded from `skills/*.skill.md`, injected into prompt
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

MVP has two sources: conversation history + skill files. Post-MVP, you add `MemoryContextSource`, `NotesContextSource`, etc. — each implements the same interface, gets registered, and the Context Manager merges them by priority.

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
- Each session holds: conversation history, user ID, authenticated providers
- Session state is serializable — this is critical for horizontal scaling post-MVP

### 4.3 Auth Flow (User-Facing)

1. User hits `/api/auth/connect?provider=google`
2. Noto redirects to Nango's OAuth flow
3. Nango handles consent, token exchange, storage
4. Nango redirects back to Noto with success/failure
5. Noto's tool registry marks Google tools as available for that user

---

## Modularity Contracts (The Important Part)

These are the interfaces that make post-MVP extension painless:

### Adding a new integration
1. Add OAuth config in Nango
2. Create `tools/{name}.ts` (exports `NotoTool[]`)
3. Create `skills/{name}.skill.md`
4. Drop them in → registry auto-discovers → done. Zero changes to core.

### Adding memory (post-MVP)
1. Implement `ContextSource` interface in a new `memory/` directory
2. Register it with Context Manager
3. Optionally add a `memory_search` tool in `tools/memory.ts`
4. Agent automatically gets memory context + tools — no changes to existing code

### Adding multi-agent (post-MVP)
1. Router evolves from pass-through to skill selector
2. Each "skill-agent" is a Pi Mono session with a filtered tool set
3. Noto orchestrator spawns skill-agents and merges results
4. Same tools, same Nango tokens — agents share infrastructure

### Adding a new LLM
1. Configure in Pi Mono's Unified LLM API (supports 18+ providers)
2. Update Router with model selection logic (e.g., Sonnet for simple, Opus for complex)
3. No changes to tools, integrations, or skills

### Upgrading to MCP later (if needed)
1. Wrap existing tool functions in an MCP server (each function becomes a tool handler)
2. The tool logic stays identical — only the transport changes
3. Useful for: enterprise deployments, compliance/audit trails, third-party agent access

---

## What's Explicitly Out of Scope for MVP

- Memory / persistent context (post-MVP, but `ContextSource` interface is ready)
- Multi-agent collaboration (post-MVP, but Router is designed for it)
- MCP protocol (can wrap tools later if enterprise needs arise)
- ClawHub skill marketplace browsing
- Mobile / native app (API-first means any client can connect later)
- Multi-tenant / multi-user (MVP is single-user; session externalization prep makes this easy later)
- Composio or any paid integration platform

---

## Success Criteria for MVP

The MVP is done when you can have this conversation:

> **You:** "Send an email to alex@example.com about the meeting tomorrow"
> **Noto:** Done. Sent "Meeting Tomorrow" to alex@example.com.
>
> **You:** "What's on my calendar this week?"
> **Noto:** You have 3 events: ...
>
> **You:** "Tell the team in #engineering on Slack that the release is delayed"
> **Noto:** Posted to #engineering: "The release is delayed."

And critically: adding a 4th integration (e.g., Notion, Discord, Outlook) requires **zero changes to core code** — just a new tool file + skill file + Nango OAuth setup.

---

## Tech Stack Summary

| Component | Choice | Why |
|-----------|--------|-----|
| Language | TypeScript | Everything else is TS-native |
| Agent Core | Pi Mono SDK | Lightweight, embeddable, SKILL.md native |
| Primary LLM | Claude (Sonnet for MVP) | Best tool-calling, cost-efficient |
| Tool Approach | CLI/direct functions | 33% more token-efficient than MCP, simpler to debug, no protocol overhead |
| OAuth | Nango (self-hosted or cloud) | Manages token lifecycle, open-source |
| Gmail + Cal | googleapis SDK | Official Google client, direct API calls |
| Slack | @slack/web-api | Official Slack client, direct API calls |
| API Framework | Hono or Fastify | Lightweight, TS-native |
| Session Store | In-memory (MVP) → Redis (scale) | Start simple, externalize later |
