# MCP Servers for SaaS Integrations -- Landscape Report (March 2026)

## Recommendation (TL;DR)

**For rapid prototyping or broad coverage:** Use **Composio** or **Pipedream** as an MCP gateway -- they give you 300-3000+ app integrations behind a single endpoint with managed OAuth, at the cost of vendor dependency and per-call pricing.

**For production with a few key integrations:** Use individual open-source MCP servers for each service. The ecosystem is now mature enough that Gmail, Google Calendar, Slack, and Outlook all have well-maintained community servers with 1k+ stars. You own the auth, the data flow, and the uptime.

**For OAuth/auth management specifically:** Use **Nango** as a backend auth layer behind your own MCP servers -- it handles token lifecycle without becoming your entire integration platform.

---

## 1. Official MCP Ecosystem (github.com/modelcontextprotocol)

The official [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) repo (80.4k stars) contains **reference implementations only** -- Filesystem, Git, Fetch, Memory, Sequential Thinking, Time. No SaaS integrations (no Gmail, Calendar, Slack, etc.) are maintained by the MCP team.

There is now an [official MCP Registry](https://registry.modelcontextprotocol.io/) for discovering community servers. The TypeScript SDK (12k+ stars) and Python SDK (22k+ stars) are both mature.

**Key takeaway:** Anthropic provides the protocol and SDKs, but relies entirely on the community and platforms for SaaS integrations.

---

## 2. Email MCP Servers

### Gmail

| Server | Stars | Language | Auth | Key Differentiator |
|--------|-------|----------|------|--------------------|
| [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) | ~1.4-1.7k | Python | OAuth 2.1 | Full Workspace: Gmail + Calendar + Docs + Sheets + Drive + more. Most feature-complete. |
| [MarkusPfundstein/mcp-gsuite](https://github.com/MarkusPfundstein/mcp-gsuite) | Active | TypeScript | OAuth 2.0 | Gmail + Calendar combo. Smithery install support. |
| [GongRzhe/Gmail-MCP-Server](https://github.com/GongRzhe/Gmail-MCP-Server) | Active | Node.js | Auto-auth | Gmail-only, easy setup via npx. |
| [david-strejc/gmail-mcp-server](https://github.com/david-strejc/gmail-mcp-server) | Moderate | Python | IMAP/SMTP | Uses IMAP instead of Google API -- simpler auth, works with non-Google email too. |

**Best pick for Gmail:** `taylorwilsdon/google_workspace_mcp` if you want the full Google suite, or `mcp-gsuite` for a lighter Gmail+Calendar combo.

### Outlook / Microsoft 365

| Server | Language | Auth | Key Differentiator |
|--------|----------|------|--------------------|
| [Softeria/ms-365-mcp-server](https://github.com/Softeria/ms-365-mcp-server) | TypeScript | MS Graph API | Most comprehensive: Mail + Calendar + OneDrive + Excel + OneNote. Org-mode for Teams/SharePoint. Read-only mode available. |
| [XenoXilus/outlook-mcp](https://github.com/XenoXilus/outlook-mcp) | Node.js | PKCE (no client secret) | Email + Calendar + SharePoint + Office doc processing. Easy DXT install for Claude Desktop. |
| [ryaker/outlook-mcp](https://github.com/ryaker/outlook-mcp) | Node.js | Azure App Reg | Email + Calendar + OneDrive + Power Automate integration. |
| [marlonluo2018/outlook-mcp-server](https://github.com/marlonluo2018/outlook-mcp-server) | Python | win32COM (local) | Fully local/offline, no cloud auth. Windows only. |

Microsoft also has an [official Outlook Mail MCP Server](https://marketplace.microsoft.com/en-us/product/saas/wa200009543) in its Frontier program (early/experimental).

**Best pick for Outlook:** `Softeria/ms-365-mcp-server` for breadth, or `XenoXilus/outlook-mcp` for easier auth (PKCE, no client secret).

---

## 3. Calendar MCP Servers

### Google Calendar

| Server | Stars | Language | Key Differentiator |
|--------|-------|----------|--------------------|
| [nspady/google-calendar-mcp](https://github.com/nspady/google-calendar-mcp) | Popular | TypeScript | Multi-account, multi-calendar, cross-account conflict detection, recurring events, free/busy, tool filtering for read-only. |
| [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) | ~1.4-1.7k | Python | Calendar as part of full Workspace suite. |
| [deciduus/calendar-mcp](https://github.com/deciduus/calendar-mcp) | Moderate | Python | Mutual free-slot finding, daily analytics. AGPLv3. |
| [guinacio/mcp-google-calendar](https://github.com/guinacio/mcp-google-calendar) | Moderate | Python | Auto timezone detection, conflict checking on create/update. |

**Best pick:** `nspady/google-calendar-mcp` for dedicated calendar use, or `taylorwilsdon/google_workspace_mcp` if you already use it for Gmail.

### Outlook Calendar

Covered by the MS 365 servers above (Softeria, XenoXilus, ryaker) -- all include calendar functionality via Microsoft Graph API.

---

## 4. Messaging MCP Servers

### Slack

| Server | Stars | Language | Key Differentiator |
|--------|-------|----------|--------------------|
| [korotovsky/slack-mcp-server](https://github.com/korotovsky/slack-mcp-server) | ~1.4k | Go | **Community leader.** 30k monthly visitors, 9k+ users. Stealth mode (no permissions needed) or OAuth. DMs, group DMs, smart history. Stdio/SSE/HTTP. Available via Homebrew. MIT license. |
| [Slack Official MCP Server](https://docs.slack.dev/ai/slack-mcp-server/) | N/A | Official | First-party from Slack. Search channels, send messages, manage canvases. |
| [ubie-oss/slack-mcp-server](https://github.com/ubie-oss/slack-mcp-server) | Active | TypeScript | Stdio + Streamable HTTP. Powerful date/content search filters. |

Note: Anthropic's original reference Slack MCP server was **deprecated** due to security vulnerabilities.

**Best pick:** `korotovsky/slack-mcp-server` for the most battle-tested open-source option, or the [official Slack MCP server](https://docs.slack.dev/ai/slack-mcp-server/) if you prefer first-party support.

### Discord

| Server | Language | Key Differentiator |
|--------|----------|--------------------|
| [barryyip0625/mcp-discord](https://github.com/barryyip0625/mcp-discord) | TypeScript | Full CRUD on channels, forums, messages, webhooks. npx/Docker install. |

Discord MCP is less mature than Slack's ecosystem -- fewer options, smaller communities.

---

## 5. Authentication / OAuth for MCP

The MCP spec itself now standardizes OAuth 2.1 for remote server auth (added March 2025). Key features:
- **PKCE mandatory** for all clients
- **Metadata discovery** for automatic endpoint configuration
- **Dynamic Client Registration** so clients can self-register with new servers
- **Third-party IdP delegation** supported natively

For implementing auth on your own MCP servers, the main approaches:
1. **Cloudflare Workers + OAuth** -- Cloudflare has built-in MCP auth support in their Agents SDK
2. **Auth0** -- Published guides for MCP + OAuth integration
3. **WorkOS** -- Tutorial for adding OAuth to MCP servers
4. **Stytch** -- End-to-end examples with Cloudflare Workers
5. **Nango** -- Handles OAuth token lifecycle as a backend service (see below)

---

## 6. Aggregator Platforms

### Composio

| Aspect | Details |
|--------|---------|
| **What it is** | MCP Gateway + unified integration platform purpose-built for AI agents |
| **Coverage** | 300+ apps via MCP, 850+ total connectors |
| **Auth** | Managed OAuth 2.0 flows, API key management |
| **Security** | SOC2 + ISO certified. Action-level RBAC, audit trails, zero data retention, sandboxed execution. |
| **Pricing** | Free tier available. $29/mo ("Ridiculously Cheap"), $229/mo ("Serious Business"), Enterprise custom. |
| **Strengths** | Single endpoint for everything. Managed auth. Built-in observability. Security certifications. |
| **Weaknesses** | Vendor lock-in. Per-call pricing at scale. You don't own the integration logic. Latency of an extra hop. |
| **Links** | [composio.dev](https://composio.dev), [mcp.composio.dev](https://mcp.composio.dev/) |

### Pipedream

| Aspect | Details |
|--------|---------|
| **What it is** | Workflow automation platform with dedicated MCP server for each of 3,000+ APIs |
| **Coverage** | 3,000+ APIs, 10,000+ prebuilt tools |
| **Auth** | Managed OAuth, encrypted credential storage |
| **Pricing** | Free for personal use. Paid tiers for teams/production. |
| **Strengths** | Massive API coverage. Free personal tier. Self-hostable. Open-source MCP chat examples. |
| **Weaknesses** | Originally a workflow platform, not agent-native. Less opinionated about MCP patterns. |
| **Links** | [pipedream.com/docs/connect/mcp](https://pipedream.com/docs/connect/mcp), [mcp.pipedream.com](https://mcp.pipedream.com/) |

### Nango

| Aspect | Details |
|--------|---------|
| **What it is** | Open-source integration platform focused on auth + data sync, with built-in MCP server |
| **Coverage** | 700+ APIs |
| **Auth** | Core competency -- manages full OAuth lifecycle, token storage, refresh |
| **Security** | SOC 2 Type II, HIPAA, GDPR compliant |
| **Pricing** | Open-source self-host option. Managed cloud with free tier + paid plans. |
| **Strengths** | Open-source. Best-in-class auth management. Used by Replit, Ramp in production. Code-first (TypeScript integration scripts). MCP endpoint at `api.nango.dev/mcp`. |
| **Weaknesses** | More of an auth/sync layer than a full MCP gateway. You still write integration logic. |
| **Links** | [nango.dev](https://nango.dev), [GitHub](https://github.com/NangoHQ/nango) |

### Klavis AI

Cross-platform MCP client for Slack/Discord + hosted MCP servers. More of a client-side play than a server platform. Relevant if you want MCP access from within Slack/Discord itself.

---

## 7. Trade-offs: Individual Servers vs. Platform

| Dimension | Individual Open-Source Servers | Aggregator Platform (Composio/Pipedream) |
|-----------|-------------------------------|------------------------------------------|
| **Setup time** | Hours per service (OAuth config, server deployment) | Minutes (single API key, one endpoint) |
| **Auth management** | You handle OAuth flows, token refresh, storage | Platform handles it all |
| **Cost at scale** | Free (self-hosted) + infra costs | Per-call pricing adds up fast |
| **Customization** | Full control -- fork and modify | Limited to what platform exposes |
| **Latency** | Direct API calls | Extra network hop through gateway |
| **Vendor risk** | Depends on community maintainers | Depends on platform's business viability |
| **Security control** | Full -- tokens stay on your infra | Tokens stored on third-party platform |
| **Maintenance** | You update each server independently | Platform handles updates |
| **Debugging** | Full visibility into server code | Black box (though Composio has observability tools) |

### When to use individual servers:
- You only need 2-4 integrations
- Security/compliance requires tokens on your own infra
- You need deep customization of tool behavior
- Cost sensitivity at high volume

### When to use a platform:
- You need 5+ integrations quickly
- Auth management is your biggest pain point
- You want managed updates and monitoring
- You're prototyping and speed matters more than control

### Hybrid approach (recommended for production):
- Use **Nango** for OAuth management (open-source, self-hostable)
- Use **individual MCP servers** for the 3-4 services you actually need
- Point the servers at Nango for token management
- This gives you: own your integration logic + managed auth + no per-call platform fees

---

## 8. Maturity Assessment

| Service | Ecosystem Maturity | Best Available Option | Confidence |
|---------|-------------------|----------------------|------------|
| Gmail | High -- multiple mature servers, 1k+ stars options | taylorwilsdon/google_workspace_mcp | High |
| Google Calendar | High -- several well-maintained options | nspady/google-calendar-mcp | High |
| Outlook/M365 Email | Medium -- newer but functional servers exist | Softeria/ms-365-mcp-server | Medium |
| Outlook Calendar | Medium -- covered by M365 servers | Softeria/ms-365-mcp-server | Medium |
| Slack | High -- strong community server + official server | korotovsky/slack-mcp-server | High |
| Discord | Low-Medium -- fewer options, smaller community | barryyip0625/mcp-discord | Medium |
| General Auth | High -- MCP spec has OAuth 2.1 baked in | Nango or platform-provided auth | High |

---

## Sources

- [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) -- Official MCP reference servers (80.4k stars)
- [Official MCP Registry](https://registry.modelcontextprotocol.io/)
- [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) -- Full Google Workspace MCP
- [MarkusPfundstein/mcp-gsuite](https://github.com/MarkusPfundstein/mcp-gsuite) -- Gmail + Calendar MCP
- [nspady/google-calendar-mcp](https://github.com/nspady/google-calendar-mcp) -- Google Calendar MCP
- [GongRzhe/Gmail-MCP-Server](https://github.com/GongRzhe/Gmail-MCP-Server) -- Gmail MCP
- [Softeria/ms-365-mcp-server](https://github.com/Softeria/ms-365-mcp-server) -- Microsoft 365 MCP
- [XenoXilus/outlook-mcp](https://github.com/XenoXilus/outlook-mcp) -- Outlook MCP (PKCE)
- [korotovsky/slack-mcp-server](https://github.com/korotovsky/slack-mcp-server) -- Slack MCP (~1.4k stars)
- [Slack Official MCP](https://docs.slack.dev/ai/slack-mcp-server/) -- First-party Slack MCP
- [barryyip0625/mcp-discord](https://github.com/barryyip0625/mcp-discord) -- Discord MCP
- [Composio MCP Gateway](https://composio.dev/mcp-gateway) -- Unified MCP platform
- [Pipedream MCP](https://pipedream.com/docs/connect/mcp) -- 3000+ API MCP servers
- [Nango](https://nango.dev/) -- Open-source integration platform with MCP
- [MCP OAuth 2.1 Spec](https://modelcontextprotocol.io/specification/draft/basic/authorization)
- [Auth0 MCP Guide](https://auth0.com/blog/an-introduction-to-mcp-and-authorization/)
- [WorkOS MCP OAuth Guide](https://workos.com/blog/how-to-add-authentication-to-your-mcp-server)
