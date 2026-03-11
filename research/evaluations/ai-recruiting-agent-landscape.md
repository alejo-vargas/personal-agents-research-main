# AI Agents for Recruiting / Candidate Matching

**Date:** 2026-03-11
**Status:** Initial research complete

---

## TL;DR

The recruiting AI agent space is mature commercially but thin on open-source. The dominant architecture is evolving from single-model RAG pipelines to **multi-agent systems** with specialized agents for parsing, matching, scoring, and outreach. For Lark/Feishu specifically, no dedicated recruiting agent exists, but the infrastructure is ready: **OpenClaw** supports multi-agent routing into Feishu group chats, **Mgrsc/lark_bot** supports MCP tool integration, and there is now an **official Lark OpenAPI MCP server**. The clearest path is: build a multi-agent recruiting system using CrewAI or LangGraph, expose it via MCP, and deploy it into Lark through OpenClaw or a custom Feishu bot.

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

## 6. Recommended Architecture for a Lark-Based Recruiting Agent

Based on this research, here is a viable architecture:

```
┌──────────────────────────────────────────────────────┐
│                    LARK / FEISHU                      │
│                                                      │
│  Group Chat: #staffing-requests                      │
│  ┌──────────────────────────────────────────┐        │
│  │ Manager: "Need a senior React dev, 5+    │        │
│  │ years, fintech experience, NYC preferred" │        │
│  └──────────────────────────────────────────┘        │
│           │ (event: im.message.receive_v1)           │
└───────────┼──────────────────────────────────────────┘
            ▼
┌──────────────────────────────────────────────────────┐
│              AGENT GATEWAY                            │
│  (OpenClaw / custom Feishu bot / Mgrsc/lark_bot)     │
│  - WebSocket connection to Lark                      │
│  - @mention detection                                │
│  - Thread-based responses                            │
│  - Redis for conversation state                      │
└───────────┼──────────────────────────────────────────┘
            ▼
┌──────────────────────────────────────────────────────┐
│           MULTI-AGENT ORCHESTRATOR                    │
│           (CrewAI / LangGraph)                        │
│                                                      │
│  Agent 1: JD Parser                                  │
│    - LLM structured extraction of requirements       │
│    - Output: structured JSON (skills, experience,    │
│      location, nice-to-haves)                        │
│                                                      │
│  Agent 2: Candidate Searcher                         │
│    - Query vector DB of candidate profiles           │
│    - Hybrid: embedding similarity + keyword filters  │
│    - MCP tools: query CRM, ATS, external sources     │
│                                                      │
│  Agent 3: Match Evaluator                            │
│    - Score each candidate against structured JD      │
│    - Explain match reasoning                         │
│    - Flag gaps and risks                             │
│                                                      │
│  Agent 4: Presenter                                  │
│    - Format top-N candidates as Lark card message    │
│    - Include match score, key skills, gaps           │
│    - Action buttons: "Schedule interview", "Pass"    │
└───────────┼──────────────────────────────────────────┘
            │
            ▼ (via MCP)
┌──────────────────────────────────────────────────────┐
│              MCP TOOL SERVERS                         │
│                                                      │
│  - Candidate DB (custom MCP server over your data)   │
│  - Lark OpenAPI MCP (for calendar, docs, messages)   │
│  - Leonar MCP (if using Leonar as CRM)               │
│  - HubSpot MCP (if using HubSpot as CRM)             │
│  - Zapier MCP (for outreach, email, etc.)            │
└──────────────────────────────────────────────────────┘
```

### Build vs. Buy Decision

| Component | Build | Buy/Use |
|-----------|-------|---------|
| Lark bot gateway | Build (or use OpenClaw/lark_bot) | -- |
| JD parsing | Build (LLM structured extraction) | -- |
| Candidate DB + embeddings | Build (ChromaDB/Pinecone + embedding model) | -- |
| Matching logic | Build (multi-agent with CrewAI/LangGraph) | -- |
| CRM integration | -- | Leonar MCP / HubSpot MCP |
| Candidate sourcing (external) | -- | Leonar (870M profiles) / Juicebox API |
| Outreach automation | -- | Zapier MCP / Leonar outreach |

### Key Risks

1. **Bias/discrimination liability** -- AI matching in hiring is a legal minefield. NYC already requires bias audits. Must have human-in-the-loop for all hiring decisions.
2. **Data privacy** -- Candidate data is PII. Need encryption, access controls, retention policies, GDPR/CCPA compliance.
3. **Matching accuracy** -- Pure embedding similarity misses nuance. Multi-agent with human review is essential.
4. **Lark ecosystem limitations** -- Fewer third-party integrations than Slack. May need custom MCP servers for Lark-specific features.
