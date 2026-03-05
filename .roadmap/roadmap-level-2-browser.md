# Level 2: Browser Automation Workflows (Week 5-7)

**Time:** 18-24 hours | **Status:** 🔲 Not Started | **Priority:** HIGH

[Back to Blueprint](./README.md) | Prev: [Level 1](./roadmap-level-1-queues.md) | Next: [Level 3](./roadmap-level-3-api-workflows.md)

---

## Why This Matters

OpenClaw-style browser automation is one of the hottest areas in AI right now. Companies building AI agents need engineers who can combine vision models with browser control.

## Concepts to Master

- Browser automation (Playwright)
- Computer use / UI automation
- Vision-based navigation (Claude with images)
- Action planning and execution
- Error recovery in automation
- Screenshot analysis

## Project: AI Browser Automation Agent

Build an agent that can navigate websites, fill forms, and extract data.

```
apps/level-2-browser-agent/
├── app/
│   ├── api/
│   │   ├── automate/route.ts           # Start automation
│   │   ├── automate/[id]/route.ts      # Get status
│   │   └── automate/[id]/stop/route.ts # Stop automation
│   ├── automate/page.tsx               # Automation UI
│   └── automate/[id]/page.tsx          # Live view
├── lib/
│   ├── browser/
│   │   ├── client.ts                   # Playwright client
│   │   ├── actions.ts                  # Browser actions
│   │   └── screenshot.ts              # Screenshot utilities
│   ├── agent/
│   │   ├── planner.ts                  # Action planning
│   │   ├── executor.ts                 # Action execution
│   │   └── vision.ts                   # Vision analysis
│   └── workflows/
│       ├── form-filler.ts              # Form filling workflow
│       ├── data-extractor.ts           # Data extraction
│       └── web-navigator.ts            # Navigation workflow
└── workers/
    └── automation-worker.ts            # Background automation
```

---

## Day 1-2: Playwright Setup (3-4 hours)

### Goal
Get Playwright running and take your first automated screenshots.

### Steps

**1. Install Playwright (15 min)**
```bash
cd apps/level-2-browser-agent
pnpm add playwright
pnpm exec playwright install chromium
```

**2. Create browser client (60 min)**

Create `lib/browser/client.ts`:
```typescript
import { chromium, Browser, Page, BrowserContext } from 'playwright';

export class BrowserClient {
  private browser: Browser | null = null;
  private context: BrowserContext | null = null;
  private page: Page | null = null;

  async launch(options?: { headless?: boolean }) {
    this.browser = await chromium.launch({
      headless: options?.headless ?? true,
    });
    this.context = await this.browser.newContext({
      viewport: { width: 1280, height: 720 },
    });
    this.page = await this.context.newPage();
    return this.page;
  }

  getPage(): Page {
    if (!this.page) throw new Error('Browser not launched');
    return this.page;
  }

  async navigate(url: string) {
    const page = this.getPage();
    await page.goto(url, { waitUntil: 'networkidle' });
  }

  async screenshot(): Promise<Buffer> {
    const page = this.getPage();
    return await page.screenshot({ type: 'png' });
  }

  async close() {
    await this.context?.close();
    await this.browser?.close();
    this.page = null;
    this.context = null;
    this.browser = null;
  }
}
```

**3. Test basic navigation (30 min)**

Create `lib/browser/test-browser.ts`:
```typescript
import { BrowserClient } from './client';

async function test() {
  const browser = new BrowserClient();
  await browser.launch({ headless: false });

  await browser.navigate('https://example.com');
  const screenshot = await browser.screenshot();

  // Save screenshot
  const fs = await import('fs');
  fs.writeFileSync('test-screenshot.png', screenshot);
  console.log('Screenshot saved!');

  await browser.close();
}

test();
```

**4. Build browser actions (60 min)**

Create `lib/browser/actions.ts`:
```typescript
import { Page } from 'playwright';

export class BrowserActions {
  constructor(private page: Page) {}

  async click(selector: string) {
    try {
      await this.page.waitForSelector(selector, { timeout: 5000 });
      await this.page.click(selector);
      await this.page.waitForLoadState('networkidle');
    } catch (error) {
      throw new Error(`Failed to click "${selector}": ${error}`);
    }
  }

  async type(selector: string, text: string) {
    try {
      await this.page.waitForSelector(selector, { timeout: 5000 });
      await this.page.fill(selector, text);
    } catch (error) {
      throw new Error(`Failed to type in "${selector}": ${error}`);
    }
  }

  async scroll(direction: 'up' | 'down', pixels: number = 500) {
    await this.page.evaluate(
      ({ dir, px }) => window.scrollBy(0, dir === 'down' ? px : -px),
      { dir: direction, px: pixels }
    );
    await this.page.waitForTimeout(500);
  }

  async getText(selector: string): Promise<string> {
    const el = await this.page.waitForSelector(selector, { timeout: 5000 });
    return (await el?.textContent()) || '';
  }

  async getTexts(selector: string): Promise<string[]> {
    return await this.page.$$eval(selector, (els) =>
      els.map((el) => el.textContent || '')
    );
  }

  async waitForNavigation() {
    await this.page.waitForLoadState('networkidle');
  }

  async getCurrentUrl(): Promise<string> {
    return this.page.url();
  }
}
```

### Day 1-2 Checkpoints
- [ ] Playwright installed and working
- [ ] Can navigate to URLs
- [ ] Can take screenshots
- [ ] Browser actions (click, type, scroll) work

---

## Day 3-5: Vision-Based Navigation (6-8 hours)

### Goal
Use Claude's vision to analyze screenshots and decide what to do next.

### Steps

**1. Build vision analyzer (90 min)**

Create `lib/agent/vision.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

export interface AgentAction {
  action: 'click' | 'type' | 'scroll' | 'extract' | 'navigate' | 'done';
  selector?: string;
  value?: string;
  url?: string;
  direction?: 'up' | 'down';
  reasoning: string;
}

export async function analyzeScreenshot(
  screenshot: Buffer,
  instruction: string,
  previousActions: string[] = []
): Promise<AgentAction> {
  const historyContext =
    previousActions.length > 0
      ? `\n\nPrevious actions taken:\n${previousActions.map((a, i) => `${i + 1}. ${a}`).join('\n')}`
      : '';

  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: 'image/png',
              data: screenshot.toString('base64'),
            },
          },
          {
            type: 'text',
            text: `You are a browser automation agent. Your task: "${instruction}"${historyContext}

Analyze this screenshot and determine the next action. Respond ONLY with valid JSON:
{
  "action": "click" | "type" | "scroll" | "navigate" | "extract" | "done",
  "selector": "CSS selector for the target element (if applicable)",
  "value": "text to type or extracted data (if applicable)",
  "url": "URL to navigate to (if action is navigate)",
  "direction": "up or down (if action is scroll)",
  "reasoning": "brief explanation of why this action"
}`,
          },
        ],
      },
    ],
  });

  const text =
    response.content[0].type === 'text' ? response.content[0].text : '';

  try {
    return JSON.parse(text) as AgentAction;
  } catch {
    // Try to extract JSON from response
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      return JSON.parse(jsonMatch[0]) as AgentAction;
    }
    return { action: 'done', reasoning: 'Could not parse response' };
  }
}
```

**2. Build the agent loop (90 min)**

Create `lib/agent/executor.ts`:
```typescript
import { BrowserClient } from '../browser/client';
import { BrowserActions } from '../browser/actions';
import { analyzeScreenshot, AgentAction } from './vision';

export interface AutomationResult {
  success: boolean;
  steps: { action: AgentAction; screenshot?: string }[];
  finalData?: any;
  error?: string;
}

export async function runAutomation(
  instruction: string,
  startUrl: string,
  options?: {
    maxSteps?: number;
    headless?: boolean;
    onStep?: (step: number, action: AgentAction) => void;
  }
): Promise<AutomationResult> {
  const maxSteps = options?.maxSteps || 15;
  const browser = new BrowserClient();
  const steps: AutomationResult['steps'] = [];
  const actionHistory: string[] = [];

  try {
    const page = await browser.launch({ headless: options?.headless ?? true });
    const actions = new BrowserActions(page);

    await browser.navigate(startUrl);

    for (let step = 0; step < maxSteps; step++) {
      // Take screenshot
      const screenshot = await browser.screenshot();

      // Analyze and plan
      const plan = await analyzeScreenshot(
        screenshot,
        instruction,
        actionHistory
      );

      // Log
      console.log(`Step ${step + 1}: ${plan.action} - ${plan.reasoning}`);
      options?.onStep?.(step, plan);

      steps.push({
        action: plan,
        screenshot: screenshot.toString('base64').slice(0, 100) + '...',
      });

      actionHistory.push(
        `${plan.action}: ${plan.reasoning}`
      );

      // Done?
      if (plan.action === 'done') {
        return { success: true, steps, finalData: plan.value };
      }

      // Execute action
      switch (plan.action) {
        case 'click':
          if (plan.selector) await actions.click(plan.selector);
          break;
        case 'type':
          if (plan.selector && plan.value)
            await actions.type(plan.selector, plan.value);
          break;
        case 'scroll':
          await actions.scroll(plan.direction || 'down');
          break;
        case 'navigate':
          if (plan.url) await browser.navigate(plan.url);
          break;
        case 'extract':
          if (plan.selector) {
            const text = await actions.getText(plan.selector);
            actionHistory.push(`Extracted: "${text.slice(0, 200)}"`);
          }
          break;
      }

      // Brief pause between actions
      await page.waitForTimeout(1000);
    }

    return {
      success: false,
      steps,
      error: `Reached max steps (${maxSteps})`,
    };
  } catch (error) {
    return {
      success: false,
      steps,
      error: error instanceof Error ? error.message : String(error),
    };
  } finally {
    await browser.close();
  }
}
```

**3. Test on real websites (60 min)**

Create `lib/agent/test-agent.ts`:
```typescript
import { runAutomation } from './executor';

async function test() {
  console.log('Starting browser automation...\n');

  const result = await runAutomation(
    'Go to Wikipedia and search for "artificial intelligence", then extract the first paragraph of the article.',
    'https://en.wikipedia.org',
    {
      maxSteps: 10,
      headless: false,
      onStep: (step, action) => {
        console.log(`  [Step ${step + 1}] ${action.action}: ${action.reasoning}`);
      },
    }
  );

  console.log('\n--- Result ---');
  console.log('Success:', result.success);
  console.log('Steps taken:', result.steps.length);
  if (result.finalData) console.log('Data:', result.finalData);
  if (result.error) console.log('Error:', result.error);
}

test();
```

### Day 3-5 Checkpoints
- [ ] Vision analysis returns valid actions
- [ ] Agent can navigate multi-step flows
- [ ] Error recovery works (bad selectors, timeouts)
- [ ] Can extract data from pages

---

## Day 6-8: Pre-Built Workflows (5-6 hours)

### Goal
Build reusable workflow templates for common automation tasks.

### Steps

**1. Form filler workflow (90 min)**

Create `lib/workflows/form-filler.ts`:
```typescript
import { BrowserClient } from '../browser/client';
import { BrowserActions } from '../browser/actions';
import { analyzeScreenshot } from '../agent/vision';

export interface FormField {
  label: string;
  value: string;
}

export async function fillForm(
  url: string,
  fields: FormField[],
  options?: { submit?: boolean; headless?: boolean }
) {
  const browser = new BrowserClient();

  try {
    const page = await browser.launch({ headless: options?.headless ?? true });
    const actions = new BrowserActions(page);

    await browser.navigate(url);

    for (const field of fields) {
      const screenshot = await browser.screenshot();

      // Ask Claude to find the field
      const plan = await analyzeScreenshot(
        screenshot,
        `Find the form field labeled "${field.label}" and type "${field.value}" into it.`
      );

      if (plan.action === 'type' && plan.selector) {
        await actions.type(plan.selector, field.value);
        console.log(`Filled "${field.label}" with "${field.value}"`);
      }
    }

    if (options?.submit) {
      const screenshot = await browser.screenshot();
      const plan = await analyzeScreenshot(
        screenshot,
        'Find and click the submit button.'
      );

      if (plan.action === 'click' && plan.selector) {
        await actions.click(plan.selector);
        console.log('Form submitted!');
      }
    }

    return { success: true };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : String(error),
    };
  } finally {
    await browser.close();
  }
}
```

**2. Data extractor workflow (90 min)**

Create `lib/workflows/data-extractor.ts`:
```typescript
import { BrowserClient } from '../browser/client';
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

export interface ExtractionSchema {
  fields: { name: string; description: string }[];
}

export async function extractData(
  url: string,
  schema: ExtractionSchema,
  options?: { headless?: boolean }
) {
  const browser = new BrowserClient();

  try {
    await browser.launch({ headless: options?.headless ?? true });
    await browser.navigate(url);

    const screenshot = await browser.screenshot();

    const fieldDescriptions = schema.fields
      .map((f) => `- ${f.name}: ${f.description}`)
      .join('\n');

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image',
              source: {
                type: 'base64',
                media_type: 'image/png',
                data: screenshot.toString('base64'),
              },
            },
            {
              type: 'text',
              text: `Extract the following data from this webpage screenshot. Return ONLY valid JSON.\n\nFields to extract:\n${fieldDescriptions}\n\nRespond with: { "data": { "fieldName": "value", ... } }`,
            },
          ],
        },
      ],
    });

    const text =
      response.content[0].type === 'text' ? response.content[0].text : '';
    const parsed = JSON.parse(text.match(/\{[\s\S]*\}/)?.[0] || '{}');

    return { success: true, data: parsed.data || parsed };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : String(error),
    };
  } finally {
    await browser.close();
  }
}
```

**3. Web navigator workflow (60 min)**

Create `lib/workflows/web-navigator.ts`:
```typescript
import { runAutomation } from '../agent/executor';

// Pre-built navigation tasks
export const navigationTasks = {
  async searchGoogle(query: string) {
    return runAutomation(
      `Search Google for "${query}" and extract the titles and URLs of the first 5 results.`,
      'https://www.google.com',
      { maxSteps: 8 }
    );
  },

  async scrapeProductInfo(url: string) {
    return runAutomation(
      'Extract the product name, price, description, and rating from this product page.',
      url,
      { maxSteps: 5 }
    );
  },

  async checkWebsiteStatus(url: string) {
    return runAutomation(
      'Check if this website is loading correctly. Report the page title and any error messages visible.',
      url,
      { maxSteps: 3 }
    );
  },
};
```

### Day 6-8 Checkpoints
- [ ] Form filler works on real forms
- [ ] Data extractor returns structured data
- [ ] Navigation workflows complete successfully

---

## Day 9-11: API & Dashboard (5-6 hours)

### Goal
Build API endpoints and a dashboard to manage automations.

### Steps

**1. API routes (90 min)**

Create `app/api/automate/route.ts`:
```typescript
import { NextResponse } from 'next/server';
import { runAutomation } from '@/lib/agent/executor';

// In-memory store (use Redis/DB in production)
const automations = new Map<
  string,
  { status: string; result?: any; error?: string }
>();

export async function POST(req: Request) {
  const { instruction, url, maxSteps } = await req.json();

  const id = crypto.randomUUID();
  automations.set(id, { status: 'running' });

  // Run in background (don't await)
  runAutomation(instruction, url, { maxSteps: maxSteps || 15 })
    .then((result) => {
      automations.set(id, {
        status: result.success ? 'completed' : 'failed',
        result,
      });
    })
    .catch((error) => {
      automations.set(id, {
        status: 'failed',
        error: error.message,
      });
    });

  return NextResponse.json({ id, status: 'running' });
}
```

Create `app/api/automate/[id]/route.ts`:
```typescript
import { NextResponse } from 'next/server';

export async function GET(
  req: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const automation = automations.get(id);

  if (!automation) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }

  return NextResponse.json({ id, ...automation });
}
```

**2. Dashboard UI (90 min)**

Create `app/automate/page.tsx`:
```typescript
'use client';
import { useState } from 'react';

export default function AutomatePage() {
  const [instruction, setInstruction] = useState('');
  const [url, setUrl] = useState('');
  const [automationId, setAutomationId] = useState<string | null>(null);
  const [status, setStatus] = useState<string>('');
  const [result, setResult] = useState<any>(null);

  async function startAutomation() {
    const res = await fetch('/api/automate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ instruction, url, maxSteps: 15 }),
    });
    const data = await res.json();
    setAutomationId(data.id);
    setStatus('running');

    // Poll for status
    const interval = setInterval(async () => {
      const statusRes = await fetch(`/api/automate/${data.id}`);
      const statusData = await statusRes.json();
      setStatus(statusData.status);

      if (statusData.status !== 'running') {
        clearInterval(interval);
        setResult(statusData.result);
      }
    }, 2000);
  }

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Browser Automation Agent</h1>

      <div className="space-y-4 mb-6">
        <input
          type="url"
          placeholder="https://example.com"
          value={url}
          onChange={(e) => setUrl(e.target.value)}
          className="w-full p-3 border rounded"
        />
        <textarea
          placeholder="What should the agent do? e.g., 'Search for AI news and extract the top 5 headlines'"
          value={instruction}
          onChange={(e) => setInstruction(e.target.value)}
          className="w-full p-3 border rounded h-24"
        />
        <button
          onClick={startAutomation}
          disabled={!instruction || !url || status === 'running'}
          className="px-6 py-3 bg-blue-600 text-white rounded disabled:opacity-50"
        >
          {status === 'running' ? 'Running...' : 'Start Automation'}
        </button>
      </div>

      {status && (
        <div className="border rounded p-4">
          <p className="font-medium">
            Status:{' '}
            <span
              className={
                status === 'completed'
                  ? 'text-green-600'
                  : status === 'failed'
                    ? 'text-red-600'
                    : 'text-yellow-600'
              }
            >
              {status}
            </span>
          </p>
          {result && (
            <pre className="mt-4 p-3 bg-gray-100 rounded text-sm overflow-auto">
              {JSON.stringify(result, null, 2)}
            </pre>
          )}
        </div>
      )}
    </div>
  );
}
```

### Day 9-11 Checkpoints
- [ ] API endpoints work
- [ ] Dashboard can start and monitor automations
- [ ] Results display correctly

---

## Final Checkpoints

- [ ] Playwright automation works reliably
- [ ] Vision-based navigation handles real websites
- [ ] Pre-built workflows (form filler, data extractor) work
- [ ] API and dashboard functional
- [ ] Error recovery handles common failures

## Study Resources

- [ ] Read: [Playwright Docs](https://playwright.dev/docs/intro)
- [ ] Read: [Anthropic Vision](https://docs.anthropic.com/en/docs/build-with-claude/vision)
- [ ] Reverse-engineer: [Browser Use](https://github.com/browser-use/browser-use)
- [ ] Study: OpenClaw architecture

## Interview Prep

- Be able to explain: "How does vision-based browser automation work?"
- Be able to explain: "How do you handle dynamic websites and SPAs?"
- Be able to demo: A working browser automation agent
- Be able to discuss: Trade-offs between selector-based vs vision-based automation
