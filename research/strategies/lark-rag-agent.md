# Lark Knowledge Agent: Architecture & Implementation Strategy

**Date:** 2026-05-02
**Status:** Strategy / Recommendation
**Question:** Best way to build an AI chatbot that answers questions over a company's Lark docs, chats, and other surfaces, returning links to the source material. Assume large corpora.

---

## TL;DR Recommendation

**Build a custom RAG agent on top of the official Lark Open Platform APIs. Don't wait for a turnkey vendor connector — none of the major enterprise search vendors (Glean, Copilot, Gemini for Workspace) ships a first-class Lark connector today, and Lark's own AI features don't yet expose a generic Q&A agent over arbitrary tenant data with citations.**

Recommended stack:

| Layer | Choice | Why |
|---|---|---|
| **Ingestion** | Official `larksuite/lark-openapi-mcp` + `larksuite/cli` for bulk pulls; webhook event subscription for incremental updates | Official, covers Wiki, Docs, IM, Drive, Bitable; OAuth + tenant tokens; pagination via `--page-all`. |
| **Storage** | Object storage (raw exports) + Postgres (metadata, ACLs) + a hybrid vector store | Separate the source-of-truth from the index so you can re-embed without re-fetching. |
| **Vector store** | **Turbopuffer** or **Qdrant** at scale (>50M chunks); **pgvector** if <10–20M and you already run Postgres | Both natively support hybrid (BM25 + vector) search and metadata filters needed for ACL enforcement. |
| **Retrieval** | Hybrid (BM25 + dense embedding) + Reciprocal Rank Fusion + cross-encoder reranker | Production default in 2026; reranking gives 10–30% top-10 quality lift. |
| **Embeddings** | Voyage-3 or OpenAI `text-embedding-3-large` for docs; same model for chat with thread-aware windowing | Strong on multilingual + code-heavy enterprise content. |
| **Orchestration** | **LlamaIndex** for ingestion/retrieval + **Claude Agent SDK** (or LangGraph) for the chat loop | LlamaIndex has the broadest doc-loader + retriever primitives; Claude Agent SDK is the lightest path to a reliable tool-using assistant. |
| **Surface** | Lark bot (event-subscription-based) so users chat with it in Lark itself; web fallback for richer citations | Bot lives where users already work; Lark deep-link URLs are stable, so citations resolve. |
| **ACL** | Per-user OAuth tokens for query-time permission filtering, OR ingest-time ACL hydration (capture viewer lists per doc/chat into chunk metadata) | Without this, the bot will leak content. Non-negotiable for a company deployment. |

**Realistic timeline:** ~3 weeks for an MVP indexing one wiki space + a handful of group chats; ~2–3 months to harden ACLs, scale ingestion, and ship a polished Lark-native bot.

**Key risk:** Lark's chat-history API requires the bot to be a member of every chat you want to index. Plan an "invite the bot" rollout (or use admin-tenant credentials if your tenant grants them) before promising coverage.

---

## Why Not Turnkey?

I checked the obvious "buy it" paths first. Summary:

- **Glean** — 100+ connectors, but **no first-class Lark/Feishu connector** as of May 2026. You'd build a [Custom Data Source](https://docs.glean.com/connectors/custom/about), which is roughly the same engineering work as building it yourself (Glean's ingestion API + permission model). Worth it if you also want Slack/Drive/Confluence in one bot; not worth it if Lark is the only surface.
- **Microsoft Copilot / Gemini for Workspace** — bound to their own ecosystems; no Lark connector.
- **Lark Aily / native AI features** — Lark's marketing pages tout AI knowledge sharing and "smart search," but there is no public, generic "build a Q&A agent over my whole tenant with citations" product. Lark's own MCP server gives the building blocks, not the finished product.
- **Open-source bots (`Mgrsc/lark_bot`, `ConnectAI-E/Lark-OpenAI`, etc.)** — these are thin proxies from Lark chat to OpenAI; they don't index your tenant. Useful as reference for the bot wiring, not as a solution.

**Conclusion:** This is a build, not a buy. The good news is the official Lark tooling is unusually mature for this kind of project.

---

## What Lark Actually Exposes

Lark (and its China-region twin Feishu) ship a comprehensive Open Platform. Relevant pieces:

### APIs you'll use

| Domain | Endpoints | Notes |
|---|---|---|
| **Wiki** | `wiki/v2/spaces` (list spaces), `wiki/v2/spaces/{id}/nodes` (tree), node→doc resolution | Best entry point for "company knowledge base." Hierarchical. |
| **Docs (Docx)** | `docx/v1/documents/{token}/raw_content`, `.../blocks` | Block-structured; pull `blocks` for fidelity, `raw_content` for fast text. |
| **IM / Messenger** | `im/v1/chats` (list), `im/v1/messages?container_id=...` (history), `im/v1/messages/{id}` | Bot must be a chat member. Supports threads and rich content. |
| **Drive** | `drive/v1/files` (list), file download, `drive/v1/permissions` | Covers files outside Wiki. |
| **Bitable** | `bitable/v1/apps/.../tables/.../records/search` | Internal CRMs / OKRs often live here — worth indexing. |
| **Calendar / Mail** | `calendar/v4/calendars`, `mail/...` | Optional; usually low signal-to-noise for a Q&A bot. |
| **Events (webhook or socket)** | `im.message.receive_v1`, `docs.update`, etc. | Drives incremental indexing. Socket mode is recommended over webhook for most teams (no public IP needed). |

### Auth model

- **App Credentials → `tenant_access_token`** = bot acts as itself, sees only what it's a member of. Easiest for chat ingestion.
- **OAuth → `user_access_token`** = bot acts as a specific user, sees what they see. **This is what you want for ACL-correct retrieval at query time.** Tokens last ~2 hours, refreshable. Scopes are granular (`docx:document:readonly`, `im:message:readonly`, `wiki:wiki:readonly`, `drive:drive:readonly`, etc.).

### Official tooling that saves you weeks

- **`larksuite/lark-openapi-mcp`** — official MCP server wrapping the Open Platform. Tool naming `biz.version.resource.method` (e.g., `im.v1.message.list`). Caveat from the README: file upload/download not yet supported, and only document **import + read** are exposed — no edit. Good news for our use case (we only need read).
- **`larksuite/cli`** — 200+ commands across Messenger, Docs, Base, Wiki, Drive, etc. Has `--page-all` for exhaustive pagination and `--format json` for piping into an indexer. Perfect for the initial backfill.
- **Node SDK (`@larksuiteoapi/node-sdk`)** — production SDK if you outgrow the CLI/MCP wrappers.

### Limits to plan for

- API rate limits return HTTP 429. The CLI/MCP wrappers don't auto-backoff aggressively; bake in token-bucket throttling and retry with jitter.
- Chat history pagination is cursor-based and per-chat; ingesting "all of company chat" linearly will take days. Parallelize per-chat with a worker pool.
- Lark's URLs are stable and predictable (`https://{tenant}.larksuite.com/wiki/{node_token}`, `.../docx/{doc_token}`, `.../im/{chat_id}` with optional `?msgId=...`), so **citation generation is trivial** once you've stored the right tokens in chunk metadata.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     INGESTION (batch + streaming)                  │
│                                                                    │
│  Lark APIs ──► Workers (Wiki, Docs, IM, Drive, Bitable)            │
│      │              │                                              │
│      │              ▼                                              │
│      │        Raw export (S3/GCS) ──► Normalizer ──► Chunker       │
│      │                                                  │          │
│      │                                                  ▼          │
│      └──► Webhook/socket subscriber ──► Delta queue ──►┤          │
│                                                         ▼          │
│                                               Embeddings + ACL     │
│                                                         │          │
│                                                         ▼          │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Hybrid index: BM25 + dense vectors + metadata (ACL, type,   │ │
│  │  source URL, timestamps) — Qdrant / Turbopuffer / pgvector   │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                       AGENT (per-user query)                       │
│                                                                    │
│  User in Lark ──► Bot (event subscription)                         │
│       │                                                            │
│       ▼                                                            │
│  Resolve user identity ──► Mint/refresh user_access_token          │
│       │                                                            │
│       ▼                                                            │
│  Claude Agent SDK loop:                                            │
│    1. Query rewrite (use chat history)                             │
│    2. Tool: hybrid_search(query, user_id, filters)                 │
│         └─► retriever filters chunks by ACL                        │
│    3. Cross-encoder rerank top-50 → top-8                          │
│    4. Claude composes answer with inline citations                 │
│    5. Map chunk → Lark URL → render as Lark interactive card       │
└────────────────────────────────────────────────────────────────────┘
```

### Ingestion details

**Backfill (one-shot per source):**
- Wiki: walk every space → every node → fetch each doc's blocks. Store `space_id`, `node_token`, `doc_token`, parent path, last-edited-at.
- IM: list chats the bot is in → page through messages per chat. Store `chat_id`, `message_id`, `thread_id`, `sender`, `timestamp`. **Roll up replies into the parent thread** during chunking — answering "what did we decide about X" needs the whole thread, not a single message.
- Drive: list files, fetch text-extractable types (docx-equivalents, PDFs, sheets summaries). Skip binaries.
- Bitable: enumerate apps and tables; index the schema + rows you care about (often only specific tables — make this a config list, not a sweep).

**Incremental:**
- Subscribe to events (`im.message.receive_v1`, `drive.file.bitable_record_changed_v1`, doc edit events). Push deltas onto a queue. A worker re-chunks/re-embeds the affected document or message thread.
- For chat, debounce by thread — don't re-index after every keystroke; wait for 30–60s of silence.

### Chunking (the part most teams get wrong)

Different surfaces need different strategies:

- **Docs / Wiki**: structural chunking by Lark block tree (heading → section), then 256–512 token windows with 10–20% overlap inside each section. Store the section header in the chunk metadata so citations name the right anchor. The 2026 benchmark default (512 tokens, 15% overlap) is a fine starting point.
- **Chat**: thread-aware windowing — group messages by `thread_id` (or by ≤30-min gap if no thread), then slide a 10–20-message window. Each chunk includes participants and time range. This dramatically beats per-message indexing for "what was decided" / "who agreed to do X" questions.
- **Bitable**: row-as-document with a header line that names the table and key fields, so dense retrieval can match on schema + values.

Use **contextual retrieval** (Anthropic's technique: prepend a 1–2 sentence LLM-generated summary of the parent doc/thread to each chunk before embedding). Roughly 35% retrieval-quality lift on long-document corpora; well worth the one-time embed cost.

### Retrieval

- **Hybrid is non-negotiable.** Chat data is full of names, project codes, and acronyms — BM25 catches what dense embeddings miss. Docs benefit from semantic. Run both and fuse with RRF (k=60 default).
- **Rerank** the top-50 with a cross-encoder (Cohere Rerank 3.5, Voyage rerank-2.5, or a small open-source model like `bge-reranker-v2-m3`). Take top-6 to top-8 into the LLM context.
- **Filter on metadata**: `source_type` (doc/chat/bitable), `time_range`, and most importantly **ACL**.

### Permission-aware retrieval (the part that stops you from getting fired)

This is the hard part. Two viable approaches:

1. **Query-time impersonation (cleanest, slower):**
   - Mint a user-scoped token for the asker.
   - When ranking chunks, call Lark to verify the user's read permission on each candidate parent doc/chat (cache aggressively — 5–15 min TTL).
   - Pros: always correct. Cons: extra latency, more API calls.

2. **Ingest-time ACL hydration (faster, eventually consistent):**
   - During ingestion, capture the viewer set for each doc/chat (via `drive/v1/permissions` and `im/v1/chats/{id}/members`).
   - Store a `viewer_ids` array (or a group-membership-expanded set) in chunk metadata.
   - At query time, filter `viewer_ids contains current_user_id`.
   - Pros: a single index lookup. Cons: stale if permissions change between ingestion runs — refresh ACLs on a faster cadence than content (e.g., every 1–2 hours).

**Recommendation:** Hybrid — ingest-time ACL hydration as the default filter, plus a short-TTL "membership cache" the agent consults for any chunk it's about to actually return as a citation. This is the model Glean and other enterprise search products converged on.

---

## Vector store sizing

Rough sizing for "a lot of docs and chats":

- 50k docs × ~10 chunks/doc = 500k doc chunks
- 5k active chats × 50k messages avg × ~1 chunk per 15 messages ≈ 16M chat chunks
- Plus Bitable rows, Drive files: budget 30–50M chunks total

At that scale:

| Store | Verdict |
|---|---|
| **pgvector / pgvectorscale** | Works up to ~50M with `pgvectorscale`; ~470 QPS at 99% recall on 50M vectors. Cheapest if you already run Postgres. Hybrid via `pg_search` extension or app-side fusion. |
| **Qdrant** | Best latency among open source (~12ms p99 at 10M), native hybrid, strong filter performance. Good default if you're going purpose-built. |
| **Turbopuffer** | Object-storage-backed → cheapest at scale (~$800/mo for 100M vs $5k–7k on Pinecone/Qdrant Cloud). Native hybrid. Some operational tradeoffs (cold-start latency on rarely-queried namespaces). |
| **Weaviate** | Native hybrid, multi-tenant model maps cleanly to "one tenant per Lark org" if you ever multi-tenant the product. |
| **Pinecone** | Easiest managed, but you pay for it. Reasonable if you want zero ops. |

**My pick:** Start on **pgvector** if you're under ~10M chunks and already have Postgres. Move to **Qdrant** or **Turbopuffer** once you cross that line. Both support the BM25 + vector + metadata-filter combo you need without app-side fusion gymnastics.

---

## Agent framework

The 2026 production pattern that keeps showing up:

> LlamaIndex for ingestion + retrieval, LangGraph (or Claude Agent SDK) for the conversational agent loop, RAGAS / LangSmith for eval.

For this project specifically:

- **LlamaIndex** owns: doc loaders for Lark exports, chunking, embedding pipeline, hybrid retriever wrapping Qdrant/pgvector. Its retrieval primitives are the most mature, and it has Lark connectors via the community packs.
- **Claude Agent SDK** owns: the chat loop, tool calls (`hybrid_search`, `fetch_full_doc`, `verify_acl`), citation formatting. It's substantially lighter than LangGraph for a single-purpose agent and is the closest fit for "answer with sources" workflows. Use LangGraph instead if you anticipate complex branching (e.g., multi-step research, plan-and-execute) — overkill for a Q&A bot.
- **Don't pick Haystack** unless you have a reason to (it's solid, but the talent pool and community examples are smaller, which slows you down).

---

## Surface: should it live in Lark itself?

Yes. Build the bot as a Lark custom app:
- Subscribe to `im.message.receive_v1` events.
- Reply with [Lark interactive cards](https://open.larksuite.com/document) so citations render as clickable chips with doc/chat titles.
- Support `@bot question` in any chat where it's invited, plus DMs.
- Side-channel web UI is optional — useful for admins reviewing answers and for richer source previews.

The user-experience reason: you already know who is asking (Lark sends you their `open_id`), so query-time ACL filtering is one lookup away.

---

## Phased plan

**Phase 1 — Spike (1 week)**
- Register a Lark app with `wiki:wiki:readonly`, `docx:document:readonly`, `im:message:readonly`, `drive:drive:readonly`.
- Use `larksuite/cli` to dump one wiki space and one chat to JSON.
- Embed with OpenAI/Voyage, load into pgvector.
- CLI Q&A loop with hybrid retrieval. **No ACL yet.** Goal: prove answers + citations work.

**Phase 2 — MVP (2–3 weeks)**
- Wrap as a Lark bot with event subscription.
- Add ACL hydration on ingest (`viewer_ids` per chunk).
- Reranker on top-50.
- Incremental updates via event subscription for the indexed sources.

**Phase 3 — Scale (1–2 months)**
- Migrate to Qdrant or Turbopuffer if needed.
- Expand to all wikis + key chats + Bitable tables.
- Eval harness (RAGAS) on a curated 100-question gold set.
- Ops: rate-limit handling, observability, ACL refresh cron.

**Phase 4 — Polish**
- Conversational memory (per-user thread state).
- Query rewrite for follow-ups.
- Admin dashboard for "what's indexed, what's stale, what failed."

---

## Things to decide before starting

1. **Coverage scope.** "All chats" is rarely actually wanted (DMs are sensitive, noisy, low signal). Recommended default: index all wikis + a curated allowlist of group chats + Drive + selected Bitable tables. Make the allowlist a config, not code.
2. **ACL stance.** Strict (impersonation at query time, never leak) vs. permissive (anything the bot can see, any user can ask about). Strict is the only defensible default for a company deployment.
3. **Region.** Lark vs. Feishu have different API hosts (`open.larksuite.com` vs. `open.feishu.cn`) and data-residency implications. Pick deliberately.
4. **Model hosting.** If chat data is sensitive, prefer Bedrock/Vertex/Azure-hosted Claude or a self-hosted embedding model over public OpenAI. Voyage and Cohere both offer enterprise/VPC deployment.

---

## Sources

Lark / Feishu platform:
- [Official Lark CLI (`larksuite/cli`)](https://github.com/larksuite/cli)
- [Official Lark OpenAPI MCP server](https://github.com/larksuite/lark-openapi-mcp)
- [Lark Open Platform documentation hub](https://open.larksuite.com/document)
- [Get Wiki spaces — Server API](https://open.larksuite.com/document/server-docs/docs/wiki-v2/space/list)
- [Get docs content — Server API](https://open.larksuite.com/document/ukTMukTMukTM/uUDN04SN0QjL1QDN/docs-v1/content/get)
- [Get chat history — Server API (Feishu)](https://open.feishu.cn/document/server-docs/im-v1/message/list)
- [Obtain access tokens — Server API](https://open.larksuite.com/document/ukTMukTMukTM/uMTNz4yM1MjLzUzM)
- [Custom bot usage guide](https://open.larksuite.com/document/client-docs/bot-v3/add-custom-bot)
- [`@larksuiteoapi/node-sdk`](https://www.npmjs.com/package/@larksuiteoapi/node-sdk)
- [Feishu/Lark knowledge base exporter (`leemysw/feishu-docx`)](https://github.com/leemysw/feishu-docx)

Architecture / RAG:
- [Hybrid Search done right (BM25 + HNSW + RRF) — Feb 2026](https://ashutoshkumars1ngh.medium.com/hybrid-search-done-right-fixing-rag-retrieval-failures-using-bm25-hnsw-reciprocal-rank-fusion-a73596652d22)
- [Optimizing RAG with Hybrid Search & Reranking — Superlinked](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking)
- [Best Chunking Strategies for RAG (2026)](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)
- [RAG Chunking Strategies & Embeddings — 2026 Benchmark](https://nandigamharikrishna.substack.com/p/rag-chunking-strategies-and-embeddings)
- [Build a RAG agent with LangChain](https://docs.langchain.com/oss/python/langchain/rag)
- [LangChain vs LlamaIndex vs Haystack 2026](https://kanerika.com/blogs/llamaindex-vs-langchain-vs-haystack/)
- [Best RAG Frameworks 2026](https://iternal.ai/blockify-rag-frameworks)

Permissions / ACL:
- [ACL Hydration: secure knowledge workflows for agentic AI](https://booboone.com/introducing-acl-hydration-secure-knowledge-workflows-for-agentic-ai/)
- [Query-Time ACL and RBAC Enforcement — Azure AI Search](https://learn.microsoft.com/en-us/azure/search/search-query-access-control-rbac-enforcement)
- [Building a Permissions System For Your RAG Application — Paragon](https://www.useparagon.com/learn/ai-knowledge-chatbot-with-permissions-chapter-2/)
- [Permission-Aware RAG (IEEE, 2026)](https://ieeexplore.ieee.org/document/11224764/)

Vector stores:
- [Vector Database Performance Compared — Vecstore](https://vecstore.app/blog/vector-database-performance-compared)
- [pgvector vs Pinecone vs Turbopuffer vs Qdrant (2026)](https://app.daily.dev/posts/pgvector-vs-pinecone-vs-turbopuffer-vs-qdrant-2026--m1dot7ras)
- [Best Vector Databases in 2026 — Firecrawl](https://www.firecrawl.dev/blog/best-vector-databases)

Buy-vs-build reference points:
- [Glean Connectors Hub](https://docs.glean.com/connectors/) (no first-class Lark connector as of May 2026)
- [Glean Custom Data Sources](https://docs.glean.com/connectors/custom/about)
