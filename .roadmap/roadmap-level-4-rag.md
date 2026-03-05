# Level 4: RAG & Knowledge Workflows (Week 11-13)

**Time:** 18-24 hours | **Status:** 🔲 Not Started | **Priority:** HIGH

[Back to Blueprint](./README.md) | Prev: [Level 3](./roadmap-level-3-api-workflows.md) | Next: [Level 5](./roadmap-level-5-agents.md)

---

## Why This Matters

RAG (Retrieval-Augmented Generation) is how you make LLMs work with your own data. Every enterprise AI product uses RAG. Understanding embeddings, vector search, and retrieval pipelines is essential for AI engineer interviews.

## Concepts to Master

- Text embeddings and vector representations
- Vector databases (Supabase pgvector)
- Text chunking strategies
- Semantic search vs keyword search
- Hybrid search and reranking
- RAG pipeline architecture
- Source citations

## Project: Document Q&A System

Build a system that ingests documents, creates embeddings, and answers questions with citations.

```
apps/level-4-rag/
├── app/
│   ├── api/
│   │   ├── upload/route.ts            # Upload documents
│   │   ├── chat/route.ts             # Q&A with streaming
│   │   └── search/route.ts           # Semantic search
│   ├── page.tsx                       # Upload + Chat UI
│   └── components/
│       ├── FileUpload.tsx             # Document upload
│       ├── ChatWithSources.tsx        # Chat with citations
│       └── SearchResults.tsx          # Search results
├── lib/
│   ├── rag/
│   │   ├── chunker.ts                # Text chunking
│   │   ├── embeddings.ts             # Embedding generation
│   │   ├── vectorstore.ts            # Vector storage
│   │   ├── retriever.ts              # Search & retrieval
│   │   └── pipeline.ts               # Full RAG pipeline
│   └── db/
│       └── schema.sql                # Supabase schema
```

---

## Day 1-2: Embeddings & Vector Store (4-6 hours)

### Goal
Understand embeddings and set up vector storage with Supabase.

### Steps

**1. Setup Supabase (30 min)**
- Create free project at https://supabase.com
- Enable pgvector extension:
```sql
-- Run in Supabase SQL editor
create extension if not exists vector;
```

- Create documents table:
```sql
create table documents (
  id bigserial primary key,
  content text not null,
  metadata jsonb default '{}',
  embedding vector(1536),
  created_at timestamptz default now()
);

-- Create index for fast similarity search
create index on documents using ivfflat (embedding vector_cosine_ops)
  with (lists = 100);

-- Create search function
create or replace function match_documents(
  query_embedding vector(1536),
  match_threshold float default 0.7,
  match_count int default 5
)
returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language sql stable
as $$
  select
    id,
    content,
    metadata,
    1 - (embedding <=> query_embedding) as similarity
  from documents
  where 1 - (embedding <=> query_embedding) > match_threshold
  order by embedding <=> query_embedding
  limit match_count;
$$;
```

**2. Install dependencies (10 min)**
```bash
pnpm add @supabase/supabase-js openai ai @ai-sdk/anthropic
```

**3. Build embedding generator (60 min)**

Create `lib/rag/embeddings.ts`:
```typescript
import OpenAI from 'openai';

// OpenAI has the best embedding models for now
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function generateEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  });

  return response.data[0].embedding;
}

export async function generateEmbeddings(
  texts: string[]
): Promise<number[][]> {
  // Batch embeddings (max 2048 per request)
  const batches: string[][] = [];
  for (let i = 0; i < texts.length; i += 100) {
    batches.push(texts.slice(i, i + 100));
  }

  const allEmbeddings: number[][] = [];

  for (const batch of batches) {
    const response = await openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: batch,
    });

    allEmbeddings.push(...response.data.map((d) => d.embedding));
  }

  return allEmbeddings;
}
```

**4. Build vector store client (60 min)**

Create `lib/rag/vectorstore.ts`:
```typescript
import { createClient } from '@supabase/supabase-js';
import { generateEmbedding, generateEmbeddings } from './embeddings';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

export interface Document {
  content: string;
  metadata?: Record<string, any>;
}

export interface SearchResult {
  id: number;
  content: string;
  metadata: Record<string, any>;
  similarity: number;
}

export async function storeDocuments(documents: Document[]) {
  // Generate embeddings for all documents
  const embeddings = await generateEmbeddings(
    documents.map((d) => d.content)
  );

  // Insert into Supabase
  const rows = documents.map((doc, i) => ({
    content: doc.content,
    metadata: doc.metadata || {},
    embedding: embeddings[i],
  }));

  const { error } = await supabase.from('documents').insert(rows);

  if (error) throw new Error(`Failed to store documents: ${error.message}`);

  return rows.length;
}

export async function searchDocuments(
  query: string,
  options?: { threshold?: number; limit?: number }
): Promise<SearchResult[]> {
  const { threshold = 0.7, limit = 5 } = options || {};

  // Generate embedding for query
  const queryEmbedding = await generateEmbedding(query);

  // Search using Supabase RPC
  const { data, error } = await supabase.rpc('match_documents', {
    query_embedding: queryEmbedding,
    match_threshold: threshold,
    match_count: limit,
  });

  if (error) throw new Error(`Search failed: ${error.message}`);

  return data as SearchResult[];
}
```

### Day 1-2 Checkpoints
- [ ] Supabase project created with pgvector
- [ ] Can generate embeddings via OpenAI
- [ ] Can store documents with embeddings
- [ ] Can search by similarity

---

## Day 3-4: Text Chunking (3-4 hours)

### Goal
Build a robust text chunking system that splits documents intelligently.

### Steps

**1. Build chunker (90 min)**

Create `lib/rag/chunker.ts`:
```typescript
export interface Chunk {
  content: string;
  metadata: {
    source: string;
    chunkIndex: number;
    startChar: number;
    endChar: number;
  };
}

export interface ChunkOptions {
  chunkSize?: number;      // Target chunk size in characters
  chunkOverlap?: number;   // Overlap between chunks
  separator?: string;      // Primary separator
}

/**
 * Split text into overlapping chunks.
 * Uses paragraph boundaries when possible, falls back to sentences.
 */
export function chunkText(
  text: string,
  source: string,
  options?: ChunkOptions
): Chunk[] {
  const {
    chunkSize = 1000,
    chunkOverlap = 200,
    separator = '\n\n',
  } = options || {};

  const chunks: Chunk[] = [];

  // First split by paragraphs
  const paragraphs = text.split(separator).filter((p) => p.trim());

  let currentChunk = '';
  let currentStart = 0;
  let charOffset = 0;

  for (const paragraph of paragraphs) {
    // If adding this paragraph exceeds chunk size, save current chunk
    if (currentChunk.length + paragraph.length > chunkSize && currentChunk.length > 0) {
      chunks.push({
        content: currentChunk.trim(),
        metadata: {
          source,
          chunkIndex: chunks.length,
          startChar: currentStart,
          endChar: charOffset,
        },
      });

      // Start new chunk with overlap
      const overlapText = currentChunk.slice(-chunkOverlap);
      currentChunk = overlapText + separator + paragraph;
      currentStart = charOffset - chunkOverlap;
    } else {
      currentChunk += (currentChunk ? separator : '') + paragraph;
    }

    charOffset += paragraph.length + separator.length;
  }

  // Don't forget the last chunk
  if (currentChunk.trim()) {
    chunks.push({
      content: currentChunk.trim(),
      metadata: {
        source,
        chunkIndex: chunks.length,
        startChar: currentStart,
        endChar: charOffset,
      },
    });
  }

  return chunks;
}

/**
 * Split by sentences for finer-grained chunks.
 */
export function chunkBySentence(
  text: string,
  source: string,
  options?: ChunkOptions
): Chunk[] {
  const { chunkSize = 500, chunkOverlap = 100 } = options || {};

  // Split by sentence boundaries
  const sentences = text.match(/[^.!?]+[.!?]+/g) || [text];

  const chunks: Chunk[] = [];
  let currentChunk = '';
  let currentStart = 0;
  let charOffset = 0;

  for (const sentence of sentences) {
    if (currentChunk.length + sentence.length > chunkSize && currentChunk.length > 0) {
      chunks.push({
        content: currentChunk.trim(),
        metadata: {
          source,
          chunkIndex: chunks.length,
          startChar: currentStart,
          endChar: charOffset,
        },
      });

      const overlapText = currentChunk.slice(-chunkOverlap);
      currentChunk = overlapText + ' ' + sentence;
      currentStart = charOffset - chunkOverlap;
    } else {
      currentChunk += (currentChunk ? ' ' : '') + sentence;
    }

    charOffset += sentence.length;
  }

  if (currentChunk.trim()) {
    chunks.push({
      content: currentChunk.trim(),
      metadata: {
        source,
        chunkIndex: chunks.length,
        startChar: currentStart,
        endChar: charOffset,
      },
    });
  }

  return chunks;
}
```

**2. Test chunking (30 min)**

Create `lib/rag/test-chunker.ts`:
```typescript
import { chunkText } from './chunker';

const sampleText = `
Artificial intelligence (AI) is intelligence demonstrated by machines.
AI research has been defined as the field of study of intelligent agents.

The traditional goals of AI research include reasoning, knowledge
representation, planning, learning, natural language processing,
perception, and support for robotics.

General intelligence is among the field's long-term goals. To solve
these problems, AI researchers have adapted and integrated a wide
range of problem-solving techniques.

Machine learning is a subset of AI that focuses on building systems
that learn from data. Deep learning is a subset of machine learning
that uses neural networks with many layers.
`.trim();

const chunks = chunkText(sampleText, 'test.txt', {
  chunkSize: 200,
  chunkOverlap: 50,
});

chunks.forEach((chunk, i) => {
  console.log(`\n--- Chunk ${i} (${chunk.content.length} chars) ---`);
  console.log(chunk.content);
  console.log('Metadata:', chunk.metadata);
});
```

**3. Build document processor (60 min)**

Create `lib/rag/processor.ts`:
```typescript
import { chunkText, Chunk } from './chunker';
import { storeDocuments } from './vectorstore';

export async function processDocument(
  content: string,
  filename: string,
  options?: { chunkSize?: number; chunkOverlap?: number }
) {
  console.log(`Processing: ${filename} (${content.length} chars)`);

  // 1. Chunk the document
  const chunks = chunkText(content, filename, options);
  console.log(`Created ${chunks.length} chunks`);

  // 2. Store chunks with embeddings
  const documents = chunks.map((chunk) => ({
    content: chunk.content,
    metadata: {
      ...chunk.metadata,
      filename,
    },
  }));

  const stored = await storeDocuments(documents);
  console.log(`Stored ${stored} documents`);

  return { chunks: chunks.length, stored };
}

export async function processTextFile(filePath: string) {
  const fs = await import('fs');
  const path = await import('path');

  const content = fs.readFileSync(filePath, 'utf-8');
  const filename = path.basename(filePath);

  return processDocument(content, filename);
}
```

### Day 3-4 Checkpoints
- [ ] Text chunking works with overlap
- [ ] Can process documents end-to-end
- [ ] Chunks are stored with embeddings in Supabase

---

## Day 5-7: RAG Pipeline (6-8 hours)

### Goal
Build the full RAG pipeline: query → search → context → generate answer with citations.

### Steps

**1. Build retriever with hybrid search (90 min)**

Create `lib/rag/retriever.ts`:
```typescript
import { searchDocuments, SearchResult } from './vectorstore';

export interface RetrievalResult {
  content: string;
  source: string;
  similarity: number;
  chunkIndex: number;
}

/**
 * Retrieve relevant documents for a query.
 * Uses semantic search + keyword filtering.
 */
export async function retrieve(
  query: string,
  options?: {
    limit?: number;
    threshold?: number;
    sourceFilter?: string;
  }
): Promise<RetrievalResult[]> {
  const { limit = 5, threshold = 0.6, sourceFilter } = options || {};

  // Semantic search
  let results = await searchDocuments(query, { threshold, limit: limit * 2 });

  // Filter by source if specified
  if (sourceFilter) {
    results = results.filter((r) =>
      r.metadata.source?.includes(sourceFilter)
    );
  }

  // Deduplicate overlapping chunks
  const deduplicated = deduplicateChunks(results);

  return deduplicated.slice(0, limit).map((r) => ({
    content: r.content,
    source: r.metadata.source || 'unknown',
    similarity: r.similarity,
    chunkIndex: r.metadata.chunkIndex || 0,
  }));
}

function deduplicateChunks(results: SearchResult[]): SearchResult[] {
  const seen = new Set<string>();
  return results.filter((r) => {
    // Create a fingerprint from first 100 chars
    const fingerprint = r.content.slice(0, 100);
    if (seen.has(fingerprint)) return false;
    seen.add(fingerprint);
    return true;
  });
}
```

**2. Build RAG pipeline (90 min)**

Create `lib/rag/pipeline.ts`:
```typescript
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { retrieve, RetrievalResult } from './retriever';

export interface RAGResponse {
  stream: ReadableStream;
  sources: RetrievalResult[];
}

export async function ragQuery(
  question: string,
  conversationHistory?: { role: 'user' | 'assistant'; content: string }[]
): Promise<RAGResponse> {
  // 1. Retrieve relevant documents
  const sources = await retrieve(question, { limit: 5, threshold: 0.5 });

  // 2. Build context from sources
  const context = sources
    .map(
      (s, i) =>
        `[Source ${i + 1}: ${s.source}]\n${s.content}`
    )
    .join('\n\n---\n\n');

  // 3. Build system prompt
  const systemPrompt = `You are a helpful assistant that answers questions based on the provided documents.

RULES:
- Answer ONLY based on the provided context
- If the context doesn't contain enough information, say so
- Always cite your sources using [Source N] format
- Be concise and direct

CONTEXT:
${context}`;

  // 4. Build messages
  const messages = [
    ...(conversationHistory || []),
    { role: 'user' as const, content: question },
  ];

  // 5. Stream response
  const result = await streamText({
    model: anthropic('claude-sonnet-4-20250514'),
    system: systemPrompt,
    messages,
    maxTokens: 1024,
  });

  return {
    stream: result.toDataStream(),
    sources,
  };
}
```

**3. Build API routes (60 min)**

Create `app/api/upload/route.ts`:
```typescript
import { processDocument } from '@/lib/rag/processor';

export async function POST(req: Request) {
  try {
    const formData = await req.formData();
    const file = formData.get('file') as File;

    if (!file) {
      return Response.json({ error: 'No file provided' }, { status: 400 });
    }

    const content = await file.text();
    const result = await processDocument(content, file.name);

    return Response.json({
      message: `Processed ${result.chunks} chunks`,
      ...result,
    });
  } catch (error) {
    console.error('Upload error:', error);
    return Response.json({ error: 'Upload failed' }, { status: 500 });
  }
}
```

Create `app/api/chat/route.ts`:
```typescript
import { ragQuery } from '@/lib/rag/pipeline';

export async function POST(req: Request) {
  try {
    const { messages } = await req.json();
    const lastMessage = messages[messages.length - 1];

    const { stream, sources } = await ragQuery(
      lastMessage.content,
      messages.slice(0, -1)
    );

    // Return stream with sources in header
    return new Response(stream, {
      headers: {
        'Content-Type': 'text/event-stream',
        'X-Sources': JSON.stringify(
          sources.map((s) => ({ source: s.source, similarity: s.similarity }))
        ),
      },
    });
  } catch (error) {
    console.error('Chat error:', error);
    return Response.json({ error: 'Chat failed' }, { status: 500 });
  }
}
```

### Day 5-7 Checkpoints
- [ ] Semantic search returns relevant results
- [ ] RAG pipeline generates accurate answers
- [ ] Source citations work correctly
- [ ] Streaming responses with sources

---

## Day 8-10: Hybrid Search & UI (6-8 hours)

### Goal
Improve search quality with hybrid search and build a polished UI.

### Steps

**1. Add keyword search (60 min)**

Update `lib/rag/retriever.ts` - add keyword search:
```typescript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

export async function keywordSearch(
  query: string,
  limit: number = 10
): Promise<SearchResult[]> {
  // Use Supabase full-text search
  const { data, error } = await supabase
    .from('documents')
    .select('id, content, metadata')
    .textSearch('content', query, { type: 'websearch' })
    .limit(limit);

  if (error) throw error;

  return (data || []).map((d) => ({
    ...d,
    similarity: 0.5, // Default score for keyword matches
  }));
}

/**
 * Hybrid search: combine semantic + keyword results.
 * Uses Reciprocal Rank Fusion (RRF) to merge rankings.
 */
export async function hybridSearch(
  query: string,
  options?: { limit?: number; semanticWeight?: number }
): Promise<RetrievalResult[]> {
  const { limit = 5, semanticWeight = 0.7 } = options || {};
  const keywordWeight = 1 - semanticWeight;

  // Run both searches in parallel
  const [semanticResults, keywordResults] = await Promise.all([
    retrieve(query, { limit: limit * 2 }),
    keywordSearch(query, limit * 2),
  ]);

  // RRF scoring
  const scores = new Map<string, { score: number; result: any }>();
  const k = 60; // RRF constant

  semanticResults.forEach((r, rank) => {
    const key = r.content.slice(0, 100);
    const rrf = semanticWeight * (1 / (k + rank + 1));
    const existing = scores.get(key);
    scores.set(key, {
      score: (existing?.score || 0) + rrf,
      result: r,
    });
  });

  keywordResults.forEach((r, rank) => {
    const key = r.content.slice(0, 100);
    const rrf = keywordWeight * (1 / (k + rank + 1));
    const existing = scores.get(key);
    scores.set(key, {
      score: (existing?.score || 0) + rrf,
      result: existing?.result || {
        content: r.content,
        source: r.metadata?.source || 'unknown',
        similarity: r.similarity,
        chunkIndex: r.metadata?.chunkIndex || 0,
      },
    });
  });

  // Sort by combined score
  return Array.from(scores.values())
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map((s) => s.result);
}
```

**2. Build Upload UI (60 min)**

Create `app/components/FileUpload.tsx`:
```typescript
'use client';
import { useState } from 'react';

export function FileUpload({ onUploadComplete }: { onUploadComplete?: () => void }) {
  const [uploading, setUploading] = useState(false);
  const [result, setResult] = useState<string | null>(null);

  async function handleUpload(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    setUploading(true);
    setResult(null);

    try {
      const formData = new FormData();
      formData.append('file', file);

      const res = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
      });

      const data = await res.json();
      setResult(`Processed ${data.chunks} chunks from ${file.name}`);
      onUploadComplete?.();
    } catch (error) {
      setResult('Upload failed');
    } finally {
      setUploading(false);
    }
  }

  return (
    <div className="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center">
      <input
        type="file"
        accept=".txt,.md,.csv"
        onChange={handleUpload}
        disabled={uploading}
        className="mb-2"
      />
      {uploading && <p className="text-blue-500">Processing...</p>}
      {result && <p className="text-green-600">{result}</p>}
    </div>
  );
}
```

**3. Build Chat with Sources UI (90 min)**

Create `app/components/ChatWithSources.tsx`:
```typescript
'use client';
import { useChat } from 'ai/react';
import { useState, useEffect } from 'react';

interface Source {
  source: string;
  similarity: number;
}

export function ChatWithSources() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat();
  const [sources, setSources] = useState<Source[]>([]);

  return (
    <div className="flex gap-4 h-full">
      {/* Chat area */}
      <div className="flex-1 flex flex-col">
        <div className="flex-1 overflow-y-auto space-y-3 mb-4">
          {messages.map((m) => (
            <div
              key={m.id}
              className={`p-3 rounded-lg ${
                m.role === 'user'
                  ? 'bg-blue-100 ml-auto max-w-[80%]'
                  : 'bg-gray-100 mr-auto max-w-[80%]'
              }`}
            >
              <p className="whitespace-pre-wrap">{m.content}</p>
            </div>
          ))}
          {isLoading && (
            <div className="bg-gray-100 p-3 rounded-lg mr-auto">
              <p className="animate-pulse">Searching documents...</p>
            </div>
          )}
        </div>

        <form onSubmit={handleSubmit} className="flex gap-2">
          <input
            value={input}
            onChange={handleInputChange}
            placeholder="Ask about your documents..."
            className="flex-1 border rounded-lg px-4 py-2"
          />
          <button
            type="submit"
            disabled={isLoading}
            className="bg-blue-500 text-white px-4 py-2 rounded-lg disabled:opacity-50"
          >
            Ask
          </button>
        </form>
      </div>

      {/* Sources panel */}
      <div className="w-64 border-l pl-4">
        <h3 className="font-bold mb-2">Sources</h3>
        {sources.length === 0 ? (
          <p className="text-gray-400 text-sm">
            Sources will appear here after you ask a question
          </p>
        ) : (
          <ul className="space-y-2">
            {sources.map((s, i) => (
              <li key={i} className="text-sm p-2 bg-gray-50 rounded">
                <p className="font-medium">{s.source}</p>
                <p className="text-gray-500">
                  Relevance: {(s.similarity * 100).toFixed(0)}%
                </p>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}
```

### Day 8-10 Checkpoints
- [ ] Hybrid search improves result quality
- [ ] File upload works
- [ ] Chat UI shows streaming answers with source citations
- [ ] Can upload multiple documents and query across them

---

## Final Checkpoints

- [ ] Full RAG pipeline: upload → chunk → embed → store → search → answer
- [ ] Hybrid search (semantic + keyword) working
- [ ] Source citations in answers
- [ ] Streaming responses
- [ ] Can explain RAG architecture in an interview

## Study Resources

- [ ] Read: [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [ ] Read: [Supabase Vector Docs](https://supabase.com/docs/guides/ai)
- [ ] Study: [LangChain RAG](https://js.langchain.com/docs/use_cases/question_answering/)
- [ ] Reverse-engineer: [Vercel AI Chatbot](https://github.com/vercel/ai-chatbot)

## Interview Prep

- Explain: "How does RAG work end-to-end?"
- Explain: "What are different chunking strategies and when to use each?"
- Explain: "How do you improve RAG accuracy? (hybrid search, reranking, chunk size)"
- Explain: "What's the difference between semantic search and keyword search?"
- Design: "How would you build a RAG system for 10M documents?"
