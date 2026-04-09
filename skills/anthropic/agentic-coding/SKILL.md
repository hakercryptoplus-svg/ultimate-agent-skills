---
  name: agentic-coding
  description: "Best practices for AI agents writing, editing, and managing code autonomously. Load this skill when building software, fixing bugs, or working in any codebase."
  ---

  # Agentic Coding — كيف يبرمج الـ Agent باحترافية

  ## Core Behavior Rules

  ### 1. Read Before You Write
  ALWAYS read a file before editing it. Never assume its contents.

  ```
  Before editing any file:
  1. Read the file completely (or grep for relevant sections)
  2. Understand the existing patterns and conventions
  3. Make the minimal change that solves the problem
  4. Verify the change doesn't break other parts
  ```

  ### 2. Minimal Footprint Principle
  Do exactly what was asked — no more, no less.

  ```
  ✅ Fix only the reported bug
  ✅ Add only the requested feature
  ✅ Modify only files directly related to the task
  ❌ Don't refactor unrelated code
  ❌ Don't add unrequested features
  ❌ Don't change files that aren't needed
  ```

  ### 3. Prefer Reversible Actions
  Order your actions by reversibility:

  ```
  SAFEST  → Create new file (easy to delete)
  SAFE    → Edit existing file (can revert)
  RISKY   → Delete code (harder to recover)
  DANGER  → Delete data/database (may be permanent — always confirm)
  ```

  ### 4. Verify Before Destructive Operations
  Before deleting files, databases, or critical data:
  - Confirm with the user explicitly
  - State exactly what will be deleted
  - Offer to create a backup first

  ---

  ## Code Quality Standards

  ### TypeScript/JavaScript
  ```typescript
  // ✅ Always use explicit types
  function getUser(id: string): Promise<User | null> {
    return db.users.findUnique({ where: { id } });
  }

  // ✅ Handle all error cases
  async function safeGetUser(id: string): Promise<User> {
    const user = await getUser(id);
    if (!user) throw new NotFoundError(`User ${id} not found`);
    return user;
  }

  // ✅ Use const by default, let only when reassignment is needed
  const MAX_RETRIES = 3;
  let retryCount = 0;

  // ❌ Never use any without justification
  function process(data: any) {} // BAD
  function process(data: UserInput) {} // GOOD
  ```

  ### Error Handling
  ```typescript
  // ✅ Always handle errors explicitly — never swallow them
  try {
    const result = await riskyOperation();
    return { success: true, data: result };
  } catch (error) {
    // Log with full context
    logger.error('Operation failed', {
      error: error instanceof Error ? error.message : String(error),
      stack: error instanceof Error ? error.stack : undefined,
      context: { userId, operation: 'riskyOperation', timestamp: new Date().toISOString() }
    });
    
    // Re-throw as domain error (don't leak internal details)
    throw new AppError('Operation failed. Please try again.', { cause: error });
  }

  // ❌ Never do this:
  try {
    await riskyOperation();
  } catch (e) {
    // silently ignored - TERRIBLE
  }
  ```

  ### Naming Conventions
  ```typescript
  // Variables and functions: camelCase
  const userEmail = 'user@example.com';
  function getUserById(id: string) {}

  // Classes and types: PascalCase
  class UserService {}
  interface UserProfile {}
  type ApiResponse<T> = { data: T; error?: string };

  // Constants: SCREAMING_SNAKE_CASE
  const MAX_LOGIN_ATTEMPTS = 5;
  const DEFAULT_TIMEOUT_MS = 30_000;

  // Files: kebab-case
  // user-service.ts, auth-middleware.ts, api-routes.ts
  ```

  ---

  ## Parallel Execution Strategy

  When multiple independent operations are needed, run them in parallel:

  ```javascript
  // ❌ SLOW — sequential
  const user = await getUser(id);
  const posts = await getPosts(id);
  const settings = await getSettings(id);

  // ✅ FAST — parallel
  const [user, posts, settings] = await Promise.all([
    getUser(id),
    getPosts(id),
    getSettings(id)
  ]);
  ```

  Same for file operations — read multiple files at once, don't chain them.

  ---

  ## Working with Existing Codebases

  ### Exploration First
  ```bash
  # 1. Understand structure
  ls -la
  cat package.json

  # 2. Find relevant code (use grep, not cat)
  grep -r "functionName" --include="*.ts" -l
  grep -r "ComponentName" --include="*.tsx" -n

  # 3. Read only what you need
  # Use offset/limit for large files
  ```

  ### Respect Existing Patterns
  ```
  Before adding code, ask:
  - What naming convention does this codebase use?
  - How are errors handled here?
  - What's the import style (default vs named)?
  - Is there an existing utility for what I'm building?

  Then match those patterns exactly.
  ```

  ---

  ## Verification Checklist

  After every code change, verify:

  ```
  □ Does it build without errors? (npm run build / tsc --noEmit)
  □ Do existing tests still pass? (npm test)
  □ Are all edge cases handled? (null, undefined, empty array, network failure)
  □ Is sensitive data protected? (no passwords in logs, no secrets hardcoded)
  □ Does the change actually solve the original problem?
  ```

  ---

  ## Self-Review Pattern

  Before submitting any code, re-read it as if you're a reviewer:

  ```
  "Would I approve this PR if a junior developer submitted it?"

  Look for:
  - Unhandled null/undefined access
  - Missing error handling
  - Hardcoded values that should be env vars
  - Race conditions in async code
  - Security vulnerabilities (SQL injection, XSS, etc.)
  - Performance issues (N+1 queries, unnecessary re-renders)
  ```
  