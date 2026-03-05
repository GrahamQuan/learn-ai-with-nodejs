# Level 3: API Integration Workflows (Week 8-10)

**Time:** 18-24 hours | **Status:** 🔲 Not Started | **Priority:** HIGH

[Back to Blueprint](./README.md) | Prev: [Level 2](./roadmap-level-2-browser.md) | Next: [Level 4](./roadmap-level-4-rag.md)

---

## Why This Matters

Zapier/n8n-style API workflows are the bread and butter of workflow automation. Companies need engineers who can orchestrate multiple APIs, handle data transformation, and build reliable execution engines.

## Concepts to Master

- API integration patterns
- Data transformation & mapping
- Webhook handling
- OAuth & authentication flows
- Conditional logic & branching
- Error handling & retries
- Workflow execution engines

## Project: API Workflow Builder

Build a mini Zapier/n8n - a visual workflow builder that connects APIs.

```
apps/level-3-api-workflows/
├── app/
│   ├── workflows/
│   │   ├── page.tsx                    # Workflow list
│   │   ├── [id]/page.tsx              # Workflow editor
│   │   └── [id]/runs/page.tsx         # Execution history
│   ├── api/
│   │   ├── workflows/route.ts         # CRUD workflows
│   │   ├── execute/route.ts           # Execute workflow
│   │   └── webhooks/[id]/route.ts     # Webhook triggers
├── lib/
│   ├── workflow/
│   │   ├── engine.ts                  # Workflow execution engine
│   │   ├── nodes/
│   │   │   ├── trigger.ts             # Trigger nodes
│   │   │   ├── action.ts             # Action nodes
│   │   │   ├── condition.ts          # Conditional nodes
│   │   │   └── transform.ts          # Data transform nodes
│   │   └── executor.ts               # Node executor
│   ├── integrations/
│   │   ├── github.ts                  # GitHub API
│   │   ├── slack.ts                   # Slack API
│   │   ├── notion.ts                  # Notion API
│   │   └── openai.ts                 # OpenAI API
│   └── db/
│       └── schema.ts                  # Workflow storage
```

---

## Day 1-2: Core Workflow Engine (4-6 hours)

### Goal
Build the execution engine that runs multi-step workflows.

### Steps

**1. Define workflow types (30 min)**

Create `lib/workflow/types.ts`:
```typescript
export interface WorkflowNode {
  id: string;
  type: 'trigger' | 'action' | 'condition' | 'transform' | 'llm';
  name: string;
  config: Record<string, any>;
}

export interface WorkflowEdge {
  from: string;
  to: string;
  condition?: string; // For conditional branches
}

export interface Workflow {
  id: string;
  name: string;
  nodes: WorkflowNode[];
  edges: WorkflowEdge[];
  createdAt: Date;
  updatedAt: Date;
}

export interface ExecutionContext {
  workflowId: string;
  runId: string;
  data: Record<string, any>;
  variables: Record<string, any>;
  logs: ExecutionLog[];
}

export interface ExecutionLog {
  nodeId: string;
  status: 'running' | 'completed' | 'failed' | 'skipped';
  input: any;
  output: any;
  error?: string;
  startedAt: Date;
  completedAt?: Date;
}
```

**2. Build the engine (90 min)**

Create `lib/workflow/engine.ts`:
```typescript
import { Workflow, WorkflowNode, ExecutionContext, ExecutionLog } from './types';
import { executeAction } from './nodes/action';
import { evaluateCondition } from './nodes/condition';
import { transformData } from './nodes/transform';
import { callLLM } from './nodes/llm';

export class WorkflowEngine {
  async execute(workflow: Workflow, triggerData: any): Promise<ExecutionContext> {
    const context: ExecutionContext = {
      workflowId: workflow.id,
      runId: crypto.randomUUID(),
      data: triggerData,
      variables: {},
      logs: [],
    };

    // Find trigger node (entry point)
    const triggerNode = workflow.nodes.find((n) => n.type === 'trigger');
    if (!triggerNode) throw new Error('No trigger node found');

    // Start execution from trigger
    await this.executeNode(triggerNode, workflow, context);

    return context;
  }

  private async executeNode(
    node: WorkflowNode,
    workflow: Workflow,
    context: ExecutionContext
  ) {
    const log: ExecutionLog = {
      nodeId: node.id,
      status: 'running',
      input: context.data,
      output: null,
      startedAt: new Date(),
    };

    try {
      // Execute based on node type
      const result = await this.runNode(node, context);
      log.output = result;
      log.status = 'completed';
      log.completedAt = new Date();
      context.logs.push(log);

      // Update context data with result
      context.data = result;

      // Find and execute next nodes
      const nextEdges = workflow.edges.filter((e) => e.from === node.id);

      for (const edge of nextEdges) {
        // Check condition if present
        if (edge.condition) {
          const shouldContinue = evaluateCondition(edge.condition, context);
          if (!shouldContinue) continue;
        }

        const nextNode = workflow.nodes.find((n) => n.id === edge.to);
        if (nextNode) {
          await this.executeNode(nextNode, workflow, context);
        }
      }
    } catch (error) {
      log.status = 'failed';
      log.error = error instanceof Error ? error.message : String(error);
      log.completedAt = new Date();
      context.logs.push(log);
      throw error;
    }
  }

  private async runNode(node: WorkflowNode, context: ExecutionContext) {
    switch (node.type) {
      case 'trigger':
        return context.data; // Pass through trigger data
      case 'action':
        return await executeAction(node, context);
      case 'condition':
        return context.data; // Conditions are handled in edges
      case 'transform':
        return transformData(node, context);
      case 'llm':
        return await callLLM(node, context);
      default:
        throw new Error(`Unknown node type: ${node.type}`);
    }
  }
}
```

**3. Implement node types (60 min)**

Create `lib/workflow/nodes/action.ts`:
```typescript
import { WorkflowNode, ExecutionContext } from '../types';

export async function executeAction(
  node: WorkflowNode,
  context: ExecutionContext
) {
  const { integration, method, params } = node.config;

  // Replace template variables in params
  const resolvedParams = resolveTemplates(params, context);

  switch (integration) {
    case 'http':
      return await httpRequest(resolvedParams);
    case 'github':
      return await githubAction(method, resolvedParams);
    case 'slack':
      return await slackAction(method, resolvedParams);
    default:
      throw new Error(`Unknown integration: ${integration}`);
  }
}

async function httpRequest(params: {
  url: string;
  method: string;
  headers?: Record<string, string>;
  body?: any;
}) {
  const response = await fetch(params.url, {
    method: params.method || 'GET',
    headers: {
      'Content-Type': 'application/json',
      ...params.headers,
    },
    body: params.body ? JSON.stringify(params.body) : undefined,
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return await response.json();
}

async function githubAction(method: string, params: any) {
  const token = process.env.GITHUB_TOKEN;
  if (!token) throw new Error('GITHUB_TOKEN not set');

  const baseUrl = 'https://api.github.com';
  const headers = {
    Authorization: `Bearer ${token}`,
    Accept: 'application/vnd.github.v3+json',
  };

  switch (method) {
    case 'createIssue':
      const res = await fetch(`${baseUrl}/repos/${params.repo}/issues`, {
        method: 'POST',
        headers,
        body: JSON.stringify({
          title: params.title,
          body: params.body,
          labels: params.labels,
        }),
      });
      return await res.json();

    case 'listIssues':
      const listRes = await fetch(
        `${baseUrl}/repos/${params.repo}/issues?state=${params.state || 'open'}`,
        { headers }
      );
      return await listRes.json();

    default:
      throw new Error(`Unknown GitHub method: ${method}`);
  }
}

async function slackAction(method: string, params: any) {
  const token = process.env.SLACK_TOKEN;
  if (!token) throw new Error('SLACK_TOKEN not set');

  switch (method) {
    case 'sendMessage':
      const res = await fetch('https://slack.com/api/chat.postMessage', {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          channel: params.channel,
          text: params.text,
        }),
      });
      return await res.json();

    default:
      throw new Error(`Unknown Slack method: ${method}`);
  }
}

// Replace {{variable}} templates with context values
function resolveTemplates(obj: any, context: ExecutionContext): any {
  if (typeof obj === 'string') {
    return obj.replace(/\{\{(\w+(?:\.\w+)*)\}\}/g, (_, path) => {
      const parts = path.split('.');
      let value: any = { ...context.data, ...context.variables };
      for (const part of parts) {
        value = value?.[part];
      }
      return value ?? '';
    });
  }
  if (Array.isArray(obj)) return obj.map((item) => resolveTemplates(item, context));
  if (typeof obj === 'object' && obj !== null) {
    return Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, resolveTemplates(v, context)])
    );
  }
  return obj;
}
```

Create `lib/workflow/nodes/transform.ts`:
```typescript
import { WorkflowNode, ExecutionContext } from '../types';

export function transformData(node: WorkflowNode, context: ExecutionContext) {
  const { operation, config } = node.config;

  switch (operation) {
    case 'pick':
      // Pick specific fields
      return pick(context.data, config.fields);

    case 'map':
      // Map array items
      if (!Array.isArray(context.data)) throw new Error('Data is not an array');
      return context.data.map((item: any) => pick(item, config.fields));

    case 'filter':
      // Filter array
      if (!Array.isArray(context.data)) throw new Error('Data is not an array');
      return context.data.filter((item: any) => {
        const value = item[config.field];
        switch (config.operator) {
          case 'eq': return value === config.value;
          case 'neq': return value !== config.value;
          case 'contains': return String(value).includes(config.value);
          case 'gt': return value > config.value;
          case 'lt': return value < config.value;
          default: return true;
        }
      });

    case 'template':
      // String template
      return { result: resolveTemplate(config.template, context.data) };

    default:
      return context.data;
  }
}

function pick(obj: any, fields: string[]) {
  return Object.fromEntries(fields.map((f) => [f, obj[f]]));
}

function resolveTemplate(template: string, data: any): string {
  return template.replace(/\{\{(\w+)\}\}/g, (_, key) => data[key] ?? '');
}
```

Create `lib/workflow/nodes/condition.ts`:
```typescript
import { ExecutionContext } from '../types';

export function evaluateCondition(
  condition: string,
  context: ExecutionContext
): boolean {
  // Simple condition parser: "field operator value"
  // e.g., "status eq open", "count gt 5", "title contains bug"
  const parts = condition.split(' ');
  if (parts.length < 3) return true;

  const [field, operator, ...valueParts] = parts;
  const value = valueParts.join(' ');
  const actual = context.data[field];

  switch (operator) {
    case 'eq': return String(actual) === value;
    case 'neq': return String(actual) !== value;
    case 'gt': return Number(actual) > Number(value);
    case 'lt': return Number(actual) < Number(value);
    case 'contains': return String(actual).includes(value);
    case 'exists': return actual !== undefined && actual !== null;
    default: return true;
  }
}
```

Create `lib/workflow/nodes/llm.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { WorkflowNode, ExecutionContext } from '../types';

const client = new Anthropic();

export async function callLLM(node: WorkflowNode, context: ExecutionContext) {
  const { prompt, systemPrompt, outputFormat } = node.config;

  // Resolve template variables in prompt
  const resolvedPrompt = prompt.replace(
    /\{\{(\w+(?:\.\w+)*)\}\}/g,
    (_: string, path: string) => {
      const parts = path.split('.');
      let value: any = { ...context.data, ...context.variables };
      for (const part of parts) value = value?.[part];
      return typeof value === 'object' ? JSON.stringify(value) : String(value ?? '');
    }
  );

  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: resolvedPrompt },
  ];

  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: systemPrompt || undefined,
    messages,
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '';

  // Parse output if JSON format requested
  if (outputFormat === 'json') {
    try {
      return JSON.parse(text);
    } catch {
      return { raw: text };
    }
  }

  return { result: text, tokens: response.usage };
}
```

### Day 1-2 Checkpoints
- [ ] Workflow types defined
- [ ] Engine executes multi-step workflows
- [ ] Action, transform, condition, LLM nodes work
- [ ] Template variable resolution works

---

## Day 3-4: API Integrations (4-6 hours)

### Goal
Build real API integrations and test end-to-end workflows.

### Steps

**1. Build integration registry (60 min)**

Create `lib/integrations/registry.ts`:
```typescript
export interface Integration {
  id: string;
  name: string;
  description: string;
  methods: IntegrationMethod[];
  authType: 'api-key' | 'oauth' | 'none';
}

export interface IntegrationMethod {
  id: string;
  name: string;
  description: string;
  params: { name: string; type: string; required: boolean }[];
}

export const integrations: Integration[] = [
  {
    id: 'github',
    name: 'GitHub',
    description: 'GitHub API integration',
    authType: 'api-key',
    methods: [
      {
        id: 'createIssue',
        name: 'Create Issue',
        description: 'Create a new GitHub issue',
        params: [
          { name: 'repo', type: 'string', required: true },
          { name: 'title', type: 'string', required: true },
          { name: 'body', type: 'string', required: false },
        ],
      },
      {
        id: 'listIssues',
        name: 'List Issues',
        description: 'List repository issues',
        params: [
          { name: 'repo', type: 'string', required: true },
          { name: 'state', type: 'string', required: false },
        ],
      },
    ],
  },
  {
    id: 'slack',
    name: 'Slack',
    description: 'Slack messaging',
    authType: 'api-key',
    methods: [
      {
        id: 'sendMessage',
        name: 'Send Message',
        description: 'Send a message to a channel',
        params: [
          { name: 'channel', type: 'string', required: true },
          { name: 'text', type: 'string', required: true },
        ],
      },
    ],
  },
  {
    id: 'http',
    name: 'HTTP Request',
    description: 'Make any HTTP request',
    authType: 'none',
    methods: [
      {
        id: 'request',
        name: 'HTTP Request',
        description: 'Make an HTTP request',
        params: [
          { name: 'url', type: 'string', required: true },
          { name: 'method', type: 'string', required: true },
          { name: 'body', type: 'object', required: false },
        ],
      },
    ],
  },
];
```

**2. Test a real workflow (60 min)**

Create `lib/workflow/test-workflow.ts`:
```typescript
import { WorkflowEngine } from './engine';
import { Workflow } from './types';

const testWorkflow: Workflow = {
  id: 'test-1',
  name: 'GitHub Issues → LLM Summary',
  nodes: [
    {
      id: 'trigger',
      type: 'trigger',
      name: 'Manual Trigger',
      config: {},
    },
    {
      id: 'fetch-issues',
      type: 'action',
      name: 'Fetch GitHub Issues',
      config: {
        integration: 'github',
        method: 'listIssues',
        params: { repo: '{{repo}}', state: 'open' },
      },
    },
    {
      id: 'summarize',
      type: 'llm',
      name: 'Summarize Issues',
      config: {
        prompt: 'Summarize these GitHub issues in 3 bullet points:\n\n{{data}}',
        outputFormat: 'text',
      },
    },
  ],
  edges: [
    { from: 'trigger', to: 'fetch-issues' },
    { from: 'fetch-issues', to: 'summarize' },
  ],
  createdAt: new Date(),
  updatedAt: new Date(),
};

async function test() {
  const engine = new WorkflowEngine();
  const result = await engine.execute(testWorkflow, {
    repo: 'vercel/ai',
  });

  console.log('Execution logs:');
  for (const log of result.logs) {
    console.log(`  ${log.nodeId}: ${log.status} (${log.completedAt!.getTime() - log.startedAt.getTime()}ms)`);
  }
  console.log('\nFinal result:', result.data);
}

test();
```

**3. Add webhook triggers (60 min)**

Create `app/api/webhooks/[id]/route.ts`:
```typescript
import { WorkflowEngine } from '@/lib/workflow/engine';
import { getWorkflowById } from '@/lib/db/schema';

export async function POST(
  req: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;

  try {
    const body = await req.json();
    const workflow = await getWorkflowById(id);

    if (!workflow) {
      return new Response('Workflow not found', { status: 404 });
    }

    const engine = new WorkflowEngine();
    const result = await engine.execute(workflow, body);

    return Response.json({
      runId: result.runId,
      status: 'completed',
      logs: result.logs.map((l) => ({
        node: l.nodeId,
        status: l.status,
        duration: l.completedAt
          ? l.completedAt.getTime() - l.startedAt.getTime()
          : null,
      })),
      result: result.data,
    });
  } catch (error) {
    return Response.json(
      { error: error instanceof Error ? error.message : 'Unknown error' },
      { status: 500 }
    );
  }
}
```

### Day 3-4 Checkpoints
- [ ] Integration registry defined
- [ ] Real GitHub/Slack API calls work
- [ ] Webhook triggers functional
- [ ] End-to-end workflow tested

---

## Day 5-7: Workflow Storage & API (4-6 hours)

### Goal
Persist workflows and build CRUD API routes.

### Steps

**1. Simple file-based storage (60 min)**

Create `lib/db/schema.ts`:
```typescript
import { readFile, writeFile, mkdir } from 'fs/promises';
import { join } from 'path';
import { Workflow } from '../workflow/types';

const DATA_DIR = join(process.cwd(), '.data');
const WORKFLOWS_FILE = join(DATA_DIR, 'workflows.json');

async function ensureDataDir() {
  await mkdir(DATA_DIR, { recursive: true });
}

async function readWorkflows(): Promise<Workflow[]> {
  try {
    const data = await readFile(WORKFLOWS_FILE, 'utf-8');
    return JSON.parse(data);
  } catch {
    return [];
  }
}

async function saveWorkflows(workflows: Workflow[]) {
  await ensureDataDir();
  await writeFile(WORKFLOWS_FILE, JSON.stringify(workflows, null, 2));
}

export async function getAllWorkflows(): Promise<Workflow[]> {
  return await readWorkflows();
}

export async function getWorkflowById(id: string): Promise<Workflow | null> {
  const workflows = await readWorkflows();
  return workflows.find((w) => w.id === id) || null;
}

export async function createWorkflow(
  workflow: Omit<Workflow, 'id' | 'createdAt' | 'updatedAt'>
): Promise<Workflow> {
  const workflows = await readWorkflows();
  const newWorkflow: Workflow = {
    ...workflow,
    id: crypto.randomUUID(),
    createdAt: new Date(),
    updatedAt: new Date(),
  };
  workflows.push(newWorkflow);
  await saveWorkflows(workflows);
  return newWorkflow;
}

export async function updateWorkflow(
  id: string,
  updates: Partial<Workflow>
): Promise<Workflow | null> {
  const workflows = await readWorkflows();
  const index = workflows.findIndex((w) => w.id === id);
  if (index === -1) return null;

  workflows[index] = {
    ...workflows[index],
    ...updates,
    updatedAt: new Date(),
  };
  await saveWorkflows(workflows);
  return workflows[index];
}

export async function deleteWorkflow(id: string): Promise<boolean> {
  const workflows = await readWorkflows();
  const filtered = workflows.filter((w) => w.id !== id);
  if (filtered.length === workflows.length) return false;
  await saveWorkflows(filtered);
  return true;
}
```

**2. CRUD API routes (60 min)**

Create `app/api/workflows/route.ts`:
```typescript
import { getAllWorkflows, createWorkflow } from '@/lib/db/schema';

export async function GET() {
  const workflows = await getAllWorkflows();
  return Response.json(workflows);
}

export async function POST(req: Request) {
  const body = await req.json();
  const workflow = await createWorkflow(body);
  return Response.json(workflow, { status: 201 });
}
```

### Day 5-7 Checkpoints
- [ ] Workflows persist to disk
- [ ] CRUD API routes work
- [ ] Can create, list, update, delete workflows

---

## Day 8-10: Workflow Builder UI (6-8 hours)

### Goal
Build a simple visual workflow editor.

### Steps

**1. Install React Flow (15 min)**
```bash
pnpm add @xyflow/react
```

**2. Build workflow editor page**
- Drag-and-drop node creation
- Connect nodes with edges
- Node configuration panels
- Save/load workflows

**3. Build workflow list page**
- List all workflows
- Create new workflow
- Run workflow manually
- View execution history

### Day 8-10 Checkpoints
- [ ] Visual workflow editor works
- [ ] Can create workflows visually
- [ ] Can run workflows from UI
- [ ] Execution results displayed

---

## Final Checkpoints

- [ ] Workflow engine executes multi-step workflows
- [ ] At least 3 API integrations working (GitHub, Slack, HTTP)
- [ ] LLM node processes data with Claude
- [ ] Webhook triggers functional
- [ ] Workflows persist and can be managed via API
- [ ] Basic visual editor works

## Study Resources

- [ ] Read: [n8n source code](https://github.com/n8n-io/n8n) - Study workflow execution
- [ ] Read: [Trigger.dev source](https://github.com/triggerdotdev/trigger.dev) - Study job orchestration
- [ ] Read: [Inngest Docs](https://www.inngest.com/docs) - Alternative approach
- [ ] Study: How Zapier handles webhooks and OAuth

## Interview Prep

- Explain: "How would you design a workflow execution engine?"
- Explain: "How do you handle failures in multi-step workflows?"
- Explain: "How do you implement webhook triggers?"
- Code: Build a simple workflow engine from scratch
