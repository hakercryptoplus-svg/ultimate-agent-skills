---
  name: memory-and-persistence
  description: "Patterns for giving AI agents persistent memory across sessions. Load when building stateful agents, saving project context, or implementing agent memory systems."
  ---

  # Memory & Persistence — الذاكرة والاستمرارية

  ## Memory Types

  ```
  IN-CONTEXT (fast, temporary):
    - Current conversation
    - Loaded files
    - Variables in session
    ✅ Instantly accessible
    ❌ Lost when session ends

  EXTERNAL (persistent, unlimited):
    - Database records
    - JSON files
    - Vector stores  
    ✅ Survives restarts
    ❌ Requires retrieval step

  PARAMETRIC (baked-in knowledge):
    - Model training data
    - Skills and capabilities
    ✅ Always available
    ❌ Cannot be updated
  ```

  ## File-Based Agent State

  ```typescript
  // .agent-state.json — project memory file
  interface AgentState {
    projectGoal: string;
    conventions: {
      language: string;          // "TypeScript strict"
      framework: string;         // "Express 5 + Zod"
      testFramework: string;     // "Vitest"
      namingStyle: string;       // "camelCase files, PascalCase classes"
    };
    decisions: ArchitectureDecision[];
    completedMilestones: string[];
    currentTask: string | null;
    blockers: string[];
  }

  class AgentMemory {
    private file = '.agent-state.json';
    
    async save(state: AgentState): Promise<void> {
      await writeFile(this.file, JSON.stringify(state, null, 2));
    }
    
    async load(): Promise<AgentState | null> {
      try {
        const content = await readFile(this.file);
        return JSON.parse(content);
      } catch {
        return null;  // First run
      }
    }
    
    async addDecision(decision: ArchitectureDecision): Promise<void> {
      const state = await this.load() ?? defaultState();
      state.decisions.push({ ...decision, timestamp: new Date().toISOString() });
      await this.save(state);
    }
  }
  ```

  ## Architecture Decision Records (ADR)

  Save important decisions so future sessions don't re-debate them:

  ```markdown
  # ADR-001: Authentication Strategy
  Date: 2025-01-15
  Status: Accepted

  ## Decision
  Use JWT with refresh token rotation (NOT sessions/cookies)

  ## Rationale
  - API is consumed by mobile app + web frontend
  - JWT allows stateless horizontal scaling
  - Refresh token rotation prevents token theft

  ## Consequences
  + No server-side session storage needed
  + Works across domains
  - Must implement refresh token storage in DB
  - Must handle token revocation for logout

  ## Alternatives Rejected
  - Sessions: Doesn't work well with mobile clients
  - OAuth only: Adds complexity for simple auth needs
  ```

  ## Vector Memory for Semantic Search

  ```typescript
  // For large amounts of project context
  class SemanticMemory {
    async store(content: string, tags: string[]) {
      const embedding = await embed(content);
      await vectorDb.upsert({
        id: hashContent(content),
        embedding,
        metadata: { content, tags, stored_at: new Date().toISOString() }
      });
    }
    
    async recall(query: string, limit = 5): Promise<string[]> {
      const embedding = await embed(query);
      const results = await vectorDb.query({ embedding, topK: limit });
      return results.map(r => r.metadata.content);
    }
  }

  // Usage
  await memory.store("Decided to use PostgreSQL for ACID compliance", ["database", "decision"])
  const relevant = await memory.recall("Which database are we using?")
  ```

  ## Checkpoint Pattern

  ```typescript
  // Save state before risky operations
  async function withCheckpoint<T>(
    label: string,
    operation: () => Promise<T>
  ): Promise<T> {
    const state = await agentMemory.load();
    await agentMemory.saveCheckpoint(label, state);
    
    try {
      return await operation();
    } catch (error) {
      console.log(`Operation failed. Checkpoint '${label}' available for rollback.`);
      throw error;
    }
  }
  ```

  ## Session Start Protocol

  At the beginning of every session:
  ```
  1. Read .agent-state.json (if exists)
  2. Read DECISIONS.md for architectural choices
  3. Read current-task.md for what was in-progress
  4. Brief user: "Last session: X. Current state: Y. Ready to continue?"
  ```

  ## Session End Protocol

  At the end of every session:
  ```
  1. Update .agent-state.json with completed milestones
  2. Add any new decisions to DECISIONS.md
  3. Note any blockers discovered
  4. Write next steps to current-task.md
  ```
  