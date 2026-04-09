---
name: devops
description: "Complete DevOps skill: Docker, CI/CD with GitHub Actions, environment management, and deployment. Load when deploying or setting up pipelines."
---

# DevOps & Deployment — النشر والـ CI/CD

## Dockerfile (Node.js Best Practice)

```dockerfile
FROM node:20-alpine AS base
RUN corepack enable
WORKDIR /app

FROM base AS deps
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod

FROM base AS builder
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm run build

FROM base AS runner
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser
USER appuser

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/api/healthz || exit 1

CMD ["node", "dist/index.js"]
```

## Docker Compose

```yaml
version: '3.9'
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  postgres_data:
```

## GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm run typecheck
      - run: pnpm run lint

  test:
    runs-on: ubuntu-latest
    needs: quality
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ["5432:5432"]
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      JWT_SECRET: test-secret-minimum-32-characters-long
      NODE_ENV: test
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm run db:migrate
      - run: pnpm test -- --coverage

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myapp
            docker compose pull
            docker compose up -d
            docker image prune -f
```

## Environment Variables Best Practices

```bash
# .env.example — COMMIT THIS FILE
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
JWT_SECRET=generate-with-openssl-rand-base64-32
PORT=3000
NODE_ENV=development
CORS_ORIGIN=http://localhost:5173

# Generate secure secrets:
# openssl rand -base64 32
```

```
NEVER commit:
- .env files with real values
- Private keys
- API keys or tokens
- Database passwords

ALWAYS use:
- Environment variables for all config
- Secrets manager in production (AWS Secrets Manager, Vault, etc.)
- .env.example with placeholder values
- Different credentials per environment
```

## Zero-Downtime Deployment

```bash
# Blue-Green deployment
docker compose up -d --scale app=2  # Start new containers
sleep 30                             # Wait for health checks to pass
docker compose up -d --scale app=1  # Remove old containers
```

## Production Checklist

```
Before deploying to production:
□ All tests pass in CI
□ TypeScript has no errors
□ Environment variables documented in .env.example
□ Database migrations are backward compatible
□ Health check endpoint works
□ Error tracking configured (Sentry)
□ Logs going to aggregation service
□ Backup strategy for database
□ Rate limiting enabled
□ HTTPS enforced
□ Security headers set (helmet)
□ CORS restricted to known origins
```

## Monitoring Must-Haves

```
Minimum monitoring setup:
1. Uptime monitoring: UptimeRobot (free) — alerts on downtime
2. Error tracking: Sentry (free tier) — captures exceptions
3. Health endpoint: GET /api/healthz — checks DB + app
4. Log aggregation: Papertrail or CloudWatch

Metrics to alert on:
- Error rate > 1% of requests
- Response time p95 > 2 seconds
- Database connection pool > 80% used
- Disk usage > 80%
- Memory usage > 85%
```
