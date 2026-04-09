---
  name: long-context
  description: "Techniques for working effectively with large codebases, long files, and extensive context. Load when analyzing large projects or working with files over 500 lines."
  ---

  # Long Context Handling — التعامل مع السياق الطويل

  ## The Lost-in-the-Middle Problem

  Research shows AI models remember:
  - Beginning of context: ⭐⭐⭐⭐⭐ Excellent  
  - End of context: ⭐⭐⭐⭐⭐ Excellent
  - Middle of context: ⭐⭐ Poor

  **Solution: Put critical information at START and END of context.**

  ## File Reading Strategy by Size

  ```
  < 300 lines:   Read fully (no problem)
  300-1000 lines: Read with smart offset/limit
  1000-5000 lines: Grep first, then read sections
  > 5000 lines:  Map-reduce approach (summarize chunks)
  ```

  ## Grep Before Reading

  ```bash
  # Find the function you need first
  grep("handleUserRegistration", path="src", type="ts", output_mode="content", "-n": true, "-C": 3)

  # Now you know it's at line 247 in auth.service.ts
  # Read only that section:
  readFile("src/auth.service.ts", offset=240, limit=80)

  # This uses 10x less context than reading the full file
  ```

  ## Large Project Analysis: Map-Reduce

  ```typescript
  async function analyzeProject(dir: string) {
    // STEP 1: MAP — analyze each file independently
    const files = await glob("**/*.ts", { path: dir });
    
    const summaries = await Promise.all(
      files.map(file => summarizeFile(file))
    );
    
    // STEP 2: REDUCE — synthesize findings
    return {
      architecture: identifyPatterns(summaries),
      dependencies: buildDependencyGraph(summaries),
      issues: collectIssues(summaries),
      entryPoints: findEntryPoints(summaries)
    };
  }

  async function summarizeFile(path: string): Promise<FileSummary> {
    // Read file in sections if large
    const content = await readFile(path);
    return {
      path,
      exports: extractExports(content),
      imports: extractImports(content),
      mainPurpose: inferPurpose(path, content),
      lineCount: content.split('\n').length
    };
  }
  ```

  ## Context Priority Order

  When working on a task, load context in this order:

  ```
  LOAD FIRST (highest priority):
    1. The specific task/bug description
    2. The file(s) directly involved
    3. Relevant tests

  LOAD IF NEEDED:
    4. Files that the involved files import
    5. Type definitions used
    6. Related configuration

  LOAD LAST (lower priority):  
    7. Documentation
    8. Unrelated utility files
  ```

  ## Hierarchical Summarization

  For very large projects:

  ```
  Level 1: Project summary (what it does, tech stack)
    Level 2: Module summaries (auth, payments, UI, etc.)
      Level 3: File summaries (what each file exports)
        Level 4: Function-level details (load on demand)
  ```

  ## Avoiding Context Pollution

  ```
  ✅ DO:
  - Read only files relevant to current task
  - Use grep to find specific sections
  - Summarize bash output instead of including it raw
  - Remove outdated context when switching tasks

  ❌ DON'T:
  - Read entire large files when you need one function
  - Include full error logs when a summary would do
  - Re-read files you already have in context
  - Load documentation you won't reference
  ```

  ## Long Document Processing

  ```python
  def process_long_document(doc: str, question: str) -> str:
      """Answer a question about a document too long to fit in context."""
      
      # Split into overlapping chunks
      chunks = split_with_overlap(doc, chunk_size=4000, overlap=200)
      
      # Extract relevant info from each chunk
      relevant_parts = [
          extract_relevant(chunk, question)
          for chunk in chunks
          if is_relevant(chunk, question)
      ]
      
      # Synthesize final answer
      return synthesize(relevant_parts, question)
  ```
  