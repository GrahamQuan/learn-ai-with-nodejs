# Level 0: Foundation + Streaming (Week 1-2)

**Time:** 12-16 hours | **Status:** 🔲 Not Started | **Priority:** CRITICAL

[Back to Blueprint](./README.md) | Next: [Level 1 - Job Queues](./roadmap-level-1-queues.md)

---

## Why This Matters

Companies need engineers who understand LLM APIs, streaming, and async patterns. This is table-stakes knowledge for any AI engineer role.

## Concepts to Master

- LLM API fundamentals (Claude/GPT)
- Streaming responses (SSE, ReadableStream)
- Async/await patterns
- Token management and pricing
- Error handling and retries
- Rate limiting basics

## Project: Streaming Web Chatbot

Skip the basic CLI. Build a web chatbot with streaming from day 1.

```
apps/level-0-streaming-chat/
├── app/
│   ├── api/chat/route.ts          # Streaming API endpoint
│   ├── page.tsx                    # Chat UI
│   └── components/
│       ├── ChatMessage.tsx         # Message component
│       └── StreamingText.tsx       # Streaming text display
├── lib/
│   ├── llm-client.ts              # LLM API wrapper
│   ├── stream-handler.ts          # Stream utilities
│   └── token-counter.ts           # Token tracking
└── package.json
```

---

## Day 1: Setup & First API Call (2 hours)

### Goal
Make your first successful call to Claude API and see a streaming response.

### Steps

**1. Create the project (15 min)**
```bash
cd apps
pnpm create next-app level-0-streaming-chat --typescript --tailwind --app
cd level-0-streaming-chat
```

**2. Install dependencies (5 min)**
```bash
pnpm add @anthropic-ai/sdk ai @ai-sdk/anthropic zod
pnpm add -D @types/node
```

**3. Get API key (10 min)**
- Go to https://console.anthropic.com/
- Create account / sign in
- Go to API Keys → Create new key → Copy it

**4. Setup environment (5 min)**
```bash
echo "ANTHROPIC_API_KEY=your-key-here" > .env.local
```

**5. Test basic API call (45 min)**

Create `lib/test-api.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';

async function testAPI() {
  const client = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY,
  });

  console.log('Calling Claude API...');

  const message = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: 'Hello! Can you explain what you are in one sentence?',
    }],
  });

  console.log('Response:', message.content[0].text);
  console.log('Tokens used:', message.usage);
}

testAPI();
```

Run: `tsx lib/test-api.ts`

**6. Test streaming (40 min)**

Create `lib/test-streaming.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';

async function testStreaming() {
  const client = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY,
  });

  console.log('Streaming response...\n');

  const stream = await client.messages.stream({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: 'Write a haiku about coding.',
    }],
  });

  for await (const chunk of stream) {
    if (
      chunk.type === 'content_block_delta' &&
      chunk.delta.type === 'text_delta'
    ) {
      process.stdout.write(chunk.delta.text);
    }
  }

  console.log('\n\nDone!');
}

testStreaming();
```

Run: `tsx lib/test-streaming.ts`

### Day 1 Checkpoints
- [ ] API key works
- [ ] Can make basic API call
- [ ] Can stream responses
- [ ] Understand tokens and usage

---

## Day 2: Streaming API Route (2-3 hours)

### Goal
Create a Next.js API route that streams Claude responses.

### Steps

**1. Create API route (60 min)**

Create `app/api/chat/route.ts`:
```typescript
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export const runtime = 'edge';

export async function POST(req: Request) {
  try {
    const { messages } = await req.json();

    const result = await streamText({
      model: anthropic('claude-sonnet-4-20250514'),
      messages,
      maxTokens: 1024,
    });

    return result.toDataStreamResponse();
  } catch (error) {
    console.error('Chat API error:', error);
    return new Response('Error processing request', { status: 500 });
  }
}
```

**2. Test with curl (15 min)**
```bash
pnpm dev
# In another terminal:
curl -X POST http://localhost:3000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Hello!"}]}'
```

**3. Add error handling & rate limiting (60 min)**

Create `lib/rate-limit.ts`:
```typescript
const rateLimit = new Map<string, { count: number; resetAt: number }>();

export function checkRateLimit(
  identifier: string,
  limit: number = 10,
  windowMs: number = 60000
): boolean {
  const now = Date.now();
  const record = rateLimit.get(identifier);

  if (!record || now > record.resetAt) {
    rateLimit.set(identifier, { count: 1, resetAt: now + windowMs });
    return true;
  }

  if (record.count >= limit) return false;
  record.count++;
  return true;
}
```

### Day 2 Checkpoints
- [ ] API route streams responses
- [ ] Error handling works
- [ ] Rate limiting implemented

---

## Day 3-4: Chat UI (3-4 hours)

### Goal
Build a polished chat interface with streaming text display.

### Steps

**1. Build chat page with useChat hook (90 min)**

Update `app/page.tsx`:
```typescript
'use client';
import { useChat } from 'ai/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } = useChat();

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">AI Chat</h1>

      <div className="flex-1 overflow-y-auto space-y-4 mb-4">
        {messages.map((m) => (
          <div
            key={m.id}
            className={`p-3 rounded-lg ${
              m.role === 'user'
                ? 'bg-blue-100 ml-auto max-w-[80%]'
                : 'bg-gray-100 mr-auto max-w-[80%]'
            }`}
          >
            <p className="text-sm font-medium mb-1">
              {m.role === 'user' ? 'You' : 'AI'}
            </p>
            <p className="whitespace-pre-wrap">{m.content}</p>
          </div>
        ))}

        {isLoading && (
          <div className="bg-gray-100 p-3 rounded-lg mr-auto">
            <p className="text-sm text-gray-500">Thinking...</p>
          </div>
        )}
      </div>

      {error && (
        <div className="bg-red-100 text-red-700 p-2 rounded mb-2">
          Error: {error.message}
        </div>
      )}

      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Type a message..."
          className="flex-1 p-2 border rounded-lg"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || !input.trim()}
          className="px-4 py-2 bg-blue-500 text-white rounded-lg disabled:opacity-50"
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

**2. Style and polish (60 min)**
- Add Tailwind styling
- Add auto-scroll to bottom
- Add keyboard shortcuts (Enter to send)
- Mobile responsive layout

**3. Add streaming text animation (30 min)**
- Cursor blinking effect while streaming
- Smooth text appearance

### Day 3-4 Checkpoints
- [ ] Chat UI renders messages
- [ ] Streaming text displays in real-time
- [ ] Loading and error states work
- [ ] UI is responsive and polished

---

## Day 5: Token Tracking & Cost (2 hours)

### Goal
Track token usage and display costs.

### Steps

**1. Build token counter (60 min)**

Create `lib/token-counter.ts`:
```typescript
// Claude pricing (per 1M tokens)
const PRICING = {
  'claude-sonnet-4-20250514': { input: 3, output: 15 },
} as const;

export interface TokenUsage {
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  estimatedCost: number;
}

export function calculateCost(
  model: string,
  inputTokens: number,
  outputTokens: number
): TokenUsage {
  const pricing = PRICING[model as keyof typeof PRICING] || { input: 3, output: 15 };

  const inputCost = (inputTokens / 1_000_000) * pricing.input;
  const outputCost = (outputTokens / 1_000_000) * pricing.output;

  return {
    inputTokens,
    outputTokens,
    totalTokens: inputTokens + outputTokens,
    estimatedCost: inputCost + outputCost,
  };
}
```

**2. Display usage in UI (60 min)**
- Show token count per message
- Running total for session
- Estimated cost display

### Day 5 Checkpoints
- [ ] Token counting works
- [ ] Cost calculation accurate
- [ ] Usage displayed in UI

---

## Week 1 Summary Checklist

- [ ] API key setup and working
- [ ] Streaming API route implemented
- [ ] Chat UI built and polished
- [ ] Token tracking added
- [ ] Error handling works
- [ ] Rate limiting implemented
- [ ] Understand async/streaming patterns

---

## Study Resources

- [ ] Read: [Anthropic Streaming Docs](https://docs.anthropic.com/en/api/streaming)
- [ ] Read: [MDN ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream)
- [ ] Read: [Vercel AI SDK Docs](https://sdk.vercel.ai/docs)
- [ ] Reverse-engineer: [Vercel AI Chatbot](https://github.com/vercel/ai-chatbot)

## Interview Prep

- Be able to explain: "How does streaming work in LLM APIs?"
- Be able to explain: "Why use streaming vs waiting for full response?"
- Be able to explain: "What are tokens and how does pricing work?"
- Be able to code: A streaming endpoint from scratch

---

**Next:** [Level 1 - Job Queues & Background Processing](./roadmap-level-1-queues.md)
