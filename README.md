# Agent Engineer - a course for software engineers

Learn the fundamentals of AI agents and how to build them with Google Cloud AI.

## Who is this for?

Software engineers who want to understand what AI agents are, how they work, and how to build them. No prior AI/ML experience required - just curiosity and some Python knowledge.

## Course overview

This course is split into three parts:

**Part 1: Fundamentals (101)** - Understand the core concepts behind AI agents. These lessons are platform-agnostic and focused on building your mental model.

**Part 2: Building and shipping (201)** - Put those fundamentals into practice using Google Cloud AI, Vertex AI, and the Agent Development Kit (ADK).

**Part 3: Deep dives (301)** - Go deeper on specific topics that matter for real-world agent development.

## Lessons

### Part 1: fundamentals

| # | Lesson | What you will learn |
|---|--------|-------------------|
| 01 | [What are AI agents?](./01-what-are-ai-agents/README.md) | The big picture - what agents are, why they matter, and when to use them |
| 02 | [How agents think](./02-how-agents-think/README.md) | LLMs as the reasoning engine - how models plan, decide, and generate |
| 03 | [Tools - giving agents hands](./03-tools-giving-agents-hands/README.md) | Function calling, tool design, and connecting agents to the real world |
| 04 | [Agentic design patterns](./04-agentic-design-patterns/README.md) | ReAct, reflection, planning, and other core patterns |
| 05 | [Memory and context](./05-memory-and-context/README.md) | How agents remember things - sessions, context windows, and long-term memory |
| 06 | [Planning and reasoning](./06-planning-and-reasoning/README.md) | How agents break down complex tasks and make decisions |
| 07 | [Multi-agent systems](./07-multi-agent-systems/README.md) | When one agent is not enough - coordination, delegation, and teamwork |
| 08 | [Agentic RAG](./08-agentic-rag/README.md) | Going beyond basic retrieval - agents that search, evaluate, and refine |
| 09 | [Evaluating and testing agents](./09-evaluating-and-testing-agents/README.md) | How to know if your agent actually works - metrics, evals, and observability |
| 10 | [Guardrails and safety](./10-guardrails-and-safety/README.md) | Keeping agents trustworthy - security, alignment, and responsible AI |

### Part 2: building and shipping

| # | Lesson | What you will learn |
|---|--------|-------------------|
| 11 | [From prototype to production](./11-from-prototype-to-production/README.md) | The journey from demo to deployed - CI/CD, rollout, and operations |
| 12 | [Getting started with Vertex AI and ADK](./12-getting-started-with-vertex-and-adk/README.md) | The Google Cloud AI stack for agents - what is available and how it fits together |
| 13 | [Building your first agent](./13-building-your-first-agent/README.md) | Hands-on - build a working agent with ADK step by step |
| 14 | [Agent protocols - MCP and A2A](./14-agent-protocols-mcp-and-a2a/README.md) | How agents talk to tools and to each other using open standards |

### Part 3: deep dives

| # | Lesson | What you will learn |
|---|--------|-------------------|
| 15 | [AGENTS.md](./15-agents-md/README.md) | Giving AI coding agents context about your project with a standard config file |
| 16 | [MCP deep dive](./16-mcp-deep-dive/README.md) | How MCP works under the hood, MCP vs. CLI tools, and security considerations |
| 17 | [Agent skills](./17-agent-skills/README.md) | Packaging reusable domain expertise as portable skill modules |
| 18 | [Orchestrators](./18-orchestrators/README.md) | Managing agent control flow - patterns, frameworks, and best practices |
| 19 | [Where to go from here](./19-where-to-go-from-here/README.md) | Resources, codelabs, community, and next steps |

## How to use this course

- **Read in order** if you are new to agents. Each lesson builds on the previous one.
- **Jump around** if you already know the basics. Each lesson is self-contained enough to read on its own.
- **Follow the links** to official docs, codelabs, and tutorials for hands-on practice. We intentionally link out to maintained resources rather than duplicating API docs or code samples that go stale.

## Philosophy

This course follows a few principles:

- **Analogies first.** We use everyday comparisons to explain complex concepts before diving into technical details.
- **Fundamentals over frameworks.** Understand the "why" before the "how." Frameworks change, but the core ideas stick around.
- **Link, don't duplicate.** For API references, code samples, and setup instructions, we point to official Google Cloud docs and codelabs. This keeps our content focused on concepts and ensures you always see up-to-date information.
- **Honest about trade-offs.** Every architectural choice has costs. We try to show both sides.

## Prerequisites

- Basic Python knowledge (functions, classes, HTTP requests)
- A Google Cloud account ([free trial available](https://cloud.google.com/free))
- Familiarity with REST APIs and JSON

## Additional resources

- [Google Cloud AI documentation](https://cloud.google.com/ai)
- [Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
- [Agent Development Kit (ADK) documentation](https://google.github.io/adk-docs/)
- [Google Cloud AI codelabs](https://codelabs.developers.google.com/?cat=AI)
- [Gemini API documentation](https://ai.google.dev/docs)

## Contributing

Found a typo? Have a suggestion? PRs and issues are welcome. See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the Apache 2.0 License - see the [LICENSE](./LICENSE) file for details.