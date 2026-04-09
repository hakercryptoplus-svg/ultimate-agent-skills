---
name: clean-code
description: "Clean code principles, naming conventions, function design, SOLID principles, and code organization. Load when writing new code or reviewing existing code for quality."
---

# Clean Code — كتابة كود نظيف ومتسق

## Naming (The Most Important Skill)

```typescript
// Variables: nouns, specific, reveal intent
// ❌ Bad
const d = new Date()
const lst = users.filter(u => u.a)
const temp = calculate(x, y)

// ✅ Good
const currentDate = new Date()
const activeUsers = users.filter(user => user.isActive)
const totalOrderAmount = calculateTotalAmount(items, discount)

// Boolean variables: is/has/can/should prefix
// ❌ Bad
const loading = true
const admin = user.role === 'admin'
const email = user.emailVerified

// ✅ Good
const isLoading = true
const isAdmin = user.role === 'admin'
const hasVerifiedEmail = user.emailVerified
```

## Functions: Do One Thing

```typescript
// ❌ Function doing too many things
async function registerUser(email: string, password: string, name: string) {
  // Validates input
  if (!email.includes('@')) throw new Error('Invalid email')
  if (password.length < 8) throw new Error('Password too short')
  
  // Hashes password
  const hash = await bcrypt.hash(password, 12)
  
  // Creates user in DB
  const user = await db.insert(users).values({ email, name, passwordHash: hash })
  
  // Sends welcome email
  await mailer.send({ to: email, subject: 'Welcome!' })
  
  // Creates Stripe customer
  const customer = await stripe.customers.create({ email })
  
  // Logs analytics
  analytics.track('user_registered', { userId: user.id })
  
  return user
}

// ✅ Each function has one clear responsibility
async function registerUser(input: RegisterInput) {
  const validated = RegisterSchema.parse(input)
  const user = await UserService.create(validated)
  await onUserRegistered(user)  // Orchestrates side effects
  return user
}

async function onUserRegistered(user: User) {
  await Promise.all([
    EmailService.sendWelcome(user.email),
    BillingService.createCustomer(user),
    Analytics.track('user_registered', { userId: user.id })
  ])
}
```

## Small Functions Rule

```
A function should:
✅ Do ONE thing
✅ Be < 20 lines (ideally < 10)
✅ Have < 3 parameters (use objects for more)
✅ Have a clear, descriptive name
✅ Be at the same level of abstraction

If you need to comment WHAT the code does — extract it into a named function
```

## Comments: Why, Not What

```typescript
// ❌ Comment explaining WHAT (obvious from code)
// Increment counter by 1
counter++

// Check if user is admin
if (user.role === 'admin') { }

// ✅ Comment explaining WHY (not obvious)
// Use cost factor 12: industry standard balancing security vs performance
// Lower (< 10) is too fast for brute force, higher (> 14) is too slow for UX
const hash = await bcrypt.hash(password, 12)

// Deliberate 200ms delay: prevents timing attacks that reveal if email exists
await new Promise(resolve => setTimeout(resolve, 200))
await res.json({ message: 'If email exists, reset link was sent' })
```

## DRY (Don't Repeat Yourself)

```typescript
// ❌ Repeated logic
router.get('/users', async (req, res) => {
  try {
    const users = await UserService.getAll()
    res.json(users)
  } catch (err) {
    logger.error('Error in GET /users', { error: err })
    res.status(500).json({ error: 'Internal server error' })
  }
})

router.post('/users', async (req, res) => {
  try {
    const user = await UserService.create(req.body)
    res.status(201).json(user)
  } catch (err) {
    logger.error('Error in POST /users', { error: err })
    res.status(500).json({ error: 'Internal server error' })
  }
})

// ✅ Extract repeated pattern
function asyncHandler(fn: RequestHandler): RequestHandler {
  return async (req, res, next) => {
    try {
      await fn(req, res, next)
    } catch (err) {
      next(err)  // Global error handler will handle it
    }
  }
}

router.get('/users', asyncHandler(async (req, res) => {
  const users = await UserService.getAll()
  res.json(users)
}))
```

## Avoid Deep Nesting

```typescript
// ❌ Pyramid of doom
async function processOrder(orderId: string) {
  const order = await getOrder(orderId)
  if (order) {
    if (order.status === 'pending') {
      const payment = await processPayment(order)
      if (payment.success) {
        const updated = await updateOrderStatus(orderId, 'paid')
        if (updated) {
          await sendConfirmationEmail(order.userId)
        }
      }
    }
  }
}

// ✅ Early returns flatten the code
async function processOrder(orderId: string) {
  const order = await getOrder(orderId)
  if (!order) throw new NotFoundError('Order not found')
  if (order.status !== 'pending') throw new BusinessError('Order not in pending state')
  
  const payment = await processPayment(order)
  if (!payment.success) throw new PaymentError(payment.error)
  
  await updateOrderStatus(orderId, 'paid')
  await sendConfirmationEmail(order.userId)
}
```

## SOLID Principles Summary

```
S — Single Responsibility: each module/class/function does one thing
O — Open/Closed: open for extension, closed for modification
L — Liskov Substitution: subtypes must be substitutable for base types
I — Interface Segregation: prefer small, focused interfaces
D — Dependency Inversion: depend on abstractions, not concretions

Most important for web dev: S and D
```
