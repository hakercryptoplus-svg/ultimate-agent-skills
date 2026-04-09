---
name: anthropic-sdk
description: "Anthropic Claude SDK usage skill: messages API, streaming, tool use, vision, and best practices for building Claude-powered applications."
---

# Anthropic SDK — استخدام Claude في تطبيقاتك

## Setup

```typescript
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
  timeout: 60 * 1000,  // 60 seconds for longer tasks
  maxRetries: 2
})
```

## Basic Messages

```typescript
// Simple message
const message = await anthropic.messages.create({
  model: 'claude-opus-4-5',
  max_tokens: 1024,
  messages: [
    { role: 'user', content: 'Write a TypeScript function to reverse a string' }
  ]
})

console.log(message.content[0].text)

// With system prompt
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-5',  // claude-sonnet-4-5 for most tasks, claude-opus-4-5 for complex
  max_tokens: 2048,
  system: 'You are a senior TypeScript developer. Write clean, typed, production-ready code.',
  messages: [
    { role: 'user', content: 'Build a retry utility function with exponential backoff' }
  ]
})
```

## Streaming

```typescript
// Stream in Node.js
async function streamResponse(prompt: string) {
  const stream = await anthropic.messages.create({
    model: 'claude-sonnet-4-5',
    max_tokens: 2000,
    stream: true,
    messages: [{ role: 'user', content: prompt }]
  })
  
  let fullText = ''
  
  for await (const event of stream) {
    if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
      fullText += event.delta.text
      process.stdout.write(event.delta.text)  // Real-time output
    }
  }
  
  return fullText
}

// Express SSE streaming endpoint
app.post('/api/chat/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  
  const stream = anthropic.messages.stream({
    model: 'claude-sonnet-4-5',
    max_tokens: 2000,
    messages: req.body.messages
  })
  
  stream.on('text', (text) => {
    res.write(`data: ${JSON.stringify({ text })}\n\n`)
  })
  
  await stream.finalMessage()
  res.write('data: [DONE]\n\n')
  res.end()
})
```

## Tool Use

```typescript
const tools: Anthropic.Messages.Tool[] = [
  {
    name: 'search_database',
    description: 'Search the product database for items matching a query',
    input_schema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Search query' },
        category: { type: 'string', enum: ['electronics', 'clothing', 'books'] },
        maxResults: { type: 'number', default: 10 }
      },
      required: ['query']
    }
  }
]

async function agentLoop(userMessage: string) {
  const messages: Anthropic.Messages.MessageParam[] = [
    { role: 'user', content: userMessage }
  ]
  
  while (true) {
    const response = await anthropic.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 4096,
      tools,
      messages
    })
    
    messages.push({ role: 'assistant', content: response.content })
    
    if (response.stop_reason === 'end_turn') {
      // Extract final text response
      const textBlock = response.content.find(b => b.type === 'text')
      return textBlock?.text ?? ''
    }
    
    if (response.stop_reason === 'tool_use') {
      const toolResults: Anthropic.Messages.ToolResultBlockParam[] = []
      
      for (const block of response.content) {
        if (block.type === 'tool_use') {
          const result = await executeTool(block.name, block.input)
          toolResults.push({
            type: 'tool_result',
            tool_use_id: block.id,
            content: JSON.stringify(result)
          })
        }
      }
      
      messages.push({ role: 'user', content: toolResults })
    }
  }
}
```

## Vision (Image Analysis)

```typescript
// Analyze image from URL
const response = await anthropic.messages.create({
  model: 'claude-opus-4-5',
  max_tokens: 1024,
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'image',
          source: {
            type: 'url',
            url: 'https://example.com/diagram.png'
          }
        },
        {
          type: 'text',
          text: 'Describe this architecture diagram and identify any issues'
        }
      ]
    }
  ]
})

// Analyze local image (base64)
import { readFileSync } from 'fs'

const imageData = readFileSync('./screenshot.png').toString('base64')

const response = await anthropic.messages.create({
  model: 'claude-opus-4-5',
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
            data: imageData
          }
        },
        { type: 'text', text: 'What bugs do you see in this UI?' }
      ]
    }
  ]
})
```

## Model Selection Guide

```
claude-opus-4-5:
  Use for: Complex reasoning, nuanced analysis, sophisticated coding
  Cost: Highest
  Speed: Slower

claude-sonnet-4-5:
  Use for: Most tasks — good balance of quality and speed
  Cost: Medium
  Speed: Fast
  
claude-haiku-3-5:
  Use for: Simple classification, extraction, fast responses
  Cost: Cheapest
  Speed: Fastest

Decision rule:
- Default: claude-sonnet-4-5
- Need best quality: claude-opus-4-5
- Need speed/low cost: claude-haiku-3-5
```

## Rate Limiting and Error Handling

```typescript
async function robustClaudeCall(params: Anthropic.Messages.MessageCreateParams) {
  const maxRetries = 3
  let lastError: Error
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await anthropic.messages.create(params)
    } catch (error) {
      lastError = error as Error
      
      if (error instanceof Anthropic.RateLimitError) {
        const waitTime = Math.pow(2, attempt) * 1000  // Exponential backoff
        await new Promise(r => setTimeout(r, waitTime))
        continue
      }
      
      if (error instanceof Anthropic.APIError && error.status >= 500) {
        await new Promise(r => setTimeout(r, 2000))
        continue
      }
      
      throw error  // Don't retry client errors (4xx)
    }
  }
  
  throw lastError!
}
```
