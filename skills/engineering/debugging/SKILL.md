---
name: debugging
description: "Systematic debugging skill: root cause analysis, debugging strategies, reading stack traces, using debuggers, logging, and common bug patterns. Load when investigating bugs."
---

# Debugging Mastery — إتقان البحث عن الأخطاء

## The Debugging Mindset

```
Debugging is science — not guessing
1. Observe the symptoms clearly
2. Form a hypothesis (one at a time)
3. Test the hypothesis
4. Learn from results
5. Repeat until root cause found

Golden rule: Understand WHY it's broken, not just how to make it stop crashing
```

## Systematic Debugging Process

```
1. Reproduce the bug consistently
   - Get exact steps to reproduce
   - Identify what data triggers it
   - Determine if it's consistent or intermittent

2. Isolate the problem
   - Binary search through the code
   - Eliminate possibilities one by one
   - Create a minimal reproduction case

3. Understand the root cause
   - Don't fix symptoms, fix the disease
   - Read the full stack trace
   - Add logging to confirm assumptions

4. Fix and verify
   - Make the smallest change needed
   - Test the fix works
   - Test you haven't broken anything else

5. Prevent recurrence
   - Add a test that would have caught this
   - Fix similar patterns elsewhere
   - Document if non-obvious
```

## Reading Stack Traces

```typescript
// This error trace:
Error: Cannot read property 'email' of undefined
    at UserController.getProfile (/app/src/controllers/user.controller.ts:45:29)
    at Layer.handle [as handle_request] (/app/node_modules/express/lib/router/layer.js:95:5)
    at next (/app/node_modules/express/lib/router/route.js:137:13)

// Tells you:
// 1. The error: trying to access .email on undefined value
// 2. Where: user.controller.ts, line 45, column 29
// 3. The call stack: your code -> express router
// 4. Root cause: user is undefined at line 45

// How to read:
// Read from top to bottom
// First line = the error
// Next lines = call stack (top = where error occurred)
// Ignore library frames (node_modules) initially
// Focus on YOUR code lines
```

## Common Bug Patterns and Fixes

### Async/Await Bugs

```typescript
// Bug 1: Forgetting await
async function getUser(id: string) {
  const user = db.query.users.findFirst(...)  // BUG: Promise not awaited!
  return user  // Returns Promise object, not the user
}

// Fix: Always await async operations
const user = await db.query.users.findFirst(...)

// Bug 2: Unhandled promise rejection
function doWork() {
  fetchData()    // BUG: Not awaited, error will be unhandled!
  return 'done'
}

// Fix: Either await it or catch it
async function doWork() {
  try {
    await fetchData()
  } catch (error) {
    logger.error('fetchData failed', error)
  }
}
```

### TypeScript/JavaScript Type Bugs

```typescript
// Bug 1: Loose equality with type coercion
if (userId == null) { }   // BUG: catches both null AND undefined AND 0 AND ''
// Fix: Use strict equality
if (userId === null || userId === undefined) { }
// Or: if (userId == null) { }  — actually okay for null + undefined check

// Bug 2: Array from undefined
const users = null
const emails = users.map(u => u.email)  // BUG: Cannot read properties of null
// Fix: Guard against null/undefined
const emails = users?.map(u => u.email) ?? []

// Bug 3: parseInt without radix
parseInt('09')     // BUG: In older environments, 09 parsed as octal!
parseInt('09', 10) // Fix: Always specify radix 10
```

### Race Conditions

```typescript
// Bug: Multiple async operations modifying shared state
let count = 0

async function increment() {
  const current = count        // Read
  await delay(10)              // This creates a race window!
  count = current + 1          // Write — may overwrite another increment
}

// Fix: Use atomic operations or mutex
import { Mutex } from 'async-mutex'
const mutex = new Mutex()

async function increment() {
  await mutex.runExclusive(async () => {
    count += 1
  })
}
```

## Debugging Tools

### Console Debugging

```typescript
// More than just console.log
console.log({ user, token })    // Object inspection
console.table(users)            // Format arrays as table
console.time('query')           // Start timer
const result = await query()
console.timeEnd('query')        // End timer: "query: 245ms"
console.trace('Where am I?')    // Print call stack
console.group('User Auth')      // Group related logs
console.log('Step 1')
console.groupEnd()

// Add context to logs
const logger = {
  info: (msg: string, ctx?: object) => console.log(JSON.stringify({ level: 'info', msg, ...ctx, ts: new Date().toISOString() })),
  error: (msg: string, ctx?: object) => console.error(JSON.stringify({ level: 'error', msg, ...ctx, ts: new Date().toISOString() }))
}

// Use structured logging in production
logger.info('User logged in', { userId: user.id, ip: req.ip })
```

### Node.js Debugger

```bash
# Start with debugger
node --inspect-brk src/index.ts

# Then open in Chrome:
chrome://inspect

# Or use VS Code launch.json:
{
  "type": "node",
  "request": "launch",
  "name": "Debug API",
  "program": "${workspaceFolder}/src/index.ts",
  "runtimeArgs": ["--loader", "tsx"],
  "sourceMaps": true
}
```

### Database Query Debugging

```typescript
// Log all queries in development
const db = drizzle(pool, {
  logger: process.env.NODE_ENV === 'development' ? {
    logQuery: (query, params) => {
      console.log('SQL:', query)
      console.log('Params:', params)
    }
  } : undefined
})
```

## Production Debugging

```typescript
// Structured error logging for production
import { createLogger } from 'winston'

const logger = createLogger({
  format: winston.format.json(),
  transports: [
    new winston.transports.Console()
  ]
})

// Log errors with full context
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error('Unhandled error', {
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
    userId: req.user?.id,
    requestId: req.headers['x-request-id'],
    body: req.body  // Be careful with sensitive data
  })
  
  res.status(500).json({ error: 'Internal server error' })
})

// Correlation IDs to trace requests
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] ?? crypto.randomUUID()
  res.setHeader('x-request-id', req.id)
  next()
})
```

## Debugging Checklist

```
When stuck on a bug:
□ Can you reproduce it consistently?
□ What's the exact error message + stack trace?
□ What were the INPUTS when it failed?
□ What is the EXPECTED behavior?
□ When did it last work? What changed?
□ Have you checked the logs?
□ Have you added logging to confirm your assumptions?
□ Have you tried isolating the problem to a minimal case?
□ Have you searched the codebase for similar patterns?
□ Have you read the docs for the library involved?
□ Have you asked: "What's the SIMPLEST explanation?"
```
