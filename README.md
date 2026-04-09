# 🧠 Ultimate Agent Skills

  A comprehensive library of **real, actionable SKILL.md files** for AI agents.
  Load any skill and immediately level up your AI assistant's capabilities.

  > Works with: Claude, OpenCode, OpenClaw, Cursor, Copilot, and any agent that can read files.

  ## 📦 Skills Directory (24 Skills)

  ### 🏛️ Anthropic Official Skills (2025)
  | Skill | Description |
  |-------|-------------|
  | [agentic-coding](skills/anthropic/agentic-coding/SKILL.md) | Best practices for AI-assisted software development |
  | [computer-use](skills/anthropic/computer-use/SKILL.md) | Control computers: browser, desktop, mouse, keyboard |
  | [tool-use](skills/anthropic/tool-use/SKILL.md) | Function calling and tool integration patterns |
  | [multi-agent](skills/anthropic/multi-agent/SKILL.md) | Orchestrating multiple AI agents |
  | [memory](skills/anthropic/memory/SKILL.md) | Memory systems: in-context, external, and hybrid |
  | [long-context](skills/anthropic/long-context/SKILL.md) | Working effectively with large contexts (200K tokens) |

  ### ⚡ SuperAgent Skills — Web Development
  | Skill | Description |
  |-------|-------------|
  | [fullstack-web](skills/superagent/fullstack-web/SKILL.md) | Complete full-stack web development patterns |
  | [api-design](skills/superagent/api-design/SKILL.md) | REST API design, validation, error handling |
  | [database](skills/superagent/database/SKILL.md) | PostgreSQL, Drizzle ORM, migrations, optimization |
  | [auth](skills/superagent/auth/SKILL.md) | JWT, sessions, OAuth, security best practices |
  | [devops](skills/superagent/devops/SKILL.md) | Docker, CI/CD, deployment, monitoring |
  | [testing](skills/superagent/testing/SKILL.md) | Unit, integration, E2E testing with Vitest + Playwright |
  | [react-frontend](skills/superagent/react-frontend/SKILL.md) | React, TanStack Query, Zustand, forms |
  | [typescript](skills/superagent/typescript/SKILL.md) | TypeScript strict mode, generics, utility types |

  ### 🔧 Engineering Excellence Skills
  | Skill | Description |
  |-------|-------------|
  | [security](skills/engineering/security/SKILL.md) | OWASP Top 10, injection prevention, secrets management |
  | [clean-code](skills/engineering/clean-code/SKILL.md) | Naming, SOLID principles, DRY, readable code |
  | [performance](skills/engineering/performance/SKILL.md) | Bundle optimization, caching, N+1 queries, Core Web Vitals |
  | [code-review](skills/engineering/code-review/SKILL.md) | What to look for, how to give feedback |
  | [debugging](skills/engineering/debugging/SKILL.md) | Systematic debugging, stack traces, common patterns |

  ### 🤖 AI Integration Skills
  | Skill | Description |
  |-------|-------------|
  | [openai](skills/ai/openai/SKILL.md) | OpenAI API: chat, streaming, function calling, embeddings |
  | [anthropic-sdk](skills/ai/anthropic-sdk/SKILL.md) | Claude SDK: messages, streaming, tool use, vision |
  | [rag](skills/ai/rag/SKILL.md) | RAG systems: ingestion, chunking, retrieval, generation |
  | [agents](skills/ai/agents/SKILL.md) | Building AI agents: planning, tools, memory, safety |
  | [streaming](skills/ai/streaming/SKILL.md) | SSE, WebSockets, streaming AI responses |

  ## 🚀 How to Use

  ### Option 1: Load a skill in your prompt
  ```
  Read the file at skills/superagent/fullstack-web/SKILL.md and follow its instructions.
  ```

  ### Option 2: Reference in system prompt
  ```
  You are an expert web developer. Follow the patterns in:
  - skills/superagent/fullstack-web/SKILL.md
  - skills/engineering/security/SKILL.md
  - skills/superagent/testing/SKILL.md
  ```

  ### Option 3: Clone and use locally
  ```bash
  git clone https://github.com/hakercryptoplus-svg/ultimate-agent-skills
  ```

  ## 🎯 Skill Selection Guide

  | Task | Skills to Load |
  |------|----------------|
  | Build a REST API | api-design + database + auth + security |
  | Build React app | react-frontend + typescript + testing |
  | Add AI features | openai OR anthropic-sdk + streaming |
  | Build an AI agent | agents + tool-use + memory |
  | Semantic search | rag + openai |
  | Security audit | security + code-review |
  | Fix a bug | debugging + clean-code |
  | Optimize performance | performance + database |
  | Deploy to production | devops + security |

  ## 📋 Skill Format

  Every SKILL.md follows this structure:
  ```yaml
  ---
  name: skill-name
  description: "What this skill does and when to load it"
  ---

  # Title

  ## Practical code examples
  ## Checklists
  ## Anti-patterns to avoid
  ## Best practices
  ```

  ## 📊 Repository Stats

  - **24 total skills** across 4 categories
  - Each skill: **100-300 lines** of actionable content
  - Focus: **code examples over theory**
  - Updated: **2025** (Anthropic official + modern best practices)

  ---

  ⭐ **Star this repo** if it helps you build better software with AI!

  *Made with ❤️ for the AI engineering community*
  