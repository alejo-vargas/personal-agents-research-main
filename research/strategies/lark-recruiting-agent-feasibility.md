# Lark Recruiting Agent — Feasibility & Architecture Research

**Date:** 2026-03-11
**Status:** Feasible — Recommended to build
**TL;DR:** Yes, you can build an AI agent that lives inside Lark as a bot "employee," monitors a group chat for job opportunities, searches your candidate database in Bitable, and responds with matches. All the APIs exist. The official Lark MCP server covers most of what you need, and Bitable has a dedicated MCP server for SQL-like queries.

---

## The Use Case

1. Job opportunities get posted in a **Lark group chat**
2. An AI agent (appearing as a bot/team member) **monitors that chat**
3. When it detects a new opportunity, it **parses the job requirements**
4. It **searches the candidate database** (stored in Lark Bitable) for matches
5. It **responds in the group chat** with candidate recommendations

---

## Verdict: Fully Feasible

Every piece of this workflow has API support:

| Capability | Supported? | How |
|-----------|-----------|-----|
| Bot appears as team member in group chat | ✅ Yes | Lark Custom App with Bot feature |
| Bot receives ALL messages in group (not just @mentions) | ✅ Yes | `im:message.group_msg` permission (sensitive — needs admin approval) |
| Bot sends messages/replies in group chat | ✅ Yes | `im:message:send_as_bot` permission |
| Bot sends rich formatted responses (cards, tables) | ✅ Yes | Lark Interactive Message Cards |
| Read/search candidate data from Bitable | ✅ Yes | Bitable Search Records API + Bitable MCP server |
| Real-time event subscription (new messages) | ✅ Yes | WebSocket (recommended) or Webhook |
| Parse unstructured job descriptions | ✅ Yes | LLM-powered (Claude/GPT) |

---

## Lark Bot Framework — Key Findings

### How Bots Work in Lark
- You create a **Custom App** on the [Lark Open Platform](https://open.larksuite.com)
- Enable the **Bot** feature on the app
- The bot gets added to group chats like any team member
- It appears with its own name and avatar in the chat

### Two Bot Types
1. **ChatBot** — Full-featured, requires App ID + App Secret, can send/receive messages, subscribe to events
2. **NotificationBot** — Webhook-only, simpler but limited (can only send, not receive)

**You need a ChatBot** for this use case.

### Receiving Messages
- **Default behavior:** Bots only receive messages where they're `@mentioned`
- **What you need:** The `im:message.group_msg` permission to receive ALL messages in the group
- **Caveat:** This is a **sensitive permission** — requires enterprise admin approval
- **Alternative:** You could require users to `@bot` when posting opportunities. Simpler permission model but less seamless

### Event Subscription
Two methods:
1. **WebSocket (Long Connection)** — Recommended. No public server needed, no SSL certs. Bot connects outbound to Lark's servers
2. **Webhook** — Lark pushes events to your server URL. Requires public endpoint. Needed for Lark International in some cases

Key events to subscribe to:
- `im.message.receive_v1` — New message received
- `im.chat.member.bot.added_v1` — Bot added to a group
- `im.message.message_read_v1` — Message read receipts

### Sending Messages
- Bot sends via `POST /open-apis/im/v1/messages`
- Supports text, rich text, images, and **Interactive Message Cards** (best for showing candidate matches — supports tables, buttons, links)
- Can reply to specific messages (threaded)
- Can mention specific users with `@`

### Required Permissions
| Permission | Purpose |
|-----------|---------|
| `im:message.group_msg` | Read ALL group messages (sensitive) |
| `im:message.group_at_msg:readonly` | Read @bot messages (fallback) |
| `im:message:send_as_bot` | Send messages as the bot |
| `im:message` | General message access |
| `im:resource` | Read/upload images and files |
| `bitable:app:readonly` | Read Bitable data |

### Authentication
- **App Access Token:** `POST /open-apis/auth/v3/app_access_token/internal` with `app_id` + `app_secret`
- **Tenant Access Token:** For org-wide access (recommended for this use case)
- Tokens expire every 2 hours, need refresh

---

## Lark Bitable — Candidate Database Access

### What Is Bitable?
Lark's database-spreadsheet hybrid (like Airtable). Perfect for structured candidate data with fields like name, skills, experience, location, availability, etc.

### API Capabilities
- **Search Records:** `POST /open-apis/bitable/v1/apps/:app_token/tables/:table_id/records/search`
- **List Records:** Paginated listing of all records
- **Get Record:** Fetch a specific record by ID
- **Filter System:** Supports `AND`/`OR` conjunctions with field-level conditions

### Filter Example (Candidate Search)
```json
{
  "filter": {
    "conjunction": "and",
    "conditions": [
      {
        "field_name": "Skills",
        "operator": "contains",
        "value": ["Python"]
      },
      {
        "field_name": "Status",
        "operator": "is",
        "value": ["Available"]
      },
      {
        "field_name": "Experience_Years",
        "operator": "isGreater",
        "value": ["3"]
      }
    ]
  },
  "page_size": 20
}
```

### Limitations
- Search API supports up to **500 rows per query** with pagination
- Filter values limited to **10 per condition**
- No full-text search across all fields — you filter by specific field names
- **This is why an LLM layer matters:** The LLM parses the job description into structured filter criteria

---

## MCP Server Landscape for Lark

### 1. Official Lark OpenAPI MCP Server ⭐ (Recommended)
- **Repo:** [larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp)
- **Maintainer:** Lark/ByteDance (official)
- **Coverage:** Wraps almost ALL Feishu/Lark APIs — messaging, groups, docs, calendar, Bitable, contacts, tasks
- **Key tools for this project:**
  - `im.v1.message.create` — Send messages
  - `im.v1.message.list` — List messages
  - `im.v1.chat.create` — Create/manage chats
  - `preset.base.batch` — Bitable batch operations
  - `preset.doc.default` — Document operations
- **Transport:** stdio (for Claude/Cursor) or SSE (HTTP)
- **Config:** Requires `APP_ID` and `APP_SECRET` env vars
- **Limitations:** No file upload/download. No direct editing of cloud docs (read + import only)
- **Selective tools:** Use `-t` flag to enable only needed tools (reduces token usage)

### 2. Bitable MCP Server (by lloydzhou)
- **Repo:** [lloydzhou/bitable-mcp](https://github.com/lloydzhou/bitable-mcp)
- **Listed in:** Official MCP servers repository
- **Focus:** Dedicated Bitable access with SQL-like queries
- **Tools:**
  - `list_table` — List all tables in a Bitable base
  - `describe_table` — Get column headers and data types
  - `read_query` — Execute SQL-like queries against Bitable data
- **Read-only** (safe — agent can't accidentally modify candidate data)
- **Config:** `PERSONAL_BASE_TOKEN` + `APP_TOKEN` env vars
- **Great for:** Your candidate database queries — more natural than raw filter API

### 3. Feishu MCP (by cso1z) — Document Specialist
- **Focus:** Deep Feishu Docs integration (block-level editing, search, folder management)
- **Best for:** If candidate data is in Lark Docs rather than Bitable
- **Less relevant** if data is in Bitable

### 4. Other Community Servers
- **kone-net/mcp_server_lark** — Sheets only, 2 stars, very early stage. Skip.
- **loonghao/feishu-bot-mcp-server** — Bot messaging focused, Python. Early stage.

### Recommendation
Use the **Official Lark MCP server** for messaging + chat operations, and the **Bitable MCP server** for candidate database queries. Together they cover the full workflow.

---

## Proposed Architecture

### Option A: MCP-Native Agent (Simpler, Recommended for MVP)

```
┌─────────────────────────────────────────────┐
│              Lark Group Chat                │
│  (Job opportunities posted here)            │
└──────────────┬──────────────────────────────┘
               │ Event: im.message.receive_v1
               │ (WebSocket or Webhook)
               ▼
┌─────────────────────────────────────────────┐
│           Event Handler Service             │
│  (Node.js / Python — lightweight)           │
│                                             │
│  1. Receives new message event              │
│  2. Checks if it's a job opportunity        │
│     (keyword heuristic or always forward)   │
│  3. Forwards to AI Agent                    │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│            AI Agent (Claude)                │
│                                             │
│  MCP Servers connected:                     │
│  ├── lark-openapi-mcp (messaging)           │
│  └── bitable-mcp (candidate DB)             │
│                                             │
│  Workflow:                                  │
│  1. Parse job description → extract:        │
│     - Required skills                       │
│     - Experience level                      │
│     - Location/timezone                     │
│     - Contract type                         │
│     - Industry                              │
│  2. Query Bitable for matching candidates   │
│  3. Rank/score matches                      │
│  4. Send response to group chat via MCP     │
└─────────────────────────────────────────────┘
```

**Pros:**
- Simple architecture — event handler + AI agent with MCP tools
- Claude (or GPT) handles the hard part: parsing unstructured job posts and matching logic
- MCP servers are pre-built, well-maintained
- Read-only Bitable access = safe

**Cons:**
- Requires a running service for event handling (can't be purely MCP-triggered)
- LLM costs per job posting (but likely low volume)

### Option B: Full Custom Bot (More Control, More Work)

```
┌──────────────────────────────────────────────┐
│           Custom Lark Bot Service            │
│  (Python/Node.js — always running)           │
│                                              │
│  Components:                                 │
│  ├── Lark SDK (go-lark, lark-node, etc.)     │
│  ├── Event subscriber (WebSocket)            │
│  ├── Bitable client (direct API calls)       │
│  ├── LLM client (Claude API)                 │
│  └── Message formatter (Interactive Cards)   │
│                                              │
│  Flow:                                       │
│  1. Subscribe to group chat events           │
│  2. On new message → send to Claude for      │
│     classification + requirement extraction  │
│  3. Query Bitable API with extracted filters  │
│  4. Format results as Interactive Card        │
│  5. Send card reply to group chat             │
└──────────────────────────────────────────────┘
```

**Pros:**
- Full control over every step
- Can optimize API calls (batch, cache, pre-filter)
- Can use Interactive Message Cards for rich UI (tables with candidate info, action buttons)
- No MCP dependency

**Cons:**
- More code to write and maintain
- Need to handle auth token refresh, rate limiting, error recovery
- Duplicates what MCP servers already provide

### Recommendation: Start with Option A, Evolve to Option B

MVP with MCP servers to validate the concept. If you need richer UI (Interactive Cards), custom caching, or tighter control, migrate to a custom bot. The event handler service is needed either way.

---

## Implementation Plan

### Phase 1: Proof of Concept (1-2 days)
1. Create a Custom App on [Lark Open Platform](https://open.larksuite.com)
2. Enable Bot feature, configure permissions
3. Request `im:message.group_msg` from admin (or start with @mention-only)
4. Set up WebSocket event subscription
5. Build minimal event handler that logs incoming messages
6. Connect Claude with Bitable MCP server to your candidate database
7. Test: manually trigger a query, verify it returns sensible matches

### Phase 2: Core Agent (2-3 days)
1. Build the event handler service (Node.js or Python)
2. Integrate Claude API for job description parsing
3. Connect Bitable MCP or direct API for candidate search
4. Build response formatting (start with plain text, upgrade to cards)
5. Deploy and add bot to the target group chat
6. Test end-to-end: post a job → agent responds with candidates

### Phase 3: Polish (ongoing)
1. Interactive Message Cards with candidate details, profile links
2. Feedback loop — "Was this match helpful?" buttons
3. Candidate ranking/scoring refinement
4. Handle edge cases: vague job posts, no matches, too many matches
5. Optional: agent asks clarifying questions in the chat before searching

---

## Key Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| `im:message.group_msg` permission denied by admin | Agent can't auto-detect job posts | Fall back to @mention trigger — users `@agent` when posting |
| Bitable data structure is messy/inconsistent | Poor search results | Pre-clean data; use LLM to fuzzy-match despite inconsistencies |
| LLM misparses job requirements | Wrong candidates surfaced | Include confidence scores; let users refine with follow-up messages |
| Rate limiting on Bitable API | Slow responses during burst | Cache recent queries; batch operations; low-volume use case unlikely to hit limits |
| Lark International vs Feishu differences | WebSocket may not work on Lark Intl | Use Webhook as fallback; check your specific Lark edition |

---

## Existing Similar Projects

### On Lark/Feishu
- **Feishu Hire** — Lark's built-in recruitment module. Full ATS, but it's a separate product, not a custom agent. Doesn't do the "monitor group chat → match candidates" workflow you want
- **Lark-OpenAI** ([ConnectAI-E/Lark-OpenAI](https://github.com/ConnectAI-E/Lark-OpenAI)) — GPT-4 bot for Lark. Good reference for bot architecture but not recruiting-specific
- **lark_bot** ([Mgrsc/lark_bot](https://github.com/Mgrsc/lark_bot)) — Configurable AI assistant for Lark with MCP support. Could be adapted

### AI Recruiting Agents (General)
- The market is exploding with AI recruiting tools (HireVue, Paradox, SeekOut), but they're all SaaS platforms, not custom agents in your chat
- The custom "agent-as-employee in chat" pattern is more common on Slack (numerous Slack bots for recruiting), less explored on Lark
- No one appears to have built exactly what you're describing — a Lark chat-native agent that matches candidates from Bitable. This is a differentiated approach

### Architecture Patterns for Candidate Matching
- **Embedding-based:** Embed both job descriptions and candidate profiles, find nearest neighbors. Good for large databases (1000+ candidates)
- **LLM-based parsing + structured search:** Parse job → extract criteria → query database with filters. Better for smaller databases with structured data (your Bitable case)
- **Hybrid:** LLM parses the job, structured search narrows candidates, LLM re-ranks the shortlist. Best results

**For your case:** LLM-based parsing + Bitable structured search is the right starting point. If the candidate database grows large, add embeddings later.

---

## MCP Server Configuration Quick Reference

### Official Lark MCP Server
```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": [
        "-y", "@anthropic-ai/lark-openapi-mcp",
        "-t", "im.v1.message.create,im.v1.message.list,preset.base.batch"
      ],
      "env": {
        "LARK_APP_ID": "your-app-id",
        "LARK_APP_SECRET": "your-app-secret"
      }
    }
  }
}
```

### Bitable MCP Server
```json
{
  "mcpServers": {
    "bitable": {
      "command": "uvx",
      "args": ["bitable-mcp"],
      "env": {
        "PERSONAL_BASE_TOKEN": "your-personal-base-token",
        "APP_TOKEN": "your-app-token"
      }
    }
  }
}
```

---

## Open Questions for Alejandro

1. **Is your Lark instance "Lark" (international) or "Feishu" (China)?** This affects WebSocket availability and some API endpoints
2. **What does your candidate data look like in Bitable?** Column names, data types, how many records? This shapes the search strategy
3. **Do you have Lark admin access** to approve the `im:message.group_msg` sensitive permission? Or do you need to go through an IT admin?
4. **Volume:** How many job opportunities get posted per day/week? This affects whether you need caching or can just query on every message
5. **Should the agent respond to every message or only detected job posts?** This shapes the classification step
6. **Do you want the agent to @mention specific people** (e.g., "Hey @recruiter, these candidates might fit") or just post results?

---

## Sources

- [Lark Open Platform — Bot Overview](https://open.larksuite.com/document/client-docs/bot-v3/bot-overview)
- [Lark Receive Message Event](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/reference/im-v1/message/events/receive)
- [Lark Bitable Search Records API](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/reference/bitable-v1/app-table-record/search)
- [Lark Bitable Filter Guide](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/reference/bitable-v1/app-table-record/record-filter-guide)
- [Official Lark OpenAPI MCP — GitHub](https://github.com/larksuite/lark-openapi-mcp)
- [Bitable MCP Server — GitHub](https://github.com/lloydzhou/bitable-mcp)
- [go-lark SDK — GitHub](https://github.com/go-lark/lark)
- [Lark-OpenAI Bot — GitHub](https://github.com/ConnectAI-E/Lark-OpenAI)
- [cso1z Feishu MCP — PulseMCP](https://www.pulsemcp.com/servers/cso1z-feishu)
