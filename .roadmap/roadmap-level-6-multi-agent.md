# Level 6: Multi-Agent Systems (Week 17-19)

**Time:** 18-24 hours | **Status:** 🔲 Not Started | **Priority:** HIGH

[Back to Blueprint](./README.md) | Prev: [Level 5](./roadmap-level-5-agents.md) | Next: [Level 7](./roadmap-level-7-production.md)

---

## Why This Matters

Real-world AI products rarely use a single agent. Complex workflows need specialized agents that collaborate - a researcher, a writer, a reviewer, a coder. This is how tools like Devin, OpenClaw, and enterprise AI platforms work.

## Concepts to Master

- Agent orchestration patterns
- Specialized agent roles
- Inter-agent communication
- Task delegation and routing
- Supervisor/worker pattern
- Pipeline pattern (sequential agents)
- Debate pattern (agents critique each other)
- Shared state and context passing

## Project: Multi-Agent Content Pipeline

Build a system where multiple agents collaborate to research a topic, write content, review it, and publish.

```
apps/level-6-multi-agent/
├── app/
│   ├── api/
│   │   ├── pipeline/route.ts          # Start pipeline
│   │   └── pipeline/[id]/route.ts     # Get status
│   ├── page.tsx                        # Pipeline UI
│   └── components/
│       ├── PipelineView.tsx            # Pipeline visualization
│       ├── AgentCard.tsx               # Individual agent status
│       └── MessageLog.tsx              # Inter-agent messages
├── lib/
│   ├── agents/
│   │   ├── base-agent.ts              # Base agent class
│   │   ├── researcher.ts              # Research agent
│   │   ├── writer.ts                  # Writing agent
│   │   ├── reviewer.ts                # Review/critique agent
│   │   └── publisher.ts               # Publishing agent
│   ├── orchestrator/
│   │   ├── supervisor.ts              # Supervisor orchestrator
│   │   ├── pipeline.ts                # Pipeline orchestrator
│   │   └── router.ts                  # Task router
│   └── shared/
│       ├── message-bus.ts             # Agent communication
│       ├── state.ts                   # Shared state
│       └── types.ts                   # Shared types
```

---

## Day 1-2: Base Agent & Communication (4-6 hours)

### Goal
Build a reusable base agent class and a message bus for agent communication.

### Steps

**1. Define shared types (30 min)**

Create `lib/shared/types.ts`:
```typescript
export interface AgentMessage {
  id: string;
  from: string;       // Agent name
  to: string;         // Target agent or 'broadcast'
  type: 'task' | 'result' | 'feedback' | 'error';
  content: string;
  data?: Record<string, any>;
  timestamp: Date;
}

export interface AgentRole {
  name: string;
  description: string;
  systemPrompt: string;
  tools?: ToolDefinition[];
}

export interface PipelineState {
  id: string;
  status: 'running' | 'completed' | 'failed';
  currentAgent: string;
  messages: AgentMessage[];
  results: Record<string, any>;
  startedAt: Date;
  completedAt?: Date;
}

export interface ToolDefinition {
  name: string;
  description: string;
  parameters: Record<string, any>;
  execute: (params: any) => Promise<string>;
}
```

**2. Build message bus (45 min)**

Create `lib/shared/message-bus.ts`:
```typescript
import { AgentMessage } from './types';

type MessageHandler = (message: AgentMessage) => void;

export class MessageBus {
  private handlers: Map<string, MessageHandler[]> = new Map();
  private allMessages: AgentMessage[] = [];

  subscribe(agentName: string, handler: MessageHandler) {
    const existing = this.handlers.get(agentName) || [];
    existing.push(handler);
    this.handlers.set(agentName, existing);
  }

  publish(message: AgentMessage) {
    this.allMessages.push(message);

    // Direct message
    if (message.to !== 'broadcast') {
      const handlers = this.handlers.get(message.to) || [];
      handlers.forEach((h) => h(message));
      return;
    }

    // Broadcast to all except sender
    for (const [name, handlers] of this.handlers) {
      if (name !== message.from) {
        handlers.forEach((h) => h(message));
      }
    }
  }

  getHistory(agentName?: string): AgentMessage[] {
    if (!agentName) return [...this.allMessages];
    return this.allMessages.filter(
      (m) => m.from === agentName || m.to === agentName || m.to === 'broadcast'
    );
  }

  clear() {
    this.allMessages = [];
    this.handlers.clear();
  }
}
```

**3. Build base agent class (90 min)**

Create `lib/agents/base-agent.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { AgentMessage, AgentRole, ToolDefinition } from '../shared/types';
import { MessageBus } from '../shared/message-bus';

const client = new Anthropic();

export class BaseAgent {
  protected role: AgentRole;
  protected bus: MessageBus;
  protected inbox: AgentMessage[] = [];

  constructor(role: AgentRole, bus: MessageBus) {
    this.role = role;
    this.bus = bus;

    // Subscribe to messages
    bus.subscribe(role.name, (msg) => {
      this.inbox.push(msg);
    });
  }

  async process(task: string, context?: Record<string, any>): Promise<string> {
    const messages = this.buildMessages(task, context);

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      system: this.role.systemPrompt,
      messages,
      ...(this.role.tools?.length
        ? {
            tools: this.role.tools.map((t) => ({
              name: t.name,
              description: t.description,
              input_schema: {
                type: 'object' as const,
                properties: t.parameters,
              },
            })),
          }
        : {}),
    });

    // Handle tool use in a loop
    let result = '';
    let currentResponse = response;

    while (currentResponse.stop_reason === 'tool_use') {
      const toolResults = [];

      for (const block of currentResponse.content) {
        if (block.type === 'text') {
          result += block.text;
        } else if (block.type === 'tool_use') {
          const tool = this.role.tools?.find((t) => t.name === block.name);
          if (tool) {
            const toolResult = await tool.execute(block.input);
            toolResults.push({
              type: 'tool_result' as const,
              tool_use_id: block.id,
              content: toolResult,
            });
          }
        }
      }

      // Continue conversation with tool results
      currentResponse = await client.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 4096,
        system: this.role.systemPrompt,
        messages: [
          ...messages,
          { role: 'assistant', content: currentResponse.content },
          { role: 'user', content: toolResults },
        ],
      });
    }

    // Extract final text
    for (const block of currentResponse.content) {
      if (block.type === 'text') {
        result += block.text;
      }
    }

    return result;
  }

  private buildMessages(task: string, context?: Record<string, any>) {
    const parts: string[] = [];

    // Add context from other agents
    if (context) {
      parts.push('## Context from other agents:');
      for (const [key, value] of Object.entries(context)) {
        parts.push(`### ${key}:\n${value}`);
      }
      parts.push('');
    }

    // Add inbox messages
    if (this.inbox.length > 0) {
      parts.push('## Messages from other agents:');
      for (const msg of this.inbox) {
        parts.push(`[${msg.from}]: ${msg.content}`);
      }
      parts.push('');
    }

    parts.push(`## Your task:\n${task}`);

    return [{ role: 'user' as const, content: parts.join('\n') }];
  }

  sendMessage(to: string, type: AgentMessage['type'], content: string, data?: Record<string, any>) {
    this.bus.publish({
      id: crypto.randomUUID(),
      from: this.role.name,
      to,
      type,
      content,
      data,
      timestamp: new Date(),
    });
  }
}
```

### Day 1-2 Checkpoints
- [ ] Base agent class works with Claude API
- [ ] Message bus enables agent communication
- [ ] Agents can send/receive messages
- [ ] Tool use works within agents

---

## Day 3-5: Specialized Agents (6-8 hours)

### Goal
Build 4 specialized agents that each handle a different part of the content pipeline.

### Steps

**1. Research Agent (60 min)**

Create `lib/agents/researcher.ts`:
```typescript
import { BaseAgent } from './base-agent';
import { MessageBus } from '../shared/message-bus';

export function createResearcher(bus: MessageBus) {
  return new BaseAgent(
    {
      name: 'researcher',
      description: 'Researches topics and gathers information',
      systemPrompt: `You are a research specialist. Your job is to:
1. Break down the research topic into key questions
2. Use available tools to find information
3. Organize findings into a structured research brief
4. Include sources and key data points
5. Flag any conflicting information or gaps

Output format:
## Research Brief: [Topic]
### Key Findings
- Finding 1 (source)
- Finding 2 (source)
### Data Points
- Stat 1
- Stat 2
### Open Questions
- Question 1
### Sources
- Source 1`,
      tools: [
        {
          name: 'web_search',
          description: 'Search the web for information',
          parameters: {
            query: { type: 'string', description: 'Search query' },
          },
          execute: async (params: { query: string }) => {
            // In production, use a real search API (Tavily, Serper, etc.)
            // For learning, simulate results
            return `Search results for "${params.query}":\n1. [Result 1] Relevant information about ${params.query}\n2. [Result 2] Additional data on ${params.query}\n3. [Result 3] Expert analysis of ${params.query}`;
          },
        },
      ],
    },
    bus
  );
}
```

**2. Writer Agent (60 min)**

Create `lib/agents/writer.ts`:
```typescript
import { BaseAgent } from './base-agent';
import { MessageBus } from '../shared/message-bus';

export function createWriter(bus: MessageBus) {
  return new BaseAgent(
    {
      name: 'writer',
      description: 'Writes content based on research',
      systemPrompt: `You are a professional content writer. Your job is to:
1. Read the research brief provided
2. Create well-structured, engaging content
3. Use data points and citations from the research
4. Write in a clear, professional tone
5. Include an introduction, body sections, and conclusion

Guidelines:
- Use short paragraphs
- Include subheadings
- Back claims with data from the research
- Make it actionable and practical`,
    },
    bus
  );
}
```

**3. Reviewer Agent (60 min)**

Create `lib/agents/reviewer.ts`:
```typescript
import { BaseAgent } from './base-agent';
import { MessageBus } from '../shared/message-bus';

export function createReviewer(bus: MessageBus) {
  return new BaseAgent(
    {
      name: 'reviewer',
      description: 'Reviews and critiques content',
      systemPrompt: `You are a strict content reviewer. Your job is to:
1. Check factual accuracy against the research brief
2. Evaluate writing quality and clarity
3. Identify logical gaps or weak arguments
4. Suggest specific improvements
5. Rate the content on a scale of 1-10

Output format:
## Review
### Score: X/10
### Strengths
- ...
### Issues Found
- Issue 1: [description] → Suggested fix: [fix]
- Issue 2: [description] → Suggested fix: [fix]
### Verdict
APPROVE / NEEDS_REVISION / REJECT`,
    },
    bus
  );
}
```

**4. Publisher Agent (45 min)**

Create `lib/agents/publisher.ts`:
```typescript
import { BaseAgent } from './base-agent';
import { MessageBus } from '../shared/message-bus';

export function createPublisher(bus: MessageBus) {
  return new BaseAgent(
    {
      name: 'publisher',
      description: 'Formats and publishes final content',
      systemPrompt: `You are a publishing specialist. Your job is to:
1. Take the approved content
2. Format it for the target platform (blog, social, docs)
3. Add metadata (title, description, tags)
4. Create a summary/excerpt
5. Generate SEO-friendly elements

Output format:
## Published Content
### Metadata
- Title: ...
- Description: ...
- Tags: ...
- Excerpt: ...
### Final Content
[formatted content]`,
    },
    bus
  );
}
```

### Day 3-5 Checkpoints
- [ ] All 4 agents created with clear roles
- [ ] Each agent has a focused system prompt
- [ ] Research agent can use tools
- [ ] Agents produce structured output

---

## Day 6-8: Orchestration Patterns (6-8 hours)

### Goal
Build orchestrators that coordinate multiple agents.

### Steps

**1. Pipeline Orchestrator (90 min)**

Sequential execution: Research → Write → Review → (Revise?) → Publish

Create `lib/orchestrator/pipeline.ts`:
```typescript
import { MessageBus } from '../shared/message-bus';
import { PipelineState } from '../shared/types';
import { createResearcher } from '../agents/researcher';
import { createWriter } from '../agents/writer';
import { createReviewer } from '../agents/reviewer';
import { createPublisher } from '../agents/publisher';

export class PipelineOrchestrator {
  private bus: MessageBus;
  private state: PipelineState;

  constructor() {
    this.bus = new MessageBus();
    this.state = {
      id: crypto.randomUUID(),
      status: 'running',
      currentAgent: '',
      messages: [],
      results: {},
      startedAt: new Date(),
    };
  }

  async run(topic: string): Promise<PipelineState> {
    const researcher = createResearcher(this.bus);
    const writer = createWriter(this.bus);
    const reviewer = createReviewer(this.bus);
    const publisher = createPublisher(this.bus);

    try {
      // Step 1: Research
      this.state.currentAgent = 'researcher';
      console.log('🔍 Researcher is working...');
      const research = await researcher.process(
        `Research the following topic thoroughly: ${topic}`
      );
      this.state.results.research = research;

      // Step 2: Write
      this.state.currentAgent = 'writer';
      console.log('✍️  Writer is working...');
      const draft = await writer.process(
        `Write a comprehensive article about: ${topic}`,
        { research }
      );
      this.state.results.draft = draft;

      // Step 3: Review
      this.state.currentAgent = 'reviewer';
      console.log('🔎 Reviewer is working...');
      const review = await reviewer.process(
        'Review the following article for accuracy, quality, and completeness.',
        { research, draft }
      );
      this.state.results.review = review;

      // Step 4: Revise if needed
      if (review.includes('NEEDS_REVISION')) {
        this.state.currentAgent = 'writer';
        console.log('✍️  Writer is revising...');
        const revised = await writer.process(
          'Revise the article based on the reviewer feedback.',
          { draft, review }
        );
        this.state.results.revised = revised;
        this.state.results.finalDraft = revised;
      } else {
        this.state.results.finalDraft = draft;
      }

      // Step 5: Publish
      this.state.currentAgent = 'publisher';
      console.log('📤 Publisher is working...');
      const published = await publisher.process(
        'Format and prepare this content for publishing.',
        { content: this.state.results.finalDraft }
      );
      this.state.results.published = published;

      this.state.status = 'completed';
      this.state.completedAt = new Date();
    } catch (error) {
      this.state.status = 'failed';
      this.state.completedAt = new Date();
      throw error;
    }

    return this.state;
  }

  getState(): PipelineState {
    return { ...this.state };
  }
}
```

**2. Supervisor Orchestrator (90 min)**

A supervisor agent that dynamically decides which agent to call next.

Create `lib/orchestrator/supervisor.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { MessageBus } from '../shared/message-bus';
import { BaseAgent } from '../agents/base-agent';
import { createResearcher } from '../agents/researcher';
import { createWriter } from '../agents/writer';
import { createReviewer } from '../agents/reviewer';
import { createPublisher } from '../agents/publisher';

const client = new Anthropic();

export class SupervisorOrchestrator {
  private bus: MessageBus;
  private agents: Map<string, BaseAgent>;
  private results: Record<string, string> = {};

  constructor() {
    this.bus = new MessageBus();
    this.agents = new Map([
      ['researcher', createResearcher(this.bus)],
      ['writer', createWriter(this.bus)],
      ['reviewer', createReviewer(this.bus)],
      ['publisher', createPublisher(this.bus)],
    ]);
  }

  async run(task: string, maxRounds: number = 8): Promise<Record<string, string>> {
    let round = 0;

    while (round < maxRounds) {
      round++;
      console.log(`\n--- Round ${round} ---`);

      // Ask supervisor what to do next
      const decision = await this.decide(task);

      if (decision.action === 'done') {
        console.log('✅ Supervisor says: Done!');
        break;
      }

      // Delegate to chosen agent
      const agent = this.agents.get(decision.agent);
      if (!agent) {
        console.error(`Unknown agent: ${decision.agent}`);
        continue;
      }

      console.log(`📋 Delegating to ${decision.agent}: ${decision.instruction}`);
      const result = await agent.process(decision.instruction, this.results);
      this.results[`${decision.agent}_round${round}`] = result;

      console.log(`✅ ${decision.agent} completed`);
    }

    return this.results;
  }

  private async decide(
    originalTask: string
  ): Promise<{ action: 'delegate' | 'done'; agent: string; instruction: string }> {
    const summaryOfWork = Object.entries(this.results)
      .map(([key, val]) => `### ${key}:\n${val.slice(0, 500)}...`)
      .join('\n\n');

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      system: `You are a project supervisor managing a team of AI agents:
- researcher: Gathers information and data
- writer: Creates written content
- reviewer: Reviews and critiques work
- publisher: Formats and publishes content

Decide which agent should work next, or if the task is complete.
Respond in JSON: {"action": "delegate"|"done", "agent": "name", "instruction": "what to do"}`,
      messages: [
        {
          role: 'user',
          content: `Original task: ${originalTask}\n\nWork completed so far:\n${summaryOfWork || 'None yet'}`,
        },
      ],
    });

    const text = response.content[0].type === 'text' ? response.content[0].text : '';
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      return { action: 'done', agent: '', instruction: '' };
    }

    return JSON.parse(jsonMatch[0]);
  }
}
```

**3. Test both patterns (60 min)**

Create `lib/orchestrator/test-pipeline.ts`:
```typescript
import { PipelineOrchestrator } from './pipeline';

async function test() {
  const pipeline = new PipelineOrchestrator();
  const result = await pipeline.run(
    'The impact of AI agents on software development workflows in 2026'
  );

  console.log('\n=== Pipeline Complete ===');
  console.log('Status:', result.status);
  console.log('Agents used:', Object.keys(result.results));
  console.log('\nPublished content preview:');
  console.log(result.results.published?.slice(0, 500));
}

test();
```

### Day 6-8 Checkpoints
- [ ] Pipeline orchestrator runs all agents sequentially
- [ ] Supervisor orchestrator makes dynamic decisions
- [ ] Agents pass context to each other
- [ ] Review → revision loop works

---

## Day 9-11: Router & Advanced Patterns (4-6 hours)

### Goal
Build a task router that automatically selects the right agent for incoming requests.

### Steps

**1. Task Router (90 min)**

Create `lib/orchestrator/router.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { MessageBus } from '../shared/message-bus';
import { BaseAgent } from '../agents/base-agent';

const client = new Anthropic();

export class TaskRouter {
  private bus: MessageBus;
  private agents: Map<string, { agent: BaseAgent; description: string }>;

  constructor() {
    this.bus = new MessageBus();
    this.agents = new Map();
  }

  register(name: string, agent: BaseAgent, description: string) {
    this.agents.set(name, { agent, description });
  }

  async route(userRequest: string): Promise<{ agent: string; result: string }> {
    // Use LLM to classify the request
    const agentList = Array.from(this.agents.entries())
      .map(([name, { description }]) => `- ${name}: ${description}`)
      .join('\n');

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 256,
      system: `You are a task router. Given a user request, choose the best agent to handle it.
Available agents:\n${agentList}\n\nRespond with JSON: {"agent": "name", "task": "refined task for the agent"}`,
      messages: [{ role: 'user', content: userRequest }],
    });

    const text = response.content[0].type === 'text' ? response.content[0].text : '';
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (!jsonMatch) throw new Error('Failed to route task');

    const { agent: agentName, task } = JSON.parse(jsonMatch[0]);
    const entry = this.agents.get(agentName);
    if (!entry) throw new Error(`Unknown agent: ${agentName}`);

    const result = await entry.agent.process(task);
    return { agent: agentName, result };
  }
}
```

**2. Debate Pattern (60 min)**

Two agents argue different perspectives, then a judge decides.

Create `lib/orchestrator/debate.ts`:
```typescript
import { BaseAgent } from '../agents/base-agent';
import { MessageBus } from '../shared/message-bus';

export class DebateOrchestrator {
  private bus: MessageBus;

  constructor() {
    this.bus = new MessageBus();
  }

  async debate(
    topic: string,
    rounds: number = 3
  ): Promise<{ rounds: { pro: string; con: string }[]; verdict: string }> {
    const proAgent = new BaseAgent(
      {
        name: 'pro',
        description: 'Argues in favor',
        systemPrompt: `You argue IN FAVOR of the given position. Be persuasive, use evidence, and directly counter opposing arguments. Keep responses focused and under 300 words.`,
      },
      this.bus
    );

    const conAgent = new BaseAgent(
      {
        name: 'con',
        description: 'Argues against',
        systemPrompt: `You argue AGAINST the given position. Be persuasive, use evidence, and directly counter opposing arguments. Keep responses focused and under 300 words.`,
      },
      this.bus
    );

    const judge = new BaseAgent(
      {
        name: 'judge',
        description: 'Judges the debate',
        systemPrompt: `You are an impartial judge. Evaluate both sides of the debate fairly. Consider: strength of arguments, use of evidence, logical consistency, and persuasiveness. Give a clear verdict with reasoning.`,
      },
      this.bus
    );

    const debateRounds: { pro: string; con: string }[] = [];
    let proArg = '';
    let conArg = '';

    for (let i = 0; i < rounds; i++) {
      console.log(`\n--- Debate Round ${i + 1} ---`);

      proArg = await proAgent.process(
        i === 0
          ? `Argue in favor of: ${topic}`
          : `Continue arguing in favor of: ${topic}. Counter this opposing argument: ${conArg}`,
        i > 0 ? { previousRounds: JSON.stringify(debateRounds) } : undefined
      );

      conArg = await conAgent.process(
        `Argue against: ${topic}. Counter this argument: ${proArg}`,
        { previousRounds: JSON.stringify(debateRounds) }
      );

      debateRounds.push({ pro: proArg, con: conArg });
    }

    // Judge decides
    const verdict = await judge.process(
      `Judge this debate on: "${topic}"`,
      { fullDebate: JSON.stringify(debateRounds) }
    );

    return { rounds: debateRounds, verdict };
  }
}
```

### Day 9-11 Checkpoints
- [ ] Task router classifies and delegates requests
- [ ] Debate pattern produces structured arguments
- [ ] Understand when to use each pattern

---

## Day 12-14: API & UI (4-6 hours)

### Goal
Build the API and a simple UI to visualize multi-agent workflows.

### Steps

**1. API Route (60 min)**

Create `app/api/pipeline/route.ts`:
```typescript
import { PipelineOrchestrator } from '@/lib/orchestrator/pipeline';
import { SupervisorOrchestrator } from '@/lib/orchestrator/supervisor';

export async function POST(req: Request) {
  const { topic, mode } = await req.json();

  if (mode === 'supervisor') {
    const supervisor = new SupervisorOrchestrator();
    const results = await supervisor.run(topic);
    return Response.json({ mode: 'supervisor', results });
  }

  // Default: pipeline mode
  const pipeline = new PipelineOrchestrator();
  const state = await pipeline.run(topic);
  return Response.json({ mode: 'pipeline', state });
}
```

**2. Pipeline Visualization UI (90 min)**

Build a page that shows:
- Which agent is currently active
- Messages between agents
- Results from each step
- Final output

**3. Compare patterns (30 min)**

Run the same task through both pipeline and supervisor modes. Compare:
- Quality of output
- Number of LLM calls
- Total tokens used
- Time to complete

---

## Final Checkpoints

- [ ] Pipeline orchestrator: sequential agent execution
- [ ] Supervisor orchestrator: dynamic agent delegation
- [ ] Task router: automatic agent selection
- [ ] Debate pattern: multi-perspective analysis
- [ ] Agents communicate via message bus
- [ ] Context passes correctly between agents
- [ ] UI visualizes the multi-agent workflow

## Study Resources

- [ ] Read: [Anthropic Multi-Agent Patterns](https://docs.anthropic.com/en/docs/build-with-claude/agent-patterns)
- [ ] Read: [LangGraph Multi-Agent](https://langchain-ai.github.io/langgraph/)
- [ ] Study: [CrewAI](https://github.com/joaomdmoura/crewAI) - Python but great patterns
- [ ] Study: [AutoGen](https://github.com/microsoft/autogen) - Microsoft's multi-agent framework

## Interview Prep

- Explain: "What are different multi-agent orchestration patterns?"
- Explain: "When would you use a supervisor vs a pipeline?"
- Design: "How would you build a multi-agent system for [X]?"
- Trade-offs: "What are the costs and risks of multi-agent systems?"
