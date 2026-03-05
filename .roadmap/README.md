# AI Agents Developer Roadmap - 6 Month Plan

> This is the high-level blueprint. Each level has its own detailed workflow file.

## Your Profile

- **Goal:** AI Engineer role specializing in workflow automation
- **Background:** Full-stack developer (Next.js, Hono), intermediate backend
- **Timeline:** 6 months (1-2 hours/day, 10-14 hours/week)
- **Focus:** Workflows (80%) + Chatbots (20%)
- **Learning Style:** Build from scratch + reverse-engineer code

**Skills to Strengthen:**
- Async/streaming patterns in Node.js
- Job queues & background processing
- Vector databases & embeddings
- Agent decision-making & planning

---

## 6-Month Timeline

### Month 1-2: Foundation (Levels 0-2)

| Level | Topic | Weeks | Deliverable |
|-------|-------|-------|-------------|
| [Level 0](./roadmap-level-0-streaming.md) | Streaming & Async | 1-2 | Working streaming chatbot |
| [Level 1](./roadmap-level-1-queues.md) | Job Queues & Workers | 3-4 | Async workflow system |
| [Level 2](./roadmap-level-2-browser.md) | Browser Automation | 5-7 | OpenClaw-style browser agent |

### Month 3: Workflow Automation (Level 3)

| Level | Topic | Weeks | Deliverable |
|-------|-------|-------|-------------|
| [Level 3](./roadmap-level-3-api-workflows.md) | API Workflows | 8-10 | Zapier/n8n-style workflow platform |

### Month 4: RAG & Knowledge (Level 4)

| Level | Topic | Weeks | Deliverable |
|-------|-------|-------|-------------|
| [Level 4](./roadmap-level-4-rag.md) | RAG Workflows | 11-13 | Document Q&A system |

### Month 5: AI Agents (Levels 5-6)

| Level | Topic | Weeks | Deliverable |
|-------|-------|-------|-------------|
| [Level 5](./roadmap-level-5-agents.md) | Single Agent | 14-16 | Task-completing agent |
| [Level 6](./roadmap-level-6-multi-agent.md) | Multi-Agent | 17-19 | Multi-agent workflow |

### Month 6: Production & Job Prep (Levels 7-8)

| Level | Topic | Weeks | Deliverable |
|-------|-------|-------|-------------|
| [Level 7](./roadmap-level-7-production.md) | Production Patterns | 20-22 | Production-ready system |
| [Level 8](./roadmap-level-8-portfolio.md) | Portfolio & Job Hunt | 23-26 | Job offers! |

---

## Key Technologies

**Core Stack:** Next.js 16 (App Router), TypeScript, pnpm + Turborepo

**AI/ML:** Anthropic Claude (primary), OpenAI (embeddings), Vercel AI SDK

**Infrastructure:** BullMQ + Redis, Supabase (database + vectors), Playwright, Upstash Redis

---

## Success Metrics

**Technical Skills:**
- [ ] Can build streaming LLM applications
- [ ] Can implement job queues and background workers
- [ ] Can build browser automation agents
- [ ] Can create API workflow systems
- [ ] Can implement RAG pipelines
- [ ] Can build autonomous AI agents
- [ ] Can deploy production-ready systems

**Portfolio:**
- [ ] 2-3 polished projects on GitHub
- [ ] Technical blog posts explaining your work
- [ ] Live demos deployed on Vercel

**Job Readiness:**
- [ ] Resume highlights AI/workflow experience
- [ ] Can explain agent architectures in interviews
- [ ] Can code AI systems in technical interviews

---

## Weekly Schedule Template

**Weekdays (Mon-Fri):** 1-2 hours after work
- 30 min: Study/read documentation
- 60-90 min: Hands-on coding

**Weekends (Sat-Sun):** 3-4 hours (optional)
- Longer coding sessions
- Reverse-engineer open source projects

---

## Resources

**Documentation:**
- [Anthropic Docs](https://docs.anthropic.com/)
- [Vercel AI SDK](https://sdk.vercel.ai/docs)
- [BullMQ Guide](https://docs.bullmq.io/)
- [Playwright Docs](https://playwright.dev/)
- [Supabase Vector](https://supabase.com/docs/guides/ai)

**Open Source to Study:**
- [Vercel AI Chatbot](https://github.com/vercel/ai-chatbot)
- [n8n](https://github.com/n8n-io/n8n)
- [Trigger.dev](https://github.com/triggerdotdev/trigger.dev)
- [LangChain.js](https://github.com/langchain-ai/langchainjs)

**Communities:**
- [Anthropic Discord](https://discord.gg/anthropic)
- [AI Engineer Discord](https://discord.gg/aiengineer)
- [r/LocalLLaMA](https://reddit.com/r/LocalLLaMA)

---

## Interview Prep Topics

**Core Concepts:**
- How LLMs work (tokens, context windows, temperature)
- Streaming vs batch processing
- RAG architecture and optimization
- Agent decision-making patterns (ReAct, Chain-of-Thought)
- Vector databases and similarity search

**System Design:**
- Design a workflow automation system
- Design an AI agent architecture
- Design a RAG system for enterprise docs
- Handle rate limits and costs at scale

**Coding:**
- Implement streaming endpoint
- Build simple agent loop
- Create vector search function
- Handle async workflows

---

## Notes & Updates

### 2026-03-05
- Initial roadmap created
- 9 levels, workflow-focused (80%)
- Designed for 1-2 hours daily

### Future Updates
_Add notes here as you learn and as the AI landscape evolves_

---

**Last Updated:** 2026-03-05
