# Level 8: Portfolio & Job Prep (Week 23-26)

**Time:** 20-28 hours | **Status:** 🔲 Not Started | **Priority:** CRITICAL

[Back to Blueprint](./README.md) | Prev: [Level 7](./roadmap-level-7-production.md)

---

## Why This Matters

You've built the skills. Now you need to package them into a compelling story that gets you hired. This level is about turning your projects into portfolio pieces, preparing for interviews, and landing the job.

## What to Focus On

- Polish 2-3 best projects for portfolio
- Write technical content about what you built
- Prepare for system design and coding interviews
- Build your professional presence
- Apply strategically

---

## Week 1: Portfolio Projects (8-10 hours)

### Goal
Select and polish your best 2-3 projects into impressive portfolio pieces.

### Which Projects to Polish

Pick 2-3 from your completed levels. Recommended:

**1. Workflow Automation Platform (from Levels 1+3)**
- Combines job queues + API workflows
- Shows you can build infrastructure
- Directly relevant to workflow companies

**2. AI Browser Agent (from Level 2)**
- Hot topic, very impressive in demos
- Shows vision + automation skills
- Relevant to OpenClaw-style companies

**3. Multi-Agent System (from Level 6)**
- Shows advanced AI architecture skills
- Demonstrates orchestration knowledge
- Relevant to AI platform companies

### Day 1-2: Clean Up Code (4-5 hours)

For each project:

- [ ] Clean up folder structure
- [ ] Remove dead code and console.logs
- [ ] Add proper TypeScript types (no `any` where avoidable)
- [ ] Add error handling where missing
- [ ] Make sure it runs with `pnpm dev`
- [ ] Add `.env.example` with all required variables

### Day 3: Write READMEs (3-4 hours)

Each project needs a strong README:

```markdown
# Project Name

One-line description of what it does.

## Demo

[Screenshot or GIF of it working]
[Link to live demo if deployed]

## What I Learned

- Key concept 1 and how I applied it
- Key concept 2 and how I applied it
- Challenge I faced and how I solved it

## Architecture

[Simple diagram or description]
- How the system works
- Key design decisions
- Tech stack and why

## Quick Start

\`\`\`bash
git clone ...
pnpm install
cp .env.example .env.local
# Fill in API keys
pnpm dev
\`\`\`

## Key Files

- `lib/agent/core.ts` - Agent loop implementation
- `lib/workflow/engine.ts` - Workflow execution engine
- etc.
```

### Day 4: Deploy Demos (2-3 hours)

- [ ] Deploy to Vercel (free tier)
- [ ] Set up environment variables
- [ ] Test live demos work
- [ ] Record a short demo video (Loom or screen recording)

---

## Week 2: Technical Content (6-8 hours)

### Goal
Write 2-3 technical blog posts that demonstrate your knowledge.

### Why Write?

- Shows depth of understanding (not just copy-paste)
- Attracts recruiters and hiring managers
- Builds your professional brand
- Forces you to solidify your knowledge

### Blog Post Ideas

**Post 1: "Building a Workflow Automation Engine in Node.js"**
- How workflow engines work (nodes, edges, execution)
- Job queues with BullMQ
- Error handling and retries
- Code examples from your project

**Post 2: "How I Built an AI Browser Agent with Playwright + Claude"**
- Vision-based navigation explained
- The agent loop pattern
- Challenges with dynamic websites
- Demo and results

**Post 3: "Multi-Agent Systems: Making AI Agents Work Together"**
- Why single agents aren't enough
- Orchestration patterns (supervisor, pipeline, debate)
- Inter-agent communication
- Real examples from your project

### Where to Publish

- **Dev.to** - Easy to publish, good developer audience
- **Hashnode** - Good SEO, custom domain support
- **Your own blog** - If you have one
- **Medium** - Wider audience but paywalled

### Day 5-6: Write Posts (4-5 hours)

Structure for each post:
1. Hook - Why should someone care?
2. Problem - What are you solving?
3. Solution - How you built it (with code)
4. Results - What you learned
5. Call to action - Link to repo

### Day 7: Publish & Share (2-3 hours)

- [ ] Publish on chosen platform
- [ ] Share on Twitter/X with key takeaways
- [ ] Share on LinkedIn
- [ ] Post in relevant Discord communities

---

## Week 3: Interview Prep (6-8 hours)

### Goal
Prepare for AI engineer interviews - system design, coding, and behavioral.

### Day 8-9: System Design Practice (4 hours)

Practice designing these systems (whiteboard or paper):

**Design 1: "Design a workflow automation platform like Zapier"**
- Trigger system (webhooks, schedules, events)
- Workflow execution engine
- Node types and data flow
- Error handling and retries
- Scaling considerations
- Database schema

**Design 2: "Design an AI agent system"**
- Agent loop architecture
- Tool registry and execution
- Memory management
- Multi-agent orchestration
- Cost and rate limiting
- Monitoring and observability

**Design 3: "Design a RAG system for enterprise documents"**
- Document ingestion pipeline
- Chunking strategy
- Embedding and vector storage
- Retrieval and reranking
- Answer generation with citations
- Scaling to millions of documents

For each design, practice explaining:
- Requirements and constraints
- High-level architecture
- Key components and their interactions
- Data flow
- Trade-offs you made
- How you'd scale it

### Day 10: Coding Interview Prep (2 hours)

Practice coding these from scratch (no IDE help):

**Exercise 1: Streaming endpoint**
```typescript
// Implement a streaming chat endpoint that:
// 1. Accepts messages array
// 2. Calls Claude API with streaming
// 3. Returns SSE stream to client
// 4. Handles errors gracefully
```

**Exercise 2: Simple agent loop**
```typescript
// Implement a ReAct agent that:
// 1. Takes a task description
// 2. Has access to 2-3 tools
// 3. Reasons about what to do
// 4. Executes tools
// 5. Returns final answer
```

**Exercise 3: Vector search**
```typescript
// Implement a function that:
// 1. Takes a query string
// 2. Generates embedding
// 3. Searches vector store
// 4. Returns top-k results with similarity scores
```

### Day 11: Behavioral Prep (2 hours)

Prepare stories for these questions:

- "Tell me about a complex technical project you built"
  → Your multi-agent system or workflow engine

- "How do you approach learning new technologies?"
  → This 6-month roadmap, building from scratch

- "Describe a time you solved a difficult bug"
  → Pick a real debugging story from your projects

- "How do you make technical decisions?"
  → Trade-offs you made in your projects (e.g., BullMQ vs Inngest)

- "Where do you see AI going in the next 2 years?"
  → Your perspective on agents, workflows, automation

---

## Week 4: Job Hunt (4-6 hours)

### Goal
Start applying strategically.

### Day 12: Resume & LinkedIn (2-3 hours)

**Resume updates:**
- [ ] Add "AI Engineer" or "AI/ML Engineer" to target title
- [ ] List key projects with impact:
  - "Built workflow automation engine processing N+ concurrent jobs with BullMQ"
  - "Developed AI browser agent using Claude vision + Playwright"
  - "Implemented multi-agent orchestration system with supervisor pattern"
- [ ] List relevant skills: LLM APIs, RAG, Vector Databases, Agent Architecture, Streaming, Workflow Automation
- [ ] Link to GitHub and blog posts

**LinkedIn updates:**
- [ ] Update headline: "Full-Stack Developer | AI Engineer | Workflow Automation"
- [ ] Add projects to Featured section
- [ ] Write a post about your learning journey
- [ ] Connect with AI engineers and recruiters

### Day 13-14: Apply (2-3 hours)

**Where to look:**
- **AI-focused companies:** Anthropic, OpenAI, Vercel, LangChain, n8n, Trigger.dev
- **Startups building AI products:** Search "AI agent" or "workflow automation" on:
  - Y Combinator jobs (workatastartup.com)
  - AI-specific job boards (ai-jobs.net)
  - LinkedIn Jobs
  - AngelList / Wellfound
- **Enterprise companies with AI teams:** Look for "AI Platform Engineer" roles

**Application strategy:**
- Apply to 3-5 jobs per week
- Customize cover letter mentioning relevant projects
- Include links to live demos and blog posts
- Follow up after 1 week if no response

**Networking:**
- Attend AI meetups (local or virtual)
- Engage in AI Discord communities
- Comment on AI-related posts on Twitter/LinkedIn
- Reach out to engineers at target companies

---

## Final Checkpoints

- [ ] 2-3 polished projects on GitHub with strong READMEs
- [ ] Projects deployed with live demos
- [ ] 2-3 technical blog posts published
- [ ] Resume updated with AI experience
- [ ] LinkedIn optimized
- [ ] Can whiteboard AI system designs
- [ ] Can code agent/streaming/RAG from scratch
- [ ] Applying to 3-5 jobs per week

---

## Study Resources

- [ ] Read: [Interviewing.io AI interview guide](https://interviewing.io/)
- [ ] Practice: System design on Excalidraw
- [ ] Watch: AI engineer interview videos on YouTube
- [ ] Read: Job postings to understand what companies want

---

## Interview Prep

- Explain: Your learning journey and motivation
- Explain: Architecture of each portfolio project
- Demo: Live walkthrough of your best project
- Code: Agent loop, streaming endpoint, vector search from scratch
- Design: Workflow platform, RAG system, agent architecture

---

**You made it! 6 months of focused learning. You've built real AI systems, written about them, and prepared to interview. Now go get that job.**
