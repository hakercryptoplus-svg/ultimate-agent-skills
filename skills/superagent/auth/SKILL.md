---
  name: auth
  description: "Complete authentication and authorization skill. Covers JWT, refresh tokens, sessions, OAuth, role-based access, and security best practices. Load for any auth implementation."
  ---

  # Auth Systems — أنظمة المصادقة الكاملة

  ## JWT Auth Flow

  ```
  Registration:
  User → POST /auth/register → hash password → create user → return tokens

  Login:
  User → POST /auth/login → verify password → return access + refresh token

  Access Protected Route:
  User → GET /api/data + Authorization: Bearer <access_token>
       → middleware verifies JWT → req.user populated → handler runs

  Token Refresh (when access token expires):
  User → POST /auth/refresh + { refreshToken }
       → verify refresh token in DB → rotate tokens → return new pair

  Logout:
  User → POST /auth/logout → delete refresh token from DB
  ```

  ## Complete Auth Service

  ```typescript
  import bcrypt from 'bcrypt'
  import jwt from 'jsonwebtoken'
  import crypto from 'crypto'
  import { db } from '../db'
  import { users, refreshTokens } from '../db/schema'
  import { eq, and, gt } from 'drizzle-orm'

  const BCRYPT_ROUNDS = 12
  const ACCESS_TTL = '15m'
  const REFRESH_TTL = 30 * 24 * 60 * 60 * 1000  // 30 days in ms

  interface TokenPair {
    accessToken: string
    refreshToken: string
  }

  interface AuthUser {
    id: string
    email: string
    name: string
    role: string
  }

  export class AuthService {
    
    static async register(email: string, password: string, name: string): Promise<{ user: AuthUser } & TokenPair> {
      // Check uniqueness
      const exists = await db.query.users.findFirst({ where: eq(users.email, email.toLowerCase()) })
      if (exists) throw Object.assign(new Error('Email already registered'), { statusCode: 409, code: 'EMAIL_TAKEN' })
      
      // Create user
      const [user] = await db.insert(users).values({
        email: email.toLowerCase(),
        name,
        passwordHash: await bcrypt.hash(password, BCRYPT_ROUNDS)
      }).returning({ id: users.id, email: users.email, name: users.name, role: users.role })
      
      const tokens = await this.generateTokens(user)
      return { user, ...tokens }
    }
    
    static async login(email: string, password: string): Promise<{ user: AuthUser } & TokenPair> {
      const user = await db.query.users.findFirst({
        where: eq(users.email, email.toLowerCase())
      })
      
      // Use constant-time comparison (bcrypt) to prevent timing attacks
      const passwordValid = user && await bcrypt.compare(password, user.passwordHash)
      
      // Same error for wrong email OR wrong password (don't reveal which)
      if (!user || !passwordValid) {
        throw Object.assign(new Error('Invalid email or password'), { statusCode: 401, code: 'INVALID_CREDENTIALS' })
      }
      
      await db.update(users).set({ lastLoginAt: new Date() }).where(eq(users.id, user.id))
      
      const safeUser = { id: user.id, email: user.email, name: user.name, role: user.role }
      const tokens = await this.generateTokens(safeUser)
      return { user: safeUser, ...tokens }
    }
    
    static async refresh(token: string): Promise<TokenPair> {
      const stored = await db.query.refreshTokens.findFirst({
        where: and(
          eq(refreshTokens.token, token),
          gt(refreshTokens.expiresAt, new Date())
        ),
        with: { user: true }
      })
      
      if (!stored) throw Object.assign(new Error('Invalid or expired refresh token'), { statusCode: 401, code: 'INVALID_REFRESH' })
      
      // Rotate: delete old, create new (prevents reuse)
      await db.delete(refreshTokens).where(eq(refreshTokens.id, stored.id))
      
      const user = { id: stored.user.id, email: stored.user.email, name: stored.user.name, role: stored.user.role }
      return this.generateTokens(user)
    }
    
    static async logout(token: string): Promise<void> {
      await db.delete(refreshTokens).where(eq(refreshTokens.token, token))
    }
    
    private static async generateTokens(user: AuthUser): Promise<TokenPair> {
      const accessToken = jwt.sign(
        { sub: user.id, email: user.email, role: user.role },
        process.env.JWT_SECRET!,
        { expiresIn: ACCESS_TTL, issuer: 'myapp' }
      )
      
      const refreshToken = crypto.randomBytes(40).toString('hex')
      
      await db.insert(refreshTokens).values({
        userId: user.id,
        token: refreshToken,
        expiresAt: new Date(Date.now() + REFRESH_TTL)
      })
      
      return { accessToken, refreshToken }
    }
  }
  ```

  ## Auth Middleware

  ```typescript
  import jwt from 'jsonwebtoken'
  import { Request, Response, NextFunction } from 'express'

  interface JwtPayload {
    sub: string
    email: string
    role: string
    iat: number
    exp: number
  }

  declare global {
    namespace Express {
      interface Request {
        user?: { id: string; email: string; role: string }
      }
    }
  }

  export function requireAuth(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization
    
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'Authentication required', code: 'MISSING_TOKEN' })
    }
    
    const token = authHeader.slice(7)
    
    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET!, {
        issuer: 'myapp'
      }) as JwtPayload
      
      req.user = { id: payload.sub, email: payload.email, role: payload.role }
      next()
    } catch (err) {
      if (err instanceof jwt.TokenExpiredError) {
        return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' })
      }
      return res.status(401).json({ error: 'Invalid token', code: 'INVALID_TOKEN' })
    }
  }

  export function optionalAuth(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization
    if (!authHeader?.startsWith('Bearer ')) return next()
    
    try {
      const payload = jwt.verify(authHeader.slice(7), process.env.JWT_SECRET!) as JwtPayload
      req.user = { id: payload.sub, email: payload.email, role: payload.role }
    } catch {
      // Token invalid — continue as unauthenticated
    }
    next()
  }

  export function requireRole(...roles: string[]) {
    return (req: Request, res: Response, next: NextFunction) => {
      if (!req.user) return res.status(401).json({ error: 'Authentication required' })
      if (!roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Insufficient permissions', code: 'FORBIDDEN' })
      }
      next()
    }
  }
  ```

  ## Frontend Auth Hook

  ```typescript
  // hooks/useAuth.ts
  import { create } from 'zustand'
  import { persist } from 'zustand/middleware'
  import { api } from '@/lib/api'

  interface AuthState {
    user: User | null
    accessToken: string | null
    refreshToken: string | null
    login: (email: string, password: string) => Promise<void>
    logout: () => Promise<void>
    refresh: () => Promise<string | null>
  }

  export const useAuth = create<AuthState>()(
    persist(
      (set, get) => ({
        user: null,
        accessToken: null,
        refreshToken: null,
        
        login: async (email, password) => {
          const { data } = await api.post('/auth/login', { email, password })
          set({ user: data.user, accessToken: data.accessToken, refreshToken: data.refreshToken })
        },
        
        logout: async () => {
          try {
            await api.post('/auth/logout', { refreshToken: get().refreshToken })
          } finally {
            set({ user: null, accessToken: null, refreshToken: null })
          }
        },
        
        refresh: async () => {
          const { refreshToken } = get()
          if (!refreshToken) return null
          
          try {
            const { data } = await api.post('/auth/refresh', { refreshToken })
            set({ accessToken: data.accessToken, refreshToken: data.refreshToken })
            return data.accessToken
          } catch {
            set({ user: null, accessToken: null, refreshToken: null })
            return null
          }
        }
      }),
      {
        name: 'auth-store',
        partialize: (state) => ({
          user: state.user,
          refreshToken: state.refreshToken
          // Don't persist accessToken — get fresh one on load
        })
      }
    )
  )

  // Auto-refresh on 401 responses
  api.interceptors.response.use(
    (response) => response,
    async (error) => {
      if (error.response?.status === 401 && error.config.url !== '/auth/refresh') {
        const newToken = await useAuth.getState().refresh()
        if (newToken) {
          error.config.headers.Authorization = `Bearer ${newToken}`
          return api.request(error.config)
        }
      }
      return Promise.reject(error)
    }
  )
  ```

  ## Password Reset Flow

  ```typescript
  // 1. Request reset
  router.post('/forgot-password', async (req, res) => {
    const { email } = req.body
    const user = await db.query.users.findFirst({ where: eq(users.email, email) })
    
    // Always return success (don't reveal if email exists)
    if (user) {
      const token = crypto.randomBytes(32).toString('hex')
      const expires = new Date(Date.now() + 60 * 60 * 1000)  // 1 hour
      
      await db.insert(passwordResets).values({ userId: user.id, token, expiresAt: expires })
      await sendEmail(email, 'Reset your password', resetEmailTemplate(token))
    }
    
    res.json({ message: 'If that email exists, you will receive a reset link.' })
  })

  // 2. Verify and reset
  router.post('/reset-password', async (req, res) => {
    const { token, newPassword } = req.body
    
    const reset = await db.query.passwordResets.findFirst({
      where: and(eq(passwordResets.token, token), gt(passwordResets.expiresAt, new Date())),
      with: { user: true }
    })
    
    if (!reset) return res.status(400).json({ error: 'Invalid or expired reset token' })
    
    await Promise.all([
      db.update(users)
        .set({ passwordHash: await bcrypt.hash(newPassword, 12) })
        .where(eq(users.id, reset.userId)),
      db.delete(passwordResets).where(eq(passwordResets.userId, reset.userId)),
      db.delete(refreshTokens).where(eq(refreshTokens.userId, reset.userId))
      // Delete all sessions on password reset
    ])
    
    res.json({ message: 'Password reset successfully. Please log in again.' })
  })
  ```
  