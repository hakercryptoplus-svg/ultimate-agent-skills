---
name: streaming
description: "Complete streaming skill for AI applications: Server-Sent Events (SSE), WebSockets, streaming LLM responses, and real-time UI updates. Load when implementing any streaming feature."
---

# Streaming — البث الفوري للبيانات

## Choosing the Right Protocol

```
SSE (Server-Sent Events):
✅ Simple — HTTP based, automatic reconnect
✅ One-directional: server → client
✅ Great for: AI responses, live feeds, notifications
❌ Client can't send after connection (use REST for that)

WebSocket:
✅ Bidirectional — client and server both send
✅ Great for: chat, live collaboration, games
❌ More complex — need to handle reconnection
❌ More server resources

Long Polling:
✅ Works everywhere, simple
❌ Slower, more server load
❌ Avoid if SSE or WebSocket work
```

## SSE — Server-Sent Events

### Express Backend

```typescript
// routes/stream.ts
router.post('/chat/stream', requireAuth, async (req, res) => {
  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')
  res.flushHeaders()
  
  const sendEvent = (data: unknown) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`)
  }
  
  const sendError = (message: string) => {
    res.write(`event: error\ndata: ${JSON.stringify({ message })}\n\n`)
    res.end()
  }
  
  // Handle client disconnect
  req.on('close', () => {
    // Clean up any resources (cancel AI request, etc.)
  })
  
  try {
    const stream = await anthropic.messages.create({
      model: 'claude-sonnet-4-5',
      max_tokens: 2000,
      stream: true,
      messages: req.body.messages
    })
    
    for await (const event of stream) {
      if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
        sendEvent({ type: 'text', content: event.delta.text })
      }
      
      if (event.type === 'message_stop') {
        sendEvent({ type: 'done' })
        res.end()
      }
    }
  } catch (error) {
    sendError('Failed to generate response')
  }
})
```

### React Frontend

```typescript
// hooks/useStreamingChat.ts
import { useState, useCallback, useRef } from 'react'

interface Message {
  role: 'user' | 'assistant'
  content: string
}

export function useStreamingChat() {
  const [messages, setMessages] = useState<Message[]>([])
  const [isStreaming, setIsStreaming] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const abortControllerRef = useRef<AbortController | null>(null)
  
  const sendMessage = useCallback(async (userMessage: string) => {
    const newMessages = [...messages, { role: 'user' as const, content: userMessage }]
    setMessages(newMessages)
    setIsStreaming(true)
    setError(null)
    
    // Add empty assistant message to start filling
    setMessages(prev => [...prev, { role: 'assistant', content: '' }])
    
    abortControllerRef.current = new AbortController()
    
    try {
      const response = await fetch('/api/chat/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ messages: newMessages }),
        signal: abortControllerRef.current.signal
      })
      
      if (!response.ok) throw new Error('Failed to connect')
      
      const reader = response.body!.getReader()
      const decoder = new TextDecoder()
      let buffer = ''
      
      while (true) {
        const { done, value } = await reader.read()
        if (done) break
        
        buffer += decoder.decode(value, { stream: true })
        const lines = buffer.split('\n')
        buffer = lines.pop() ?? ''  // Keep incomplete line in buffer
        
        for (const line of lines) {
          if (!line.startsWith('data: ')) continue
          const data = line.slice(6)
          
          try {
            const parsed = JSON.parse(data)
            
            if (parsed.type === 'text') {
              // Append to the last (assistant) message
              setMessages(prev => {
                const updated = [...prev]
                const last = updated[updated.length - 1]
                if (last.role === 'assistant') {
                  updated[updated.length - 1] = {
                    ...last,
                    content: last.content + parsed.content
                  }
                }
                return updated
              })
            }
            
            if (parsed.type === 'done') {
              setIsStreaming(false)
            }
          } catch {
            // Ignore parse errors
          }
        }
      }
    } catch (err) {
      if ((err as Error).name !== 'AbortError') {
        setError('Failed to get response')
        // Remove the empty assistant message
        setMessages(prev => prev.slice(0, -1))
      }
    } finally {
      setIsStreaming(false)
    }
  }, [messages])
  
  const stop = useCallback(() => {
    abortControllerRef.current?.abort()
    setIsStreaming(false)
  }, [])
  
  return { messages, isStreaming, error, sendMessage, stop }
}
```

### Chat UI Component

```tsx
// components/Chat.tsx
export function Chat() {
  const { messages, isStreaming, error, sendMessage, stop } = useStreamingChat()
  const [input, setInput] = useState('')
  const messagesEndRef = useRef<HTMLDivElement>(null)
  
  // Auto-scroll to bottom
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!input.trim() || isStreaming) return
    const message = input.trim()
    setInput('')
    await sendMessage(message)
  }
  
  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto">
      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg, i) => (
          <div key={i} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`max-w-[80%] rounded-lg px-4 py-2 ${
              msg.role === 'user' ? 'bg-blue-500 text-white' : 'bg-gray-100'
            }`}>
              {msg.content}
              {isStreaming && i === messages.length - 1 && msg.role === 'assistant' && (
                <span className="animate-pulse">▋</span>  // Typing cursor
              )}
            </div>
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>
      
      {/* Error */}
      {error && <div className="text-red-500 px-4">{error}</div>}
      
      {/* Input */}
      <form onSubmit={handleSubmit} className="p-4 border-t flex gap-2">
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          disabled={isStreaming}
          placeholder="Type a message..."
          className="flex-1 border rounded-lg px-4 py-2"
        />
        {isStreaming ? (
          <button type="button" onClick={stop} className="px-4 py-2 bg-red-500 text-white rounded-lg">
            Stop
          </button>
        ) : (
          <button type="submit" disabled={!input.trim()} className="px-4 py-2 bg-blue-500 text-white rounded-lg">
            Send
          </button>
        )}
      </form>
    </div>
  )
}
```

## WebSocket with Socket.io

```typescript
// Server
import { Server } from 'socket.io'

const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL }
})

io.on('connection', (socket) => {
  console.log(`User connected: ${socket.id}`)
  
  socket.on('join-room', (roomId: string) => {
    socket.join(roomId)
    socket.to(roomId).emit('user-joined', { socketId: socket.id })
  })
  
  socket.on('send-message', (data: { roomId: string; message: string }) => {
    // Broadcast to all in room except sender
    socket.to(data.roomId).emit('new-message', {
      from: socket.id,
      message: data.message,
      timestamp: new Date().toISOString()
    })
  })
  
  socket.on('disconnect', () => {
    console.log(`User disconnected: ${socket.id}`)
  })
})

// Client
import { io } from 'socket.io-client'

const socket = io(process.env.NEXT_PUBLIC_WS_URL!)

socket.on('connect', () => console.log('Connected'))
socket.on('new-message', (data) => updateUI(data))
socket.emit('send-message', { roomId: 'room-123', message: 'Hello!' })
```
