---
  name: api-design
  description: "Complete guide for designing and building professional REST APIs. Covers routing conventions, versioning, validation, error handling, pagination, and documentation. Load for any backend API work."
  ---

  # API Design & Build — تصميم وبناء APIs احترافية

  ## REST Conventions

  ```
  Resource naming (always plural, lowercase, kebab-case):
  GET    /api/users              — list all users
  POST   /api/users              — create user
  GET    /api/users/:id          — get one user
  PUT    /api/users/:id          — replace user
  PATCH  /api/users/:id          — update user partially
  DELETE /api/users/:id          — delete user

  Nested resources:
  GET    /api/users/:id/posts    — get user's posts
  POST   /api/users/:id/posts    — create post for user

  Actions (when CRUD doesn't fit):
  POST   /api/users/:id/verify   — send verification email
  POST   /api/auth/refresh       — refresh token
  POST   /api/orders/:id/cancel  — cancel order
  ```

  ## HTTP Status Codes

  ```
  200 OK            — GET, PATCH success (with body)
  201 Created       — POST success (return created resource)
  204 No Content    — DELETE success (no body)
  400 Bad Request   — Malformed request / invalid input
  401 Unauthorized  — Missing or invalid authentication
  403 Forbidden     — Authenticated but not authorized
  404 Not Found     — Resource doesn't exist
  409 Conflict      — Duplicate (email already exists, etc.)
  422 Unprocessable — Validation failed (wrong data format)
  429 Too Many Req  — Rate limit exceeded
  500 Server Error  — Unexpected server failure
  503 Unavailable   — Service temporarily down
  ```

  ## Standard Response Format

  ```typescript
  // Success responses
  interface ApiSuccess<T> {
    data: T;
    meta?: {
      page?: number;
      limit?: number;
      total?: number;
      totalPages?: number;
    };
  }

  // Error responses
  interface ApiError {
    error: string;           // Human-readable message
    code?: string;           // Machine-readable code (e.g., "USER_NOT_FOUND")
    details?: {              // Validation errors
      field: string;
      message: string;
    }[];
    requestId?: string;      // For debugging
  }

  // Examples:
  // GET /users/123 — not found
  {
    "error": "User not found",
    "code": "USER_NOT_FOUND",
    "requestId": "req_abc123"
  }

  // POST /users — validation failed
  {
    "error": "Validation failed",
    "code": "VALIDATION_ERROR",
    "details": [
      { "field": "email", "message": "Must be a valid email address" },
      { "field": "password", "message": "Must be at least 8 characters" }
    ]
  }
  ```

  ## Validation Middleware with Zod

  ```typescript
  import { z, ZodSchema } from 'zod/v4'
  import { Request, Response, NextFunction } from 'express'

  export function validateBody<T>(schema: ZodSchema<T>) {
    return (req: Request, res: Response, next: NextFunction) => {
      const result = schema.safeParse(req.body)
      
      if (!result.success) {
        return res.status(422).json({
          error: 'Validation failed',
          code: 'VALIDATION_ERROR',
          details: result.error.issues.map(issue => ({
            field: issue.path.join('.'),
            message: issue.message
          }))
        })
      }
      
      req.body = result.data  // Use parsed (clean) data
      next()
    }
  }

  export function validateQuery<T>(schema: ZodSchema<T>) {
    return (req: Request, res: Response, next: NextFunction) => {
      const result = schema.safeParse(req.query)
      if (!result.success) {
        return res.status(422).json({ error: 'Invalid query params', details: result.error.issues })
      }
      req.query = result.data as any
      next()
    }
  }
  ```

  ## Pagination Pattern

  ```typescript
  const PaginationSchema = z.object({
    page:   z.coerce.number().int().min(1).default(1),
    limit:  z.coerce.number().int().min(1).max(100).default(20),
    sort:   z.enum(['createdAt', 'name', 'email']).default('createdAt'),
    order:  z.enum(['asc', 'desc']).default('desc')
  })

  router.get('/', validateQuery(PaginationSchema), async (req, res) => {
    const { page, limit, sort, order } = req.query as PaginationParams
    const offset = (page - 1) * limit
    
    const [items, total] = await Promise.all([
      db.query.users.findMany({
        limit,
        offset,
        orderBy: order === 'desc' ? desc(users[sort]) : asc(users[sort])
      }),
      db.select({ count: count() }).from(users).then(r => Number(r[0].count))
    ])
    
    res.json({
      data: items,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit)
      }
    })
  })
  ```

  ## Rate Limiting

  ```typescript
  import rateLimit from 'express-rate-limit'

  // General API rate limit
  export const apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                   // 100 requests per window
    standardHeaders: true,
    legacyHeaders: false,
    message: { error: 'Too many requests', code: 'RATE_LIMITED' }
  })

  // Strict limit for auth endpoints
  export const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,  // Only 5 login attempts per 15 minutes
    skipSuccessfulRequests: true,  // Don't count successful logins
    message: { error: 'Too many login attempts', code: 'AUTH_RATE_LIMITED' }
  })

  // Apply
  app.use('/api/', apiLimiter)
  app.use('/api/auth/login', authLimiter)
  app.use('/api/auth/register', authLimiter)
  ```

  ## Auth Middleware

  ```typescript
  import jwt from 'jsonwebtoken'

  export function requireAuth(req: Request, res: Response, next: NextFunction) {
    const token = req.headers.authorization?.replace('Bearer ', '')
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required', code: 'MISSING_TOKEN' })
    }
    
    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload
      req.user = { id: payload.sub, email: payload.email, role: payload.role }
      next()
    } catch (err) {
      if (err instanceof jwt.TokenExpiredError) {
        return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' })
      }
      return res.status(401).json({ error: 'Invalid token', code: 'INVALID_TOKEN' })
    }
  }

  export function requireRole(...roles: string[]) {
    return (req: Request, res: Response, next: NextFunction) => {
      if (!req.user || !roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Insufficient permissions', code: 'FORBIDDEN' })
      }
      next()
    }
  }

  // Usage:
  router.delete('/:id', requireAuth, requireRole('admin'), deleteUser)
  ```

  ## Request Logging

  ```typescript
  import { randomUUID } from 'crypto'

  app.use((req, res, next) => {
    const requestId = randomUUID()
    const start = Date.now()
    
    req.id = requestId
    res.setHeader('X-Request-ID', requestId)
    
    res.on('finish', () => {
      const duration = Date.now() - start
      logger.info('Request completed', {
        requestId,
        method: req.method,
        path: req.path,
        status: res.statusCode,
        duration: `${duration}ms`,
        userAgent: req.headers['user-agent'],
        ip: req.ip
      })
    })
    
    next()
  })
  ```

  ## OpenAPI/Swagger Documentation

  ```typescript
  import swaggerJsdoc from 'swagger-jsdoc'
  import swaggerUi from 'swagger-ui-express'

  const spec = swaggerJsdoc({
    definition: {
      openapi: '3.0.0',
      info: {
        title: 'My API',
        version: '1.0.0',
        description: 'API documentation'
      },
      components: {
        securitySchemes: {
          bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' }
        }
      }
    },
    apis: ['./src/api/routes/*.ts']
  })

  app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(spec))
  ```
  