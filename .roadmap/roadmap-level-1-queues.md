# Level 1: Job Queues & Background Processing (Week 3-4)

**Time:** 14-18 hours | **Status:** 🔲 Not Started | **Priority:** CRITICAL

[Back to Blueprint](./README.md) | Prev: [Level 0](./roadmap-level-0-streaming.md) | Next: [Level 2](./roadmap-level-2-browser.md)

---

## Why This Matters

Workflow automation requires background jobs. Real AI workflows are long-running - you can't block an HTTP request for 30 seconds while an agent works. Every workflow platform (n8n, Zapier, Trigger.dev) uses job queues under the hood.

## Concepts to Master

- Job queues (BullMQ)
- Background workers
- Task scheduling and retries
- Event-driven architecture
- Webhook handling
- Long-running task management

## Project: Async Workflow Engine

Build a system that processes long-running AI tasks in the background.

```
apps/level-1-workflow-engine/
├── app/
│   ├── api/
│   │   ├── workflows/create/route.ts    # Create workflow
│   │   ├── workflows/[id]/route.ts      # Get status
│   │   └── webhooks/complete/route.ts   # Completion webhook
│   ├── workflows/page.tsx               # Workflow dashboard
│   └── workflows/[id]/page.tsx          # Workflow detail
├── lib/
│   ├── queue/
│   │   ├── client.ts                    # Queue client (BullMQ)
│   │   ├── workers.ts                   # Worker definitions
│   │   └── jobs.ts                      # Job types
│   ├── workflows/
│   │   ├── executor.ts                  # Workflow execution
│   │   └── steps.ts                     # Workflow steps
│   └── db/
│       └── schema.ts                    # Database schema
└── workers/
    └── workflow-worker.ts               # Background worker process
```

---

## Day 1-2: Setup BullMQ (3-4 hours)

### Goal
Get a job queue running with Redis and process your first background job.

### Steps

**1. Setup Redis (15 min)**
```bash
# Using Docker
docker run -d --name redis -p 6379:6379 redis

# Or install locally on macOS
brew install redis
brew services start redis
```

**2. Install dependencies (10 min)**
```bash
cd apps/level-1-workflow-engine
pnpm add bullmq ioredis
pnpm add -D @types/node
```

**3. Create queue client (45 min)**

Create `lib/queue/client.ts`:
```typescript
import { Queue, Worker, Job } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis({
  host: process.env.REDIS_HOST || 'localhost',
  port: Number(process.env.REDIS_PORT) || 6379,
  maxRetriesPerRequest: null,
});

export const workflowQueue = new Queue('workflows', { connection });

export async function addWorkflowJob(data: {
  name: string;
  steps: WorkflowStep[];
  input: Record<string, any>;
}) {
  return await workflowQueue.add('process-workflow', data, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: { count: 100 },
    removeOnFail: { count: 50 },
  });
}

export interface WorkflowStep {
  id: string;
  type: 'llm-call' | 'transform' | 'api-call' | 'condition';
  config: Record<string, any>;
}
```

**4. Create a simple worker (60 min)**

Create `workers/workflow-worker.ts`:
```typescript
import { Worker, Job } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis({
  host: process.env.REDIS_HOST || 'localhost',
  port: Number(process.env.REDIS_PORT) || 6379,
  maxRetriesPerRequest: null,
});

const worker = new Worker(
  'workflows',
  async (job: Job) => {
    console.log(`Processing job ${job.id}: ${job.data.name}`);

    const { steps, input } = job.data;
    let currentData = input;

    for (let i = 0; i < steps.length; i++) {
      const step = steps[i];

      // Update progress
      await job.updateProgress((i / steps.length) * 100);
      await job.log(`Executing step ${i + 1}/${steps.length}: ${step.type}`);

      // Execute step
      currentData = await executeStep(step, currentData);
    }

    await job.updateProgress(100);
    return currentData;
  },
  {
    connection,
    concurrency: 5,
  }
);

async function executeStep(
  step: { type: string; config: Record<string, any> },
  data: Record<string, any>
) {
  switch (step.type) {
    case 'llm-call':
      // Simulate LLM call for now
      console.log(`  LLM call with prompt: ${step.config.prompt}`);
      await new Promise((r) => setTimeout(r, 2000));
      return { ...data, llmResult: `Response for: ${step.config.prompt}` };

    case 'transform':
      console.log(`  Transforming data`);
      return { ...data, transformed: true };

    case 'api-call':
      console.log(`  API call to: ${step.config.url}`);
      await new Promise((r) => setTimeout(r, 1000));
      return { ...data, apiResult: 'ok' };

    default:
      return data;
  }
}

worker.on('completed', (job) => {
  console.log(`Job ${job.id} completed`);
});

worker.on('failed', (job, err) => {
  console.error(`Job ${job?.id} failed:`, err.message);
});

console.log('Worker started, waiting for jobs...');
```

**5. Test it (30 min)**

Create `lib/queue/test-queue.ts`:
```typescript
import { addWorkflowJob, workflowQueue } from './client';

async function test() {
  const job = await addWorkflowJob({
    name: 'Test Workflow',
    steps: [
      { id: '1', type: 'llm-call', config: { prompt: 'Summarize this text' } },
      { id: '2', type: 'transform', config: {} },
      { id: '3', type: 'api-call', config: { url: 'https://api.example.com' } },
    ],
    input: { text: 'Hello world' },
  });

  console.log(`Job created: ${job.id}`);

  // Poll for status
  const interval = setInterval(async () => {
    const state = await job.getState();
    const progress = job.progress;
    console.log(`Status: ${state}, Progress: ${progress}%`);

    if (state === 'completed' || state === 'failed') {
      clearInterval(interval);
      const result = await job.returnvalue;
      console.log('Result:', result);
      process.exit(0);
    }
  }, 1000);
}

test();
```

Run in two terminals:
```bash
# Terminal 1: Start worker
tsx workers/workflow-worker.ts

# Terminal 2: Submit job
tsx lib/queue/test-queue.ts
```

### Day 1-2 Checkpoints
- [ ] Redis running locally
- [ ] Can create and process jobs
- [ ] Worker handles failures and retries
- [ ] Understand job lifecycle (waiting → active → completed/failed)

---

## Day 3-4: Real LLM Integration (4-5 hours)

### Goal
Replace simulated steps with real LLM calls and build actual workflow steps.

### Steps

**1. Create LLM step executor (60 min)**

Create `lib/workflows/steps.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

export async function executeLLMStep(
  config: { prompt: string; systemPrompt?: string },
  data: Record<string, any>
) {
  // Interpolate data into prompt
  const prompt = config.prompt.replace(
    /\{\{(\w+)\}\}/g,
    (_, key) => data[key] || ''
  );

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: config.systemPrompt || 'You are a helpful assistant.',
    messages: [{ role: 'user', content: prompt }],
  });

  return response.content[0].type === 'text'
    ? response.content[0].text
    : '';
}

export async function executeTransformStep(
  config: { operation: 'extract' | 'filter' | 'map'; expression: string },
  data: Record<string, any>
) {
  switch (config.operation) {
    case 'extract':
      // Use LLM to extract structured data
      const extracted = await executeLLMStep(
        {
          prompt: `Extract the following from this text: ${config.expression}\n\nText: ${JSON.stringify(data)}`,
          systemPrompt: 'Extract data and return as JSON only.',
        },
        data
      );
      return JSON.parse(extracted);

    case 'filter':
      return Object.fromEntries(
        Object.entries(data).filter(([key]) =>
          config.expression.split(',').includes(key)
        )
      );

    case 'map':
      return data;

    default:
      return data;
  }
}

export async function executeAPIStep(
  config: { url: string; method: string; headers?: Record<string, string> },
  data: Record<string, any>
) {
  const response = await fetch(config.url, {
    method: config.method || 'GET',
    headers: {
      'Content-Type': 'application/json',
      ...config.headers,
    },
    body: config.method !== 'GET' ? JSON.stringify(data) : undefined,
  });

  if (!response.ok) {
    throw new Error(`API call failed: ${response.statusText}`);
  }

  return await response.json();
}
```

**2. Build workflow executor (60 min)**

Create `lib/workflows/executor.ts`:
```typescript
import { executeLLMStep, executeTransformStep, executeAPIStep } from './steps';
import type { WorkflowStep } from '../queue/client';

export interface ExecutionContext {
  data: Record<string, any>;
  logs: string[];
  stepResults: Record<string, any>;
}

export async function executeWorkflow(
  steps: WorkflowStep[],
  input: Record<string, any>,
  onProgress?: (step: number, total: number, log: string) => void
): Promise<ExecutionContext> {
  const context: ExecutionContext = {
    data: input,
    logs: [],
    stepResults: {},
  };

  for (let i = 0; i < steps.length; i++) {
    const step = steps[i];
    const log = `Step ${i + 1}/${steps.length}: ${step.type} (${step.id})`;
    context.logs.push(log);
    onProgress?.(i + 1, steps.length, log);

    try {
      let result: any;

      switch (step.type) {
        case 'llm-call':
          result = await executeLLMStep(step.config as any, context.data);
          break;
        case 'transform':
          result = await executeTransformStep(step.config as any, context.data);
          break;
        case 'api-call':
          result = await executeAPIStep(step.config as any, context.data);
          break;
        case 'condition':
          // Evaluate condition - skip next step if false
          break;
        default:
          throw new Error(`Unknown step type: ${step.type}`);
      }

      context.stepResults[step.id] = result;
      context.data = { ...context.data, [step.id]: result };
    } catch (error) {
      context.logs.push(`Error in step ${step.id}: ${error}`);
      throw error;
    }
  }

  return context;
}
```

**3. Update worker to use real executor (30 min)**

Update `workers/workflow-worker.ts` to import and use `executeWorkflow`.

**4. Create example workflows (60 min)**

Create `lib/workflows/examples.ts`:
```typescript
import type { WorkflowStep } from '../queue/client';

// Workflow: Summarize a URL's content
export const summarizeUrlWorkflow: WorkflowStep[] = [
  {
    id: 'fetch',
    type: 'api-call',
    config: { url: '{{url}}', method: 'GET' },
  },
  {
    id: 'summarize',
    type: 'llm-call',
    config: {
      prompt: 'Summarize this content in 3 bullet points:\n\n{{fetch}}',
    },
  },
];

// Workflow: Analyze and categorize text
export const analyzeTextWorkflow: WorkflowStep[] = [
  {
    id: 'analyze',
    type: 'llm-call',
    config: {
      prompt: 'Analyze the sentiment and key topics of this text:\n\n{{text}}',
      systemPrompt: 'Return JSON with fields: sentiment, topics[], summary',
    },
  },
  {
    id: 'extract',
    type: 'transform',
    config: { operation: 'extract', expression: 'sentiment, topics' },
  },
];

// Workflow: Multi-step content generation
export const contentPipelineWorkflow: WorkflowStep[] = [
  {
    id: 'outline',
    type: 'llm-call',
    config: {
      prompt: 'Create an outline for a blog post about: {{topic}}',
    },
  },
  {
    id: 'draft',
    type: 'llm-call',
    config: {
      prompt: 'Write a blog post based on this outline:\n\n{{outline}}',
    },
  },
  {
    id: 'review',
    type: 'llm-call',
    config: {
      prompt: 'Review this blog post for clarity and suggest improvements:\n\n{{draft}}',
      systemPrompt: 'You are an editor. Be concise and actionable.',
    },
  },
];
```

### Day 3-4 Checkpoints
- [ ] Real LLM calls work in workflow steps
- [ ] Workflow executor chains steps together
- [ ] Data flows between steps correctly
- [ ] Example workflows run end-to-end

---

## Day 5-6: API Routes & Status Tracking (4-5 hours)

### Goal
Build API routes to create workflows and track their status.

### Steps

**1. Create workflow API (60 min)**

Create `app/api/workflows/create/route.ts`:
```typescript
import { addWorkflowJob } from '@/lib/queue/client';

export async function POST(req: Request) {
  try {
    const { name, steps, input } = await req.json();

    const job = await addWorkflowJob({ name, steps, input });

    return Response.json({
      id: job.id,
      name,
      status: 'queued',
      createdAt: new Date().toISOString(),
    });
  } catch (error) {
    return Response.json(
      { error: 'Failed to create workflow' },
      { status: 500 }
    );
  }
}
```

Create `app/api/workflows/[id]/route.ts`:
```typescript
import { workflowQueue } from '@/lib/queue/client';

export async function GET(
  req: Request,
  { params }: { params: { id: string } }
) {
  try {
    const job = await workflowQueue.getJob(params.id);

    if (!job) {
      return Response.json({ error: 'Job not found' }, { status: 404 });
    }

    const state = await job.getState();
    const logs = await job.log();

    return Response.json({
      id: job.id,
      name: job.data.name,
      status: state,
      progress: job.progress,
      logs: logs.logs,
      result: state === 'completed' ? job.returnvalue : null,
      failedReason: state === 'failed' ? job.failedReason : null,
      createdAt: job.timestamp,
    });
  } catch (error) {
    return Response.json(
      { error: 'Failed to get workflow status' },
      { status: 500 }
    );
  }
}
```

**2. Create workflow list API (30 min)**

Create `app/api/workflows/route.ts`:
```typescript
import { workflowQueue } from '@/lib/queue/client';

export async function GET() {
  const [waiting, active, completed, failed] = await Promise.all([
    workflowQueue.getWaiting(0, 20),
    workflowQueue.getActive(0, 20),
    workflowQueue.getCompleted(0, 20),
    workflowQueue.getFailed(0, 20),
  ]);

  const jobs = [...waiting, ...active, ...completed, ...failed]
    .sort((a, b) => b.timestamp - a.timestamp)
    .map((job) => ({
      id: job.id,
      name: job.data.name,
      progress: job.progress,
      timestamp: job.timestamp,
    }));

  return Response.json({ jobs });
}
```

### Day 5-6 Checkpoints
- [ ] Can create workflows via API
- [ ] Can poll job status
- [ ] Can list all workflows

---

## Day 7: Dashboard UI (3-4 hours)

### Goal
Build a workflow dashboard showing active, completed, and failed jobs.

### Steps

**1. Build workflow dashboard (90 min)**

Create `app/workflows/page.tsx`:
```typescript
'use client';
import { useEffect, useState } from 'react';

interface WorkflowJob {
  id: string;
  name: string;
  progress: number;
  timestamp: number;
}

export default function WorkflowDashboard() {
  const [jobs, setJobs] = useState<WorkflowJob[]>([]);

  useEffect(() => {
    const fetchJobs = async () => {
      const res = await fetch('/api/workflows');
      const data = await res.json();
      setJobs(data.jobs);
    };

    fetchJobs();
    const interval = setInterval(fetchJobs, 2000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Workflow Dashboard</h1>

      <div className="space-y-3">
        {jobs.map((job) => (
          <a
            key={job.id}
            href={`/workflows/${job.id}`}
            className="block p-4 border rounded-lg hover:bg-gray-50"
          >
            <div className="flex justify-between items-center">
              <span className="font-medium">{job.name}</span>
              <span className="text-sm text-gray-500">
                {new Date(job.timestamp).toLocaleString()}
              </span>
            </div>
            <div className="mt-2 bg-gray-200 rounded-full h-2">
              <div
                className="bg-blue-500 rounded-full h-2 transition-all"
                style={{ width: `${job.progress}%` }}
              />
            </div>
          </a>
        ))}
      </div>
    </div>
  );
}
```

**2. Build workflow detail page (60 min)**

Create `app/workflows/[id]/page.tsx` with real-time status, logs, and results.

**3. Add "Create Workflow" form (30 min)**

Add a simple form to trigger example workflows from the dashboard.

### Day 7 Checkpoints
- [ ] Dashboard shows all jobs with progress
- [ ] Detail page shows logs and results
- [ ] Can create workflows from UI
- [ ] Real-time updates via polling

---

## Final Checkpoints

- [ ] Jobs process in background
- [ ] Worker handles failures and retries
- [ ] Can monitor job status in real-time
- [ ] Understand event-driven architecture
- [ ] Real LLM calls work in workflow steps

## Study Resources

- [ ] Read: [BullMQ Guide](https://docs.bullmq.io/)
- [ ] Read: [Inngest Docs](https://www.inngest.com/docs) (alternative approach)
- [ ] Reverse-engineer: [Trigger.dev source](https://github.com/triggerdotdev/trigger.dev)
- [ ] Study: n8n workflow execution code

## Interview Prep

- Be able to explain: "How do job queues work?"
- Be able to explain: "How do you handle failed jobs and retries?"
- Be able to design: A workflow system architecture on a whiteboard
- Be able to explain: "Why not just use async/await for long-running tasks?"
