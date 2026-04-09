---
name: openai-integration
description: "Complete OpenAI integration skill: chat completions, streaming, function calling, embeddings, image generation, and best practices. Load for any OpenAI API work."
---

# OpenAI Integration — تكامل OpenAI

## Setup

```typescript
import OpenAI from 'openai'

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  timeout: 30 * 1000,  // 30 seconds
  maxRetries: 2
})
```

## Chat Completions

```typescript
async function chat(messages: OpenAI.Chat.ChatCompletionMessageParam[]) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages,
    temperature: 0.7,
    max_tokens: 2000
  })
  
  return response.choices[0].message.content
}

// Conversation with system prompt
const response = await chat([
  { role: 'system', content: 'You are a helpful coding assistant.' },
  { role: 'user', content: 'Explain async/await in JavaScript' }
])
```

## Streaming Responses

```typescript
// Server-side streaming
async function streamChat(req: Request, res: Response) {
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')
  
  const stream = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: req.body.messages,
    stream: true
  })
  
  for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content
    if (content) {
      res.write(`data: ${JSON.stringify({ content })}\n\n`)
    }
  }
  
  res.write('data: [DONE]\n\n')
  res.end()
}

// Frontend consuming stream
async function streamResponse(messages: Message[]) {
  const response = await fetch('/api/chat/stream', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages })
  })
  
  const reader = response.body!.getReader()
  const decoder = new TextDecoder()
  let fullContent = ''
  
  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    
    const chunk = decoder.decode(value)
    const lines = chunk.split('\n').filter(line => line.startsWith('data: '))
    
    for (const line of lines) {
      const data = line.slice(6)
      if (data === '[DONE]') return fullContent
      
      const parsed = JSON.parse(data)
      fullContent += parsed.content
      onChunk(parsed.content)  // Update UI in real-time
    }
  }
  
  return fullContent
}
```

## Function Calling (Tool Use)

```typescript
const tools: OpenAI.Chat.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'get_weather',
      description: 'Get current weather for a city',
      parameters: {
        type: 'object',
        properties: {
          city: { type: 'string', description: 'City name' },
          unit: { type: 'string', enum: ['celsius', 'fahrenheit'] }
        },
        required: ['city']
      }
    }
  }
]

async function chatWithTools(userMessage: string) {
  const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
    { role: 'user', content: userMessage }
  ]
  
  while (true) {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o',
      messages,
      tools,
      tool_choice: 'auto'
    })
    
    const message = response.choices[0].message
    messages.push(message)
    
    if (!message.tool_calls) {
      return message.content  // No more tool calls, return final answer
    }
    
    // Execute each tool call
    for (const toolCall of message.tool_calls) {
      const args = JSON.parse(toolCall.function.arguments)
      const result = await executeTool(toolCall.function.name, args)
      
      messages.push({
        role: 'tool',
        tool_call_id: toolCall.id,
        content: JSON.stringify(result)
      })
    }
    // Loop continues to get final response
  }
}
```

## Embeddings

```typescript
async function getEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
    encoding_format: 'float'
  })
  return response.data[0].embedding
}

// Batch embeddings (more efficient)
async function getBatchEmbeddings(texts: string[]): Promise<number[][]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: texts  // Up to 2048 inputs at once
  })
  return response.data.map(d => d.embedding)
}

// Cosine similarity for semantic search
function cosineSimilarity(a: number[], b: number[]): number {
  const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0)
  const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0))
  const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0))
  return dotProduct / (magnitudeA * magnitudeB)
}
```

## Structured Output

```typescript
import { zodResponseFormat } from 'openai/helpers/zod'
import { z } from 'zod'

const CodeReviewSchema = z.object({
  summary: z.string(),
  issues: z.array(z.object({
    severity: z.enum(['critical', 'high', 'medium', 'low']),
    type: z.enum(['security', 'performance', 'maintainability', 'bug']),
    line: z.number().optional(),
    description: z.string(),
    fix: z.string()
  })),
  score: z.number().min(0).max(10),
  approved: z.boolean()
})

async function reviewCode(code: string) {
  const response = await openai.beta.chat.completions.parse({
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: 'You are an expert code reviewer.' },
      { role: 'user', content: `Review this code:\n\`\`\`\n${code}\n\`\`\`` }
    ],
    response_format: zodResponseFormat(CodeReviewSchema, 'code_review')
  })
  
  return response.choices[0].message.parsed  // Fully typed!
}
```

## Rate Limiting and Retry

```typescript
import PQueue from 'p-queue'

// Queue to respect rate limits
const queue = new PQueue({
  concurrency: 5,         // Max 5 concurrent requests
  interval: 1000,         // Per second
  intervalCap: 10         // Max 10 requests per second
})

async function rateLimitedEmbed(texts: string[]) {
  return Promise.all(
    texts.map(text => queue.add(() => getEmbedding(text)))
  )
}
```

## Cost Optimization

```typescript
// Track token usage
let totalTokensUsed = 0

const response = await openai.chat.completions.create({ /* ... */ })
totalTokensUsed += response.usage?.total_tokens ?? 0

// Choose the right model
const MODEL_COSTS = {
  'gpt-4o': { input: 0.0025, output: 0.01 },      // Per 1K tokens
  'gpt-4o-mini': { input: 0.00015, output: 0.0006 }, // 15x cheaper!
  'gpt-3.5-turbo': { input: 0.0005, output: 0.0015 }
}

// Use gpt-4o-mini for: classification, extraction, simple Q&A
// Use gpt-4o for: complex reasoning, coding, analysis
```
