# Lesson 16: AGENTS.md - giving agents project context

## Introduction

When a new engineer joins your team, you do not just hand them the codebase and say "good luck." You give them onboarding docs, explain the build system, point out where the tests live, and warn them about the parts of the code they should not touch.

AI coding agents need the same thing. Without project context, an agent will guess at your conventions, use the wrong test runner, miss your branching strategy, and generally produce code that does not fit your project. AGENTS.md solves this by giving agents a structured onboarding document they can read before they start working.

### ELI5: Think of AGENTS.md like a welcome packet

Imagine your company gives every new hire a one-page cheat sheet on day one. It lists the WiFi password, how to run the build, where to find the style guide, and what never to touch. AGENTS.md is that cheat sheet - but for AI agents working on your codebase.

> **Key takeaway:** AGENTS.md is a simple Markdown file you place in your repo to tell AI agents how to work on your project. Most major AI coding tools read it automatically.

---

## What is AGENTS.md

AGENTS.md is an open-format Markdown file that lives in your repository. It provides instructions and context to AI coding agents - tools like Claude Code, Cursor, GitHub Copilot, Gemini, and others.

There is no special schema, no YAML frontmatter requirement, no tooling dependency. It is plain Markdown. You write it like you would write instructions for a human, because that is essentially what it is - instructions for an AI that reads like a human.

### The problem it solves

Without AGENTS.md, every time you use an AI coding agent you end up repeating yourself:

- "We use pytest, not unittest"
- "Run `make lint` before committing"
- "The API code is in `src/api/`, the frontend is in `web/`"
- "Never modify files in `vendor/`"

AGENTS.md captures this once, in one place, so every agent that touches your code starts with the right context.

### How it works

1. You place an `AGENTS.md` file in the root of your repository
2. AI coding agents automatically detect and read it when they start working on the project
3. The instructions inform how the agent writes code, runs tests, and makes decisions
4. You can also place additional `AGENTS.md` files in subdirectories for area-specific guidance

That is it. No configuration, no build step, no integration work.

---

## A brief history

AGENTS.md emerged in August 2025 from a collaboration between OpenAI (Codex), Amp, Google (Jules), Cursor, and Factory. Rather than each company creating its own proprietary format, they agreed on a shared, open standard.

In December 2025, AGENTS.md was contributed to the [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation) under the Linux Foundation, alongside Anthropic's Model Context Protocol (MCP) and Block's Goose. It is now a foundation-stewarded open standard.

As of early 2026, over 60,000 open-source repositories include an AGENTS.md file.

---

## What goes in an AGENTS.md file

Analysis of thousands of real repositories has identified six core areas that make the biggest difference:

### 1. Commands

The exact commands to build, lint, test, and run your project. Be specific - include flags and options, not just tool names.

```markdown
## Commands

- Build: `npm run build`
- Lint: `npm run lint -- --fix`
- Test all: `npm test`
- Test single file: `npm test -- --testPathPattern=<filename>`
- Dev server: `npm run dev` (runs on port 3000)
- Type check: `npx tsc --noEmit`
```

Why this matters: agents that know the exact commands can verify their own work by running tests and linting before presenting results.

### 2. Testing

How tests are organized, what framework you use, and any conventions the agent should follow when writing new tests.

```markdown
## Testing

- Framework: pytest with pytest-asyncio for async tests
- Test location: tests mirror src structure (src/api/users.py -> tests/api/test_users.py)
- Naming: test files start with `test_`, test functions start with `test_`
- Fixtures: shared fixtures live in tests/conftest.py
- Run a single test: `pytest tests/api/test_users.py::test_create_user -v`
```

### 3. Project structure

A map of where things live. Agents work better when they know which directories to look in.

```markdown
## Project Structure

- `src/api/` - REST API endpoints (FastAPI)
- `src/core/` - Business logic, domain models
- `src/db/` - Database models and migrations (SQLAlchemy + Alembic)
- `src/workers/` - Background task processors
- `web/` - React frontend (Vite + TypeScript)
- `infra/` - Terraform infrastructure definitions
- `scripts/` - Developer utility scripts
```

### 4. Code style

Naming conventions, formatting rules, and preferred patterns. A short code snippet showing your style is worth more than paragraphs of description.

```markdown
## Code Style

- Python: Black formatting, isort for imports, Google-style docstrings
- TypeScript: Prettier + ESLint, functional components with hooks
- Naming: snake_case for Python, camelCase for TypeScript
- Prefer explicit over implicit - no magic imports or star exports
- Error handling: use custom exception classes from src/core/exceptions.py
```

### 5. Git workflow

Branching strategy, commit message conventions, and PR requirements.

```markdown
## Git Workflow

- Branch from `main`, prefix with `feat/`, `fix/`, or `chore/`
- Commit messages: conventional commits format (e.g., "feat: add user search endpoint")
- Squash merge PRs
- All PRs require passing CI and one approval
```

### 6. Boundaries

What the agent should never touch. This is one of the most important sections.

```markdown
## Do Not Modify

- `vendor/` - third-party code, managed externally
- `.env` files - contain secrets, never commit
- `infra/production/` - production infrastructure, requires manual review
- `src/db/migrations/` - generate migrations with Alembic, do not write by hand
- `package-lock.json` - only modify via npm install
```

---

## Hierarchical AGENTS.md for monorepos

You can place AGENTS.md files at multiple levels of your directory tree. Agents read the nearest file in the current directory or its parents. This is useful for monorepos where different areas have different conventions.

```
my-monorepo/
  AGENTS.md              # Shared conventions (git workflow, CI, etc.)
  services/
    api/
      AGENTS.md          # Python-specific: pytest, Black, FastAPI patterns
    frontend/
      AGENTS.md          # TypeScript-specific: Vitest, Prettier, React patterns
    ml-pipeline/
      AGENTS.md          # Python + notebooks: data conventions, model testing
```

Each sub-project's AGENTS.md can focus on what is unique to that area. The root-level file covers shared practices.

---

## AGENTS.md vs. other agent config files

Multiple AI tools have their own instruction file formats. The content across all of them overlaps significantly - build commands, coding standards, project structure. The differences are tool-specific features.

| File | Tool | Special Features |
|------|------|-----------------|
| **AGENTS.md** | Cross-tool standard | Universal, no special syntax, read by most agents |
| **CLAUDE.md** | Claude Code | Supports `@path` imports for modular instructions |
| **.cursorrules / .mdc** | Cursor | YAML frontmatter with activation modes (Always, Auto, Agent Requested) |
| **.github/copilot-instructions.md** | GitHub Copilot | Scoped `.instructions.md` files with glob patterns |
| **GEMINI.md** | Gemini | Gemini-specific instructions |

### The practical approach

Put shared instructions in AGENTS.md. Most tools read it. Use tool-specific files only when you need features unique to that tool.

If you already have a CLAUDE.md or .cursorrules file, you do not necessarily need to duplicate everything into AGENTS.md. But if you want your instructions to work across multiple tools, AGENTS.md is the common denominator.

---

## Best practices

### Keep it concise

Aim for 150 lines or fewer. Long files bury important information and waste agent context window tokens. If your AGENTS.md is longer than your README, it is probably too long.

### Be specific and actionable

Bad: "We use a modern JavaScript stack"
Good: "React 18 with TypeScript 5.3, Vite 5, and Tailwind CSS 3.4"

Bad: "Follow standard testing practices"
Good: "Run `npm test -- --coverage` and maintain >80% line coverage on new code"

### Show, do not tell

A code example communicates style more effectively than a paragraph of description.

```markdown
## API Endpoint Pattern

New endpoints should follow this structure:

\```python
@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(
    request: CreateUserRequest,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
) -> UserResponse:
    """Create a new user account."""
    user = await user_service.create(db, request)
    return UserResponse.from_orm(user)
\```
```

### Iterate based on agent behavior

Start with a minimal AGENTS.md. When you notice an agent making the same mistake repeatedly, add an instruction to address it. The best AGENTS.md files grow through iteration, not upfront planning.

### Treat it as code

Update AGENTS.md in the same PR when you change build processes, test conventions, or project structure. Stale instructions are worse than no instructions, because they actively mislead agents.

### Include the "why" when it matters

For non-obvious rules, a brief explanation helps the agent apply the rule correctly in edge cases.

```markdown
- Do not use `datetime.now()` directly. Use `src/core/clock.py` instead.
  This allows tests to control time without monkeypatching.
```

---

## A complete example

Here is a realistic AGENTS.md for a Python web application:

```markdown
# AGENTS.md

## Project

Order management API built with FastAPI and PostgreSQL.
Python 3.12, managed with Poetry.

## Commands

- Install dependencies: `poetry install`
- Run dev server: `poetry run uvicorn src.main:app --reload --port 8000`
- Run all tests: `poetry run pytest`
- Run single test: `poetry run pytest tests/path/to/test.py -v`
- Lint: `poetry run ruff check src tests`
- Format: `poetry run ruff format src tests`
- Type check: `poetry run mypy src`
- Generate migration: `poetry run alembic revision --autogenerate -m "description"`
- Apply migrations: `poetry run alembic upgrade head`

## Project Structure

- `src/api/` - API route handlers
- `src/core/` - Business logic and domain models
- `src/db/` - SQLAlchemy models and Alembic migrations
- `src/services/` - External service integrations
- `tests/` - Mirrors src structure

## Code Style

- Ruff for linting and formatting (config in pyproject.toml)
- Google-style docstrings on public functions
- Type hints on all function signatures
- Prefer `async def` for all route handlers
- Use dependency injection via FastAPI's `Depends()`

## Testing

- pytest with pytest-asyncio
- Use factories from `tests/factories.py` to create test data
- Tests run against a real PostgreSQL database (not mocks)
- Each test function gets a fresh transaction that rolls back

## Git

- Branch naming: `feat/`, `fix/`, `chore/` prefixes
- Conventional commits
- Squash merge to main

## Do Not Modify

- `alembic/versions/` - Do not edit migration files by hand
- `.env` and `.env.local` - Contains secrets
- `src/core/legacy_adapter.py` - Scheduled for removal, do not add new code here
```

---

## When AGENTS.md is not enough

AGENTS.md handles project-level context well. But there are situations where you need more:

- **Dynamic tool access** - If agents need to query databases, call APIs, or interact with external services, you need [MCP servers](../14-agent-protocols-mcp-and-a2a/README.md) or tools, not just instructions.
- **Reusable workflows** - If you want to package multi-step processes that agents can invoke, look at [Agent Skills](../18-agent-skills/README.md).
- **Cross-agent coordination** - If multiple agents need to work together, you need an [orchestration layer](../19-orchestrators/README.md).

AGENTS.md is the foundation. Think of it as the first layer of context that makes everything else work better.

---

## Try it yourself

1. Create an `AGENTS.md` file in one of your projects
2. Start with just the Commands and Project Structure sections
3. Use an AI coding agent on that project and see if it follows your instructions
4. When you notice the agent making mistakes, add instructions to address them
5. Keep refining until the agent consistently produces code that fits your project

---

## Key takeaways

- AGENTS.md is a plain Markdown file that tells AI agents how to work on your project
- It is an open standard supported by most major AI coding tools
- Focus on six areas: commands, testing, project structure, code style, git workflow, and boundaries
- Keep it concise (under 150 lines), specific, and up to date
- Start simple and iterate based on what the agent gets wrong
- Use hierarchical files in monorepos for area-specific guidance
- Put shared instructions in AGENTS.md, tool-specific features in their respective config files

---

## Further reading

- [AGENTS.md official site](https://agents.md/)
- [AGENTS.md specification on GitHub](https://github.com/agentsmd/agents.md)
- [How to write a great AGENTS.md - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
- [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [Custom instructions with AGENTS.md - OpenAI Codex](https://developers.openai.com/codex/guides/agents-md)

---

[Next Lesson: MCP Deep Dive ->](../17-mcp-deep-dive/README.md)
