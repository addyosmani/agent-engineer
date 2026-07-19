# Lesson 17: agent skills - reusable knowledge for agents

## Introduction

In [Lesson 3](../03-tools-giving-agents-hands/), we learned about tools - functions that let agents take actions like calling APIs, querying databases, and running code. Tools are about **doing things**.

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

<div id="skill-loading-timeline" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 900px; margin: 2rem auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 2px 12px rgba(0,0,0,0.08); padding: 24px; box-sizing: border-box;">
  <h3 style="margin: 0 0 4px 0; font-size: 1.2rem; color: #1a1a2e;">Progressive Skill Loading Timeline</h3>
  <p style="margin: 0 0 16px 0; font-size: 0.85rem; color: #666;">See how skills load incrementally to minimize context window cost.</p>

  <!-- Controls -->
  <div style="display: flex; gap: 8px; margin-bottom: 16px; flex-wrap: wrap; align-items: center;">
    <button id="slt-play" style="padding: 8px 20px; background: #4285f4; color: white; border: none; border-radius: 8px; cursor: pointer; font-size: 0.9rem; font-weight: 600; transition: background 0.2s;">&#9654; Play Animation</button>
    <button id="slt-reset" style="padding: 8px 16px; background: #e8eaed; color: #333; border: none; border-radius: 8px; cursor: pointer; font-size: 0.9rem; transition: background 0.2s;">Reset</button>
    <div style="margin-left: auto; display: flex; gap: 6px;">
      <span id="slt-phase-label" style="font-size: 0.8rem; color: #666; padding: 6px 12px; background: white; border-radius: 6px; border: 1px solid #e0e0e0;">Phase: Ready</span>
    </div>
  </div>

  <!-- Context Window Meter -->
  <div style="margin-bottom: 16px; background: white; border-radius: 10px; padding: 12px 16px; border: 1px solid #e0e0e0;">
    <div style="display: flex; justify-content: space-between; margin-bottom: 6px;">
      <span style="font-size: 0.8rem; font-weight: 600; color: #333;">Context Window Usage</span>
      <span id="slt-token-count" style="font-size: 0.8rem; color: #666;">0 / 128,000 tokens</span>
    </div>
    <div style="background: #e8eaed; border-radius: 6px; height: 20px; overflow: hidden; position: relative;">
      <div id="slt-meter" style="height: 100%; width: 0%; border-radius: 6px; transition: width 0.8s ease, background 0.5s; background: linear-gradient(90deg, #34a853, #4285f4);"></div>
    </div>
    <div style="display: flex; justify-content: space-between; margin-top: 4px;">
      <span style="font-size: 0.7rem; color: #999;">0%</span>
      <span style="font-size: 0.7rem; color: #999;">50%</span>
      <span style="font-size: 0.7rem; color: #999;">100%</span>
    </div>
  </div>

  <!-- Phase Indicators -->
  <div style="display: flex; gap: 4px; margin-bottom: 16px;">
    <div id="slt-p1-ind" style="flex: 1; height: 4px; border-radius: 2px; background: #e0e0e0; transition: background 0.3s;"></div>
    <div id="slt-p2-ind" style="flex: 1; height: 4px; border-radius: 2px; background: #e0e0e0; transition: background 0.3s;"></div>
    <div id="slt-p3-ind" style="flex: 1; height: 4px; border-radius: 2px; background: #e0e0e0; transition: background 0.3s;"></div>
  </div>

  <!-- Skills Grid -->
  <div id="slt-skills" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 12px; margin-bottom: 16px;"></div>

  <!-- SKILL.md Viewer -->
  <div id="slt-viewer" style="display: none; background: white; border-radius: 10px; padding: 16px; border: 1px solid #e0e0e0; margin-top: 12px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;">
      <span style="font-size: 0.85rem; font-weight: 600; color: #333;">SKILL.md Preview</span>
      <button id="slt-close-viewer" style="background: none; border: none; cursor: pointer; font-size: 1.1rem; color: #999; padding: 0 4px;">&times;</button>
    </div>
    <pre id="slt-viewer-content" style="background: #1e1e2e; color: #cdd6f4; padding: 14px; border-radius: 8px; font-size: 0.75rem; line-height: 1.5; overflow-x: auto; margin: 0; white-space: pre-wrap;"></pre>
  </div>
</div>

<script>
(function() {
  const skills = [
    {
      name: "Code Migration",
      icon: "&#128736;",
      desc: "Migrates codebases between frameworks, languages, or API versions.",
      color: "#4285f4",
      tokens: { l1: 95, l2: 3800, l3: 12000 },
      skillmd: `---\nname: code-migration\ndescription: >\n  Migrates codebases between frameworks,\n  languages, or API versions. Use when asked\n  to upgrade, port, or migrate code.\n---\n\n## Migration Process\n\n### 1. Analyze source code\n- Identify framework-specific patterns\n- Map dependencies to target equivalents\n- Flag breaking changes\n\n### 2. Generate migration plan\n- Order files by dependency graph\n- Estimate effort per file\n- Identify manual intervention points\n\n### 3. Execute migration\n- Transform code file by file\n- Run target framework tests\n- Validate output matches behavior`
    },
    {
      name: "Incident Response",
      icon: "&#128680;",
      desc: "Guides production incident triage, resolution, and post-mortem creation.",
      color: "#ea4335",
      tokens: { l1: 88, l2: 4200, l3: 9500 },
      skillmd: `---\nname: incident-response\ndescription: >\n  Guides incident response and post-mortem\n  creation. Use when there is a production\n  incident, outage, or service degradation.\n---\n\n## During an Incident\n\n### 1. Assess severity\n- Check monitoring dashboards\n- Determine blast radius\n- Classify as P0/P1/P2\n\n### 2. Investigate\n- Check recent deployments\n- Review error logs (last 1h)\n- Identify root cause\n\n### 3. Mitigate\n- Rollback if deployment-related\n- Scale up if load-related\n- Failover if infra-related`
    },
    {
      name: "Data Pipeline",
      icon: "&#128202;",
      desc: "Builds and debugs ETL/ELT data pipelines with quality checks.",
      color: "#34a853",
      tokens: { l1: 92, l2: 3500, l3: 15000 },
      skillmd: `---\nname: data-pipeline\ndescription: >\n  Builds and debugs ETL/ELT data pipelines\n  with quality checks. Use when asked to\n  create, fix, or optimize data workflows.\n---\n\n## Pipeline Design\n\n### 1. Source analysis\n- Identify data sources and formats\n- Check schemas and data types\n- Estimate data volumes\n\n### 2. Transform logic\n- Define transformation rules\n- Add data quality checks\n- Handle nulls and edge cases\n\n### 3. Load and validate\n- Write to destination\n- Run row count validation\n- Compare checksums`
    }
  ];

  let phase = 0;
  let activeSkill = null;
  let animating = false;
  const container = document.getElementById('slt-skills');

  function renderSkills() {
    container.innerHTML = '';
    skills.forEach((s, i) => {
      const card = document.createElement('div');
      card.id = 'slt-card-' + i;
      card.style.cssText = `background: white; border-radius: 10px; padding: 14px; border: 2px solid #e0e0e0; cursor: pointer; transition: all 0.4s ease; position: relative; overflow: hidden;`;
      card.innerHTML = getCardHTML(s, i, 0);
      card.addEventListener('click', () => showSkillMd(i));
      card.addEventListener('mouseenter', () => { card.style.borderColor = s.color; card.style.boxShadow = '0 2px 8px rgba(0,0,0,0.1)'; });
      card.addEventListener('mouseleave', () => { if (activeSkill !== i || phase < 2) { card.style.borderColor = activeSkill === i && phase >= 2 ? s.color : '#e0e0e0'; card.style.boxShadow = 'none'; } });
      container.appendChild(card);
    });
  }

  function getCardHTML(s, idx, p) {
    let html = `<div style="display:flex;align-items:center;gap:8px;margin-bottom:8px;"><span style="font-size:1.3rem;">${s.icon}</span><span style="font-weight:700;font-size:0.95rem;color:#1a1a2e;">${s.name}</span>`;
    if (p >= 2 && activeSkill === idx) html += `<span style="margin-left:auto;font-size:0.65rem;background:${s.color};color:white;padding:2px 8px;border-radius:10px;">ACTIVE</span>`;
    html += `</div>`;
    html += `<p style="font-size:0.78rem;color:#666;margin:0 0 8px 0;line-height:1.4;">${s.desc}</p>`;
    // Token bar
    let tokens = s.tokens.l1;
    let label = 'L1: Metadata';
    let barColor = '#e0e0e0';
    if (p >= 2 && activeSkill === idx) { tokens = s.tokens.l1 + s.tokens.l2; label = 'L2: Full Instructions'; barColor = s.color; }
    if (p >= 3 && activeSkill === idx) { tokens = s.tokens.l1 + s.tokens.l2 + s.tokens.l3; label = 'L3: Deep Resources'; barColor = s.color; }
    if (p >= 1) {
      const pct = Math.min((tokens / 20000) * 100, 100);
      html += `<div style="margin-top:6px;"><div style="display:flex;justify-content:space-between;margin-bottom:3px;"><span style="font-size:0.65rem;color:#999;">${label}</span><span style="font-size:0.65rem;color:#999;">${tokens.toLocaleString()} tokens</span></div>`;
      html += `<div style="background:#f0f0f0;border-radius:4px;height:6px;overflow:hidden;"><div style="width:${pct}%;height:100%;background:${barColor};border-radius:4px;transition:width 0.6s ease;"></div></div></div>`;
    }
    html += `<div style="margin-top:8px;font-size:0.65rem;color:#999;cursor:pointer;">Click to view SKILL.md</div>`;
    return html;
  }

  function updateCards(p) {
    skills.forEach((s, i) => {
      const card = document.getElementById('slt-card-' + i);
      if (card) {
        card.innerHTML = getCardHTML(s, i, p);
        if (p >= 2 && activeSkill === i) {
          card.style.borderColor = s.color;
          card.style.boxShadow = `0 2px 12px ${s.color}33`;
        } else if (p >= 3 && activeSkill !== i) {
          card.style.opacity = '0.5';
        } else {
          card.style.opacity = '1';
          card.style.borderColor = '#e0e0e0';
          card.style.boxShadow = 'none';
        }
        card.onclick = () => showSkillMd(i);
      }
    });
  }

  function updateMeter(p) {
    const meter = document.getElementById('slt-meter');
    const counter = document.getElementById('slt-token-count');
    let total = 0;
    if (p >= 1) total = skills.reduce((a, s) => a + s.tokens.l1, 0);
    if (p >= 2 && activeSkill !== null) total += skills[activeSkill].tokens.l2;
    if (p >= 3 && activeSkill !== null) total += skills[activeSkill].tokens.l3;
    const pct = (total / 128000) * 100;
    meter.style.width = pct + '%';
    if (pct < 5) meter.style.background = 'linear-gradient(90deg, #34a853, #4285f4)';
    else if (pct < 15) meter.style.background = 'linear-gradient(90deg, #4285f4, #fbbc04)';
    else meter.style.background = 'linear-gradient(90deg, #fbbc04, #ea4335)';
    counter.textContent = total.toLocaleString() + ' / 128,000 tokens';
  }

  function updatePhaseIndicators(p) {
    ['slt-p1-ind', 'slt-p2-ind', 'slt-p3-ind'].forEach((id, idx) => {
      document.getElementById(id).style.background = idx < p ? ['#4285f4', '#34a853', '#9333ea'][idx] : '#e0e0e0';
    });
    const labels = ['Ready', 'L1: Discovery', 'L2: Activation', 'L3: Deep Resources'];
    document.getElementById('slt-phase-label').textContent = 'Phase: ' + labels[p];
  }

  function showSkillMd(idx) {
    const viewer = document.getElementById('slt-viewer');
    const content = document.getElementById('slt-viewer-content');
    content.textContent = skills[idx].skillmd;
    viewer.style.display = 'block';
  }

  document.getElementById('slt-close-viewer').addEventListener('click', () => {
    document.getElementById('slt-viewer').style.display = 'none';
  });

  async function animate() {
    if (animating) return;
    animating = true;
    const btn = document.getElementById('slt-play');
    btn.disabled = true;
    btn.style.opacity = '0.6';

    // Phase 1: Discovery
    phase = 1;
    activeSkill = null;
    updatePhaseIndicators(1);
    updateCards(1);
    updateMeter(1);
    await sleep(1500);

    // Phase 2: Activation - pick Incident Response
    phase = 2;
    activeSkill = 1;
    updatePhaseIndicators(2);
    updateCards(2);
    updateMeter(2);
    await sleep(2000);

    // Phase 3: Deep Resources
    phase = 3;
    updatePhaseIndicators(3);
    updateCards(3);
    updateMeter(3);
    await sleep(1500);

    btn.disabled = false;
    btn.style.opacity = '1';
    animating = false;
  }

  function reset() {
    phase = 0;
    activeSkill = null;
    animating = false;
    updatePhaseIndicators(0);
    renderSkills();
    updateMeter(0);
    document.getElementById('slt-viewer').style.display = 'none';
    const btn = document.getElementById('slt-play');
    btn.disabled = false;
    btn.style.opacity = '1';
  }

  function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

  document.getElementById('slt-play').addEventListener('click', animate);
  document.getElementById('slt-reset').addEventListener('click', reset);

  renderSkills();
})();
</script>

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

### Write workflows, not essays

A skill that reads like an essay on best practices gets skimmed and ignored - the model generates plausible text about testing and then skips the testing. A skill that reads like a workflow gets executed: numbered steps, a checkpoint after each one, and a defined exit. If you have a 2,000-word reference document, converting it into a 400-word workflow usually makes it both shorter and far more likely to be followed. Prose describes; process directs.

### Anticipate rationalization

LLMs are excellent at rationalizing their way around steps they would rather skip - "this task is too simple to need a spec," "the tests obviously pass, no need to run them." Effective skills preempt the excuses by pairing each one with a rebuttal, right in the skill body:

```markdown
## No exceptions

| If you are thinking... | Then remember... |
|---|---|
| "This change is too small to test" | Small changes break things too. Run the tests. |
| "The requirements are obvious" | State your assumptions anyway - they may be wrong. |
| "I'll clean this up in a follow-up" | There is no follow-up. Finish the checklist now. |
```

Write down the excuses your own team makes; agents make the same ones.

### End every workflow in evidence

The last step of a skill should produce concrete proof that the work is done: a passing test run, clean lint output, a screenshot, a reviewed diff. "Seems right" is never an exit criterion. This matters because agents reliably over-report their own success - a workflow that ends with "verify your work" gets a confident "verified!", while a workflow that ends with "paste the output of the test run" gets the actual output.

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

For detailed implementation guidance, see the [ADK Skills documentation](https://adk.dev/skills/).

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
- Write workflows, not essays: anticipate the model's excuses, and end every workflow in concrete evidence
- Keep L2 instructions under 5,000 tokens; move detailed material to L3 references
- Skills are supported across multiple platforms: Claude Code, ADK, GitHub Copilot, OpenAI, Spring AI
- Skills, tools, and MCP work at different layers and complement each other

---

## Further reading

- [Agent Skills Specification](https://agentskills.io/specification)
- [ADK Skills Documentation](https://adk.dev/skills/)
- [Anthropic Skills Repository](https://github.com/anthropics/skills)
- [GitHub Copilot Agent Skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
- [Skills vs. MCP Tools - LlamaIndex](https://www.llamaindex.ai/blog/skills-vs-mcp-tools-for-agents-when-to-use-what)
- [Agent Skills - Addy Osmani](https://addyosmani.com/blog/agent-skills/) - skills as an enforced software lifecycle, with anti-rationalization patterns

---

[Previous Lesson: MCP Deep Dive](../16-mcp-deep-dive/) | [Next Lesson: Orchestrators ->](../18-orchestrators/)
