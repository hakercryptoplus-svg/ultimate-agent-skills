---
name: testing
description: "Complete testing skill covering unit tests, integration tests, E2E tests with Playwright, API testing, and TDD methodology. Load when writing or fixing tests."
---

# Testing Excellence — الاختبار الشامل

## Testing Pyramid

```
         E2E Tests (Playwright)
        /    Few, slow, catch user flows   \
       /───────────────────────────────────\
      /  Integration Tests (Supertest)      \
     /    Medium count, test API contracts   \
    /─────────────────────────────────────────\
   /   Unit Tests (Vitest)                     \
  /    Many, fast, test functions in isolation  \
 /─────────────────────────────────────────────\
```

## Unit Testing with Vitest

```typescript
// user.service.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { UserService } from './user.service'
import { db } from '../db'

// Mock external dependencies
vi.mock('../db', () => ({
  db: {
    query: {
      users: {
        findFirst: vi.fn()
      }
    }
  }
}))

describe('UserService', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })
  
  describe('getById', () => {
    it('returns user when found', async () => {
      const mockUser = { id: '123', email: 'test@example.com', name: 'Test User' }
      vi.mocked(db.query.users.findFirst).mockResolvedValue(mockUser)
      
      const result = await UserService.getById('123')
      
      expect(result).toEqual(mockUser)
      expect(db.query.users.findFirst).toHaveBeenCalledWith({
        where: expect.any(Function)
      })
    })
    
    it('throws NotFoundError when user does not exist', async () => {
      vi.mocked(db.query.users.findFirst).mockResolvedValue(null)
      
      await expect(UserService.getById('nonexistent')).rejects.toThrow('User not found')
    })
    
    it('handles database errors gracefully', async () => {
      vi.mocked(db.query.users.findFirst).mockRejectedValue(new Error('DB connection failed'))
      
      await expect(UserService.getById('123')).rejects.toThrow()
    })
  })
})
```

## Integration Testing with Supertest

```typescript
// auth.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest'
import request from 'supertest'
import { app } from '../app'
import { db } from '../db'
import { users, refreshTokens } from '../db/schema'

describe('Auth API', () => {
  beforeEach(async () => {
    // Clean state before each test
    await db.delete(refreshTokens)
    await db.delete(users)
  })

  describe('POST /api/auth/register', () => {
    it('creates user and returns tokens', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({ email: 'test@example.com', password: 'Password123!', name: 'Test User' })
      
      expect(response.status).toBe(201)
      expect(response.body).toMatchObject({
        user: { email: 'test@example.com', name: 'Test User' },
        accessToken: expect.any(String),
        refreshToken: expect.any(String)
      })
      // Passwords must never be in response
      expect(response.body.user.passwordHash).toBeUndefined()
    })
    
    it('returns 409 for duplicate email', async () => {
      await request(app)
        .post('/api/auth/register')
        .send({ email: 'test@example.com', password: 'Password123!', name: 'User 1' })
      
      const response = await request(app)
        .post('/api/auth/register')
        .send({ email: 'test@example.com', password: 'Password123!', name: 'User 2' })
      
      expect(response.status).toBe(409)
      expect(response.body.code).toBe('EMAIL_TAKEN')
    })
    
    it('returns 422 for invalid email', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({ email: 'not-an-email', password: 'Password123!', name: 'Test' })
      
      expect(response.status).toBe(422)
      expect(response.body.details).toContainEqual(
        expect.objectContaining({ field: 'email' })
      )
    })
  })
  
  describe('Authenticated Routes', () => {
    let accessToken: string
    
    beforeEach(async () => {
      const registerRes = await request(app)
        .post('/api/auth/register')
        .send({ email: 'user@example.com', password: 'Password123!', name: 'Test' })
      accessToken = registerRes.body.accessToken
    })
    
    it('returns 401 without token', async () => {
      const response = await request(app).get('/api/users/me')
      expect(response.status).toBe(401)
    })
    
    it('returns user data with valid token', async () => {
      const response = await request(app)
        .get('/api/users/me')
        .set('Authorization', `Bearer ${accessToken}`)
      
      expect(response.status).toBe(200)
      expect(response.body.email).toBe('user@example.com')
    })
  })
})
```

## E2E Testing with Playwright

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication Flow', () => {
  test('user can register and access dashboard', async ({ page }) => {
    // Visit registration page
    await page.goto('/register')
    
    // Fill form
    await page.fill('[placeholder="Your name"]', 'Ahmed Ali')
    await page.fill('[placeholder="Email address"]', 'ahmed@example.com')
    await page.fill('[placeholder="Password"]', 'SecurePass123!')
    await page.fill('[placeholder="Confirm password"]', 'SecurePass123!')
    
    // Submit
    await page.click('button[type="submit"]')
    
    // Should redirect to dashboard
    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('h1')).toContainText('Welcome, Ahmed')
  })
  
  test('user sees error for wrong credentials', async ({ page }) => {
    await page.goto('/login')
    await page.fill('[placeholder="Email"]', 'wrong@example.com')
    await page.fill('[placeholder="Password"]', 'wrongpassword')
    await page.click('button[type="submit"]')
    
    await expect(page.locator('[role="alert"]')).toBeVisible()
    await expect(page.locator('[role="alert"]')).toContainText('Invalid email or password')
    await expect(page).toHaveURL('/login')  // Stayed on login page
  })
})
```

## Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node',
    globals: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80
      },
      exclude: ['**/node_modules/**', '**/dist/**', '**/*.d.ts', '**/migrations/**']
    },
    setupFiles: ['./tests/setup.ts']
  }
})
```

```typescript
// tests/setup.ts
import { beforeAll, afterAll } from 'vitest'
import { db } from '../src/db'

beforeAll(async () => {
  // Run migrations on test database
  await runMigrations()
})

afterAll(async () => {
  // Clean up
  await db.$client.end()
})
```

## Testing Best Practices

```
Test naming: describe what it does, not how
✅ 'returns 404 when user not found'
❌ 'test getUser with null'

AAA Pattern in every test:
// ARRANGE
const mockUser = { id: '1', email: 'test@example.com' }

// ACT
const result = await UserService.getById('1')

// ASSERT
expect(result).toEqual(mockUser)

Test coverage targets:
- Unit tests: 80%+ coverage
- Integration tests: all API endpoints
- E2E tests: critical user journeys only

What to test:
✅ Happy path (everything works)
✅ Error cases (not found, invalid input, etc.)
✅ Edge cases (empty arrays, null, boundary values)
✅ Security cases (unauthorized access, injection attempts)
❌ Don't test implementation details
❌ Don't test third-party libraries
```
