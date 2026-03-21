# Lesson 17: MCP deep dive - connecting agents to the world

## Introduction

In [Lesson 14](../14-agent-protocols-mcp-and-a2a/README.md), we introduced MCP (Model Context Protocol) and A2A at a high level. This lesson goes deeper on MCP specifically - how it actually works under the hood, when it adds real value, when simpler approaches are better, and how to think about security.

We also tackle one of the most debated questions in the AI engineering community: when should you use MCP servers vs. just letting your agent use CLI tools directly?

### ELI5: Think of MCP like a power strip with safety features

Your laptop can plug directly into a wall outlet. That works fine at home. But in an office with 50 devices, you want a power strip with surge protection, individual switches, and a circuit breaker. MCP is that power strip - it adds a layer of management between the agent and the tools it uses. Whether you need that layer depends on how many tools you have, who is using them, and how much control you need.

> **Key takeaway:** MCP is a powerful protocol for connecting agents to external tools and data, but it has real trade-offs in cost and complexity. Understanding when MCP adds value versus when simpler approaches work better is a critical skill for agent builders.

---

## MCP architecture - how it actually works

MCP follows a client-server architecture with three roles:

### The three roles

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|   MCP Host       |     |   MCP Client     |     |   MCP Server     |
|   (Your app)     |---->|   (Protocol      |---->|   (Tool          |
|                  |     |    handler)       |     |    provider)     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```

- **Host** - The application where the agent runs (Claude Desktop, an IDE, your custom app). It creates and manages MCP clients.
- **Client** - Handles the protocol communication. Maintains a 1:1 connection with a single MCP server. Manages capability negotiation and message routing.
- **Server** - Exposes tools, resources, and prompts to the client. Each server typically wraps a specific service (a database, an API, a file system).

### Communication

All messages use JSON-RPC 2.0. The protocol supports two transport mechanisms:

| Transport | Use Case | How It Works |
|-----------|----------|-------------|
| **stdio** | Local tools | Server runs as a subprocess, communicates over stdin/stdout. Simple, fast, no network overhead. |
| **Streamable HTTP** | Remote tools | Uses a single HTTP endpoint with bidirectional communication. Supports serverless deployment (Lambda, Cloud Functions). |

Note: The original SSE (Server-Sent Events) transport was deprecated in the March 2025 spec revision. SSE was one-directional and required two endpoints. Streamable HTTP replaced it with a single-endpoint, bidirectional design.

### MCP primitives

MCP servers can expose three types of capabilities:

| Primitive | What It Is | Who Controls It | Example |
|-----------|-----------|----------------|---------|
| **Tools** | Functions the agent can call | The model decides when to use them | `query_database`, `send_email`, `create_file` |
| **Resources** | Data the agent can read | The application or user selects them | Database schemas, file contents, API documentation |
| **Prompts** | Reusable prompt templates | The user invokes them | "Summarize this codebase", "Review this PR" |

In practice, Tools are by far the most widely used primitive. As of late 2025, 99% of MCP clients support Tools, while Resources and Prompts have around 30-35% adoption.

---

## The MCP vs. CLI debate

This is one of the most actively discussed topics in AI engineering right now. The core question: if an AI agent can run shell commands, why does it need MCP?

### The argument for CLI

Many MCP servers are thin wrappers around tools that already have excellent CLIs. The GitHub MCP Server reimplements functionality available through `gh`. The Docker MCP Server wraps `docker` commands. The Kubernetes MCP Server wraps `kubectl`.

LLMs already know how to use these CLIs. They were trained on millions of man pages, Stack Overflow answers, and GitHub repositories. When an agent uses `gh pr list`, it uses knowledge it already has. When it uses an MCP server, it needs to load the tool schema into its context window first.

The numbers are stark:

| Metric | CLI Approach | MCP Approach |
|--------|-------------|-------------|
| **Token cost (simple query)** | ~1,400 tokens | ~44,000 tokens (32x more) |
| **Initialization cost** | Near zero | Can be 50,000+ tokens for schema loading |
| **Reliability (benchmark)** | 100% | 72% |
| **Setup required** | None (tools already installed) | Install and configure MCP server |

The token cost difference comes from MCP needing to load full tool schemas (names, descriptions, parameter types, return types) into the context window. A database MCP server with 106 tools consumed 54,600 tokens just to initialize - before any actual work happened.

### The argument for MCP

The properties that make MCP expensive are the same properties that make it governable:

**Security and authentication.** CLI tools run with the user's ambient permissions. If the agent can run `rm -rf /`, it will if it decides to. MCP provides a permission boundary. The spec mandates OAuth 2.1 with PKCE for HTTP-based servers, giving you standardized authentication, token rotation, and revocation.

**Multi-user environments.** When an agent acts as you, CLI's ambient security is fine. When an agent acts on behalf of other people - reading customers' repos, writing to their Jira, messaging their Slack - you need per-user auth, scoped permissions, and audit trails. MCP provides a framework for this.

**Tool discovery.** MCP servers advertise their capabilities through schemas. An agent can discover what tools are available at runtime without being told upfront. This matters when tools change or when different users have access to different tools.

**Structured I/O.** MCP tools have typed inputs and outputs. CLI output is unstructured text that the agent must parse. For simple tools this is fine, but for complex APIs with nested JSON responses, structured output is more reliable.

### When to use which

| Situation | Recommended Approach | Why |
|-----------|---------------------|-----|
| Developer working locally | CLI | Zero setup, the agent already knows the tools, cheapest option |
| Well-known tools (git, docker, kubectl, jq) | CLI | LLM has strong training data, reliable parsing |
| Single-user agent | CLI | Ambient permissions are acceptable |
| Multi-user / multi-tenant | MCP | Need per-user auth and scoped permissions |
| Enterprise with audit requirements | MCP | Need structured logging and access control |
| High-frequency narrow tool set | MCP | Schema cost amortizes over many calls |
| Broad tool surface, occasional use | CLI | Avoid paying schema cost for tools rarely used |
| Custom internal API with no CLI | MCP | No existing CLI to leverage |
| Tools that change frequently | MCP | Dynamic discovery handles changes automatically |

### The practical answer

Most production systems use both. Claude Code, for example, has a Bash tool for direct CLI access and also supports MCP servers. The decision is per-integration, not system-wide.

A reasonable default: **start with CLI. Add MCP when you hit a specific limitation that MCP solves** - usually authentication, multi-tenancy, or structured tool discovery.

### mcp2cli: Bridging the gap

An interesting tool called [mcp2cli](https://github.com/knowsuchagency/mcp2cli) converts any MCP server into a CLI at runtime. Instead of loading all tool schemas upfront, the agent queries `--list` and `--help` only when needed. This has shown 96-99% token reduction in benchmarks while keeping MCP's structured API underneath.

---

## MCP security - what can go wrong

MCP introduces a new attack surface. The OWASP Foundation published an [MCP Top 10](https://owasp.org/www-project-mcp-top-10/) security risk list. Here are the ones that matter most for agent builders:

### 1. Tool poisoning

A malicious or compromised MCP server can return manipulated results. If your agent trusts tool output without verification, it can be led to take harmful actions.

**Mitigation:** Validate tool outputs. Use multiple sources for critical decisions. Implement output filtering.

### 2. Tool shadowing

A malicious tool mimics a legitimate one. If an agent has access to two tools with similar names - say `query_database` from a trusted server and `query_db` from an untrusted one - it might use the wrong one.

**Mitigation:** Control which MCP servers your agent can connect to. Review tool names and descriptions. Use allowlists for tool access.

### 3. Excessive permissions

MCP does not have native scope limiting. A database MCP server might expose both read and write operations. If your agent only needs to read, it can still write.

**Mitigation:** Build or use MCP servers that expose only the operations you need. Implement server-side access controls. Use an API gateway (like Apigee) in front of MCP servers for fine-grained permission management.

### 4. Context window bloat

Too many MCP tools degrade agent performance. Each tool definition consumes context tokens. A server with 100+ tools can exhaust a significant portion of the context window before any real work begins.

**Mitigation:** Keep tool counts per server reasonable (under 20). Use multiple specialized servers instead of one large one. Consider lazy loading of tool schemas.

### 5. Secret exposure

Analysis of 5,200 open-source MCP servers found that over half rely on long-lived static API keys. Only about 8.5% use modern auth like OAuth.

**Mitigation:** Use short-lived scoped credentials. Store secrets in a secret manager (like Google Cloud Secret Manager), not in environment variables or config files. Rotate credentials regularly.

### Security checklist for MCP deployments

- [ ] Audit which MCP servers your agent connects to
- [ ] Review tool schemas for overly broad permissions
- [ ] Use OAuth 2.1 for remote MCP servers
- [ ] Store secrets in a secret manager, not env vars
- [ ] Validate and sanitize tool outputs before acting on them
- [ ] Limit the number of tools per server
- [ ] Log all tool invocations for audit trails
- [ ] Test with adversarial inputs (tool poisoning, prompt injection through tool results)
- [ ] Use an API gateway for enterprise MCP deployments
- [ ] Run MCP servers in sandboxed environments where possible

---

## MCP on Google Cloud

Google Cloud provides several integration points for MCP:

### ADK and MCP

Google's Agent Development Kit (ADK) has built-in support for MCP tools. You can connect to any MCP server and use its tools within your ADK agent.

For details on configuring MCP tools in ADK, see the [ADK MCP Tools documentation](https://google.github.io/adk-docs/tools/mcp-tools/).

### Apigee as an MCP gateway

For enterprise deployments, [Apigee](https://cloud.google.com/apigee) can serve as an API and agent gateway for MCP. This adds:

- Rate limiting and quota management
- Authentication and authorization policies
- Analytics and monitoring
- Tool registry and discovery
- Traffic management across multiple MCP servers

This is particularly useful when you have many teams deploying MCP servers and need centralized governance.

### Model Armor

[Model Armor](https://cloud.google.com/security/products/model-armor) can filter and validate inputs and outputs flowing through MCP tool calls, adding protection against prompt injection and data exfiltration through tool interactions.

---

## Building an MCP server - key decisions

If you decide to build an MCP server for your service, here are the key decisions:

### Transport choice

| Question | stdio | Streamable HTTP |
|----------|-------|----------------|
| Is the server local to the agent? | Yes | Either |
| Do you need remote access? | No | Yes |
| Do you need to deploy serverlessly? | No | Yes |
| Is latency critical? | Yes | Less so |

### Tool granularity

Prefer fine-grained tools over coarse-grained ones:

- Good: `get_user_by_id`, `list_users`, `create_user`, `update_user_email`
- Bad: `manage_users` (one tool that does everything based on a mode parameter)

Fine-grained tools give the LLM clearer choices and produce better results. But keep the total count manageable - 5-20 tools per server is a good range.

### Naming and descriptions

Tool names and descriptions are the primary way the LLM decides which tool to use. Invest time in making them clear:

- Name should describe the action: `search_documents_by_topic` not `search`
- Description should explain when to use it, what it returns, and any important constraints
- Parameter descriptions should include types, valid ranges, and examples
- Error messages should help the LLM recover: "User not found. Try searching by email instead of ID."

### Output design

Keep tool outputs concise. The output goes into the agent's context window, and large responses eat into the budget.

- Return only what the agent needs to make its next decision
- Paginate large result sets
- Summarize rather than dump raw data
- Use structured formats (JSON) for machine-parseable output

---

## Putting it all together - a decision tree

When deciding how to connect your agent to an external service:

```
Do you need to connect to an external service?
|
+-- Is there a well-known CLI for it? (git, docker, aws, gcloud, kubectl)
|   |
|   +-- Yes: Does the agent need multi-user auth or audit trails?
|   |   |
|   |   +-- No: Use the CLI directly
|   |   +-- Yes: Use MCP with OAuth
|   |
|   +-- No: Continue below
|
+-- Is there an existing MCP server for it?
|   |
|   +-- Yes: Is it actively maintained and from a trusted source?
|   |   |
|   |   +-- Yes: Use the MCP server
|   |   +-- No: Consider building your own or using CLI/API directly
|   |
|   +-- No: Continue below
|
+-- Does the service have a REST API?
    |
    +-- Yes: Build an MCP server or use ADK OpenAPI tools
    +-- No: Build a custom function tool or MCP server
```

---

## Key takeaways

- MCP provides structured tool integration with schemas, auth, and discovery - but at a token cost
- CLI tools are cheaper and often more reliable for well-known developer tools
- The decision is per-integration, not system-wide - most production agents use both MCP and CLI
- Start with CLI as the default; add MCP when you need auth, multi-tenancy, or tool discovery
- MCP security requires active attention - audit servers, limit permissions, validate outputs
- On Google Cloud, ADK supports MCP natively, and Apigee can serve as an enterprise MCP gateway
- Keep MCP servers focused: 5-20 well-described tools per server, concise outputs, clear error messages

---

## Further reading

- [MCP Specification](https://modelcontextprotocol.io/)
- [ADK MCP Tools](https://google.github.io/adk-docs/tools/mcp-tools/)
- [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/)
- [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [mcp2cli - Bridge MCP to CLI](https://github.com/knowsuchagency/mcp2cli)
- [Google Cloud Apigee](https://cloud.google.com/apigee)

---

[Previous Lesson: AGENTS.md](../16-agents-md/README.md) | [Next Lesson: Agent Skills ->](../18-agent-skills/README.md)
