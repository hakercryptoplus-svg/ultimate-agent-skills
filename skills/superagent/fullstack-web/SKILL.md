---
  name: fullstack-web
  description: "Complete skill for building full-stack web applications from scratch. Covers React frontend, Express/Node backend, PostgreSQL database, API design, auth, and deployment. Load this for any web project."
  ---

  # Full-Stack Web Development — بناء مواقع كاملة

  ## Project Setup Checklist

  ```
  New project checklist:
  □ Initialize pnpm monorepo (or single package)
  □ Configure TypeScript (strict mode)
  □ Setup ESLint + Prettier
  □ Create folder structure
  □ Setup environment variables (.env.example)
  □ Initialize git with .gitignore
  □ Configure database connection
  □ Create base API skeleton
  □ Setup health check endpoint
  ```

  ## Recommended Stack (2025)

  ```
  Frontend:   React 18+ + Vite + TypeScript
  Styling:    Tailwind CSS + shadcn/ui
  State:      TanStack Query (server) + Zustand (client)
  Router:     React Router 6+ or TanStack Router
  Backend:    Node.js + Express 5 + TypeScript
  Validation: Zod (shared between frontend/backend)
  Database:   PostgreSQL + Drizzle ORM
  Auth:       JWT + refresh tokens, or Clerk
  Testing:    Vitest + Playwright
  Deploy:     Docker + any cloud
  ```

  ## Folder Structure

  ```
  project/
  ├── src/
  │   ├── api/                 # Express routes
  │   │   ├── routes/
  │   │   │   ├── auth.ts
  │   │   │   ├── users.ts
  │   │   │   └── index.ts
  │   │   └── middleware/
  │   │       ├── auth.ts      # JWT verification
  │   │       ├── validate.ts  # Zod validation
  │   │       └── rateLimit.ts
  │   ├── services/            # Business logic
  │   │   ├── auth.service.ts
  │   │   └── user.service.ts
  │   ├── db/
  │   │   ├── schema.ts        # Drizzle schema
  │   │   ├── migrations/
  │   │   └── index.ts         # DB connection
  │   ├── lib/
  │   │   ├── jwt.ts
  │   │   ├── hash.ts
  │   │   └── email.ts
  │   └── types/
  │       └── index.ts         # Shared types
  └── client/
      ├── src/
      │   ├── pages/
      │   ├── components/
      │   ├── hooks/
      │   ├── lib/
      │   └── App.tsx
      └── vite.config.ts
  ```

  ## API Design Pattern

  ```typescript
  // Every route follows this pattern:
  import { Router } from 'express'
  import { z } from 'zod/v4'
  import { validateBody, requireAuth } from '../middleware'
  import { UserService } from '../services/user.service'

  const router = Router()

  // Schema (can be shared with frontend)
  const CreateUserSchema = z.object({
    email: z.string().email(),
    name: z.string().min(2).max(100),
    password: z.string().min(8)
  })

  // Route
  router.post('/', validateBody(CreateUserSchema), async (req, res) => {
    const user = await UserService.create(req.body)
    res.status(201).json(user)
  })

  export default router
  ```

  ## Database Schema Pattern

  ```typescript
  // Drizzle ORM schema
  import { pgTable, uuid, text, timestamp, boolean } from 'drizzle-orm/pg-core'

  export const users = pgTable('users', {
    id:           uuid('id').primaryKey().defaultRandom(),
    email:        text('email').notNull().unique(),
    name:         text('name').notNull(),
    passwordHash: text('password_hash').notNull(),
    role:         text('role', { enum: ['user', 'admin'] }).default('user'),
    verified:     boolean('verified').default(false),
    createdAt:    timestamp('created_at').defaultNow(),
    updatedAt:    timestamp('updated_at').defaultNow()
  })

  export const sessions = pgTable('sessions', {
    id:           uuid('id').primaryKey().defaultRandom(),
    userId:       uuid('user_id').references(() => users.id, { onDelete: 'cascade' }),
    refreshToken: text('refresh_token').notNull().unique(),
    expiresAt:    timestamp('expires_at').notNull(),
    createdAt:    timestamp('created_at').defaultNow()
  })
  ```

  ## Auth Implementation

  ```typescript
  // Complete JWT auth service
  import jwt from 'jsonwebtoken'
  import bcrypt from 'bcrypt'
  import { db } from '../db'
  import { users, sessions } from '../db/schema'

  const BCRYPT_ROUNDS = 12
  const ACCESS_TOKEN_EXPIRY = '15m'
  const REFRESH_TOKEN_EXPIRY = '7d'

  export class AuthService {
    static async register(email: string, password: string, name: string) {
      // Check if user exists
      const existing = await db.query.users.findFirst({ where: eq(users.email, email) })
      if (existing) throw new ConflictError('Email already registered')
      
      // Hash password
      const passwordHash = await bcrypt.hash(password, BCRYPT_ROUNDS)
      
      // Create user
      const [user] = await db.insert(users).values({ email, name, passwordHash }).returning()
      
      return this.createTokens(user)
    }
    
    static async login(email: string, password: string) {
      const user = await db.query.users.findFirst({ where: eq(users.email, email) })
      if (!user) throw new UnauthorizedError('Invalid credentials')
      
      const valid = await bcrypt.compare(password, user.passwordHash)
      if (!valid) throw new UnauthorizedError('Invalid credentials')
      
      return this.createTokens(user)
    }
    
    private static async createTokens(user: User) {
      const accessToken = jwt.sign(
        { sub: user.id, email: user.email, role: user.role },
        process.env.JWT_SECRET!,
        { expiresIn: ACCESS_TOKEN_EXPIRY }
      )
      
      const refreshToken = crypto.randomUUID()
      
      // Store refresh token
      await db.insert(sessions).values({
        userId: user.id,
        refreshToken,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
      })
      
      return {
        accessToken,
        refreshToken,
        user: { id: user.id, email: user.email, name: user.name }
      }
    }
  }
  ```

  ## React Frontend Pattern

  ```typescript
  // Data fetching with TanStack Query
  import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
  import { api } from '@/lib/api'

  // Query hook
  export function useUsers() {
    return useQuery({
      queryKey: ['users'],
      queryFn: () => api.get('/users').then(r => r.data)
    })
  }

  // Mutation hook
  export function useCreateUser() {
    const queryClient = useQueryClient()
    
    return useMutation({
      mutationFn: (data: CreateUserInput) => 
        api.post('/users', data).then(r => r.data),
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['users'] })
      }
    })
  }

  // Component usage
  function UserList() {
    const { data: users, isLoading, error } = useUsers()
    const createUser = useCreateUser()
    
    if (isLoading) return <Spinner />
    if (error) return <ErrorMessage error={error} />
    
    return (
      <div>
        {users.map(user => <UserCard key={user.id} user={user} />)}
        <CreateUserForm onSubmit={createUser.mutate} />
      </div>
    )
  }
  ```

  ## Environment Variables

  ```bash
  # .env.example — commit this, never commit .env
  DATABASE_URL=postgresql://user:password@localhost:5432/dbname
  JWT_SECRET=your-super-secret-key-min-32-chars
  JWT_REFRESH_SECRET=another-secret-key-min-32-chars
  PORT=3000
  NODE_ENV=development
  CORS_ORIGIN=http://localhost:5173

  # Email (optional)
  SMTP_HOST=smtp.gmail.com
  SMTP_PORT=587
  SMTP_USER=your@email.com
  SMTP_PASS=app-specific-password

  # Redis (optional, for caching)
  REDIS_URL=redis://localhost:6379
  ```

  ## Error Handling Middleware

  ```typescript
  // Global error handler — add LAST in Express
  app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
    // Known app errors
    if (err instanceof AppError) {
      return res.status(err.statusCode).json({
        error: err.message,
        code: err.code
      })
    }
    
    // Zod validation errors
    if (err instanceof ZodError) {
      return res.status(422).json({
        error: 'Validation failed',
        details: err.issues.map(i => ({
          field: i.path.join('.'),
          message: i.message
        }))
      })
    }
    
    // Unexpected errors — don't leak details
    req.log.error('Unhandled error', { error: err.message, stack: err.stack })
    res.status(500).json({ error: 'Internal server error' })
  })
  ```

  ## CORS Configuration

  ```typescript
  import cors from 'cors'

  app.use(cors({
    origin: process.env.CORS_ORIGIN?.split(',') ?? ['http://localhost:5173'],
    credentials: true,  // Required for cookies
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization']
  }))
  ```

  ## Security Headers

  ```typescript
  import helmet from 'helmet'

  app.use(helmet())  // Sets 11 security headers automatically
  app.use(helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    }
  }))
  ```

  ## Health Check Endpoint (Always Include)

  ```typescript
  router.get('/healthz', async (req, res) => {
    const checks = {
      status: 'ok',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version,
      database: 'checking...',
      uptime: process.uptime()
    }
    
    try {
      await db.execute(sql`SELECT 1`)
      checks.database = 'connected'
    } catch {
      checks.database = 'disconnected'
      return res.status(503).json(checks)
    }
    
    res.json(checks)
  })
  ```
  