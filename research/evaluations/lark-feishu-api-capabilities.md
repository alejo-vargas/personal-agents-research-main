# Lark (Feishu) Open Platform API — Comprehensive Evaluation

**Date:** 2026-03-11
**Status:** Complete
**Confidence:** High (based on official docs, SDKs, and community sources)

---

## TL;DR

Lark's Open Platform is a mature, full-featured API that supports building bots that behave like employees in group chats. Bots can send/receive messages, react with emoji, mention users, read message history, and subscribe to real-time events via either webhooks or WebSocket. The Docs, Sheets, and Bitable APIs provide full programmatic CRUD access. Authentication uses three token types (app, tenant, user). Rate limits are per-API/per-app/per-tenant, with specific numbers varying by endpoint and subscription tier.

**Bottom line:** You can absolutely build an AI agent that monitors Lark group chats, reads documents/databases, and responds intelligently. The platform is well-documented and has official SDKs in Go, Node.js, Java, and Python.

---

## 1. Lark Bot Framework

### Can a bot act like an employee in group chats?

**Yes.** A Lark bot can be added to group chats and behave as a participant. Capabilities include:

| Capability | Supported | API/Method |
|---|---|---|
| Send text messages | Yes | `POST /open-apis/im/v1/messages` |
| Send rich text (markdown, cards) | Yes | Interactive Message Cards, Rich Text format |
| Read incoming messages | Yes | Event subscription (`im.message.receive_v1`) |
| Read message history | Yes | `GET /open-apis/im/v1/messages` (list) |
| Mention specific users | Yes | `PostTextMention` / `<at>` tag in message content |
| Mention @all | Yes | `PostTextMentionAll` |
| React with emoji | Yes | `AddReaction(messageID, emojiType)` |
| Reply to specific messages | Yes | `reply_in_thread` / parent message ID |
| Receive @mentions | Yes | Event with `mention` field |
| Send proactive messages | Yes | Bot can initiate messages, not just respond |
| List groups bot belongs to | Yes | `GET /open-apis/im/v1/chats` |

### Bot Types

1. **Chat Bot (Custom App Bot)** — Full-featured. Requires App ID + App Secret. Can send/receive messages, subscribe to events, call any API.
2. **Notification Bot (Webhook Bot)** — Simpler. Created from within a group chat. Only sends messages via webhook URL. Cannot receive messages or subscribe to events. Limited to 5 QPS / 100 QPM.

### Can bots proactively monitor group chats?

**Yes, two ways:**
- **Real-time events**: Subscribe to `im.message.receive_v1` to get notified instantly when any message is posted in a group the bot is in.
- **Polling history**: Use `GET /open-apis/im/v1/messages?container_id={chat_id}` to fetch message history on-demand.

The event-driven approach is strongly recommended over polling.

---

## 2. Lark Docs / Sheets / Bitable API

### Docs API

| Operation | Endpoint | Notes |
|---|---|---|
| Get document content | `GET /open-apis/docx/v1/documents/{document_id}/raw_content` | Returns doc content |
| Get document blocks | `GET /open-apis/docx/v1/documents/{document_id}/blocks` | Block-level access |
| Create document | `POST /open-apis/docx/v1/documents` | Create new docs programmatically |
| Search docs | Via Drive API search | Search by title/content across workspace |

### Sheets API

| Operation | Endpoint |
|---|---|
| Create spreadsheet | `POST /open-apis/sheets/v3/spreadsheets` |
| Read cell values | `GET /open-apis/sheets/v2/spreadsheets/{token}/values/{range}` |
| Write cell values | `PUT /open-apis/sheets/v2/spreadsheets/{token}/values` |
| Query sheet metadata | `GET /open-apis/sheets/v3/spreadsheets/{token}/sheets/query` |
| Find cells | `POST /open-apis/sheets/v3/spreadsheets/{token}/sheets/{sheet_id}/find` |

### Bitable API (Database Product)

Bitable is Lark's Airtable-equivalent. Full CRUD API available:

| Operation | Endpoint |
|---|---|
| List tables | `GET /open-apis/bitable/v1/apps/{app_token}/tables` |
| List records | `GET /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records` |
| Create records | `POST /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records` |
| Update records | `PUT /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/{record_id}` |
| Batch delete records | `POST /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/batch_delete` |
| Create/manage views | `POST /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/views` |
| Get field metadata | `GET /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields` |

Authentication: Requires `tenant_access_token` or `user_access_token`, plus the bot/app must have editing permissions on the specific Bitable.

---

## 3. Messaging API — Real-Time Message Access

### Event-Driven (Recommended)

Subscribe to `im.message.receive_v1` event. When any message is posted in a group chat where the bot is a member, the bot receives a callback with:
- Message content (text, rich text, image, file, etc.)
- Sender info (user ID, name)
- Chat ID (group identifier)
- Message ID
- Timestamp
- Mention info (if the bot was @mentioned)

### Two Delivery Modes

| Mode | Webhook | WebSocket (Long Connection) |
|---|---|---|
| Setup complexity | Need public URL, SSL, challenge verification | Just integrate SDK, no public URL needed |
| Security | Must handle encryption + signature verification | Auth only at connection time, plaintext after |
| Development speed | ~1 week typical | ~5 minutes with SDK |
| Infrastructure | Requires public server | Works from local/private network |
| Scaling | Multiple servers can receive | Only ONE client receives per app (no broadcast) |

### Critical Constraint: 3-Second Response Rule

When the bot receives a message event, it **must return HTTP 200 within 3 seconds**. If it doesn't, Lark considers delivery failed and retries at: 5 seconds, 5 minutes, 1 hour, and 6 hours. After that, it gives up.

**Implication for AI agents:** You cannot do LLM inference in the event handler. You must immediately acknowledge receipt, then process asynchronously and send a reply via the Send Message API.

### Message History API

`GET /open-apis/im/v1/messages?container_id_type=chat&container_id={chat_id}`

Returns paginated message history. The bot must be a member of the chat.

---

## 4. Event Subscription System

### Available Event Categories

| Category | Example Events |
|---|---|
| Messages | `im.message.receive_v1`, `im.message.message_read_v1`, `im.message.recalled_v1` |
| Reactions | `im.message.reaction.created_v1`, `im.message.reaction.deleted_v1` |
| Chat membership | `im.chat.member.bot.added_v1`, `im.chat.member.bot.deleted_v1`, `im.chat.member.user.added_v1`, `im.chat.member.user.deleted_v1` |
| Chat lifecycle | `im.chat.created_v1`, `im.chat.disbanded_v1`, `im.chat.updated_v1` |
| Docs | Document change events (content updated, permission changed) |
| Calendar | Event created, updated, deleted |
| Approval | Approval status changes |

### Event Schema Versions

- **v1.0** (legacy) — Older format, still works for existing bots
- **v2.0** (current) — Applied automatically to new bots. Recommended.

### Configuration Steps

1. Go to Lark Developer Console > your app
2. Navigate to "Events & Callbacks"
3. Choose delivery mode (Webhook or Long Connection)
4. For webhook: configure Encrypt Key and Request URL
5. Add specific events (e.g., `im.message.receive_v1`)
6. Publish app version

### Subscribing to Specific Groups

The bot receives events from **all groups it's a member of**. There's no API to selectively subscribe to specific groups — you filter on your end by checking the `chat_id` in incoming events. To stop receiving events from a group, remove the bot from that group.

---

## 5. Authentication

### Three Token Types

| Token | Purpose | How to Get | Expiry |
|---|---|---|---|
| `app_access_token` | App-level identity | `POST /open-apis/auth/v3/app_access_token/internal` with `app_id` + `app_secret` | 2 hours |
| `tenant_access_token` | App acting on behalf of a tenant/org | `POST /open-apis/auth/v3/tenant_access_token/internal` with `app_id` + `app_secret` | 2 hours |
| `user_access_token` | Acting as a specific user | OAuth 2.0 flow (user grants consent) | 2 hours (refreshable) |

### Which Token for What

- **Bot sending messages, reading chats, accessing Bitable**: `tenant_access_token` (most common for bot use cases)
- **Accessing a user's personal docs/calendar**: `user_access_token`
- **App-level operations**: `app_access_token`

Official SDKs (Node.js, Go, Java, Python) handle token refresh automatically.

### Required Permission Scopes

For a bot that monitors group chats and accesses documents:

| Scope | Purpose |
|---|---|
| `im:message` | Read messages |
| `im:message:send_as_bot` | Send messages as bot |
| `im:chat` | Access chat info |
| `im:chat:readonly` | Read chat metadata |
| `im:message:readonly` | Read message content/history |
| `im:resource` | Access message attachments |
| `docx:document:readonly` | Read doc content |
| `sheets:spreadsheet` | Read/write sheets |
| `bitable:app` | Access Bitable |
| `wiki:wiki:readonly` | Read wiki content |

Permissions are configured in the Lark Developer Console under "Permissions & Scopes." Some scopes require admin approval.

### App Publishing

The app must be published and approved (by tenant admin) before it can receive messages in production. During development, you can use the app in test mode with a limited set of users.

---

## 6. Rate Limits & Restrictions

### General Rate Limiting Model

- Rate limits are applied **per-API, per-app, per-tenant**
- Write APIs have lower limits than read APIs
- Limits vary by enterprise subscription tier (Starter, Pro, Enterprise)
- Specific limit values can change; Lark notifies via changelog

### Known Specific Limits

| Scope | Limit |
|---|---|
| Webhook/Notification Bot | 5 QPS, 100 QPM per tenant per bot |
| Document basic info APIs | 5 QPS per app |
| Document modification APIs | 5 QPS + 10,000 calls/day |
| Starter plan apps | 10,000 API calls/month total |
| General messaging APIs | Varies by tier (check official docs for current numbers) |

### Rate Limit Response

When exceeded:
- HTTP status: **429**
- Error code: `99991400`
- Message: `"request trigger frequency limit"`
- Headers: `x-ogw-ratelimit-limit` (window), `x-ogw-ratelimit-reset` (seconds to wait)

### Other Notable Restrictions

1. **3-second event response rule**: Must acknowledge events within 3 seconds or face retries.
2. **WebSocket single-client**: In long connection mode, only ONE client per app receives events (no fan-out/broadcast).
3. **Bot must be in the group**: To read messages or receive events from a group chat, the bot must be added as a member.
4. **External groups**: Bots can be added to external groups (with users from other orgs), but this requires specific configuration.
5. **Message types**: Bots can send text, rich text, images, files, and interactive cards. Some message types (e.g., audio, video) may have restrictions.
6. **API call billing**: Lark has introduced API call limit adjustments for custom apps — higher-tier plans get more API quota.

---

## 7. SDKs & Developer Tools

### Official SDKs

| Language | Package | Coverage |
|---|---|---|
| Node.js | `@larksuiteoapi/node-sdk` | Full API + Events |
| Go | `github.com/larksuite/oapi-sdk-go` | Full API + Events |
| Java | `larksuite/oapi-sdk-java` | Full API + Events |
| Python | `larksuite/oapi-sdk-python` | Full API + Events |

### Notable Community SDKs

| Language | Package | Notes |
|---|---|---|
| Go | `github.com/chyroc/lark` | Supports ALL Open API and Event Callbacks |
| Go | `github.com/go-lark/lark` | IM-focused, battle-tested by ~650 ByteDance devs |

### MCP Integration (2025+)

Lark has released an official MCP server (`larksuite/lark-openapi-mcp`) that wraps all Open Platform APIs as MCP tools. This allows AI agents (via Claude, Cursor, etc.) to directly call Lark APIs for document processing, conversation management, and calendar scheduling.

### API Explorer

Lark provides an interactive API Explorer at `open.larksuite.com` where you can test any API call directly in the browser with your credentials.

---

## Key Architectural Considerations for Building a Lark AI Agent

1. **Use WebSocket (Long Connection) mode** for event subscription during development — no public URL needed, 5-minute setup.
2. **Always acknowledge events immediately** (return 200), then process asynchronously. The 3-second timeout is strict.
3. **Use `tenant_access_token`** for most bot operations. The SDKs handle token refresh automatically.
4. **Filter events by `chat_id`** on your side if you only care about specific groups.
5. **Consider the single-client WebSocket limitation** — if you need HA, use webhook mode with multiple servers behind a load balancer.
6. **Bitable is excellent for structured data** — use it as a lightweight database for your agent's state, logs, or knowledge base.
7. **The official Node.js or Python SDK** is the fastest path to a working bot.

---

## Sources

- [Lark Open Platform — Developer Portal](https://open.larksuite.com/)
- [Lark Event Subscription Overview](https://open.larksuite.com/document/server-docs/event-subscription/overview-of-event-subscription)
- [Lark Bot Overview](https://open.larksuite.com/document/client-docs/bot-v3/bot-overview)
- [Lark Send Message API](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/reference/im-v1/message/create)
- [Lark Get Chat History API](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/reference/im-v1/message/list)
- [Lark Receive Message Event](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/reference/im-v1/message/events/receive)
- [Lark Bitable Overview](https://open.larksuite.com/document/ukTMukTMukTM/uUDN04SN0QjL1QDN/bitable-overview)
- [Lark Sheets API](https://open.larksuite.com/document/ukTMukTMukTM/uUDN04SN0QjL1QDN/sheets-v3/spreadsheet-sheet/query)
- [Lark Authentication — Access Credentials](https://open.larksuite.com/document/home/introduction-to-scope-and-authorization/access-credentials)
- [Lark Rate Limits](https://open.larksuite.com/document/ukTMukTMukTM/uUzN04SN3QjL1cDN)
- [Lark Custom App API Call Limits](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/api-call-guide/api-billing)
- [go-lark SDK (GitHub)](https://github.com/go-lark/lark)
- [Official Node.js SDK (npm)](https://www.npmjs.com/package/@larksuiteoapi/node-sdk)
- [Lark OpenAPI MCP Server (GitHub)](https://github.com/larksuite/lark-openapi-mcp)
- [Feishu Rate Limits (Chinese)](https://open.feishu.cn/document/server-docs/api-call-guide/frequency-control)
