---
name: typescript
description: "TypeScript excellence skill: strict typing, generics, utility types, type guards, and advanced patterns. Load for any TypeScript project to ensure type safety."
---

# TypeScript Excellence — التميز في TypeScript

## Always Use Strict Mode

```json
// tsconfig.json — non-negotiable settings
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

## Type Inference vs Explicit Types

```typescript
// ✅ Let TypeScript infer simple cases
const count = 42                           // inferred: number
const name = 'Alice'                       // inferred: string
const isActive = true                      // inferred: boolean
const items = [1, 2, 3]                    // inferred: number[]

// ✅ Be explicit for function signatures and public APIs
function getUserById(id: string): Promise<User | null> {
  return db.query.users.findFirst({ where: eq(users.id, id) })
}

// ✅ Be explicit when inference would be wrong/ambiguous
const config: Record<string, unknown> = {}
const items: string[] = []  // Not [] (inferred as never[])
```

## Generics — Power Types

```typescript
// Generic function
function first<T>(arr: T[]): T | undefined {
  return arr[0]
}

// Generic with constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

// Generic Result type (avoid throwing everywhere)
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E }

async function safeGetUser(id: string): Promise<Result<User>> {
  try {
    const user = await getUserById(id)
    if (!user) return { success: false, error: new Error('Not found') }
    return { success: true, data: user }
  } catch (error) {
    return { success: false, error: error as Error }
  }
}

// Usage — exhaustive handling
const result = await safeGetUser(id)
if (result.success) {
  console.log(result.data.email)  // TypeScript knows data exists
} else {
  console.error(result.error.message)  // TypeScript knows error exists
}
```

## Utility Types (Built-in)

```typescript
interface User {
  id: string
  email: string
  name: string
  passwordHash: string
  role: 'user' | 'admin'
  createdAt: Date
}

// Partial — all fields optional
type UpdateUserInput = Partial<Pick<User, 'name' | 'email'>>

// Required — all fields required
type RequiredUser = Required<User>

// Pick — select specific fields
type PublicUser = Pick<User, 'id' | 'email' | 'name' | 'role'>

// Omit — exclude specific fields
type CreateUserInput = Omit<User, 'id' | 'createdAt'>

// Record — key-value pairs
type RolePermissions = Record<User['role'], string[]>

// Readonly — prevent mutation
type ImmutableUser = Readonly<User>

// ReturnType — extract function return type
type GetUsersReturn = ReturnType<typeof getUsers>

// Awaited — unwrap Promise
type ResolvedUser = Awaited<ReturnType<typeof getUserById>>
```

## Type Guards

```typescript
// Type predicate
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value &&
    typeof (value as User).email === 'string'
  )
}

// Discriminated union with type guard
type ApiResult<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; message: string }

function handleResult<T>(result: ApiResult<T>) {
  if (result.status === 'success') {
    return result.data  // TypeScript knows this is T
  }
  throw new Error(result.message)  // TypeScript knows message exists
}

// Assertion function
function assertNonNull<T>(value: T, name: string): asserts value is NonNullable<T> {
  if (value === null || value === undefined) {
    throw new Error(`Expected ${name} to be defined, got ${value}`)
  }
}

const user = await getUserById(id)
assertNonNull(user, 'user')
// After this line, TypeScript knows user is not null
console.log(user.email)
```

## Enums vs Union Types

```typescript
// ✅ Use const enum for performance
const enum Direction {
  Up = 'UP',
  Down = 'DOWN',
  Left = 'LEFT',
  Right = 'RIGHT'
}

// ✅ Or use union types (simpler, better for serialization)
type Status = 'pending' | 'active' | 'suspended' | 'deleted'
type Role = 'user' | 'admin' | 'moderator'

// ✅ Extract the type from an object for the best of both worlds
const STATUS = {
  PENDING: 'pending',
  ACTIVE: 'active',
  SUSPENDED: 'suspended'
} as const

type Status = typeof STATUS[keyof typeof STATUS]
// Status = 'pending' | 'active' | 'suspended'
```

## Branded Types (Advanced)

```typescript
// Prevent mixing up IDs of different types
type UserId = string & { readonly _brand: 'UserId' }
type PostId = string & { readonly _brand: 'PostId' }

function makeUserId(id: string): UserId {
  return id as UserId
}

function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

const userId = makeUserId('abc123')
const postId = 'def456' as PostId

getUser(userId)   // ✅ Works
getUser(postId)   // ❌ TypeScript error: PostId is not UserId
```

## Error Handling Types

```typescript
// Custom error classes with TypeScript
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
    public readonly details?: unknown
  ) {
    super(message)
    this.name = 'AppError'
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, 'NOT_FOUND', 404)
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(message, 'UNAUTHORIZED', 401)
  }
}

class ValidationError extends AppError {
  constructor(details: { field: string; message: string }[]) {
    super('Validation failed', 'VALIDATION_ERROR', 422, details)
  }
}
```
