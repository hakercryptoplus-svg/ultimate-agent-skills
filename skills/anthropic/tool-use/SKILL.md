---
  name: tool-use
  description: "Best practices for using tools efficiently in agentic workflows. Load this skill to maximize speed and avoid common mistakes when using bash, file tools, search, and APIs."
  ---

  # Tool Use Mastery — إتقان استخدام الأدوات

  ## The Golden Rules

  ```
  1. Use the RIGHT tool for each job
  2. Parallelize INDEPENDENT operations  
  3. Read files ONCE — make all edits in one pass
  4. Never use bash to do what a dedicated tool does better
  ```

  ## Tool Selection Guide

  | Task | Use This | Not This |
  |------|----------|----------|
  | Read a file | read_file() | bash cat |
  | Find files by name | glob() | bash find |
  | Search code | grep() | bash grep |
  | Small code edit | edit() | write() |
  | Run commands | bash() | — |

  ## Parallel Execution (Critical for Speed)

  ```javascript
  // ❌ SLOW — 3 sequential reads = 3x time
  const a = await readFile('src/app.ts')
  const b = await readFile('src/auth.ts')  
  const c = await readFile('src/db.ts')

  // ✅ FAST — parallel reads = 1x time
  const [a, b, c] = await Promise.all([
    readFile('src/app.ts'),
    readFile('src/auth.ts'),
    readFile('src/db.ts')
  ])
  ```

  ## File Reading Strategy

  ```
  File size          → Strategy
  < 500 lines        → read full file
  500–2000 lines     → read with offset/limit
  > 2000 lines       → grep first, then read sections

  Example:
  grep("authMiddleware", type="ts", output_mode="content", "-C": 5)
  # Then read only the relevant section:
  readFile("src/middleware/auth.ts", offset=45, limit=30)
  ```

  ## Editing Files (edit vs write)

  ```
  Use edit() when:
  - Changing specific lines/functions in existing file
  - Renaming variables
  - Fixing a bug in place

  Use write() when:
  - Creating a new file
  - Completely rewriting a file (< 200 lines)
  - File doesn't exist yet

  edit() is ALWAYS safer — it preserves unchanged content.
  ```

  ## Bash Timeouts

  ```javascript
  bash("ls", { timeout: 5000 })           // File listing: 5s
  bash("pnpm add zod", { timeout: 120000 }) // Install: 2min
  bash("pnpm build", { timeout: 120000 })   // Build: 2min
  bash("pnpm test", { timeout: 120000 })    // Tests: 2min
  ```

  ## Bash Anti-Patterns

  ```bash
  # ❌ Wrong tools in bash
  bash("cat file.ts")             # Use read_file()
  bash("find . -name '*.ts'")     # Use glob()
  bash("grep 'pattern' src/")     # Use grep tool
  bash("cat .env")                # NEVER read env files!

  # ✅ Correct bash usage  
  bash("pnpm run dev")            # Starting servers
  bash("pnpm run build")          # Building
  bash("git status")              # Git operations
  bash("curl localhost:3000/api/health")  # API testing
  ```

  ## Grep Patterns

  ```javascript
  // Find all files containing a pattern
  grep("UserService", type="ts", output_mode="files_with_matches")

  // Find with context around matches (5 lines before/after)
  grep("handleLogin", output_mode="content", "-C": 5)

  // Case insensitive search
  grep("(?i)todo|fixme|hack", output_mode="content", "-n": true)

  // Multiline pattern
  grep("export.*interface.*User", type="ts", multiline: true)
  ```

  ## Context Efficiency Rules

  ```
  To avoid filling context with noise:
  □ Use head_limit in grep to cap results
  □ Don't read files you won't modify
  □ Summarize large bash outputs instead of reproducing fully
  □ Use output_mode: "files_with_matches" for initial discovery
  □ Then use output_mode: "content" only for files you'll actually use
  ```
  