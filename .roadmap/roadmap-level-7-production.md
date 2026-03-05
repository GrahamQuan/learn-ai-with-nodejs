# Level 7: Production Patterns (Week 20-22)

**Time:** 18-24 hours | **Status:** 🔲 Not Started | **Priority:** HIGH

[Back to Blueprint](./README.md) | Prev: [Level 6](./roadmap-level-6-multi-agent.md) | Next: [Level 8](./roadmap-level-8-portfolio.md)

---

## Why This Matters

Building AI demos is easy. Making them production-ready is what companies pay for. Caching, rate limiting, cost control, monitoring, and error handling separate junior from senior AI engineers.

## Concepts to Master

- Response caching (semantic cache)
- Rate limiting and throttling
- Cost tracking and budgets
- Observability and logging
- Error handling and fallbacks
- Model fallback chains
- Prompt caching (Anthropic)
- Streaming error recovery
- Load testing AI endpoints

## Project: Production-Ready AI API

Take your best project from previous levels and make it production-grade.

```
apps/level-7-production/
├── app/
│   ├── api/
│   │   ├── chat/route.ts             # Production chat endpoint
│   │   └── admin/
│   │       ├── usage/route.ts         # Usage dashboard API
│   │       └── health/route.ts        # Health check
│   ├── admin/page.tsx                  # Admin dashboard
│   └── page.tsx                        # Chat UI
├── lib/
│   ├── cache/
│   │   ├── semantic-cache.ts          # Semantic response cache
│   │   └── redis-client.ts            # Redis client
│   ├── middleware/
│   │   ├── rate-limiter.ts            # Rate limiting
│   │   ├── cost-guard.ts             # Cost budget enforcement
│   │   └── auth.ts                    # API key auth
│   ├── observability/
│   │   ├── logger.ts                  # Structured logging
│   │   ├── metrics.ts                 # Usage metrics
│   │   └── tracer.ts                  # Request tracing
│   ├── resilience/
│   │   ├── fallback.ts                # Model fallback chain
│   │   ├── retry.ts                   # Smart retry logic
│   │   └── circuit-breaker.ts         # Circuit breaker
│   └── llm/
│       ├── client.ts                  # Production LLM client
│       └── prompt-cache.ts            # Prompt caching
```

---

## Day 1-2: Caching Layer (4-6 hours)

### Goal
Build a semantic cache that avoids redundant LLM calls for similar questions.

### Steps

**1. Setup Upstash Redis (20 min)**
- Create free account at https://upstash.com
- Create a Redis database
- Copy `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN`

```bash
pnpm add @upstash/redis
```

**2. Build Redis client (30 min)**

Create `lib/cache/redis-client.ts`:
```typescript
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Helper for typed cache operations
export async function cacheGet<T>(key: string): Promise<T | null> {
  return await redis.get<T>(key);
}

export async function cacheSet(
  key: string,
  value: any,
  ttlSeconds: number = 3600
) {
  await redis.set(key, value, { ex: ttlSeconds });
}

export async function cacheDelete(key: string) {
  await redis.del(key);
}
```

**3. Build semantic cache (90 min)**

Create `lib/cache/semantic-cache.ts`:
```typescript
import { redis } from './redis-client';
import { generateEmbedding } from '../llm/client';

interface CacheEntry {
  query: string;
  response: string;
  embedding: number[];
  createdAt: number;
  hitCount: number;
}

export class SemanticCache {
  private prefix: string;
  private similarityThreshold: number;
  private ttlSeconds: number;

  constructor(options?: {
    prefix?: string;
    similarityThreshold?: number;
    ttlSeconds?: number;
  }) {
    this.prefix = options?.prefix || 'sem-cache';
    this.similarityThreshold = options?.similarityThreshold || 0.95;
    this.ttlSeconds = options?.ttlSeconds || 3600;
  }

  async get(query: string): Promise<string | null> {
    // Generate embedding for query
    const queryEmbedding = await generateEmbedding(query);

    // Get all cache keys
    const keys = await redis.keys(`${this.prefix}:*`);
    if (keys.length === 0) return null;

    // Check similarity against cached entries
    let bestMatch: { key: string; similarity: number; response: string } | null = null;

    for (const key of keys) {
      const entry = await redis.get<CacheEntry>(key);
      if (!entry) continue;

      const similarity = cosineSimilarity(queryEmbedding, entry.embedding);

      if (
        similarity >= this.similarityThreshold &&
        (!bestMatch || similarity > bestMatch.similarity)
      ) {
        bestMatch = { key, similarity, response: entry.response };
      }
    }

    if (bestMatch) {
      // Update hit count
      const entry = await redis.get<CacheEntry>(bestMatch.key);
      if (entry) {
        entry.hitCount++;
        await redis.set(bestMatch.key, entry, { ex: this.ttlSeconds });
      }
      return bestMatch.response;
    }

    return null;
  }

  async set(query: string, response: string) {
    const embedding = await generateEmbedding(query);
    const key = `${this.prefix}:${crypto.randomUUID()}`;

    const entry: CacheEntry = {
      query,
      response,
      embedding,
      createdAt: Date.now(),
      hitCount: 0,
    };

    await redis.set(key, entry, { ex: this.ttlSeconds });
  }
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }

  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

**4. Integrate with chat endpoint (60 min)**

Create `lib/llm/client.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai';
import { SemanticCache } from '../cache/semantic-cache';

const anthropic = new Anthropic();
const openai = new OpenAI();
const cache = new SemanticCache({ similarityThreshold: 0.93 });

export async function generateEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  });
  return response.data[0].embedding;
}

export async function chat(
  messages: { role: 'user' | 'assistant'; content: string }[],
  options?: { useCache?: boolean }
) {
  const lastMessage = messages[messages.length - 1]?.content || '';

  // Check cache first
  if (options?.useCache !== false) {
    const cached = await cache.get(lastMessage);
    if (cached) {
      return { text: cached, cached: true, tokens: 0 };
    }
  }

  // Call LLM
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    messages,
  });

  const text =
    response.content[0].type === 'text' ? response.content[0].text : '';

  // Store in cache
  if (options?.useCache !== false) {
    await cache.set(lastMessage, text);
  }

  return {
    text,
    cached: false,
    tokens: response.usage.input_tokens + response.usage.output_tokens,
  };
}
```

### Day 1-2 Checkpoints
- [ ] Redis connected and working
- [ ] Semantic cache stores and retrieves responses
- [ ] Cache hit rate visible
- [ ] Similar questions return cached responses

---

## Day 3-4: Rate Limiting & Cost Control (4-6 hours)

### Goal
Protect your API from abuse and control costs.

### Steps

**1. Build rate limiter (60 min)**

Create `lib/middleware/rate-limiter.ts`:
```typescript
import { redis } from '../cache/redis-client';

interface RateLimitResult {
  allowed: boolean;
  remaining: number;
  resetAt: number;
  limit: number;
}

export async function rateLimit(
  identifier: string,
  options?: { limit?: number; windowSeconds?: number }
): Promise<RateLimitResult> {
  const { limit = 20, windowSeconds = 60 } = options || {};
  const key = `rate-limit:${identifier}`;
  const now = Math.floor(Date.now() / 1000);
  const windowStart = now - windowSeconds;

  // Use Redis sorted set for sliding window
  const pipeline = redis.pipeline();

  // Remove old entries
  pipeline.zremrangebyscore(key, 0, windowStart);
  // Add current request
  pipeline.zadd(key, { score: now, member: `${now}:${Math.random()}` });
  // Count requests in window
  pipeline.zcard(key);
  // Set expiry
  pipeline.expire(key, windowSeconds);

  const results = await pipeline.exec();
  const count = results[2] as number;

  return {
    allowed: count <= limit,
    remaining: Math.max(0, limit - count),
    resetAt: now + windowSeconds,
    limit,
  };
}
```

**2. Build cost guard (60 min)**

Create `lib/middleware/cost-guard.ts`:
```typescript
import { redis } from '../cache/redis-client';

// Pricing per 1M tokens (approximate)
const PRICING: Record<string, { input: number; output: number }> = {
  'claude-sonnet-4-20250514': { input: 3, output: 15 },
  'claude-haiku-4-5-20251001': { input: 0.25, output: 1.25 },
  'gpt-4o': { input: 2.5, output: 10 },
  'gpt-4o-mini': { input: 0.15, output: 0.6 },
};

interface UsageRecord {
  inputTokens: number;
  outputTokens: number;
  cost: number;
  requests: number;
}

export class CostGuard {
  private budgetKey: string;
  private dailyBudget: number; // in dollars

  constructor(options?: { budgetKey?: string; dailyBudget?: number }) {
    this.budgetKey = options?.budgetKey || 'cost-guard';
    this.dailyBudget = options?.dailyBudget || 10; // $10/day default
  }

  async checkBudget(userId: string): Promise<{
    allowed: boolean;
    spent: number;
    remaining: number;
  }> {
    const today = new Date().toISOString().split('T')[0];
    const key = `${this.budgetKey}:${userId}:${today}`;

    const usage = await redis.get<UsageRecord>(key);
    const spent = usage?.cost || 0;

    return {
      allowed: spent < this.dailyBudget,
      spent,
      remaining: Math.max(0, this.dailyBudget - spent),
    };
  }

  async recordUsage(
    userId: string,
    model: string,
    inputTokens: number,
    outputTokens: number
  ) {
    const today = new Date().toISOString().split('T')[0];
    const key = `${this.budgetKey}:${userId}:${today}`;

    const pricing = PRICING[model] || { input: 3, output: 15 };
    const cost =
      (inputTokens / 1_000_000) * pricing.input +
      (outputTokens / 1_000_000) * pricing.output;

    const existing = (await redis.get<UsageRecord>(key)) || {
      inputTokens: 0,
      outputTokens: 0,
      cost: 0,
      requests: 0,
    };

    const updated: UsageRecord = {
      inputTokens: existing.inputTokens + inputTokens,
      outputTokens: existing.outputTokens + outputTokens,
      cost: existing.cost + cost,
      requests: existing.requests + 1,
    };

    // Expire at end of day
    const secondsUntilMidnight =
      86400 - (Date.now() % 86400000) / 1000;
    await redis.set(key, updated, {
      ex: Math.ceil(secondsUntilMidnight),
    });

    return { cost, totalCost: updated.cost };
  }

  async getUsageReport(
    userId: string,
    days: number = 7
  ): Promise<{ date: string; usage: UsageRecord }[]> {
    const report = [];

    for (let i = 0; i < days; i++) {
      const date = new Date(Date.now() - i * 86400000)
        .toISOString()
        .split('T')[0];
      const key = `${this.budgetKey}:${userId}:${date}`;
      const usage = await redis.get<UsageRecord>(key);

      if (usage) {
        report.push({ date, usage });
      }
    }

    return report;
  }
}
```

**3. Build model fallback chain (60 min)**

Create `lib/resilience/fallback.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

interface FallbackModel {
  model: string;
  maxTokens: number;
  priority: number; // Lower = try first
}

const FALLBACK_CHAIN: FallbackModel[] = [
  { model: 'claude-sonnet-4-20250514', maxTokens: 4096, priority: 1 },
  { model: 'claude-haiku-4-5-20251001', maxTokens: 4096, priority: 2 },
];

export async function chatWithFallback(
  messages: { role: 'user' | 'assistant'; content: string }[],
  options?: { systemPrompt?: string }
) {
  const sorted = [...FALLBACK_CHAIN].sort((a, b) => a.priority - b.priority);
  let lastError: Error | null = null;

  for (const { model, maxTokens } of sorted) {
    try {
      const response = await client.messages.create({
        model,
        max_tokens: maxTokens,
        ...(options?.systemPrompt ? { system: options.systemPrompt } : {}),
        messages,
      });

      return {
        text:
          response.content[0].type === 'text'
            ? response.content[0].text
            : '',
        model,
        usage: response.usage,
        fallback: model !== sorted[0].model,
      };
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));
      console.warn(`Model ${model} failed, trying next...`, lastError.message);
      continue;
    }
  }

  throw lastError || new Error('All models failed');
}
```

**4. Build circuit breaker (60 min)**

Create `lib/resilience/circuit-breaker.ts`:
```typescript
interface CircuitState {
  failures: number;
  lastFailure: number;
  state: 'closed' | 'open' | 'half-open';
}

export class CircuitBreaker {
  private state: CircuitState = {
    failures: 0,
    lastFailure: 0,
    state: 'closed',
  };

  constructor(
    private options: {
      failureThreshold?: number;
      resetTimeoutMs?: number;
    } = {}
  ) {}

  private get failureThreshold() {
    return this.options.failureThreshold || 5;
  }

  private get resetTimeoutMs() {
    return this.options.resetTimeoutMs || 30000;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Check if circuit is open
    if (this.state.state === 'open') {
      const timeSinceLastFailure = Date.now() - this.state.lastFailure;

      if (timeSinceLastFailure < this.resetTimeoutMs) {
        throw new Error('Circuit breaker is open - service unavailable');
      }

      // Try half-open
      this.state.state = 'half-open';
    }

    try {
      const result = await fn();

      // Success - reset
      if (this.state.state === 'half-open') {
        this.state = { failures: 0, lastFailure: 0, state: 'closed' };
      }

      return result;
    } catch (error) {
      this.state.failures++;
      this.state.lastFailure = Date.now();

      if (this.state.failures >= this.failureThreshold) {
        this.state.state = 'open';
      }

      throw error;
    }
  }

  getState() {
    return { ...this.state };
  }
}
```

### Day 3-4 Checkpoints
- [ ] Rate limiting works per user
- [ ] Cost tracking records every request
- [ ] Daily budget enforcement works
- [ ] Model fallback chain handles failures
- [ ] Circuit breaker prevents cascading failures

---

## Day 5-6: Observability (4-6 hours)

### Goal
Add structured logging, metrics, and request tracing.

### Steps

**1. Build structured logger (45 min)**

Create `lib/observability/logger.ts`:
```typescript
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface LogEntry {
  level: LogLevel;
  message: string;
  timestamp: string;
  traceId?: string;
  data?: Record<string, any>;
}

class Logger {
  private minLevel: LogLevel;
  private levels: Record<LogLevel, number> = {
    debug: 0,
    info: 1,
    warn: 2,
    error: 3,
  };

  constructor(minLevel: LogLevel = 'info') {
    this.minLevel = minLevel;
  }

  private log(level: LogLevel, message: string, data?: Record<string, any>) {
    if (this.levels[level] < this.levels[this.minLevel]) return;

    const entry: LogEntry = {
      level,
      message,
      timestamp: new Date().toISOString(),
      ...data,
    };

    // In production, send to logging service
    // For now, structured console output
    const output = JSON.stringify(entry);

    switch (level) {
      case 'error':
        console.error(output);
        break;
      case 'warn':
        console.warn(output);
        break;
      default:
        console.log(output);
    }
  }

  debug(message: string, data?: Record<string, any>) {
    this.log('debug', message, data);
  }
  info(message: string, data?: Record<string, any>) {
    this.log('info', message, data);
  }
  warn(message: string, data?: Record<string, any>) {
    this.log('warn', message, data);
  }
  error(message: string, data?: Record<string, any>) {
    this.log('error', message, data);
  }

  // Create child logger with trace context
  withTrace(traceId: string) {
    const parent = this;
    return {
      debug: (msg: string, data?: Record<string, any>) =>
        parent.log('debug', msg, { ...data, traceId }),
      info: (msg: string, data?: Record<string, any>) =>
        parent.log('info', msg, { ...data, traceId }),
      warn: (msg: string, data?: Record<string, any>) =>
        parent.log('warn', msg, { ...data, traceId }),
      error: (msg: string, data?: Record<string, any>) =>
        parent.log('error', msg, { ...data, traceId }),
    };
  }
}

export const logger = new Logger(
  (process.env.LOG_LEVEL as LogLevel) || 'info'
);
```

**2. Build metrics collector (45 min)**

Create `lib/observability/metrics.ts`:
```typescript
import { redis } from '../cache/redis-client';

export class Metrics {
  async recordLatency(endpoint: string, durationMs: number) {
    const key = `metrics:latency:${endpoint}`;
    const bucket = this.getTimeBucket();

    await redis.lpush(`${key}:${bucket}`, durationMs);
    await redis.expire(`${key}:${bucket}`, 86400);
  }

  async recordTokens(model: string, input: number, output: number) {
    const bucket = this.getTimeBucket();
    const key = `metrics:tokens:${model}:${bucket}`;

    await redis.hincrby(key, 'input', input);
    await redis.hincrby(key, 'output', output);
    await redis.hincrby(key, 'requests', 1);
    await redis.expire(key, 86400);
  }

  async recordCacheHit(hit: boolean) {
    const bucket = this.getTimeBucket();
    const key = `metrics:cache:${bucket}`;

    await redis.hincrby(key, hit ? 'hits' : 'misses', 1);
    await redis.expire(key, 86400);
  }

  async recordError(endpoint: string, errorType: string) {
    const bucket = this.getTimeBucket();
    const key = `metrics:errors:${endpoint}:${bucket}`;

    await redis.hincrby(key, errorType, 1);
    await redis.expire(key, 86400);
  }

  async getLatencyStats(
    endpoint: string
  ): Promise<{ avg: number; p50: number; p95: number; p99: number }> {
    const bucket = this.getTimeBucket();
    const key = `metrics:latency:${endpoint}:${bucket}`;

    const values = await redis.lrange(key, 0, -1);
    const nums = values.map(Number).sort((a, b) => a - b);

    if (nums.length === 0) {
      return { avg: 0, p50: 0, p95: 0, p99: 0 };
    }

    return {
      avg: nums.reduce((a, b) => a + b, 0) / nums.length,
      p50: nums[Math.floor(nums.length * 0.5)],
      p95: nums[Math.floor(nums.length * 0.95)],
      p99: nums[Math.floor(nums.length * 0.99)],
    };
  }

  private getTimeBucket(): string {
    // Hourly buckets
    const now = new Date();
    return `${now.toISOString().split('T')[0]}:${now.getHours()}`;
  }
}

export const metrics = new Metrics();
```

**3. Build production chat endpoint (60 min)**

Create `app/api/chat/route.ts`:
```typescript
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { rateLimit } from '@/lib/middleware/rate-limiter';
import { CostGuard } from '@/lib/middleware/cost-guard';
import { SemanticCache } from '@/lib/cache/semantic-cache';
import { logger } from '@/lib/observability/logger';
import { metrics } from '@/lib/observability/metrics';

const costGuard = new CostGuard({ dailyBudget: 10 });
const cache = new SemanticCache();

export async function POST(req: Request) {
  const startTime = Date.now();
  const traceId = crypto.randomUUID();
  const log = logger.withTrace(traceId);

  try {
    // 1. Auth
    const userId = req.headers.get('x-user-id') || 'anonymous';
    log.info('Chat request received', { userId });

    // 2. Rate limit
    const rateLimitResult = await rateLimit(userId, {
      limit: 20,
      windowSeconds: 60,
    });
    if (!rateLimitResult.allowed) {
      log.warn('Rate limited', { userId });
      return new Response('Rate limit exceeded', {
        status: 429,
        headers: {
          'X-RateLimit-Remaining': String(rateLimitResult.remaining),
          'X-RateLimit-Reset': String(rateLimitResult.resetAt),
        },
      });
    }

    // 3. Cost check
    const budget = await costGuard.checkBudget(userId);
    if (!budget.allowed) {
      log.warn('Budget exceeded', { userId, spent: budget.spent });
      return new Response('Daily budget exceeded', { status: 402 });
    }

    // 4. Parse request
    const { messages } = await req.json();
    const lastMessage = messages[messages.length - 1]?.content || '';

    // 5. Check cache
    const cached = await cache.get(lastMessage);
    if (cached) {
      log.info('Cache hit', { userId });
      await metrics.recordCacheHit(true);
      await metrics.recordLatency('/api/chat', Date.now() - startTime);
      return new Response(cached);
    }
    await metrics.recordCacheHit(false);

    // 6. Stream response
    const result = await streamText({
      model: anthropic('claude-sonnet-4-20250514'),
      messages,
      maxTokens: 2048,
      onFinish: async ({ usage, text }) => {
        // Record usage
        await costGuard.recordUsage(
          userId,
          'claude-sonnet-4-20250514',
          usage.promptTokens,
          usage.completionTokens
        );
        await metrics.recordTokens(
          'claude-sonnet-4-20250514',
          usage.promptTokens,
          usage.completionTokens
        );
        await metrics.recordLatency('/api/chat', Date.now() - startTime);

        // Cache response
        await cache.set(lastMessage, text);

        log.info('Chat completed', {
          userId,
          tokens: usage.promptTokens + usage.completionTokens,
          latencyMs: Date.now() - startTime,
        });
      },
    });

    return result.toDataStreamResponse({
      headers: {
        'X-Trace-Id': traceId,
        'X-RateLimit-Remaining': String(rateLimitResult.remaining),
      },
    });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'Unknown error';
    log.error('Chat failed', { error: message });
    await metrics.recordError('/api/chat', message);
    return new Response('Internal server error', { status: 500 });
  }
}
```

### Day 5-6 Checkpoints
- [ ] Structured logging with trace IDs
- [ ] Latency and token metrics collected
- [ ] Cache hit rate tracked
- [ ] Error rates monitored

---

## Day 7-9: Admin Dashboard (4-6 hours)

### Goal
Build a dashboard to monitor usage, costs, and performance.

### Steps

- [ ] Create admin API routes for usage data
- [ ] Build dashboard UI with charts:
  - Requests per hour
  - Token usage over time
  - Cost per day
  - Cache hit rate
  - Latency percentiles (p50, p95, p99)
  - Error rate
- [ ] Add real-time updates with polling
- [ ] Add alerts for budget thresholds

### Day 7-9 Checkpoints
- [ ] Dashboard shows real-time metrics
- [ ] Cost tracking visible
- [ ] Can identify performance issues

---

## Final Checkpoints

- [ ] Semantic cache reduces LLM calls by 20%+
- [ ] Rate limiting protects API
- [ ] Cost stays within daily budget
- [ ] Model fallback handles outages
- [ ] Circuit breaker prevents cascading failures
- [ ] Structured logs enable debugging
- [ ] Metrics dashboard shows system health
- [ ] Can explain production patterns in interviews

## Study Resources
- [ ] Read: [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [ ] Read: [Upstash Rate Limiting](https://upstash.com/docs/oss/sdks/ts/ratelimit/overview)
- [ ] Study: How Vercel AI SDK handles streaming errors
- [ ] Reverse-engineer: Production AI APIs (OpenRouter, Together AI)

## Interview Prep
- Explain: "How do you handle costs in production AI systems?"
- Explain: "What's a circuit breaker and when would you use one?"
- Design: "Design a rate limiting system for an AI API"
- Design: "How would you cache LLM responses?"
