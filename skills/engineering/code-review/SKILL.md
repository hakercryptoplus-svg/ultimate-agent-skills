---
name: code-review
description: "Code review mastery skill: what to look for, how to give feedback, security checks, performance issues, and common anti-patterns. Load when reviewing PRs or self-reviewing code."
---

# Code Review Mastery — إتقان مراجعة الكود

## Code Review Mindset

```
Goal: Make the code better, not feel superior
Tone: Collaborative, not critical
Focus: The code, never the person
Be: Specific, constructive, kind

"This function is confusing" ❌
"This function could be clearer — what if we extracted lines 15-22 into getUserPermissions()?" ✅
```

## What to Look For

### 1. Correctness
```
□ Does the code do what it's supposed to do?
□ Are all edge cases handled? (empty arrays, null, negative numbers, etc.)
□ Are error cases handled explicitly?
□ Is there a race condition possible?
□ Can this fail in production in ways not caught in tests?
```

### 2. Security
```typescript
// ❌ Red flags to catch:
const query = `SELECT * FROM users WHERE id = ${req.params.id}`  // SQL injection!
const user = JSON.parse(req.body)                                // No validation
res.json({ ...user, passwordHash: user.passwordHash })           // Leaking secrets!
app.use(cors({ origin: '*' }))                                   // CORS too open
console.log('User logged in:', { email, password })              // Logging passwords!

// Check for:
□ SQL injection (raw queries with user input)
□ XSS (unsanitized user content in HTML)
□ Exposed sensitive data in responses
□ Missing auth checks on routes
□ Hardcoded secrets or API keys
□ Overly permissive CORS
□ User input not validated
```

### 3. Performance
```typescript
// ❌ Performance red flags:
for (const user of users) {
  const posts = await getPosts(user.id)  // N+1 query!
}

await step1()  // Sequential when they could be parallel
await step2()
await step3()

users.filter(u => u.active).map(u => expensiveTransform(u))  // Missing useMemo in React

// Check for:
□ N+1 database queries
□ Missing indexes mentioned in schema changes
□ Synchronous operations that should be async
□ Missing caching for expensive operations
□ Large payload responses (select only needed columns)
□ Missing pagination on list endpoints
```

### 4. Maintainability
```
□ Are functions small and focused?
□ Is the naming clear and descriptive?
□ Is there code duplication that could be extracted?
□ Is the code at a consistent level of abstraction?
□ Are magic numbers/strings replaced with named constants?
□ Is complex logic commented with WHY (not what)?
```

### 5. Tests
```
□ Are tests present for new functionality?
□ Do tests cover happy path AND error cases?
□ Are tests testing behavior, not implementation?
□ Is test code clean and readable?
□ Do tests fail for the right reasons?
```

## Code Review Feedback Templates

### Blocking Issue (Must Fix)
```
🔴 **Security Issue**: SQL injection vulnerability
The query on line 45 uses string concatenation with user input:
  `db.execute(`SELECT * FROM users WHERE id = ${userId}`)`
  
This allows attackers to inject malicious SQL. Fix:
  `db.execute(sql`SELECT * FROM users WHERE id = ${userId}`)`
```

### Improvement Suggestion (Should Fix)
```
🟡 **Performance**: N+1 query detected
In the loop on line 78, you're fetching posts for each user separately.
This makes N+1 database queries. Consider:
  `const users = await db.query.users.findMany({ with: { posts: true } })`
```

### Optional Enhancement (Consider)
```
🟢 **Readability**: Extract constant
The number 86400 appears 3 times. Consider:
  `const SECONDS_PER_DAY = 86_400`
Makes the intent clearer and easier to update.
```

### Praise (Important Too!)
```
✨ **Nice**: Excellent error handling here.
You've handled all the edge cases (null user, expired token, rate limit)
and the error messages are user-friendly without leaking internals.
```

## Self-Review Checklist

Before opening a PR, ask yourself:

```
Functionality:
□ Does it work? Did I test it manually?
□ Did I test edge cases? (empty state, error state, large data)
□ Did I break any existing functionality?

Security:
□ Is all user input validated?
□ Are there no hardcoded secrets?
□ Can unauthorized users access this?
□ Are sensitive fields excluded from responses?

Performance:
□ Are there any N+1 queries?
□ Does this scale to 10x the current load?
□ Are expensive operations cached if appropriate?

Code Quality:
□ Can a junior dev understand this in 5 minutes?
□ Is there code duplication I should extract?
□ Are all magic numbers/strings named?

Tests:
□ Are there tests for the new code?
□ Do the tests cover failure scenarios?
□ Do existing tests still pass?

Documentation:
□ Is the PR description clear?
□ Are complex algorithms commented?
□ Is the API change documented?
```

## Common Anti-Patterns to Catch

```typescript
// ❌ Empty catch blocks
try {
  await doSomething()
} catch (e) {}  // Error silently swallowed

// ❌ Any typing
function process(data: any) { }  // Loses type safety

// ❌ Synchronous file operations in async context
const content = fs.readFileSync('file.txt')  // Blocks event loop

// ❌ Mutating function parameters
function addItem(items: Item[], newItem: Item) {
  items.push(newItem)  // Mutates caller's array!
  return items
}

// ❌ Not awaiting promises
function saveUser(user: User) {
  db.insert(users).values(user)  // Promise not awaited!
  return user
}

// ❌ Boolean parameter flags (code smell)
function getUsers(includeInactive: boolean) { }  // Better: 2 separate functions

// ❌ Returning null from a function with non-null return type
// Use: User | null in the return type, or throw
```
