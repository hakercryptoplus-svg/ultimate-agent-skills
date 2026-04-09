---
  name: database
  description: "Complete database skill covering PostgreSQL, Drizzle ORM, schema design, migrations, query optimization, and common patterns. Load for any database work."
  ---

  # Database Mastery — إتقان قواعد البيانات

  ## PostgreSQL + Drizzle ORM Setup

  ```typescript
  // lib/db/index.ts
  import { drizzle } from 'drizzle-orm/node-postgres'
  import { Pool } from 'pg'
  import * as schema from './schema'

  const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    max: 20,                  // Max connections in pool
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
    ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : undefined
  })

  export const db = drizzle(pool, { schema, logger: process.env.NODE_ENV === 'development' })

  // Test connection on startup
  pool.query('SELECT 1').catch(err => {
    console.error('Database connection failed:', err)
    process.exit(1)
  })
  ```

  ## Schema Design Best Practices

  ```typescript
  import { pgTable, uuid, text, timestamp, boolean, integer, decimal, pgEnum } from 'drizzle-orm/pg-core'

  // ✅ Always use UUID primary keys (not integer auto-increment)
  // ✅ Always include createdAt + updatedAt
  // ✅ Use enums for fixed value sets
  // ✅ Add database-level constraints

  export const userRoleEnum = pgEnum('user_role', ['user', 'admin', 'moderator'])

  export const users = pgTable('users', {
    id:           uuid('id').primaryKey().defaultRandom(),
    email:        text('email').notNull().unique(),
    name:         text('name').notNull(),
    passwordHash: text('password_hash').notNull(),
    role:         userRoleEnum('role').default('user').notNull(),
    emailVerified:boolean('email_verified').default(false).notNull(),
    lastLoginAt:  timestamp('last_login_at'),
    createdAt:    timestamp('created_at').defaultNow().notNull(),
    updatedAt:    timestamp('updated_at').defaultNow().notNull()
  })

  // Foreign key with cascade
  export const posts = pgTable('posts', {
    id:        uuid('id').primaryKey().defaultRandom(),
    userId:    uuid('user_id').references(() => users.id, { onDelete: 'cascade' }).notNull(),
    title:     text('title').notNull(),
    content:   text('content').notNull(),
    published: boolean('published').default(false),
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at').defaultNow().notNull()
  })
  ```

  ## Relations Definition

  ```typescript
  import { relations } from 'drizzle-orm'

  export const usersRelations = relations(users, ({ many }) => ({
    posts: many(posts),
    sessions: many(sessions)
  }))

  export const postsRelations = relations(posts, ({ one }) => ({
    author: one(users, { fields: [posts.userId], references: [users.id] })
  }))
  ```

  ## Query Patterns

  ```typescript
  import { eq, and, or, like, gte, lte, desc, asc, count, sql } from 'drizzle-orm'

  // Single record
  const user = await db.query.users.findFirst({
    where: eq(users.email, email),
    columns: { passwordHash: false }  // Exclude sensitive fields
  })

  // With relations
  const userWithPosts = await db.query.users.findFirst({
    where: eq(users.id, userId),
    with: {
      posts: {
        where: eq(posts.published, true),
        orderBy: desc(posts.createdAt),
        limit: 10
      }
    }
  })

  // Multiple records with pagination
  const [items, [{ total }]] = await Promise.all([
    db.query.posts.findMany({
      where: and(
        eq(posts.published, true),
        gte(posts.createdAt, startDate)
      ),
      orderBy: desc(posts.createdAt),
      limit,
      offset
    }),
    db.select({ total: count() }).from(posts).where(eq(posts.published, true))
  ])

  // Raw SQL when needed
  const stats = await db.execute(sql`
    SELECT 
      date_trunc('month', created_at) as month,
      count(*) as user_count
    FROM users
    WHERE created_at >= NOW() - INTERVAL '12 months'
    GROUP BY month
    ORDER BY month DESC
  `)
  ```

  ## Migrations with Drizzle Kit

  ```typescript
  // drizzle.config.ts
  import { defineConfig } from 'drizzle-kit'

  export default defineConfig({
    schema: './src/db/schema.ts',
    out: './src/db/migrations',
    dialect: 'postgresql',
    dbCredentials: {
      url: process.env.DATABASE_URL!
    },
    verbose: true,
    strict: true
  })
  ```

  ```bash
  # Generate migration from schema changes
  npx drizzle-kit generate

  # Push to database (dev only)
  npx drizzle-kit push

  # Run migrations (production)
  npx drizzle-kit migrate

  # View database in browser
  npx drizzle-kit studio
  ```

  ## Transactions

  ```typescript
  // Always use transactions for multiple related operations
  async function transferFunds(fromId: string, toId: string, amount: number) {
    return db.transaction(async (tx) => {
      // Lock both accounts to prevent race conditions
      const [from] = await tx
        .select()
        .from(accounts)
        .where(eq(accounts.id, fromId))
        .for('update')
      
      const [to] = await tx
        .select()
        .from(accounts)
        .where(eq(accounts.id, toId))
        .for('update')
      
      if (from.balance < amount) throw new Error('Insufficient funds')
      
      await tx.update(accounts)
        .set({ balance: sql`balance - ${amount}` })
        .where(eq(accounts.id, fromId))
      
      await tx.update(accounts)
        .set({ balance: sql`balance + ${amount}` })
        .where(eq(accounts.id, toId))
      
      await tx.insert(transactions).values({
        fromAccountId: fromId,
        toAccountId: toId,
        amount,
        type: 'transfer'
      })
      
      return { success: true }
    })
  }
  ```

  ## Query Performance

  ```typescript
  // ✅ Add indexes for frequently queried columns
  import { index, uniqueIndex } from 'drizzle-orm/pg-core'

  export const posts = pgTable('posts', {
    id:        uuid('id').primaryKey().defaultRandom(),
    userId:    uuid('user_id').notNull(),
    slug:      text('slug').notNull(),
    createdAt: timestamp('created_at').defaultNow()
  }, (table) => ({
    userIdIdx:  index('posts_user_id_idx').on(table.userId),
    slugIdx:    uniqueIndex('posts_slug_idx').on(table.slug),
    createdIdx: index('posts_created_at_idx').on(table.createdAt)
  }))

  // ✅ Avoid N+1 — use eager loading
  // ❌ N+1 problem:
  const users = await db.query.users.findMany()
  for (const user of users) {
    const posts = await db.query.posts.findMany({ where: eq(posts.userId, user.id) })
    // This runs N+1 queries!
  }

  // ✅ Single query with join:
  const users = await db.query.users.findMany({
    with: { posts: true }  // Single query with LEFT JOIN
  })
  ```

  ## Database Seeding

  ```typescript
  // src/db/seed.ts
  import { db } from './index'
  import { users, posts } from './schema'
  import bcrypt from 'bcrypt'

  async function seed() {
    console.log('Seeding database...')
    
    // Clear existing data (development only)
    if (process.env.NODE_ENV !== 'production') {
      await db.delete(posts)
      await db.delete(users)
    }
    
    // Create admin user
    const [admin] = await db.insert(users).values({
      email: 'admin@example.com',
      name: 'Admin User',
      passwordHash: await bcrypt.hash('admin123', 12),
      role: 'admin',
      emailVerified: true
    }).returning()
    
    // Create sample posts
    await db.insert(posts).values([
      { userId: admin.id, title: 'First Post', content: 'Hello World!', published: true },
      { userId: admin.id, title: 'Draft Post', content: 'Work in progress...', published: false }
    ])
    
    console.log('Seeding complete!')
    process.exit(0)
  }

  seed().catch(console.error)
  ```
  