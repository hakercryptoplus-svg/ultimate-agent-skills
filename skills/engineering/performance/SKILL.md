---
name: performance
description: "Frontend and backend performance optimization skill. Covers bundle optimization, lazy loading, caching, database query optimization, CDN, and performance monitoring."
---

# Performance — تحسين الأداء

## Core Web Vitals Targets

```
LCP (Largest Contentful Paint): < 2.5s  — loading performance
FID (First Input Delay):        < 100ms — interactivity
CLS (Cumulative Layout Shift):  < 0.1   — visual stability
TTFB (Time to First Byte):      < 600ms — server response
```

## Frontend Performance

### Bundle Optimization

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-react': ['react', 'react-dom', 'react-router-dom'],
          'vendor-ui': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'vendor-query': ['@tanstack/react-query'],
          'vendor-charts': ['recharts']
        }
      }
    },
    chunkSizeWarningLimit: 500  // Warn if chunk > 500KB
  }
})
```

### Code Splitting

```typescript
// Route-level code splitting
import { lazy, Suspense } from 'react'

const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))
const Reports = lazy(() => import('./pages/Reports'))

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/reports" element={<Reports />} />
      </Routes>
    </Suspense>
  )
}
```

### Image Optimization

```tsx
// Use modern formats + lazy loading
function OptimizedImage({ src, alt, width, height }: ImageProps) {
  return (
    <picture>
      <source srcSet={src.replace('.jpg', '.webp')} type="image/webp" />
      <img
        src={src}
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
        decoding="async"
        style={{ aspectRatio: `${width}/${height}` }}  // Prevent CLS
      />
    </picture>
  )
}
```

### Memoization

```typescript
// Memoize expensive computations
const sortedAndFilteredItems = useMemo(() => {
  return items
    .filter(item => item.status === 'active')
    .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
    .slice(0, 100)
}, [items])

// Stable callbacks (prevent unnecessary re-renders)
const handleSubmit = useCallback(
  async (data: FormData) => {
    await submitForm(data)
    onSuccess?.()
  },
  [submitForm, onSuccess]
)

// Memoize components
const UserCard = memo(function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>
}, (prev, next) => prev.user.id === next.user.id)
```

## Backend Performance

### Database Query Optimization

```typescript
// ❌ N+1 Query problem
const users = await db.query.users.findMany()
for (const user of users) {
  // This fires one query PER USER = N+1 queries total
  const posts = await db.query.posts.findMany({
    where: eq(posts.userId, user.id)
  })
}

// ✅ Single query with eager loading
const users = await db.query.users.findMany({
  with: { posts: true }  // JOIN — single query
})

// ✅ Select only what you need
const users = await db.query.users.findMany({
  columns: {
    id: true,
    name: true,
    email: true
    // Omit passwordHash, createdAt, etc.
  }
})
```

### Caching with Redis

```typescript
import { Redis } from 'ioredis'
import { createHash } from 'crypto'

const redis = new Redis(process.env.REDIS_URL!)

class CacheService {
  static async get<T>(key: string): Promise<T | null> {
    const value = await redis.get(key)
    return value ? JSON.parse(value) : null
  }
  
  static async set<T>(key: string, value: T, ttlSeconds = 300): Promise<void> {
    await redis.setex(key, ttlSeconds, JSON.stringify(value))
  }
  
  static async invalidate(pattern: string): Promise<void> {
    const keys = await redis.keys(pattern)
    if (keys.length > 0) await redis.del(...keys)
  }
}

// Cache database queries
async function getCachedUser(id: string): Promise<User> {
  const cacheKey = `user:${id}`
  
  // Check cache first
  const cached = await CacheService.get<User>(cacheKey)
  if (cached) return cached
  
  // Cache miss — query DB
  const user = await db.query.users.findFirst({ where: eq(users.id, id) })
  if (!user) throw new NotFoundError('User not found')
  
  // Store in cache (5 minutes TTL)
  await CacheService.set(cacheKey, user, 300)
  
  return user
}
```

### Response Compression

```typescript
import compression from 'compression'

// Enable gzip/brotli compression
app.use(compression({
  filter: (req, res) => {
    // Don't compress small responses or already-compressed files
    if (req.headers['x-no-compression']) return false
    return compression.filter(req, res)
  },
  threshold: 1024  // Only compress responses > 1KB
}))
```

### Database Connection Pooling

```typescript
// Optimal pool configuration
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,              // Max 20 connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000
})

// Monitor pool health
setInterval(async () => {
  const { totalCount, idleCount, waitingCount } = pool
  logger.info('DB pool stats', { totalCount, idleCount, waitingCount })
  
  if (waitingCount > 5) {
    logger.warn('DB pool under pressure', { waitingCount })
  }
}, 60000)
```

## Performance Monitoring

```typescript
// Measure API endpoint performance
app.use((req, res, next) => {
  const start = Date.now()
  
  res.on('finish', () => {
    const duration = Date.now() - start
    
    if (duration > 1000) {
      logger.warn('Slow request detected', {
        method: req.method,
        path: req.path,
        duration: `${duration}ms`,
        statusCode: res.statusCode
      })
    }
    
    metrics.recordLatency(req.path, req.method, duration)
  })
  
  next()
})
```

## Performance Checklist

```
Frontend:
□ Bundle size < 200KB initial (gzipped)
□ All images lazy-loaded and sized correctly
□ Routes code-split
□ No render-blocking resources
□ Core Web Vitals green in Lighthouse

Backend:
□ No N+1 queries (verified with query logs)
□ Indexes on all JOIN and WHERE columns
□ Response caching for expensive operations
□ Compression enabled
□ Connection pooling configured

Database:
□ EXPLAIN ANALYZE on slow queries
□ Indexes cover query patterns
□ Pagination on all list endpoints
□ Soft deletes not slowing down common queries
```
