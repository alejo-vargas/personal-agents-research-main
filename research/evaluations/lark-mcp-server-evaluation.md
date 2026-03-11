# Lark MCP Server Evaluation

**Date:** 2026-03-11
**Status:** Complete
**Verdict:** There is an **official Lark MCP server from ByteDance/Larksuite** plus a rich ecosystem of community alternatives. The official one is the right starting point for most use cases.

---

## 1. What Is the Lark MCP Server?

The Lark MCP server is a Model Context Protocol bridge that exposes Lark/Feishu Open Platform APIs as MCP "tools" that AI agents (Claude, Cursor, Trae, etc.) can invoke. It lets an LLM programmatically interact with Lark's collaboration suite — messaging, documents, calendars, Bitable, tasks, wiki, and contacts.

### Is It Official?

**Yes.** ByteDance/Larksuite maintains an official MCP server:

- **Repo:** [larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp)
- **npm:** [@larksuiteoapi/lark-mcp](https://www.npmjs.com/package/@larksuiteoapi/lark-mcp)
- **License:** MIT
- **Language:** TypeScript (99.9%)
- **Stars:** ~533 | **Forks:** ~71 | **Commits:** 65
- **Latest release:** v0.5.1 (August 2025)
- **Status:** Beta (actively maintained, regular releases since ~May 2025)

There are also 6+ community-built alternatives (detailed in Section 6).

---

## 2. Tools & Capabilities (Official Server)

The official server wraps "almost all" Feishu/Lark Open Platform APIs. Tools are organized into **preset collections** you enable via the `-t` flag:

| Preset | What It Covers |
|--------|---------------|
| `preset.default` | All preset tools (default if `-t` not specified) |
| `preset.light` | Minimal set for reduced token usage |
| `preset.im.default` | Group creation, message send/list, chat management |
| `preset.base.default` | Bitable table creation, field management, record CRUD |
| `preset.base.batch` | Bitable batch create/update records |
| `preset.doc.default` | Document content reading, importing, permissions |
| `preset.task.default` | Task creation, member management |
| `preset.calendar.default` | Calendar event creation, free/busy queries |

### Specific Capabilities Confirmed

- **Send messages** in various formats (text, rich text, cards)
- **Read message history** from chats
- **Create and manage group chats**
- **Read document content** (Docx format)
- **Import documents** into Lark
- **Search wiki nodes** and cloud documents
- **Create Bitable apps, tables, fields, and records**
- **Batch operations** on Bitable records
- **Calendar event management** and scheduling
- **Contact lookups** (user ID by email/phone)
- **Task management** (create tasks, manage members)

### You Can Also Use Individual API Endpoints

Beyond presets, you can enable specific API tools by name:
```
-t im.v1.message.create,im.v1.chat.list,bitable.v1.app_table.create
```

Non-preset APIs are available but "have not undergone compatibility testing" — the AI may struggle with them.

---

## 3. Maturity Assessment

### Release History

| Version | Date | Key Changes |
|---------|------|-------------|
| v0.3.0 | May 2025 | Dev doc retrieval, `--token-mode`, SDK upgrade |
| v0.3.1 | May 2025 | `preset.light`, field type fixes |
| v0.4.0 | Jun 2025 | StreamableHTTP transport, MCP Auth, login/logout CLI, Node 20+ (breaking) |
| v0.4.1 | Jul 2025 | Auth flow fixes |
| v0.5.0 | Jul 2025 | Auto-browser login, OAuth fixes, latest API sync |
| v0.5.1 | Aug 2025 | Latest stable |

### Verdict on Maturity

- **Active development** — 6 releases in ~3 months (May–Aug 2025), 65 commits
- **Officially backed** by ByteDance/Larksuite (the `larksuite` GitHub org)
- **Beta status** — the README explicitly states "Features and APIs may change"
- **npm downloads:** ~12.7K (decent for a niche MCP server)
- **533 stars** — strong signal for a specialized integration tool
- **Open issues not excessive** — project is responsive

**Bottom line:** Mature enough for development and internal tooling. Not yet "1.0 stable" — expect API surface changes between minor versions.

---

## 4. Limitations

### Explicitly Documented

1. **No file upload/download** — cannot upload attachments or download files through MCP
2. **No direct document editing** — can import and read Feishu cloud documents, but cannot modify document content in place
3. **Non-preset APIs are untested** — enabling arbitrary Lark APIs may produce poor LLM tool-use behavior
4. **Token limits** — enabling too many tools at once can hit context window limits; use `preset.light` or selective `-t` to mitigate
5. **Beta stability** — features and APIs may change between versions

### Implied/Inferred Limitations

6. **No approval workflow support** in presets (Lark has approval APIs, but no preset covers them)
7. **No Lark email integration** visible in presets
8. **No real-time event subscriptions** — MCP is request/response, not a webhook receiver
9. **No Lark Meetings/Video API** integration mentioned
10. **Requires Node.js 20+** (as of v0.4.0)

### What the Native Lark API Can Do That MCP Cannot

| Capability | Native API | MCP Server |
|-----------|-----------|------------|
| File upload/download | Yes | No |
| Edit document content | Yes | No (read/import only) |
| Approval workflows | Yes | No preset |
| Event subscriptions (webhooks) | Yes | No (not MCP's model) |
| Lark Email API | Yes | No preset |
| Meetings/Video conferences | Yes | Not exposed |

---

## 5. Setup Guide

### Prerequisites

1. Create a Lark app at [open.larksuite.com](https://open.larksuite.com/) (international) or [open.feishu.cn](https://open.feishu.cn/) (China)
2. Note your **App ID** and **App Secret**
3. Configure appropriate **permission scopes** on the app (e.g., `im:message`, `docx:document`, `bitable:app`)
4. Install **Node.js 20+**

### Basic Config (stdio mode, tenant token)

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "npx",
      "args": [
        "-y", "@larksuiteoapi/lark-mcp", "mcp",
        "-a", "<YOUR_APP_ID>",
        "-s", "<YOUR_APP_SECRET>"
      ]
    }
  }
}
```

### With OAuth / User Access Token

```bash
# Step 1: Login (opens browser)
npx -y @larksuiteoapi/lark-mcp login -a <APP_ID> -s <APP_SECRET>

# Step 2: Configure MCP with OAuth
```

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "npx",
      "args": [
        "-y", "@larksuiteoapi/lark-mcp", "mcp",
        "-a", "<APP_ID>",
        "-s", "<APP_SECRET>",
        "--oauth",
        "--token-mode", "user_access_token"
      ]
    }
  }
}
```

### International (Lark) vs China (Feishu) Domain

Add `--domain https://open.larksuite.com` for international Lark. Default is `https://open.feishu.cn`.

### Selective Tool Loading

```json
{
  "args": [
    "-y", "@larksuiteoapi/lark-mcp", "mcp",
    "-a", "<APP_ID>", "-s", "<APP_SECRET>",
    "-t", "preset.im.default,preset.base.default"
  ]
}
```

### Transport Modes

- **stdio** — default, works with Claude Desktop, Cursor, Trae
- **StreamableHTTP / SSE** — HTTP-based, for remote/multi-client scenarios

---

## 6. All Lark/Feishu MCP Servers Compared

### Tier 1: Official

| Project | Author | Language | Stars | Focus | Maturity |
|---------|--------|----------|-------|-------|----------|
| [larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp) | ByteDance (official) | TypeScript | 533 | Full Lark API coverage | Beta, actively maintained |

### Tier 2: Notable Community

| Project | Author | Language | Stars | Focus | Maturity |
|---------|--------|----------|-------|-------|----------|
| [ztxtxwd/open-feishu-mcp-server](https://github.com/ztxtxwd/open-feishu-mcp-server) | ztxtxwd | TypeScript | 82 | Documents + OAuth on Cloudflare Workers | Active, 123 commits |
| [lloydzhou/bitable-mcp](https://github.com/lloydzhou/bitable-mcp) | lloydzhou | Python | — | Bitable read-only (SQL queries!) | Listed in official MCP servers repo |
| [Roland0511/mcp-feishu-proj](https://github.com/Roland0511/mcp-feishu-proj) | Roland0511 | Python | — | Feishu Project Management | Active |

### Tier 3: Early/Niche

| Project | Language | Focus | Notes |
|---------|----------|-------|-------|
| [kone-net/mcp_server_lark](https://github.com/kone-net/mcp_server_lark) | Python | Sheets, messages, docs | 2 commits, 2 stars — very early |
| [loonghao/feishu-bot-mcp-server](https://github.com/loonghao/feishu-bot-mcp-server) | Python | Bot messaging | 1 commit — template stage |
| [lark-helper-mcp](https://pypi.org/project/lark-helper-mcp/) (PyPI) | Python | General integration | pip installable, FastMCP-based |
| [mcp-server-my-lark-doc](https://socket.dev/pypi/package/mcp-server-my-lark-doc) (PyPI) | Python | Document search | pip installable |
| [@be-link/feishu-doc-mcp](https://www.npmjs.com/package/@be-link/feishu-doc-mcp) | TypeScript | Document operations | npm package |

### Key Differentiator: open-feishu-mcp-server vs Official

| Aspect | Official (`larksuite`) | `open-feishu-mcp-server` |
|--------|----------------------|--------------------------|
| API breadth | Broadest (nearly all Lark APIs) | Documents-focused |
| Auth UX | Manual login CLI + config | Zero-config, auto token refresh |
| Document editing | Read/import only | Block creation, updates, nested blocks |
| Deployment | Local stdio or self-hosted HTTP | Cloudflare Workers (edge) |
| Tool optimization | Raw API-to-tool mapping (can be verbose) | Deeply optimized tool schemas for LLM use |
| Best for | Broad Lark automation | Document-heavy workflows |

---

## 7. Recommendation

**For general Lark MCP integration, start with the official `@larksuiteoapi/lark-mcp`.** It has the broadest API coverage, is maintained by ByteDance, and works with all major MCP clients.

**Consider `open-feishu-mcp-server` if:**
- Your primary use case is document creation/editing (the official one can't edit docs)
- You want zero-config OAuth with automatic token refresh
- You need remote/multi-user deployment (Cloudflare Workers)

**Consider `bitable-mcp` if:**
- You only need Bitable read access with SQL-like queries
- You want a Python-based solution
- You want read-only safety guarantees

**Key gap to watch:** File upload/download and direct document editing are the biggest missing pieces in the official server. These are on the roadmap but not yet available. If those are critical, the community `open-feishu-mcp-server` partially addresses document editing.

---

## Sources

- [larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp) — Official repo
- [@larksuiteoapi/lark-mcp on npm](https://www.npmjs.com/package/@larksuiteoapi/lark-mcp) — npm package
- [ztxtxwd/open-feishu-mcp-server](https://github.com/ztxtxwd/open-feishu-mcp-server) — Community alternative
- [lloydzhou/bitable-mcp](https://github.com/lloydzhou/bitable-mcp) — Bitable-specific MCP
- [lark-helper-mcp on PyPI](https://pypi.org/project/lark-helper-mcp/) — Python pip package
- [MCP Servers Directory](https://github.com/modelcontextprotocol/servers) — Official MCP servers list
- [Lark Open Platform](https://open.larksuite.com/) — Lark developer portal
