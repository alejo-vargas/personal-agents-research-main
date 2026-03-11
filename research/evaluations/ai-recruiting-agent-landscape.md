# AI Agents for Recruiting / Candidate Matching

**Date:** 2026-03-11
**Status:** Initial research complete

---

## TL;DR

The recruiting AI agent space is mature commercially but thin on open-source. For Lark/Feishu specifically, no dedicated recruiting agent exists, but the infrastructure is ready: **Mgrsc/lark_bot** supports MCP tool integration, and there is now an **official Lark OpenAPI MCP server**. **Key insight for MVP:** candidates are already organized in Lark Docs with a structured folder per candidate (Documents, Summary, Submissions). This eliminates the need for a custom database, vector store, or embedding pipeline. The agent reads directly from Lark Docs via the Lark OpenAPI MCP, reasons over content with an LLM, and responds in Lark chat. No complex infrastructure needed — just Lark bot + LLM + Lark Docs API.

---

## 1. Existing Solutions Landscape

### Commercial Platforms (Proprietary)

| Platform | Focus | Key Capability |
|----------|-------|----------------|
| **Juicebox (PeopleGPT)** | Conversational AI sourcing | Searches 600M+ profiles via natural language |
| **hireEZ** | Agentic talent acquisition | Source, match, engage, manage at scale |
| **Eightfold AI** | Talent intelligence platform | 1.6B profiles, skills graph, career trajectory prediction |
| **Lindy** | No-code AI agent builder | Build custom recruiting agents (outreach, scheduling, etc.) |
| **Paradox (Workday)** | Conversational AI for hiring | End-to-end candidate experience via chatbot/SMS |
| **Phenom** | High-volume frontline hiring | Full automation: apply, assess, schedule via chatbot |
| **Leonar** | Recruiting CRM with MCP | First recruiting CRM with native MCP server support |

**Key insight:** Most are SaaS. None are open-source. The market is ~$1.35B (2025) growing at 19% YoY. 87% of organizations now use AI-driven recruiting.

Sources:
- [TechTarget: Top AI recruiting tools 2026](https://www.techtarget.com/searchhrsoftware/tip/Top-AI-recruiting-tools-and-software-of-2022)
- [Aisera: AI Recruitment 2026 Guide](https://aisera.com/blog/ai-recruiting/)
- [Lindy: AI Recruiting Guide](https://www.lindy.ai/blog/ai-recruiting)

### Open-Source / Build-Your-Own

No dominant open-source recruiting agent exists. What does exist:

- **[crewai-job](https://github.com/drukpa1455/crewai-job)** -- CrewAI + LangChain for job application automation (applicant side, not recruiter side)
- **[Resume-Screening-RAG-Pipeline](https://github.com/Hungreeee/Resume-Screening-RAG-Pipeline)** -- RAG chatbot for resume screening using hybrid retrieval
- **[Resume-parser (spaCy)](https://github.com/masnaashraf/Resume-parser)** -- NLP resume parsing with spaCy + HuggingFace
- **[ai-recruitment GitHub topic](https://github.com/topics/ai-recruitment)** -- Collection of smaller projects
- **[AI Hiring Multi-Agent Framework (arXiv)](https://arxiv.org/abs/2504.02870)** -- Academic: 4-agent system (extractor, evaluator, summarizer, scorer) using CrewAI + ChromaDB + DeepSeek/GPT-4o

**Gap:** There is no open-source "drop-in" recruiting agent. You would assemble one from agent frameworks (CrewAI, LangGraph) + vector DB + parsing tools + CRM integration.

---

## 2. Architecture Patterns

### Evolution: Keyword -> Embeddings -> RAG -> Multi-Agent

**Level 1: Keyword matching** -- Boolean search on skills/titles. Fast, brittle, misses synonyms.

**Level 2: Embedding-based matching** -- Convert resumes and JDs to vectors (SBERT, GTE, OpenAI embeddings), compute cosine similarity. Captures semantics ("React" near "frontend"). This is what most ATS systems now use.

**Level 3: RAG pipelines** -- Store candidate profiles in a vector DB (ChromaDB, FAISS, Pinecone). Given a JD, retrieve top-K candidates, then use an LLM to rank/explain matches. The "JobMatchr" pattern: PDF parse -> chunk -> embed -> store -> retrieve -> generate.

**Level 4: Multi-agent systems** -- Specialized agents for distinct tasks. This is the current frontier.

### Multi-Agent Architecture (Current Best Practice)

From the CVPR 2025 workshop paper and the Google ADK implementation:

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│   Resume     │ --> │   Evaluator  │ --> │  Summarizer  │ --> │ Score         │
│   Extractor  │     │   Agent      │     │  Agent       │     │ Formatter     │
│   Agent      │     │  (+ RAG)     │     │              │     │ Agent         │
└─────────────┘     └──────────────┘     └──────────────┘     └───────────────┘
      │                    │
      │              ┌─────┴──────┐
      │              │  ChromaDB  │  <-- Industry knowledge, certifications,
      │              │  (RAG)     │      university rankings, hiring criteria
      │              └────────────┘
      │
 PDF/DOCX input
 -> pdfplumber / docx parser
 -> text chunking (RecursiveCharacterTextSplitter)
 -> entity extraction (spaCy NER / LLM)
```

**Tech stack commonly used:**
- **Orchestration:** CrewAI, LangGraph, Google ADK
- **LLMs:** GPT-4o, DeepSeek-V3, Gemini 2.0
- **Embeddings:** SBERT, OpenAI text-embedding-3, Google Generative AI Embeddings
- **Vector DB:** ChromaDB, FAISS, Pinecone
- **Resume parsing:** pdfplumber, spaCy NER, HuggingFace models
- **Skills extraction:** Custom spaCy NER models (e.g., [amjad-awad/skill-extractor](https://huggingface.co/amjad-awad/skill-extractor))

### How Eightfold Does It (Production-Grade Reference)

Eightfold's architecture is the gold standard for understanding what "good" looks like at scale:

1. **Token-level embeddings** for skills, titles, companies, degrees, schools -- all mapped to N-dimensional vectors
2. **Skills graph** linking 1.6M+ skills with adjacency (what skills are easy to learn given existing ones)
3. **Career trajectory prediction** using RNNs: sequence of past titles/skills/tenures -> predicted next role vector
4. **Company similarity embeddings** to assess industry/culture fit
5. **Composite match score** (0-5) from ensemble of dimension-specific models
6. **Bias mitigation** by stripping gender, age, institution identifiers during screening

Source: [Eightfold Engineering Blog: AI-powered talent matching](https://eightfold.ai/engineering-blog/ai-powered-talent-matching-the-tech-behind-smarter-and-fairer-hiring/)

### How to Parse Unstructured Job Descriptions

Three approaches, in order of sophistication:

1. **Rule-based + NER:** spaCy with custom NER model trained on JD corpus. Extract: required skills, nice-to-haves, experience years, education, location. Fast but brittle.
2. **LLM structured extraction:** Send JD to GPT-4o/Claude with a schema prompt. Returns structured JSON with skills, requirements, responsibilities. Most flexible, highest accuracy.
3. **Hybrid:** NER for known entities (skills from a taxonomy), LLM for everything else (culture signals, implicit requirements).

### How to Search Candidate Databases

1. **Vector similarity search:** Embed candidate profiles, embed JD, retrieve top-K by cosine similarity from vector DB
2. **Hybrid search:** Combine vector similarity with keyword/filter search (location, years of experience, visa status)
3. **Re-ranking with LLM:** Take top-50 from vector search, have LLM re-rank with full context and explanation
4. **Agentic search:** Agent formulates multiple search queries, reviews results, refines -- mimicking how a human recruiter iterates

Sources:
- [arXiv: Multi-Agent Framework for Resume Screening](https://arxiv.org/abs/2504.02870)
- [Medium: Building an AI Agent to Parse Resumes](https://medium.com/@bravekjh/building-an-ai-agent-to-parse-resumes-and-job-descriptions-and-recommend-the-best-candidates-b5f4243b5254)
- [Resume2Vec paper](https://www.mdpi.com/2079-9292/14/4/794)

---

## 3. Lark/Feishu-Specific Implementations

### No Dedicated Recruiting Bot Exists

There is no open-source or well-known recruiting bot built specifically for Lark/Feishu. However, the building blocks are mature:

### Feishu Hire (Official)

Feishu has a built-in recruitment module called **Feishu Hire** -- it is a native ATS/recruiting tool within the Feishu ecosystem. It handles job posting, candidate pipeline, interview scheduling, and team collaboration. It is NOT an AI agent -- it is traditional HR software integrated into Feishu.

Source: [Feishu Hire Quick Start](https://www.feishu.cn/hc/en-US/articles/381040174542-feishu-hire-quick-start-for-hrs)

### Lark Bot Frameworks (Ready for Custom Agents)

| Project | What It Is | MCP Support | Language |
|---------|-----------|-------------|----------|
| **[Mgrsc/lark_bot](https://github.com/Mgrsc/lark_bot)** | AI assistant for Lark with OpenAI integration | Yes (native) | Python |
| **[OpenClaw](https://docs.openclaw.ai/channels/feishu)** | Multi-agent gateway with Feishu channel | Yes (via agents) | TypeScript |
| **[LangBot](https://docs.langbot.app/en/deploy/platforms/lark)** | LLM bot framework with Lark connector | Unclear | Python |
| **[go-lark](https://github.com/go-lark/lark)** | Low-level Lark SDK | No | Go |
| **[lark-bot-worker](https://github.com/lawvs/lark-bot-worker)** | Lark bot on Cloudflare Workers | No | TypeScript |

### Official Lark OpenAPI MCP Server

Larksuite has released an **official MCP server** ([larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp)) that exposes Feishu/Lark Open Platform APIs as MCP tools. This lets AI agents directly call Lark APIs for:
- Document processing
- Conversation/chat management
- Calendar scheduling
- (Potentially) Feishu Hire APIs

Source: [Lark OpenAPI MCP on PulseMCP](https://www.pulsemcp.com/servers/lark-feishu)

### Feishu Bot Technical Details

- **Connection modes:** WebSocket (recommended, no public URL needed) or Webhook
- **Event subscription:** `im.message.receive_v1` event for incoming messages
- **Permissions needed:** `im:message`, `im:message:send_as_bot`, `im:chat`
- **Group behavior:** Responds to @mentions in groups, all messages in DMs
- **Domains:** `feishu.cn` (China) vs `larksuite.com` (international) -- same protocol, different endpoints

---

## 4. Agent-as-Employee Pattern

### How It Works in Practice

The "agent-as-employee" pattern treats an AI agent as a team member in a messaging platform -- it has a name, avatar, role, and participates in conversations naturally.

### Slack (Most Mature Ecosystem)

**Salesforce Agentforce** is the flagship example:
- Agents appear as teammates you can @mention in any channel or DM
- Specialized agents: HR Agent (onboarding, benefits), Sales Agent (briefings, proposals), IT Agent (tickets)
- Grounded in both Slack conversation data AND Salesforce CRM data
- Can escalate to humans when out of scope

**Slack's own Slackbot** was rebuilt as a context-aware AI agent:
- Uses messages, files, channels, and tools you have access to
- Adapts to your style and role
- Never trains on customer data; LLMs run in Slack's VPC

Source: [Slack AI Agents](https://slack.com/ai-agents), [Salesforce Agentforce](https://www.salesforce.com/slack/agentforce/)

### Architecture Patterns for Agent-as-Employee

**Pattern 1: Event-Driven Trigger**
```
Message in channel -> Platform sends event -> Agent receives context
-> Agent processes (LLM call) -> Agent responds in thread
```
Used by: LangChain Slack integration, Runbear, custom bots

**Pattern 2: Dual Monitor/Response Agent**
- Monitor Agent watches channels for triggers (keywords, @mentions, anomalies)
- Response Agent handles the actual conversation with full context
- Used for: incident response, proactive alerts

**Pattern 3: Multi-Agent Group Chat (OpenClaw)**
- Multiple agents in one channel, each with a distinct persona
- `requireMention: true` -- agents only respond when @mentioned
- `allowBots: true` -- agents can see each other's output for chaining
- `respondWithThread: true` -- keeps channels clean
- Route different agents to different groups via bindings

**Pattern 4: Multi-Channel API Gateway**
```
Slack/Teams/Lark/Web -> API Gateway -> Auth + Rate Limit + Route
-> Agent Logic (platform-agnostic) -> Format response -> Return to platform
```
State stored in platform-agnostic DB. User can switch platforms and conversation continues.

### Key Design Decisions

| Decision | Recommendation |
|----------|---------------|
| Trigger mechanism | @mention for group chats, all messages for DMs |
| Response format | Thread replies to keep channels clean |
| Context window | Include full thread history, not just latest message |
| External data | Agent calls tools/APIs (via MCP) to fetch data on demand |
| Escalation | Explicit checkpoints for high-stakes decisions (hiring = high-stakes) |
| State management | Platform-agnostic store (Redis/Postgres) with user ID, conversation history, workflow state |

Sources:
- [OpenClaw Multi-Agent Setup](https://www.heyuan110.com/posts/ai/2026-03-05-openclaw-multi-agent-setup/)
- [Running Multiple AI Agents as Slack Teammates via OpenClaw](https://gist.github.com/rafaelquintanilha/9ca5ae6173cd0682026754cfefe26d3f)
- [Proactive Slack Incident Responder](https://chatbotkit.com/examples/proactive-slack-incident-responder)
- [Multi-Channel AI Agent Deployment](https://www.mindstudio.ai/blog/multi-channel-ai-agent-deployment-slack-teams)

---

## 5. MCP + Agent for CRM/Recruiting

### MCP Adoption in Recruiting/CRM (Rapid)

MCP (Model Context Protocol) has become the de facto standard for connecting AI agents to external tools since Anthropic released it in November 2024.

### Recruiting-Specific MCP

**Leonar** is the standout -- first recruiting CRM with native MCP:
- Operations exposed: candidate CRUD, job/pipeline management, conversations, outreach sequences, sourcing (870M+ profiles), analytics
- Setup: generate API key, add MCP server config, start using in Claude/ChatGPT
- Scoped API keys with granular permissions

Source: [Leonar: Connect AI to Your Recruiting Stack](https://www.leonar.app/blog/connect-ai-agents-recruiting-stack-mcp-api/)

### CRM MCP Servers

| Platform | MCP Status | What's Exposed |
|----------|-----------|----------------|
| **Leonar** | Native, production | Full recruiting CRM (candidates, jobs, outreach, analytics) |
| **HubSpot** | Public beta | Contacts, companies, deals, tickets, notes, engagements |
| **Salesforce** | Pilot (Agentforce 3.0) | CRM data via Agentforce + MCP client |
| **Dynamics 365** | GA (2025 Wave 2) | Customer service, case management |
| **Lark/Feishu** | Official MCP server | Docs, chats, calendar (potentially Hire APIs) |

### Workflow Automation MCP

- **Zapier MCP:** Connect AI to 8,000+ apps via MCP -- no backend needed
- **Make MCP:** Cloud MCP server gateway connecting agents to automation scenarios

### mcp-agent Framework

[LastMile AI's mcp-agent](https://github.com/lastmile-ai/mcp-agent) provides composable patterns for building agents on MCP:
- Map-reduce, orchestrator, evaluator-optimizer, router patterns
- Built on Temporal for pause/resume/recovery
- Philosophy: "MCP is all you need to build agents"

Sources:
- [HubSpot MCP Server Guide](https://www.digitalapplied.com/blog/hubspot-mcp-server-ai-agent-integration-guide)
- [Salesforce MCP Explained](https://www.salesforceben.com/salesforce-model-context-protocol-explained-how-mcp-bridges-ai-and-your-crm/)
- [Zapier MCP](https://zapier.com/mcp)

---

## 6. Existing Candidate Data Structure (Lark Docs)

**Critical context:** Candidates are already organized in Lark with a well-defined folder structure. Every candidate who has been worked on gets their own folder containing:

```
[Candidate Name]/
  Documents/       -- CV, deal sheet, transcripts, etc. (Lark Docs or uploads)
  Summary          -- Single doc: latest status with firms, law firm bio link,
                      LinkedIn link, general notes
  Submissions      -- Single doc: all write-ups to law firms / other roles
                      introduced to, history of all submissions
```

**This changes the architecture fundamentally.** There is no need to:
- Build a candidate database from scratch
- Set up a vector store / ChromaDB / FAISS
- Parse and ingest resumes into a custom system
- Build complex embedding pipelines

The agent reads directly from Lark Docs via the Lark OpenAPI MCP server. The data is already curated, structured, and maintained by the team.

---

## 7. MVP Architecture: Lark-Native Recruiting Agent

### Design Principle: Read Lark, Reason with LLM, Respond in Lark

The MVP leverages the existing Lark Docs structure as the candidate "database." The agent reads candidate folders via the Lark OpenAPI, uses an LLM to reason over the content, and responds in Lark chat.

```
Lark Group Chat
  User: @RecruitBot "We have a new M&A partner role at Kirkland.
         Need laterals with 8+ years, top-10 law school, BigLaw M&A"
         |
         | (event: im.message.receive_v1)
         v
Agent Gateway (Mgrsc/lark_bot or custom Feishu bot)
  - WebSocket connection to Lark
  - @mention detection in group chats
  - Thread-based responses
         |
         v
Agent Core (single agent for MVP)
  |
  |-- Step 1: Parse Request
  |     Extract role requirements from chat message
  |     -> practice area, seniority, firm tier, education, location
  |
  |-- Step 2: Browse Candidate Folders (via Lark Docs API)
  |     List all candidate folders
  |     Read each candidate's Summary doc
  |     Quick-scan for basic fit (practice area, seniority, status)
  |
  |-- Step 3: Deep Evaluate Shortlist
  |     For candidates passing initial screen:
  |     - Read full Summary (status, firms, notes)
  |     - Read Submissions (past write-ups, positioning)
  |     - Read Documents/CV if needed for detail
  |     - Score and rank against requirements
  |
  |-- Step 4: Present Results
  |     Format top candidates as Lark message/card
  |     Include: match reasoning, status, availability, gaps
  |     Flag: last submission date, active conversations
  |     Link back to candidate folder in Lark
  |
  v (via MCP)
MCP Tool Servers
  - Lark OpenAPI MCP (docs, folders, chat)
      List folders in candidate directory
      Read document content (Summary, Submissions)
      Download files (CVs, deal sheets)
      Send messages / card messages in chat
  - (Future: Bitable MCP for structured candidate index)
```

### MVP Capabilities

| Capability | How It Works |
|-----------|-------------|
| **"Find candidates for role X"** | Parse requirements -> scan Summaries -> evaluate matches -> present shortlist |
| **"What's the status on [Candidate]?"** | Read their Summary doc -> return current status with all firms |
| **"Who have we submitted to [Firm]?"** | Scan Submissions docs across candidates -> list all with that firm |
| **"Draft a write-up for [Candidate] to [Firm]"** | Read candidate's CV + Summary + past Submissions -> generate new write-up in their style |
| **"Which candidates are available?"** | Scan Summaries for status indicators -> return available/open candidates |

### What Makes This MVP Tractable

1. **No database to build.** Lark Docs IS the database. The Lark OpenAPI MCP server can already list folders and read documents.
2. **No embedding pipeline needed.** For a manageable number of active candidates (likely dozens to low hundreds), the agent can scan Summaries directly with the LLM. No vector search required at this scale.
3. **Write-ups have precedent.** The Submissions folder contains examples of how candidates have been positioned before. The agent can learn the house style from existing write-ups.
4. **Status is already tracked.** The Summary doc already has status with firms, so the agent doesn't need a separate state management system.

### Build vs. Buy Decision (MVP)

| Component | Approach | Notes |
|-----------|----------|-------|
| Lark bot gateway | Build on Mgrsc/lark_bot or custom | Use WebSocket, minimal setup |
| Request parsing | LLM (Claude/GPT-4o) | Structured extraction from chat message |
| Candidate data access | Lark OpenAPI MCP | Read folders, docs, files directly |
| Matching logic | LLM reasoning over doc content | No embeddings needed at this scale |
| Response formatting | Lark card messages | Rich format with links back to docs |
| Candidate DB / vector store | **Not needed for MVP** | Lark Docs is the source of truth |
| CRM integration | **Not needed for MVP** | Lark Docs already serves this role |
| External sourcing | **Not needed for MVP** | Focus on existing candidates first |

### Scaling Beyond MVP

When the candidate pool grows beyond what the LLM can scan directly (hundreds -> thousands):

1. **Add a Bitable index.** Create a Bitable table with candidate metadata (name, practice area, seniority, status, folder link). The agent queries Bitable first for filtering, then reads full docs only for shortlisted candidates.
2. **Add embeddings.** Embed Summary docs into a vector store for semantic search. Only needed at scale.
3. **Add external sourcing.** Integrate LinkedIn, Leonar, or other candidate databases for finding new candidates (not in the existing pool).
4. **Multi-agent split.** Break the single agent into specialized agents (searcher, evaluator, writer) when complexity warrants it.

### Key Risks

1. **Lark API rate limits.** Reading many candidate folders sequentially could be slow. Mitigate by: batching reads, caching recently-read Summaries, or maintaining a lightweight Bitable index.
2. **Document format inconsistency.** If Summary/Submissions docs vary in structure across candidates, the LLM needs to handle different formats. Mitigate by: establishing a template, or relying on the LLM's flexibility with freeform text.
3. **Bias/discrimination liability.** AI-assisted candidate matching in legal recruiting still carries legal risk. Must have human-in-the-loop for all decisions.
4. **Confidentiality.** Candidate data is sensitive. The LLM provider sees candidate information. Mitigate by: using a self-hosted model, or ensuring the provider has appropriate data processing agreements.
5. **Lark Docs API coverage.** Need to verify the Lark OpenAPI MCP server can actually read document content (not just metadata). May need a custom MCP server if the official one is limited.
