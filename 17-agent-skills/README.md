# Lesson 18: agent skills - reusable knowledge for agents

## Introduction

In [Lesson 3](../03-tools-giving-agents-hands/README.md), we learned about tools - functions that let agents take actions like calling APIs, querying databases, and running code. Tools are about **doing things**.

Skills are about **knowing things**. A skill packages domain expertise - instructions, best practices, decision frameworks, and reference materials - into a modular unit that an agent can discover and use when needed.

Think about the difference between giving someone a wrench (a tool) and giving them a repair manual (a skill). The wrench lets them turn bolts. The manual tells them which bolts to turn, in what order, and what to watch out for.

### ELI5: Think of skills like recipe cards in a kitchen

A professional kitchen has tools (knives, pans, ovens) and recipe cards. A new chef can pick up a knife without instructions. But to make a specific dish, they need the recipe card - it tells them which tools to use, in what order, at what temperature, and what the result should look like.

Agent skills work the same way. They are the recipe cards that tell an agent how to approach a specific type of task, which tools to use, and what good output looks like.

> **Key takeaway:** Skills encode domain expertise as portable, reusable packages. Tools let agents act. Skills tell agents how and when to act.

---

## Why skills exist

Consider this scenario: your team has an agent that helps with code reviews. You want it to follow your team's specific review checklist, flag common patterns you care about, and format its feedback in a particular way.

You could put all of this in the agent's system prompt. But system prompts get crowded fast. If you add review instructions, deployment procedures, documentation standards, and testing conventions all into one prompt, you end up with a bloated context window and an agent that is mediocre at everything.

Skills solve this by letting you:

1. **Package expertise separately** - Each skill is its own file, focused on one domain
2. **Load on demand** - Skills are only loaded when relevant, saving context window space
3. **Share across teams** - A well-written skill can be reused across projects and agents
4. **Iterate independently** - Update a skill without changing the agent's core configuration

### The context window problem

This is the key technical motivation. Every token in the context window has a cost - both in money and in attention. If you load 50,000 tokens of instructions at startup, the agent pays that cost on every single turn, even when most of those instructions are irrelevant.

Skills use progressive disclosure to keep the cost low:

- At startup, load only skill names and descriptions (~100 tokens each)
- When a skill is triggered, load its full instructions
- Only load reference materials when the instructions explicitly need them

One analysis showed this approach reducing a 150,000-token workflow to approximately 2,000 tokens at startup.

---

## The skill specification

Skills follow an open specification maintained at [agentskills.io](https://agentskills.io/specification). The format is simple:

### Directory structure

```
my-skill/
  SKILL.md          # Required: metadata + instructions
  references/       # Optional: additional documentation
  assets/           # Optional: templates, schemas, data files
  scripts/          # Optional: executable code
```

The only required file is `SKILL.md`. Everything else is optional.

### SKILL.md format

A SKILL.md file has two parts: YAML frontmatter for metadata, and Markdown content for instructions.

```markdown
---
name: code-review
description: >
  Reviews pull requests following team standards. Checks for
  security issues, test coverage, naming conventions, and
  documentation. Use when asked to review code or a PR.
---

## Code Review Process

When reviewing code, follow these steps in order:

### 1. Security check
- Look for hardcoded secrets, SQL injection, XSS vulnerabilities
- Check that user input is validated and sanitized
- Verify authentication and authorization on new endpoints

### 2. Test coverage
- New public functions should have tests
- Edge cases should be covered (empty input, null values, errors)
- Check that tests actually assert meaningful behavior

### 3. Naming and structure
- Functions and variables should have descriptive names
- Files should be in the correct directory per project conventions
- No single function should exceed 50 lines

### 4. Documentation
- Public APIs should have docstrings
- Non-obvious logic should have inline comments
- README should be updated if behavior changes

### Output format
Present findings as a list grouped by category (Security, Tests,
Style, Docs). For each finding, include the file path, line number,
severity (high/medium/low), and a suggested fix.
```

### Frontmatter fields

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Unique identifier, lowercase with hyphens (e.g., `code-review`) |
| `description` | Yes | What the skill does and when to trigger it (up to 1024 chars) |
| `license` | No | License for the skill |
| `compatibility` | No | Environment requirements (e.g., "Requires Python 3.10+") |
| `metadata` | No | Arbitrary key-value pairs (author, version, tags) |

The `description` field is critical. It is the primary way agents decide whether to activate a skill. Write it to clearly describe both what the skill does and when it should be used.

---

## Progressive disclosure - the three levels

Skills are designed to load incrementally. This is the key architectural idea that makes them practical:

### Level 1: Metadata (always loaded)

At startup, the agent loads only the `name` and `description` from the frontmatter of every installed skill. This costs roughly 100 tokens per skill. Even with 50 skills installed, the startup cost is only about 5,000 tokens.

The agent uses this metadata to decide: "Given the current task, is this skill relevant?"

### Level 2: Instructions (loaded on activation)

When the agent decides a skill is relevant, it loads the full SKILL.md body. This is where the step-by-step instructions, decision frameworks, and examples live. The recommendation is to keep this under 5,000 tokens.

### Level 3: Resources (loaded on demand)

Files in the `references/`, `assets/`, and `scripts/` directories are loaded only when the Level 2 instructions reference them. These might include:

- `references/security-checklist.md` - Extended security review criteria
- `assets/api-schema.json` - API specification for validation
- `assets/response-template.md` - Template for formatted output
- `scripts/run-linter.sh` - Script the agent can execute

This three-level approach means you can write very detailed skills without paying the context cost upfront.

```
Startup:      [L1: name + description]     ~100 tokens per skill
                        |
Task matches: [L2: full instructions]      ~2,000-5,000 tokens
                        |
As needed:    [L3: reference files]        Variable
```

---

## Skills vs. tools vs. MCP

These three concepts work at different layers. Understanding the distinction helps you decide which to use:

| Dimension | Skills | Tools / Function Calling | MCP |
|-----------|--------|------------------------|-----|
| **What it provides** | Knowledge and instructions | Executable functions | Standardized protocol for tool integration |
| **Analogy** | A recipe card | A kitchen appliance | A power outlet standard |
| **Nature** | Natural language guidance | Code that runs | JSON-RPC communication layer |
| **Execution** | LLM interprets instructions | Deterministic function call | Protocol for calling remote tools |
| **Latency** | Local (just text) | Depends on function | Network round-trip |
| **Best for** | Encoding expertise, workflows, review criteria | Taking actions (API calls, file ops, queries) | Connecting to external services with auth and discovery |
| **Context cost** | Low (progressive loading) | Medium (schema per tool) | Higher (full schemas upfront) |

### How they work together

In a typical agent, all three are used:

1. **Skills** tell the agent how to approach the task and which tools to use
2. **Tools** (function calling) let the agent execute actions
3. **MCP** provides a standard way to connect to remote tool servers

Example: A "deploy-to-staging" skill might include instructions like:
- Step 1: Run the test suite using the `run_tests` tool
- Step 2: Check the staging environment status using the Kubernetes MCP server
- Step 3: If tests pass and staging is healthy, deploy using the `deploy` tool
- Step 4: Verify the deployment by checking health endpoints

The skill provides the workflow logic. The tools and MCP servers provide the execution capability.

---

## Writing good skills

### Focus on one domain

A skill should do one thing well. Instead of a "development" skill that covers everything, create separate skills for code review, deployment, documentation, and testing.

### Write for the LLM, not a human

Skills are interpreted by a language model. Be explicit about:
- **When** to use the skill (triggering conditions)
- **What** steps to follow (ordered process)
- **How** to handle edge cases (decision points)
- **What** good output looks like (examples or templates)

### Include decision points

Real expertise includes knowing when to deviate from the standard process:

```markdown
### Handling large PRs (>500 lines changed)

If the PR changes more than 500 lines:
- Focus review on the most critical files first (API endpoints, auth, data models)
- Skip cosmetic issues (formatting, naming) unless they affect readability
- Suggest splitting the PR if the changes cover multiple unrelated concerns
```

### Show expected output

Include examples of what the skill's output should look like:

```markdown
### Example output

**Security - High**
`src/api/auth.py:45` - Password is compared using `==` instead of
`hmac.compare_digest()`. This is vulnerable to timing attacks.
Suggested fix: Replace with `hmac.compare_digest(stored_hash, provided_hash)`
```

### Keep L2 instructions under 5,000 tokens

If your instructions are getting long, move detailed reference material to L3 files in the `references/` directory and reference them from the main instructions:

```markdown
For the full security checklist, refer to `references/security-checklist.md`.
```

---

## Skills in Google ADK

Google's Agent Development Kit supports skills through the `SkillToolset` class. Here is a conceptual overview of how it works:

### File-based skills (recommended)

Place skill directories in a `skills/` folder within your agent project:

```
my-agent/
  agent.py
  skills/
    code-review/
      SKILL.md
      references/
        security-checklist.md
    deploy/
      SKILL.md
      scripts/
        pre-deploy-check.sh
```

The agent discovers and loads skills from this directory. Only the L1 metadata is loaded at startup. Full instructions load when the agent activates the skill.

### Code-based skills

For dynamic skill creation or modification, ADK also supports defining skills in code using the `Skill` model class. This is useful when skill content needs to change based on runtime conditions.

For detailed implementation guidance, see the [ADK Skills documentation](https://google.github.io/adk-docs/skills/).

---

## Skills across platforms

The Agent Skills specification has been adopted by multiple platforms:

| Platform | Support | Details |
|----------|---------|---------|
| **Claude Code** (Anthropic) | Yes | Skills as `/slash-commands`, [anthropics/skills](https://github.com/anthropics/skills) repo |
| **Google ADK** | Yes | `SkillToolset` class, file-based and code-based |
| **GitHub Copilot** | Yes | Works in VS Code, CLI, and Copilot coding agent |
| **OpenAI** | Yes | Agents SDK with skills support |
| **Spring AI** | Yes | Java ecosystem via `spring-ai-agent-utils` |

The specification is maintained by a community working group and published at [agentskills.io](https://agentskills.io/specification). Because the format is just Markdown files in a directory, skills are portable across platforms that support the spec.

---

## Practical examples

### Example 1: Database migration skill

```markdown
---
name: database-migration
description: >
  Creates and reviews database migrations. Use when the user asks to
  add, modify, or remove database tables or columns, or when reviewing
  migration files.
---

## Creating Migrations

1. Verify the current migration state: run `alembic heads` to check for conflicts
2. Create the migration: `alembic revision --autogenerate -m "description"`
3. Review the generated migration file for:
   - Correct up/down operations (both directions should work)
   - No data loss in down migration
   - Appropriate indexes for new columns
   - Nullable columns for existing tables (to avoid breaking existing rows)
4. Test the migration: `alembic upgrade head` then `alembic downgrade -1`

## Common Pitfalls

- Adding a NOT NULL column to an existing table without a default value
  will fail if the table has existing rows. Always add a default or make
  it nullable first, then backfill.
- Renaming columns requires a two-step migration: add new column, migrate
  data, drop old column. Alembic's autogenerate does not handle renames.
- Large table alterations should be done in batches on production. Add a
  note in the migration file if the table has >1M rows.
```

### Example 2: Incident response skill

```markdown
---
name: incident-response
description: >
  Guides incident response and post-mortem creation. Use when there is
  a production incident, outage, or when creating post-mortem documents.
---

## During an Incident

1. Assess severity using the service dashboard at `monitoring.internal/overview`
2. Check recent deployments: `gcloud run revisions list --service=api --limit=5`
3. Check error rates: `gcloud logging read "severity>=ERROR" --limit=50 --freshness=1h`
4. If a recent deployment is suspect, rollback:
   `gcloud run services update-traffic api --to-revisions=PREVIOUS_REVISION=100`

## After Resolution

Create a post-mortem document using the template in `assets/postmortem-template.md`
with these sections filled in:
- Timeline of events (with timestamps)
- Root cause analysis
- Impact (users affected, duration, data loss if any)
- What went well in the response
- Action items with owners and due dates
```

---

## When to use skills vs. other approaches

| Situation | Use Skills | Use Something Else |
|-----------|-----------|-------------------|
| Team has specific review criteria | Yes | - |
| Agent needs to follow a multi-step workflow | Yes | - |
| Agent needs to call an API | No | Use a tool or MCP |
| Agent needs project context (build commands, structure) | No | Use AGENTS.md |
| Workflow is simple and one-off | No | Just put it in the prompt |
| Knowledge changes rarely and is domain-specific | Yes | - |
| Knowledge changes frequently or needs live data | No | Use RAG or tools |

---

## Key takeaways

- Skills package domain expertise as portable, reusable Markdown files
- They use progressive disclosure (L1/L2/L3) to minimize context window cost
- Skills tell agents how and when to act; tools let agents actually act
- The SKILL.md file is the only required component - frontmatter for metadata, body for instructions
- Write skills focused on one domain, with clear steps, decision points, and output examples
- Keep L2 instructions under 5,000 tokens; move detailed material to L3 references
- Skills are supported across multiple platforms: Claude Code, ADK, GitHub Copilot, OpenAI, Spring AI
- Skills, tools, and MCP work at different layers and complement each other

---

## Further reading

- [Agent Skills Specification](https://agentskills.io/specification)
- [ADK Skills Documentation](https://google.github.io/adk-docs/skills/)
- [Anthropic Skills Repository](https://github.com/anthropics/skills)
- [GitHub Copilot Agent Skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
- [Skills vs. MCP Tools - LlamaIndex](https://www.llamaindex.ai/blog/skills-vs-mcp-tools-for-agents-when-to-use-what)

---

[Previous Lesson: MCP Deep Dive](../17-mcp-deep-dive/README.md) | [Next Lesson: Orchestrators ->](../19-orchestrators/README.md)
