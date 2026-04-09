---
name: rag
description: "Complete RAG (Retrieval-Augmented Generation) implementation skill. Covers document ingestion, chunking, embeddings, vector storage, retrieval, and generation. Load when building AI search or Q&A systems."
---

# RAG Systems — بناء أنظمة RAG

## What is RAG?

```
RAG = Retrieval-Augmented Generation

User Question
      ↓
Embed the question (convert to vector)
      ↓
Search vector DB for similar chunks
      ↓
Retrieve top-K relevant chunks
      ↓
Pass chunks + question to LLM
      ↓
LLM generates answer grounded in retrieved context
      ↓
Answer with citations
```

## Document Ingestion Pipeline

```typescript
interface Document {
  id: string
  source: string  // filename, URL, etc.
  content: string
  metadata: Record<string, unknown>
}

interface Chunk {
  id: string
  documentId: string
  content: string
  embedding: number[]
  chunkIndex: number
  metadata: Record<string, unknown>
}

async function ingestDocument(doc: Document): Promise<void> {
  // 1. Split into chunks
  const chunks = splitIntoChunks(doc.content, {
    chunkSize: 1000,
    chunkOverlap: 200,
    separator: '\n\n'  // Split on paragraphs
  })
  
  // 2. Generate embeddings in batch
  const embeddings = await getBatchEmbeddings(chunks.map(c => c.content))
  
  // 3. Store in vector database
  await vectorStore.upsert(
    chunks.map((chunk, i) => ({
      id: `${doc.id}-chunk-${i}`,
      embedding: embeddings[i],
      metadata: {
        content: chunk.content,
        documentId: doc.id,
        source: doc.source,
        chunkIndex: i,
        ...doc.metadata
      }
    }))
  )
}

function splitIntoChunks(text: string, options: {
  chunkSize: number
  chunkOverlap: number
  separator: string
}): { content: string; index: number }[] {
  const { chunkSize, chunkOverlap, separator } = options
  const paragraphs = text.split(separator).filter(p => p.trim())
  const chunks: { content: string; index: number }[] = []
  
  let currentChunk = ''
  let chunkIndex = 0
  
  for (const paragraph of paragraphs) {
    if ((currentChunk + paragraph).length > chunkSize && currentChunk) {
      chunks.push({ content: currentChunk.trim(), index: chunkIndex++ })
      // Keep overlap from previous chunk
      currentChunk = currentChunk.slice(-chunkOverlap) + '\n\n' + paragraph
    } else {
      currentChunk += (currentChunk ? '\n\n' : '') + paragraph
    }
  }
  
  if (currentChunk) {
    chunks.push({ content: currentChunk.trim(), index: chunkIndex })
  }
  
  return chunks
}
```

## Query and Retrieval

```typescript
async function retrieve(query: string, topK = 5): Promise<RetrievedChunk[]> {
  // 1. Embed the query
  const queryEmbedding = await getEmbedding(query)
  
  // 2. Search for similar chunks
  const results = await vectorStore.query({
    embedding: queryEmbedding,
    topK,
    filter: { /* optional metadata filter */ }
  })
  
  // 3. Return with scores
  return results.map(r => ({
    content: r.metadata.content as string,
    source: r.metadata.source as string,
    score: r.score,
    metadata: r.metadata
  }))
}
```

## Generation with Context

```typescript
async function ragQuery(question: string): Promise<{
  answer: string
  sources: string[]
}> {
  // 1. Retrieve relevant context
  const chunks = await retrieve(question, 5)
  
  // 2. Filter low-quality results
  const relevantChunks = chunks.filter(c => c.score > 0.7)
  
  if (relevantChunks.length === 0) {
    return {
      answer: "I don't have enough information to answer this question.",
      sources: []
    }
  }
  
  // 3. Build context string
  const context = relevantChunks
    .map((chunk, i) => `[Source ${i + 1}: ${chunk.source}]\n${chunk.content}`)
    .join('\n\n---\n\n')
  
  // 4. Generate answer
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content: `You are a helpful assistant. Answer questions based ONLY on the provided context.
If the context doesn't contain enough information, say so explicitly.
Always cite your sources using [Source N] references.`
      },
      {
        role: 'user',
        content: `Context:\n${context}\n\nQuestion: ${question}`
      }
    ]
  })
  
  return {
    answer: response.choices[0].message.content ?? '',
    sources: [...new Set(relevantChunks.map(c => c.source))]
  }
}
```

## Vector Database Setup (Pinecone)

```typescript
import { Pinecone } from '@pinecone-database/pinecone'

const pinecone = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY!
})

const index = pinecone.index('my-rag-index')

// Upsert vectors
await index.upsert([
  {
    id: 'chunk-1',
    values: embedding,         // number[] from embeddings API
    metadata: {
      content: 'The text content...',
      source: 'document.pdf',
      page: 1
    }
  }
])

// Query
const results = await index.query({
  vector: queryEmbedding,
  topK: 5,
  includeMetadata: true,
  filter: { source: { $eq: 'document.pdf' } }  // Optional filter
})
```

## RAG Quality Tips

```
Improve retrieval quality:
1. Chunk size matters: 500-1500 tokens is usually optimal
2. Add overlap (200 tokens) to preserve context across chunks
3. Clean text before embedding (remove headers, footers, tables)
4. Hybrid search: combine semantic (embedding) + keyword (BM25) search
5. Reranking: use a cross-encoder to rerank top results

Improve generation quality:
1. Include source attribution in every answer
2. Set a confidence threshold — refuse to answer if score too low
3. Use structured output to enforce citation format
4. Include conversation history for follow-up questions
5. Validate answers against retrieved context (hallucination check)
```
