# Skills.md — Scout (Coding Research Agent)

This file lists Scout's specialized research skills — concrete procedures for recurring analysis tasks. Reference these by name in instructions or task descriptions.

---

## Skill: compare-repos

**Trigger**: "Compare X and Y", "Which library is better for Z", "How do A and B differ?"

**Procedure**:

1. **Identify candidates** — confirm the repos/libraries to compare. If the user named specific ones, use those. If they described a problem, find the top 3-5 candidates via `WebSearch` + GitHub trending/stars.

2. **Quick triage** (eliminate weak options early):
   - Check GitHub stars, last commit date, open issues count (`gh api repos/{owner}/{repo}`)
   - Read README for scope, maturity, and stated goals
   - Eliminate anything abandoned (>6 months no commits), pre-alpha, or fundamentally wrong scope

3. **Deep comparison** (for remaining 2-3 candidates):
   - Clone to `refs/` if needed for deep code reading
   - Evaluate along these axes:

   | Axis | What to check |
   |------|---------------|
   | API ergonomics | How many lines to do the common case? Is the API intuitive? |
   | Architecture | Clean separation? Testable? Extensible without forking? |
   | Performance | Benchmarks in repo? Known perf characteristics? |
   | Bundle/binary size | Relevant for frontend/mobile/edge |
   | Type safety | TypeScript types? Rust safety? Go generics? |
   | Error handling | Panics/crashes vs. recoverable errors? |
   | Testing | Test coverage? Test quality? CI passing? |
   | Documentation | API docs? Examples? Guides? |
   | Community | Contributors? Release cadence? Issue response time? |
   | License | Permissive? Copyleft? Commercial-friendly? |

4. **Produce comparison table** — fill in the axes that matter for this specific decision (not all axes are always relevant).

5. **State recommendation** — lead with it. "Use X because..." with 1-2 sentences. Then the table. Then caveats.

6. **Save to** `research/comparisons/{topic}.md` if the comparison is substantial enough to reference later.

---

## Skill: evaluate-library

**Trigger**: "Should we use X?", "Is X production-ready?", "What's the state of X?"

**Procedure**:

1. **Gather signals**:
   ```
   gh api repos/{owner}/{repo} --jq '{stars: .stargazers_count, forks: .forks_count, open_issues: .open_issues_count, license: .license.spdx_id, pushed_at: .pushed_at, created_at: .created_at}'
   ```
   - WebSearch for "{library} production use", "{library} problems", "{library} alternatives"
   - Check npm/crates.io/PyPI for download trends if applicable

2. **Health checklist**:
   - [ ] Last release within 6 months
   - [ ] CI/CD passing on main branch
   - [ ] More than 1 active maintainer (bus factor > 1)
   - [ ] Breaking changes documented in changelogs
   - [ ] Security vulnerabilities addressed promptly
   - [ ] TypeScript types (if JS ecosystem) / API docs (if other)

3. **Risk assessment**:
   - **Low risk**: Mature, well-maintained, large community, stable API
   - **Medium risk**: Active but young, small team, API still evolving
   - **High risk**: Single maintainer, infrequent updates, known unresolved issues
   - **Avoid**: Abandoned, no tests, license issues, fundamental design flaws

4. **Deliver verdict**: One of: **Adopt** / **Trial** / **Hold** / **Avoid** (borrowing from ThoughtWorks Radar).

5. **Save to** `research/evaluations/{library}.md` if the evaluation is substantial.

---

## Skill: analyze-architecture

**Trigger**: "How is X built?", "What's the architecture of X?", "How does X handle Y?"

**Procedure**:

1. **Get the source** — clone to `refs/` if not already present.

2. **Map the structure**:
   - Entry points (main, index, app bootstrap)
   - Directory layout and module boundaries
   - Dependency graph (package.json, Cargo.toml, go.mod, etc.)
   - Configuration and environment handling

3. **Trace key flows** — identify the 2-3 most important code paths for the user's question and trace them:
   - Request/response lifecycle
   - Data flow (input → processing → storage → output)
   - Error propagation path
   - State management pattern

4. **Identify patterns**:
   - Design patterns used (repository, factory, observer, etc.)
   - Architectural style (monolith, microservices, modular monolith, serverless)
   - State management approach
   - Testing strategy

5. **Assess quality signals**:
   - Is the architecture consistent or does it have multiple conflicting patterns?
   - Are boundaries clean or is there spaghetti coupling?
   - Could you add a feature without touching unrelated code?

6. **Deliver as**:
   - ASCII diagram of the architecture
   - 3-5 bullet summary of key patterns
   - Strengths and weaknesses
   - What to steal vs. what to avoid

7. **Save to** `research/strategies/{project}-architecture.md` if substantial.

---

## Skill: research-implementation-strategy

**Trigger**: "How should we build X?", "What's the best approach for Y?", "Plan the implementation of Z"

**Procedure**:

1. **Clarify requirements** — ask one round of questions if needed:
   - What's the target platform/language/framework?
   - What are the hard constraints (performance, compatibility, team skills)?
   - What's the timeline pressure (MVP vs. production-grade)?

2. **Find prior art** — search for 3-5 projects that solved a similar problem:
   - WebSearch for tutorials, blog posts, case studies
   - GitHub search for implementations
   - Check if major frameworks have built-in solutions

3. **Analyze approaches** — for each viable approach:
   - How much code is required?
   - What dependencies are introduced?
   - What are the failure modes?
   - How does it scale?
   - How testable is it?

4. **Produce strategy document**:
   ```
   ## Recommended Approach
   [1-2 sentence summary]

   ## Why This Over Alternatives
   [Brief comparison]

   ## Implementation Outline
   1. [Step with estimated complexity: trivial/moderate/complex]
   2. ...

   ## Key Decisions
   - [Decision point]: [recommended choice] because [reason]

   ## Risks & Mitigations
   - [Risk]: [Mitigation]

   ## References
   - [Repo/article that informed this strategy]
   ```

5. **Save to** `research/strategies/{topic}.md`.

---

## Skill: mine-code-patterns

**Trigger**: "How do top projects handle X?", "What's the best practice for Y?", "Show me examples of Z"

**Procedure**:

1. **Identify target repos** — find 5-10 well-regarded projects that deal with the pattern in question. Prioritize:
   - High stars + active maintenance
   - Production-proven (used by real companies)
   - Clean, readable implementations

2. **Extract implementations** — for each repo:
   - Find the relevant code (Grep/Glob/Read)
   - Extract the core pattern (strip away project-specific details)
   - Note the approach category (e.g., "retry with exponential backoff + jitter" vs. "retry with fixed delay")

3. **Categorize approaches**:
   - **Consensus pattern**: Most projects do it this way → safe default
   - **Novel approach**: One project does it differently → interesting but risky
   - **Anti-pattern**: Some projects do it badly → learn what to avoid

4. **Synthesize**:
   - "The consensus approach is X (used by 7/10 projects). Key variant: Y (used by 2/10, better for Z use case). Avoid: W (fragile because...)."
   - Include 1-2 concrete code snippets from the best implementations

5. **Save to** `research/comparisons/{pattern}.md` if the pattern is likely to come up again.

---

## Skill: check-repo-health

**Trigger**: "Is this repo still maintained?", "Can we depend on X?", "What's the state of X?"

**Procedure**:

1. **Automated checks** (run in parallel):
   ```bash
   # GitHub API stats
   gh api repos/{owner}/{repo} --jq '{stars: .stargazers_count, forks: .forks_count, open_issues: .open_issues_count, license: .license.spdx_id, pushed_at: .pushed_at, archived: .archived}'

   # Recent commits
   gh api repos/{owner}/{repo}/commits --jq '.[0:5] | .[] | {date: .commit.author.date, message: .commit.message[:80]}'

   # Recent releases
   gh api repos/{owner}/{repo}/releases --jq '.[0:3] | .[] | {tag: .tag_name, date: .published_at, prerelease: .prerelease}'

   # Open vs closed issues ratio (last 90 days)
   gh api "repos/{owner}/{repo}/issues?state=all&per_page=100&since=$(date -v-90d +%Y-%m-%dT%H:%M:%SZ)" --jq 'group_by(.state) | map({state: .[0].state, count: length})'
   ```

2. **Manual checks**:
   - Read the latest 5 issues — are maintainers responding?
   - Check if CI badges are passing on README
   - Look for "looking for maintainers" or "archived" notices
   - Check if there's a SECURITY.md or security policy

3. **Produce health card**:
   ```
   ## {repo} Health Check — {date}

   | Signal | Status |
   |--------|--------|
   | Last commit | {date} |
   | Last release | {version} ({date}) |
   | Open issues | {count} |
   | Issue response | {responsive/slow/silent} |
   | CI status | {passing/failing/unknown} |
   | Bus factor | {N maintainers} |
   | License | {license} |

   **Verdict**: {Healthy / Caution / At Risk / Abandoned}
   ```

---

## Skill: deep-dive-codebase

**Trigger**: Handed a repo URL or local path and asked to understand it thoroughly.

**Procedure**:

1. **First pass** (5 min — get oriented):
   - Read README, CONTRIBUTING, architecture docs
   - `ls` top-level structure
   - Read package manifest (package.json, Cargo.toml, go.mod, pyproject.toml)
   - Identify entry point(s)

2. **Second pass** (map the domain):
   - Glob for key file patterns (`**/*.rs`, `**/routes/*`, `**/models/*`)
   - Identify the domain model (what entities exist?)
   - Map module boundaries and dependencies
   - Find the test structure

3. **Third pass** (trace critical paths):
   - Follow the main user-facing flow from entry to output
   - Identify where data is stored and retrieved
   - Map error handling strategy
   - Find configuration/environment handling

4. **Produce summary**:
   ```
   ## {Project} — Codebase Overview

   **Purpose**: {one sentence}
   **Stack**: {languages, frameworks, key deps}
   **Size**: {rough LoC, file count}

   ### Architecture
   {ASCII diagram}

   ### Key Modules
   - `{path}` — {what it does}

   ### Patterns Used
   - {pattern}: {where and how}

   ### Quality Signals
   - Tests: {coverage, quality}
   - Docs: {quality}
   - Code style: {consistent, messy, over-engineered}

   ### Notable Decisions
   - {interesting/unusual choice}: {why they likely did it}
   ```

---

## Workflow Reminders

- **Research before recommending.** Never recommend based on name recognition alone.
- **Cite your sources.** Every claim should trace back to code, docs, or data.
- **Save substantial findings.** If it took more than 10 minutes to research, save it to `research/`.
- **Update, don't duplicate.** Check for existing research files before creating new ones.
- **Lead with the answer.** The user wants a recommendation, not a literature review. Put the conclusion first.
- **Be honest about confidence.** "I'm 90% sure X is better, but Y has an edge case advantage for Z" is better than false certainty.
