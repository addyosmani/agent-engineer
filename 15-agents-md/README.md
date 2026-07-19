# Lesson 15: AGENTS.md - giving agents project context

## Introduction

When a new engineer joins your team, you do not just hand them the codebase and say "good luck." You give them onboarding docs, explain the build system, point out where the tests live, and warn them about the parts of the code they should not touch.

AI coding agents need the same thing. Without project context, an agent will guess at your conventions, use the wrong test runner, miss your branching strategy, and generally produce code that does not fit your project. AGENTS.md solves this by giving agents a structured onboarding document they can read before they start working.

### ELI5: Think of AGENTS.md like a welcome packet

Imagine your company gives every new hire a one-page cheat sheet on day one. It lists the WiFi password, how to run the build, where to find the style guide, and what never to touch. AGENTS.md is that cheat sheet - but for AI agents working on your codebase.

> **Key takeaway:** AGENTS.md is a simple Markdown file you place in your repo to tell AI agents how to work on your project. Most major AI coding tools read it automatically.

<div id="agents-md-builder" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 920px; margin: 2rem auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); padding: 24px; box-sizing: border-box;">
  <div style="text-align: center; margin-bottom: 16px;">
    <h3 style="margin: 0 0 4px 0; color: #1a1a2e; font-size: 1.3rem;">Interactive AGENTS.md Template Builder</h3>
    <p style="margin: 0; color: #666; font-size: 0.9rem;">Toggle sections on/off to build your AGENTS.md. Click to expand and see sample content.</p>
  </div>
  <div style="display: flex; gap: 16px; flex-wrap: wrap;">
    <!-- Left: Sections -->
    <div id="amd-sections" style="flex: 1; min-width: 290px;"></div>
    <!-- Right: Preview -->
    <div style="flex: 1; min-width: 290px;">
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04); position: relative;">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
          <div style="font-weight: 600; color: #1a1a2e; font-size: 0.95rem;">📄 AGENTS.md Preview</div>
          <button id="amd-copy" style="padding: 6px 14px; background: #4285f4; color: white; border: none; border-radius: 8px; cursor: pointer; font-weight: 600; font-size: 0.78rem; transition: all 0.2s;">Copy to Clipboard</button>
        </div>
        <pre id="amd-preview" style="background: #1a1a2e; color: #e0e0e0; padding: 14px; border-radius: 8px; font-size: 0.73rem; line-height: 1.6; overflow-x: auto; white-space: pre-wrap; margin: 0; max-height: 520px; overflow-y: auto;"></pre>
        <div id="amd-stats" style="margin-top: 8px; font-size: 0.75rem; color: #888; text-align: right;"></div>
      </div>
    </div>
  </div>
</div>

<script>
(function() {
  const sections = [
    {
      id: 'commands', title: 'Commands', icon: '⚡', importance: 'required', active: true, expanded: false,
      desc: 'Build, test, lint, and run commands for your project.',
      content: `## Commands\n\n- Build: \`npm run build\`\n- Lint: \`npm run lint -- --fix\`\n- Test all: \`npm test\`\n- Test single file: \`npm test -- --testPathPattern=<filename>\`\n- Dev server: \`npm run dev\` (runs on port 3000)\n- Type check: \`npx tsc --noEmit\``
    },
    {
      id: 'testing', title: 'Testing', icon: '🧪', importance: 'required', active: true, expanded: false,
      desc: 'Test framework, patterns, naming conventions, and how to run tests.',
      content: `## Testing\n\n- Framework: Jest with React Testing Library\n- Test location: tests mirror src structure (src/utils/foo.ts -> tests/utils/foo.test.ts)\n- Naming: test files use \`.test.ts\` suffix\n- Fixtures: shared fixtures in tests/__fixtures__/\n- Run single test: \`npm test -- --testPathPattern=foo.test.ts\`\n- Coverage: maintain >80% on new code`
    },
    {
      id: 'structure', title: 'Code Structure', icon: '📁', importance: 'required', active: true, expanded: false,
      desc: 'Directory layout and where key files live in the project.',
      content: `## Project Structure\n\n- \`src/api/\` - REST API endpoints\n- \`src/core/\` - Business logic and domain models\n- \`src/db/\` - Database models and migrations\n- \`src/services/\` - External service integrations\n- \`web/\` - React frontend (Vite + TypeScript)\n- \`scripts/\` - Developer utility scripts\n- \`tests/\` - Mirrors src structure`
    },
    {
      id: 'style', title: 'Code Style', icon: '🎨', importance: 'recommended', active: false, expanded: false,
      desc: 'Formatting, naming conventions, and preferred patterns.',
      content: `## Code Style\n\n- Prettier for formatting (config in .prettierrc)\n- ESLint with strict TypeScript rules\n- Naming: camelCase for variables/functions, PascalCase for types/components\n- Prefer explicit return types on exported functions\n- Use \`async/await\` over raw Promises\n- Imports: group by external, internal, then relative`
    },
    {
      id: 'git', title: 'Git Workflow', icon: '🌿', importance: 'recommended', active: false, expanded: false,
      desc: 'Branch naming, commit message format, and PR conventions.',
      content: `## Git Workflow\n\n- Branch from \`main\`, prefix with \`feat/\`, \`fix/\`, or \`chore/\`\n- Commit messages: conventional commits (e.g., "feat: add user search")\n- Squash merge PRs to main\n- All PRs require passing CI and one approval\n- Keep PRs focused - one feature or fix per PR`
    },
    {
      id: 'boundaries', title: 'Boundaries', icon: '🚫', importance: 'optional', active: false, expanded: false,
      desc: 'What the agent should NOT modify or touch.',
      content: `## Do Not Modify\n\n- \`vendor/\` - third-party code, managed externally\n- \`.env\` files - contain secrets, never commit\n- \`migrations/\` - generate with CLI, do not write by hand\n- \`package-lock.json\` - only modify via npm install\n- \`dist/\` - auto-generated build output`
    }
  ];

  const importColors = { required: '#ea4335', recommended: '#fbbc04', optional: '#34a853' };

  function render() {
    const container = document.getElementById('amd-sections');
    container.innerHTML = sections.map((s, i) => `
      <div style="background: white; border-radius: 12px; margin-bottom: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.04); overflow: hidden; border-left: 4px solid ${s.active ? '#4285f4' : '#e0e0e0'}; transition: all 0.3s;">
        <div style="display: flex; align-items: center; padding: 12px 14px; cursor: pointer; gap: 10px;" onclick="document.dispatchEvent(new CustomEvent('amd-toggle-expand',{detail:${i}}))">
          <span style="font-size: 1.1rem;">${s.icon}</span>
          <div style="flex: 1;">
            <div style="font-weight: 600; font-size: 0.9rem; color: #1a1a2e;">${s.title}</div>
            <div style="font-size: 0.72rem; color: #888;">${s.desc}</div>
          </div>
          <span style="font-size: 0.65rem; padding: 2px 8px; border-radius: 10px; background: ${importColors[s.importance]}18; color: ${importColors[s.importance]}; font-weight: 600; text-transform: uppercase;">${s.importance}</span>
          <label style="position: relative; display: inline-block; width: 40px; height: 22px; flex-shrink: 0;" onclick="event.stopPropagation()">
            <input type="checkbox" ${s.active ? 'checked' : ''} style="opacity: 0; width: 0; height: 0;" onchange="document.dispatchEvent(new CustomEvent('amd-toggle',{detail:${i}}))">
            <span style="position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background: ${s.active ? '#4285f4' : '#ccc'}; border-radius: 22px; transition: 0.3s;"></span>
            <span style="position: absolute; height: 16px; width: 16px; left: ${s.active ? '21px' : '3px'}; bottom: 3px; background: white; border-radius: 50%; transition: 0.3s;"></span>
          </label>
        </div>
        ${s.expanded ? `<div style="padding: 0 14px 12px; border-top: 1px solid #f0f0f0;">
          <pre style="background: #f8f9fa; padding: 10px; border-radius: 8px; font-size: 0.72rem; line-height: 1.5; margin: 8px 0 0; white-space: pre-wrap; color: #444;">${s.content}</pre>
        </div>` : ''}
      </div>
    `).join('');
    updatePreview();
  }

  function updatePreview() {
    const active = sections.filter(s => s.active);
    let md = '# AGENTS.md\n\n';
    if (active.length === 0) {
      md += '(Toggle sections on the left to build your AGENTS.md)';
    } else {
      active.forEach(s => { md += s.content + '\n\n'; });
    }
    document.getElementById('amd-preview').textContent = md.trim();
    const lines = md.trim().split('\n').length;
    document.getElementById('amd-stats').textContent = `${active.length} sections | ${lines} lines | ${active.length <= 4 ? 'Concise' : lines > 150 ? 'Consider trimming' : 'Good length'}`;
  }

  document.addEventListener('amd-toggle', e => {
    sections[e.detail].active = !sections[e.detail].active;
    render();
  });
  document.addEventListener('amd-toggle-expand', e => {
    sections[e.detail].expanded = !sections[e.detail].expanded;
    render();
  });
  document.getElementById('amd-copy').addEventListener('click', function() {
    const text = document.getElementById('amd-preview').textContent;
    navigator.clipboard.writeText(text).then(() => {
      this.textContent = 'Copied!';
      this.style.background = '#34a853';
      setTimeout(() => { this.textContent = 'Copy to Clipboard'; this.style.background = '#4285f4'; }, 2000);
    });
  });
  render();
})();
</script>

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
| **.cursor/rules/*.mdc** | Cursor | YAML frontmatter with activation modes (Always, Auto, Agent Requested). Note: the older `.cursorrules` format is deprecated. |
| **.github/copilot-instructions.md** | GitHub Copilot | Scoped `.instructions.md` files with glob patterns |
| **GEMINI.md** | Gemini | Gemini-specific instructions |

### The practical approach

Put shared instructions in AGENTS.md. Most tools read it. Use tool-specific files only when you need features unique to that tool.

If you already have a CLAUDE.md or `.cursor/rules/` files, you do not necessarily need to duplicate everything into AGENTS.md. But if you want your instructions to work across multiple tools, AGENTS.md is the common denominator.

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

### Write it by hand - do not auto-generate it

Many tools offer to generate this file for you (Claude Code's `/init`, for example). Resist the temptation to accept the output as-is. Research on real repositories points in two directions at once: human-authored files containing non-obvious project knowledge measurably speed agents up, while LLM-generated context files have been found to slightly *reduce* task success and inflate cost. The reason is redundancy - an auto-generated file mostly describes things the agent would discover on its own (directory layout, tech stack, module structure), so every line just dilutes the context without adding information.

The test for every line is the **discovery filter**: *can the agent find this out by reading the code?* If yes, delete the line.

| Line | Verdict | Why |
|---|---|---|
| "This project is a monorepo with services under `services/`" | Delete | Discoverable from the first directory listing |
| "Use `uv` for package management, not `pip`" | Keep | Not discoverable, operationally significant |
| "The frontend is React with TypeScript" | Delete | Obvious from `package.json` |
| "Run tests with `--no-cache` or you will get false positives" | Keep | A landmine the agent cannot guess |
| "`legacy/` looks dead but is imported by three production modules" | Keep | The code actively misleads without this warning |

A useful mental model: treat AGENTS.md as **a living list of codebase smells you have not fixed yet**, not a permanent configuration. If agents repeatedly stumble over something, that is often a codebase design problem wearing a documentation costume. The best fix is usually to restructure the code, add a linter rule, or improve the tests - and then delete the line.

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

- **Dynamic tool access** - If agents need to query databases, call APIs, or interact with external services, you need [MCP servers](../14-agent-protocols-mcp-and-a2a/) or tools, not just instructions.
- **Reusable workflows** - If you want to package multi-step processes that agents can invoke, look at [Agent Skills](../17-agent-skills/).
- **Cross-agent coordination** - If multiple agents need to work together, you need an [orchestration layer](../18-orchestrators/).

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
- Write it by hand and apply the discovery filter: if the agent can find it by reading the code, delete the line
- Use hierarchical files in monorepos for area-specific guidance
- Put shared instructions in AGENTS.md, tool-specific features in their respective config files

---

## Further reading

- [AGENTS.md official site](https://agents.md/)
- [AGENTS.md specification on GitHub](https://github.com/agentsmd/agents.md)
- [How to write a great AGENTS.md - GitHub Blog](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
- [Stop Using /init for AGENTS.md - Addy Osmani](https://addyosmani.com/blog/agents-md/) - the research behind the discovery filter
- [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [Custom instructions with AGENTS.md - OpenAI Codex](https://developers.openai.com/codex/guides/agents-md)

---

[Previous Lesson: Agent Protocols - MCP and A2A](../14-agent-protocols-mcp-and-a2a/) | [Next Lesson: MCP Deep Dive ->](../16-mcp-deep-dive/)
