# Level 5: AI Agents (Week 14-16)

**Time:** 18-24 hours | **Status:** 🔲 Not Started | **Priority:** HIGH

[Back to Blueprint](./README.md) | Prev: [Level 4](./roadmap-level-4-rag.md) | Next: [Level 6](./roadmap-level-6-multi-agent.md)

---

## Why This Matters

AI agents are the core of what you're aiming for. An agent is an LLM that can reason, plan, and take actions autonomously. This is the skill that separates "I can call an API" from "I can build AI products."

## Concepts to Master

- ReAct pattern (Reason + Act)
- Agent loops and decision-making
- Tool selection and execution
- Planning and task decomposition
- Memory (short-term and long-term)
- Error recovery and self-correction
- Stopping conditions

## Project: Task-Completing Agent

Build an agent that can break down complex tasks, select tools, and execute multi-step plans.

```
apps/level-5-agent/
├── app/
│   ├── api/
│   │   ├── agent/route.ts             # Start agent task
│   │   └── agent/[id]/route.ts        # Get agent status
│   ├── page.tsx                        # Agent UI
│   └── components/
│       ├── AgentChat.tsx               # Chat with agent
│       ├── ThoughtProcess.tsx          # Show reasoning
│       └── ToolExecution.tsx           # Show tool calls
├── lib/
│   ├── agent/
│   │   ├── core.ts                    # Agent loop
│   │   ├── planner.ts                 # Task planning
│   │   ├── memory.ts                  # Agent memory
│   │   └── types.ts                   # Agent types
│   └── tools/
│       ├── registry.ts                # Tool registry
│       ├── web-search.ts              # Web search tool
│       ├── calculator.ts              # Calculator tool
│       ├── file-reader.ts             # File reading tool
│       ├── code-runner.ts             # Code execution tool
│       └── summarizer.ts              # Text summarization tool
```

---

## Day 1-2: ReAct Pattern (4-6 hours)

### Goal
Implement the ReAct (Reason + Act) pattern - the foundation of all modern agents.

### Steps

**1. Define agent types (30 min)**

Create `lib/agent/types.ts`:
```typescript
export interface Tool {
  name: string;
  description: string;
  parameters: Record<string, {
    type: string;
    description: string;
    required?: boolean;
  }>;
  execute: (params: Record<string, any>) => Promise<string>;
}

export interface AgentStep {
  type: 'thought' | 'action' | 'observation' | 'answer';
  content: string;
  toolName?: string;
  toolInput?: Record<string, any>;
  toolOutput?: string;
  timestamp: Date;
}

export interface AgentState {
  taskId: string;
  task: string;
  steps: AgentStep[];
  status: 'thinking' | 'acting' | 'done' | 'error';
  finalAnswer?: string;
  maxSteps: number;
  currentStep: number;
}

export interface AgentConfig {
  model?: string;
  maxSteps?: number;
  tools: Tool[];
  systemPrompt?: string;
  verbose?: boolean;
}
```

**2. Build the agent loop (90 min)**

Create `lib/agent/core.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { Tool, AgentState, AgentStep, AgentConfig } from './types';

const client = new Anthropic();

export class Agent {
  private config: AgentConfig;
  private state: AgentState;

  constructor(config: AgentConfig) {
    this.config = {
      model: 'claude-sonnet-4-20250514',
      maxSteps: 10,
      ...config,
    };
    this.state = {
      taskId: crypto.randomUUID(),
      task: '',
      steps: [],
      status: 'thinking',
      maxSteps: this.config.maxSteps!,
      currentStep: 0,
    };
  }

  async run(task: string): Promise<AgentState> {
    this.state.task = task;

    while (
      this.state.currentStep < this.state.maxSteps &&
      this.state.status !== 'done' &&
      this.state.status !== 'error'
    ) {
      try {
        await this.step();
        this.state.currentStep++;
      } catch (error) {
        this.state.status = 'error';
        this.state.steps.push({
          type: 'thought',
          content: `Error: ${error instanceof Error ? error.message : String(error)}`,
          timestamp: new Date(),
        });
      }
    }

    return this.state;
  }

  private async step() {
    // Build messages from history
    const messages = this.buildMessages();

    // Call LLM
    const response = await client.messages.create({
      model: this.config.model!,
      max_tokens: 2048,
      system: this.buildSystemPrompt(),
      messages,
      tools: this.buildToolDefinitions(),
    });

    // Process response
    for (const block of response.content) {
      if (block.type === 'text') {
        // Agent is thinking or giving final answer
        const text = block.text;

        if (response.stop_reason === 'end_turn') {
          // Final answer
          this.state.steps.push({
            type: 'answer',
            content: text,
            timestamp: new Date(),
          });
          this.state.finalAnswer = text;
          this.state.status = 'done';
        } else {
          // Thinking
          this.state.steps.push({
            type: 'thought',
            content: text,
            timestamp: new Date(),
          });
        }
      } else if (block.type === 'tool_use') {
        // Agent wants to use a tool
        this.state.status = 'acting';

        this.state.steps.push({
          type: 'action',
          content: `Using tool: ${block.name}`,
          toolName: block.name,
          toolInput: block.input as Record<string, any>,
          timestamp: new Date(),
        });

        // Execute tool
        const tool = this.config.tools.find((t) => t.name === block.name);
        if (!tool) {
          this.state.steps.push({
            type: 'observation',
            content: `Error: Tool "${block.name}" not found`,
            timestamp: new Date(),
          });
          continue;
        }

        try {
          const result = await tool.execute(block.input as Record<string, any>);
          this.state.steps.push({
            type: 'observation',
            content: result,
            toolName: block.name,
            toolOutput: result,
            timestamp: new Date(),
          });
        } catch (error) {
          this.state.steps.push({
            type: 'observation',
            content: `Tool error: ${error instanceof Error ? error.message : String(error)}`,
            timestamp: new Date(),
          });
        }

        this.state.status = 'thinking';
      }
    }
  }

  private buildSystemPrompt(): string {
    const toolDescriptions = this.config.tools
      .map((t) => `- ${t.name}: ${t.description}`)
      .join('\n');

    return `${this.config.systemPrompt || 'You are a helpful AI agent that can use tools to complete tasks.'}

You have access to the following tools:
${toolDescriptions}

When you have enough information to answer the user's question, provide your final answer directly.
Think step by step. Use tools when you need more information.
If a tool fails, try a different approach.`;
  }

  private buildMessages(): Anthropic.MessageParam[] {
    const messages: Anthropic.MessageParam[] = [
      { role: 'user', content: this.state.task },
    ];

    // Add tool use history
    for (const step of this.state.steps) {
      if (step.type === 'action' && step.toolName) {
        messages.push({
          role: 'assistant',
          content: [
            {
              type: 'tool_use',
              id: `tool_${step.timestamp.getTime()}`,
              name: step.toolName,
              input: step.toolInput || {},
            },
          ],
        });
      } else if (step.type === 'observation') {
        messages.push({
          role: 'user',
          content: [
            {
              type: 'tool_result',
              tool_use_id: `tool_${this.findMatchingAction(step)?.timestamp.getTime()}`,
              content: step.content,
            },
          ],
        });
      }
    }

    return messages;
  }

  private findMatchingAction(observation: AgentStep): AgentStep | undefined {
    const idx = this.state.steps.indexOf(observation);
    for (let i = idx - 1; i >= 0; i--) {
      if (this.state.steps[i].type === 'action') {
        return this.state.steps[i];
      }
    }
    return undefined;
  }

  private buildToolDefinitions(): Anthropic.Tool[] {
    return this.config.tools.map((tool) => ({
      name: tool.name,
      description: tool.description,
      input_schema: {
        type: 'object' as const,
        properties: Object.fromEntries(
          Object.entries(tool.parameters).map(([key, val]) => [
            key,
            { type: val.type, description: val.description },
          ])
        ),
        required: Object.entries(tool.parameters)
          .filter(([, val]) => val.required)
          .map(([key]) => key),
      },
    }));
  }

  getState(): AgentState {
    return { ...this.state };
  }
}
```

### Day 1-2 Checkpoints
- [ ] Agent types defined
- [ ] ReAct loop implemented
- [ ] Agent can think and decide to use tools
- [ ] Tool results feed back into reasoning

---

## Day 3-4: Build Tools (4-6 hours)

### Goal
Create a set of useful tools the agent can use.

### Steps

**1. Tool registry (30 min)**

Create `lib/tools/registry.ts`:
```typescript
import { Tool } from '../agent/types';
import { calculatorTool } from './calculator';
import { webSearchTool } from './web-search';
import { fileReaderTool } from './file-reader';
import { codeRunnerTool } from './code-runner';
import { summarizerTool } from './summarizer';

export function getDefaultTools(): Tool[] {
  return [
    calculatorTool,
    webSearchTool,
    fileReaderTool,
    codeRunnerTool,
    summarizerTool,
  ];
}

export function getToolsByNames(names: string[]): Tool[] {
  const all = getDefaultTools();
  return all.filter((t) => names.includes(t.name));
}
```

**2. Calculator tool (20 min)**

Create `lib/tools/calculator.ts`:
```typescript
import { Tool } from '../agent/types';

export const calculatorTool: Tool = {
  name: 'calculator',
  description:
    'Evaluate mathematical expressions. Supports basic arithmetic, percentages, and common math functions.',
  parameters: {
    expression: {
      type: 'string',
      description: 'Math expression to evaluate, e.g. "2 + 2", "sqrt(16)", "15% of 200"',
      required: true,
    },
  },
  async execute(params) {
    const { expression } = params;

    try {
      // Simple safe eval using Function constructor
      // Only allow math operations
      const sanitized = expression
        .replace(/sqrt/g, 'Math.sqrt')
        .replace(/pow/g, 'Math.pow')
        .replace(/abs/g, 'Math.abs')
        .replace(/round/g, 'Math.round')
        .replace(/ceil/g, 'Math.ceil')
        .replace(/floor/g, 'Math.floor')
        .replace(/(\d+)%\s*of\s*(\d+)/g, '($1/100)*$2');

      // Validate - only allow numbers, operators, Math functions, parentheses
      if (!/^[\d\s+\-*/().%Math,sqrtpowabsroundceilfloor]+$/.test(sanitized)) {
        return `Error: Invalid expression "${expression}"`;
      }

      const result = new Function(`return ${sanitized}`)();
      return `${expression} = ${result}`;
    } catch (error) {
      return `Error evaluating "${expression}": ${error}`;
    }
  },
};
```

**3. Web search tool (30 min)**

Create `lib/tools/web-search.ts`:
```typescript
import { Tool } from '../agent/types';

export const webSearchTool: Tool = {
  name: 'web_search',
  description: 'Search the web for current information. Returns top results with titles and snippets.',
  parameters: {
    query: {
      type: 'string',
      description: 'Search query',
      required: true,
    },
  },
  async execute(params) {
    const { query } = params;

    // Using Serper API (free tier available)
    // Sign up at https://serper.dev
    const response = await fetch('https://google.serper.dev/search', {
      method: 'POST',
      headers: {
        'X-API-KEY': process.env.SERPER_API_KEY || '',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ q: query, num: 5 }),
    });

    if (!response.ok) {
      return `Search failed: ${response.statusText}`;
    }

    const data = await response.json();
    const results = (data.organic || [])
      .slice(0, 5)
      .map(
        (r: any, i: number) =>
          `${i + 1}. ${r.title}\n   ${r.snippet}\n   URL: ${r.link}`
      )
      .join('\n\n');

    return results || 'No results found';
  },
};
```

**4. Code runner tool (45 min)**

Create `lib/tools/code-runner.ts`:
```typescript
import { Tool } from '../agent/types';

export const codeRunnerTool: Tool = {
  name: 'run_code',
  description: 'Execute JavaScript/TypeScript code and return the result. Use for calculations, data processing, or testing logic.',
  parameters: {
    code: {
      type: 'string',
      description: 'JavaScript code to execute. Must return a value or use console.log.',
      required: true,
    },
  },
  async execute(params) {
    const { code } = params;

    try {
      // Create a sandboxed execution environment
      const logs: string[] = [];
      const mockConsole = {
        log: (...args: any[]) => logs.push(args.map(String).join(' ')),
      };

      const fn = new Function('console', `
        'use strict';
        ${code}
      `);

      const result = fn(mockConsole);

      const output = logs.length > 0 ? logs.join('\n') : String(result);
      return `Output:\n${output}`;
    } catch (error) {
      return `Error: ${error instanceof Error ? error.message : String(error)}`;
    }
  },
};
```

**5. Summarizer tool (30 min)**

Create `lib/tools/summarizer.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { Tool } from '../agent/types';

const client = new Anthropic();

export const summarizerTool: Tool = {
  name: 'summarize',
  description: 'Summarize a long piece of text into key points.',
  parameters: {
    text: {
      type: 'string',
      description: 'Text to summarize',
      required: true,
    },
    style: {
      type: 'string',
      description: 'Summary style: "brief", "detailed", or "bullet_points"',
    },
  },
  async execute(params) {
    const { text, style = 'brief' } = params;

    const styleInstructions = {
      brief: 'Provide a 2-3 sentence summary.',
      detailed: 'Provide a detailed summary covering all key points.',
      bullet_points: 'Summarize as bullet points.',
    };

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [
        {
          role: 'user',
          content: `${styleInstructions[style as keyof typeof styleInstructions] || styleInstructions.brief}\n\nText to summarize:\n${text}`,
        },
      ],
    });

    return response.content[0].type === 'text' ? response.content[0].text : 'Failed to summarize';
  },
};
```

### Day 3-4 Checkpoints
- [ ] 5 tools implemented and working
- [ ] Tool registry for easy management
- [ ] Agent can select and use tools
- [ ] Tool errors handled gracefully

---

## Day 5-7: Planning & Memory (4-6 hours)

### Goal
Add task planning and memory so the agent can handle complex multi-step tasks.

### Steps

**1. Task planner (60 min)**

Create `lib/agent/planner.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

export interface Plan {
  goal: string;
  steps: PlanStep[];
}

export interface PlanStep {
  id: number;
  description: string;
  toolHint?: string; // Suggested tool to use
  dependsOn?: number[]; // Step IDs this depends on
}

export async function createPlan(
  task: string,
  availableTools: string[]
): Promise<Plan> {
  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [
      {
        role: 'user',
        content: `Break down this task into steps. Available tools: ${availableTools.join(', ')}

Task: ${task}

Respond in JSON format:
{
  "goal": "high-level goal",
  "steps": [
    {
      "id": 1,
      "description": "what to do",
      "toolHint": "suggested tool name or null",
      "dependsOn": []
    }
  ]
}`,
      },
    ],
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '';

  // Extract JSON from response
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (!jsonMatch) throw new Error('Failed to parse plan');

  return JSON.parse(jsonMatch[0]);
}
```

**2. Agent memory (60 min)**

Create `lib/agent/memory.ts`:
```typescript
export interface MemoryEntry {
  key: string;
  value: string;
  type: 'fact' | 'result' | 'error' | 'preference';
  timestamp: Date;
}

export class AgentMemory {
  private shortTerm: MemoryEntry[] = [];
  private longTerm: Map<string, MemoryEntry> = new Map();

  // Short-term: current task context
  addShortTerm(entry: Omit<MemoryEntry, 'timestamp'>) {
    this.shortTerm.push({ ...entry, timestamp: new Date() });

    // Keep last 50 entries
    if (this.shortTerm.length > 50) {
      this.shortTerm = this.shortTerm.slice(-50);
    }
  }

  // Long-term: persisted knowledge
  addLongTerm(entry: Omit<MemoryEntry, 'timestamp'>) {
    this.longTerm.set(entry.key, { ...entry, timestamp: new Date() });
  }

  getShortTerm(): MemoryEntry[] {
    return [...this.shortTerm];
  }

  getLongTerm(key: string): MemoryEntry | undefined {
    return this.longTerm.get(key);
  }

  searchLongTerm(query: string): MemoryEntry[] {
    const results: MemoryEntry[] = [];
    const queryLower = query.toLowerCase();

    for (const entry of this.longTerm.values()) {
      if (
        entry.key.toLowerCase().includes(queryLower) ||
        entry.value.toLowerCase().includes(queryLower)
      ) {
        results.push(entry);
      }
    }

    return results;
  }

  // Format memory for LLM context
  toContext(): string {
    const shortTermStr = this.shortTerm
      .slice(-10)
      .map((e) => `[${e.type}] ${e.key}: ${e.value}`)
      .join('\n');

    const longTermStr = Array.from(this.longTerm.values())
      .slice(-10)
      .map((e) => `[${e.type}] ${e.key}: ${e.value}`)
      .join('\n');

    return `## Recent Context\n${shortTermStr}\n\n## Known Facts\n${longTermStr}`;
  }

  clear() {
    this.shortTerm = [];
  }
}
```

**3. Test the full agent (60 min)**

Create `lib/agent/test-agent.ts`:
```typescript
import { Agent } from './core';
import { getDefaultTools } from '../tools/registry';

async function testAgent() {
  const agent = new Agent({
    tools: getDefaultTools(),
    verbose: true,
  });

  console.log('--- Test 1: Simple calculation ---');
  const result1 = await agent.run(
    'What is 15% of 2847, rounded to the nearest whole number?'
  );
  console.log('Answer:', result1.finalAnswer);
  console.log('Steps:', result1.steps.length);

  console.log('\n--- Test 2: Multi-step research ---');
  const agent2 = new Agent({
    tools: getDefaultTools(),
    verbose: true,
  });
  const result2 = await agent2.run(
    'Search for the latest news about AI agents and summarize the top 3 findings.'
  );
  console.log('Answer:', result2.finalAnswer);
  console.log('Steps:', result2.steps.length);
}

testAgent();
```

### Day 5-7 Checkpoints
- [ ] Task planner breaks down complex tasks
- [ ] Memory stores and retrieves context
- [ ] Agent handles multi-step tasks
- [ ] Error recovery works (retries different approach)

---

## Day 8-10: Agent UI & API (4-6 hours)

### Goal
Build a web interface that shows the agent's thought process in real-time.

### Steps

**1. Agent API route (60 min)**

Create `app/api/agent/route.ts`:
```typescript
import { Agent } from '@/lib/agent/core';
import { getDefaultTools } from '@/lib/tools/registry';

export async function POST(req: Request) {
  const { task } = await req.json();

  const agent = new Agent({
    tools: getDefaultTools(),
    maxSteps: 15,
  });

  const result = await agent.run(task);

  return Response.json({
    taskId: result.taskId,
    status: result.status,
    steps: result.steps.map((s) => ({
      type: s.type,
      content: s.content,
      toolName: s.toolName,
      toolInput: s.toolInput,
      toolOutput: s.toolOutput,
    })),
    finalAnswer: result.finalAnswer,
  });
}
```

**2. Agent chat UI (90 min)**

Build `app/page.tsx` with:
- Task input form
- Real-time step display showing:
  - Thought bubbles (reasoning)
  - Tool calls (action + input)
  - Tool results (observation)
  - Final answer
- Collapsible step details
- Loading states per step

**3. Thought process visualization (60 min)**

Build `app/components/ThoughtProcess.tsx`:
- Color-coded steps (blue=thought, green=action, yellow=observation, purple=answer)
- Expandable tool inputs/outputs
- Step timing display
- Progress indicator

### Day 8-10 Checkpoints
- [ ] Agent accessible via API
- [ ] UI shows real-time thought process
- [ ] Tool calls visible with inputs/outputs
- [ ] Final answer displayed clearly

---

## Final Checkpoints

- [ ] ReAct pattern implemented and working
- [ ] 5+ tools available to agent
- [ ] Task planning for complex tasks
- [ ] Memory system (short-term + long-term)
- [ ] Web UI with thought process visualization
- [ ] Agent handles errors and retries
- [ ] Can complete multi-step tasks autonomously

## Study Resources

- [ ] Read: [Anthropic Tool Use Docs](https://docs.anthropic.com/en/docs/tool-use)
- [ ] Read: [ReAct Paper](https://arxiv.org/abs/2210.03629)
- [ ] Study: [LangChain Agents](https://js.langchain.com/docs/modules/agents/)
- [ ] Reverse-engineer: [Claude Code](https://github.com/anthropics/claude-code) (agent architecture)

## Interview Prep

- Explain: "What is the ReAct pattern?"
- Explain: "How does an agent decide which tool to use?"
- Explain: "How do you prevent infinite loops in agents?"
- Design: "Design an agent that can complete a multi-step research task"
- Code: Implement a simple agent loop from scratch
