---
name: security
description: "Complete security skill covering OWASP Top 10, input validation, SQL injection, XSS, CSRF, authentication hardening, secrets management, and security headers. Load for any security review or implementation."
---

# Security — تأمين التطبيقات (OWASP Top 10)

## The Security Mindset

```
Assume:
- Every input from users is malicious until proven safe
- Attackers will find any exposed endpoint
- Secrets will leak if not properly protected
- Your app will be attacked — design for it

Defense in depth:
- Multiple layers of protection
- Fail securely (deny by default)
- Least privilege principle
- Audit everything sensitive
```

## OWASP Top 10 — 2024

### A01: Broken Access Control
```typescript
// ✅ Always verify ownership before allowing access
router.get('/posts/:id', requireAuth, async (req, res) => {
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, req.params.id)
  })
  
  if (!post) return res.status(404).json({ error: 'Not found' })
  
  // CRITICAL: Verify the user owns this resource
  if (post.userId !== req.user.id && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Forbidden' })
  }
  
  res.json(post)
})

// ❌ BROKEN: Missing ownership check
router.delete('/posts/:id', requireAuth, async (req, res) => {
  await db.delete(posts).where(eq(posts.id, req.params.id))
  // Any authenticated user can delete any post!
  res.status(204).send()
})
```

### A02: Cryptographic Failures
```typescript
// ✅ Never store plain passwords
const hash = await bcrypt.hash(password, 12)  // cost=12 minimum

// ✅ Use strong random tokens
const token = crypto.randomBytes(32).toString('hex')

// ✅ Encrypt sensitive data at rest
import { encrypt, decrypt } from './encryption'
const encryptedSSN = encrypt(ssn, process.env.ENCRYPTION_KEY!)

// ❌ NEVER:
const hash = md5(password)     // MD5 is broken
const token = Math.random()    // Predictable!
const stored = { ssn: ssn }    // Plain PII in DB
```

### A03: Injection (SQL, NoSQL, Command)
```typescript
// ✅ Drizzle ORM is safe by default (parameterized)
const user = await db.query.users.findFirst({
  where: eq(users.email, userInput)  // Safe
})

// ✅ If using raw SQL, always parameterize
const result = await db.execute(
  sql`SELECT * FROM users WHERE email = ${userInput}`  // Safe
)

// ❌ NEVER concatenate user input into SQL
const query = `SELECT * FROM users WHERE email = '${userInput}'`
// SQL injection vulnerability!

// ✅ Prevent command injection
import { execFile } from 'child_process'
// Use execFile (not exec) and separate args
execFile('convert', ['-resize', '100x100', inputPath, outputPath])
```

### A04: Insecure Design
```typescript
// ✅ Rate limit sensitive endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true
})

// ✅ Same error for wrong email OR wrong password
if (!user || !await bcrypt.compare(password, user.passwordHash)) {
  throw new Error('Invalid email or password')  // Don't reveal which
}

// ✅ Account lockout after failed attempts
if (user.failedLoginAttempts >= 5) {
  if (user.lockedUntil && user.lockedUntil > new Date()) {
    throw new Error('Account temporarily locked')
  }
}
```

### A05: Security Misconfiguration
```typescript
// ✅ Set all security headers
import helmet from 'helmet'
app.use(helmet())

// ✅ Disable unnecessary features
app.disable('x-powered-by')  // Don't reveal Express

// ✅ Strict CORS
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
  credentials: true
}))

// ✅ Never expose stack traces in production
app.use((err, req, res, next) => {
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' })
  } else {
    res.status(500).json({ error: err.message, stack: err.stack })
  }
})
```

### A07: Authentication Failures
```typescript
// ✅ Enforce strong passwords
const PasswordSchema = z.string()
  .min(8, 'At least 8 characters')
  .regex(/[A-Z]/, 'At least one uppercase letter')
  .regex(/[0-9]/, 'At least one number')
  .regex(/[^A-Za-z0-9]/, 'At least one special character')

// ✅ Use secure, httpOnly cookies for refresh tokens (alternative to body)
res.cookie('refreshToken', refreshToken, {
  httpOnly: true,   // Not accessible via JavaScript
  secure: true,     // HTTPS only
  sameSite: 'strict',
  maxAge: 30 * 24 * 60 * 60 * 1000  // 30 days
})

// ✅ Invalidate all sessions on password change
await db.delete(refreshTokens).where(eq(refreshTokens.userId, userId))
```

## Security Headers Checklist

```typescript
// With helmet, all these are set automatically:
// Content-Security-Policy
// X-Content-Type-Options: nosniff
// X-Frame-Options: DENY
// X-XSS-Protection: 1; mode=block
// Strict-Transport-Security (HTTPS only)
// Referrer-Policy

// Custom CSP
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'", process.env.API_URL!],
    fontSrc: ["'self'", 'https://fonts.gstatic.com'],
    objectSrc: ["'none'"],
    frameSrc: ["'none'"]
  }
}))
```

## Secrets Management

```bash
# ✅ Use environment variables — never hardcode
JWT_SECRET=generated-with-openssl-rand-base64-32

# ✅ Generate strong secrets
openssl rand -base64 32

# ✅ Different secrets per environment
# .env.development, .env.production — never commit either

# ❌ NEVER in code:
const SECRET = 'mysecret123'           # Hardcoded
const TOKEN = process.env.TOKEN ?? ''  # Empty fallback
```

```typescript
// ✅ Validate all required env vars on startup
function validateEnv() {
  const required = ['DATABASE_URL', 'JWT_SECRET', 'PORT']
  const missing = required.filter(key => !process.env[key])
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`)
  }
}

validateEnv()  // Called before app.listen()
```

## Security Code Review Checklist

```
Before every PR, verify:
□ No secrets or credentials in code
□ All user inputs validated and sanitized
□ Parameterized queries used everywhere
□ Access control checks on every protected route
□ Error messages don't leak system details
□ Sensitive data not logged
□ HTTPS enforced in production
□ Rate limiting on public endpoints
□ File uploads validated (type + size)
□ Dependencies updated (no known CVEs)
```

## Input Sanitization

```typescript
import DOMPurify from 'isomorphic-dompurify'

// For HTML content from users
const sanitizedHtml = DOMPurify.sanitize(userHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
  ALLOWED_ATTR: ['href', 'title']
})

// For file uploads
const allowedMimeTypes = ['image/jpeg', 'image/png', 'image/webp']
const maxSize = 5 * 1024 * 1024  // 5MB

if (!allowedMimeTypes.includes(file.mimetype)) {
  throw new Error('File type not allowed')
}
if (file.size > maxSize) {
  throw new Error('File too large')
}
// Also verify magic bytes, not just mime type
```
