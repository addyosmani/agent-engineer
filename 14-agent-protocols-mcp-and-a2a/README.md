# Lesson 14: agent protocols - MCP and A2A

## Introduction

You can build an agent. You can give it tools. You can even build a team of agents that work together. But what happens when your agent needs to use a tool built by someone else? Or when your agent needs to collaborate with an agent built by a different team, at a different company, using a different framework?

Without standards, you end up writing custom integration code for every combination. That does not scale.

This lesson covers two open protocols that solve this problem: the **Model Context Protocol (MCP)** for connecting agents to tools and data, and the **Agent-to-Agent Protocol (A2A)** for enabling agents to collaborate across organizations and vendors. Together, they form the communication layer of the modern agent ecosystem.

---

## Why protocols matter

### The N x M Integration Problem

Imagine you have 5 agent frameworks and 10 tools. Without a standard protocol, each framework needs a custom connector for each tool. That is 5 x 10 = 50 custom integrations.

Now add 5 more tools. You need 25 more integrations. Add another framework and you need 15 more. The cost grows multiplicatively.

```
Without a standard protocol:

  Agent A ---custom---> Tool 1
  Agent A ---custom---> Tool 2
  Agent A ---custom---> Tool 3
  Agent B ---custom---> Tool 1
  Agent B ---custom---> Tool 2
  Agent B ---custom---> Tool 3
  ...
  (N agents x M tools = N*M integrations)


With a standard protocol:

  Agent A ---\                  /--> Tool 1
  Agent B -----> [Protocol] ---+--> Tool 2
  Agent C ---/                  \--> Tool 3
  (N + M integrations)
```

This is the same problem that USB solved for hardware. Before USB, every device had a proprietary connector. Printers needed parallel ports. Keyboards needed PS/2. Cameras needed serial cables. USB gave everyone a common interface, and the ecosystem exploded.

Protocols do the same thing for agents.

### Two kinds of communication

Agents need to communicate in two fundamentally different ways:

1. **Agent to Tool:** "Call this function with these parameters and give me the result." This is structured, specific, and synchronous. MCP handles this.

2. **Agent to Agent:** "I need help with this goal. Figure out how to accomplish it and let me know when you are done." This is open-ended, goal-oriented, and potentially asynchronous. A2A handles this.

Understanding this distinction is key to understanding why we need two protocols, not one.

---

## Model Context Protocol (MCP)

### What is MCP

MCP is an open standard for connecting language models and agents to external tools and data sources. Originally created by Anthropic and now widely adopted across the industry, MCP provides a universal interface between AI applications and the services they need to access.

Think of MCP as a **universal adapter** for AI tools. Just as a USB port lets you plug any USB device into any computer, MCP lets any agent use any MCP-compatible tool without custom integration code.

### The architecture: host, client, server

MCP uses a three-part architecture:

```
+------------------+
|      Host        |    (Your AI application - IDE, chatbot, agent)
|  +------------+  |
|  |   Client   |  |    (MCP client - manages connections to servers)
|  +-----+------+  |
+---------|---------+
          |
     MCP Protocol
     (JSON-RPC)
          |
+---------|---------+
|     Server        |    (MCP server - wraps a tool or data source)
|  +------------+   |
|  | Tool/Data  |   |    (The actual capability - database, API, file system)
|  +------------+   |
+-------------------+
```

**Host:** The application the user interacts with. This could be an IDE like VS Code, a chat interface, or your agent application. The host contains one or more MCP clients.

**Client:** The MCP client lives inside the host and manages connections to MCP servers. It handles protocol negotiation, message routing, and connection lifecycle. A single client can connect to multiple servers.

**Server:** An MCP server wraps a specific tool or data source and exposes it through the MCP protocol. There are servers for databases, file systems, APIs, SaaS products, and more. Anyone can build an MCP server.

### Key primitives

MCP defines three core primitives that servers can expose:

| Primitive | What It Is | Direction | Example |
|---|---|---|---|
| **Tools** | Functions the model can call | Model invokes, server executes | `search_database(query)`, `send_email(to, body)` |
| **Resources** | Data the model can read | Model requests, server provides | File contents, database records, API responses |
| **Prompts** | Template interactions | Server provides, user selects | "Summarize this document", "Debug this error" |

**Tools** are the most commonly used primitive. They work just like the function tools we covered in Lesson 3, but with a standardized interface that works across any MCP-compatible agent.

**Resources** provide context to the model without requiring a function call. Think of them as read-only data sources the model can reference.

**Prompts** are pre-built interaction templates that a server can offer. They help users discover what the server can do.

### The universal adapter analogy

Without MCP, connecting an agent to a new data source looks like this:

1. Read the data source's API documentation
2. Write authentication code
3. Write request/response handling
4. Write error handling
5. Define the tool schema for your specific framework
6. Test the integration

With MCP, it looks like this:

1. Install the MCP server for that data source
2. Connect your agent to it
3. Done

The MCP server handles authentication, request formatting, error handling, and schema definition. Your agent just needs to speak MCP.

### Benefits of MCP

- **Write once, use everywhere.** A tool built as an MCP server works with any MCP-compatible agent - ADK, Claude, Cursor, or any other host.

- **Dynamic tool discovery.** Agents can discover what tools are available at runtime instead of having everything hardcoded. Connect to a new MCP server and your agent automatically gains new capabilities.

- **Ecosystem leverage.** There are hundreds of community-built MCP servers for popular services. Need to connect to GitHub? Slack? A PostgreSQL database? There is probably an MCP server for it already.

- **Separation of concerns.** Tool builders focus on their tool. Agent builders focus on their agent. The protocol handles the interface between them.

### Using MCP tools in ADK

ADK has built-in support for MCP. You can connect to any MCP server and use its tools as if they were native ADK tools.

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import MCPToolset, SseServerParams

# Connect to an MCP server
mcp_tools = MCPToolset(
    connection_params=SseServerParams(
        url="http://localhost:3000/mcp",
    )
)

agent = Agent(
    name="mcp_agent",
    model="gemini-2.0-flash",
    instruction="You are a helpful assistant with access to external tools.",
    tools=[mcp_tools],
)
```

The agent discovers the available tools from the MCP server at runtime. If the server exposes a `search_database` tool and a `create_ticket` tool, your agent can use both without any additional code.

> **Learn more:** [MCP Tools in ADK](https://google.github.io/adk-docs/tools/mcp-tools/)

### Limitations and security considerations

MCP is powerful, but it comes with trade-offs you should understand:

**Tool shadowing.** If two MCP servers expose tools with similar names or descriptions, the model might get confused about which one to call. Be deliberate about which servers you connect and check for naming conflicts.

**Context window bloat.** Every connected MCP server adds tool definitions to the context window. Connect too many servers and you eat into the space available for actual conversation. Each tool definition typically consumes 100-500 tokens.

**No native scope limiting.** MCP does not have built-in fine-grained permission controls. If your agent connects to a database MCP server, it can potentially access any data that server exposes. You need to handle authorization at the server level or through guardrails.

**Trust and supply chain.** Community-built MCP servers are third-party code that your agent executes. Treat them with the same caution you would treat any open-source dependency. Review the code, check the maintainer, and run in sandboxed environments.

**Latency.** Every MCP tool call involves network communication with the MCP server. For time-sensitive applications, factor in this overhead.

| Consideration | Risk | Mitigation |
|---|---|---|
| Tool shadowing | Model calls wrong tool | Audit tool names, limit connected servers |
| Context bloat | Reduced reasoning quality | Connect only needed servers |
| No scope limits | Overly broad data access | Server-side auth, guardrails |
| Supply chain | Malicious or buggy servers | Code review, sandboxing |
| Latency | Slow tool responses | Local servers, caching |

---

## Agent-to-Agent Protocol (A2A)

### What is A2A?

A2A is an open protocol developed by Google for enabling agents to discover, communicate with, and delegate tasks to other agents - even agents built by different teams using different frameworks at different organizations.

While MCP handles the "agent talks to tool" problem, A2A handles the "agent talks to agent" problem.

### The professional network analogy

Think about how professionals collaborate in the real world. When you need legal advice, you do not become a lawyer. You find a qualified lawyer, explain what you need, and they handle it.

How do you find that lawyer?

1. **Discovery:** You look them up - maybe through a directory, a referral, or a professional network
2. **Capability check:** You review their profile to see if they handle your type of case
3. **Engagement:** You describe your situation and what you need
4. **Delegation:** They go away and work on it, sending you updates
5. **Delivery:** They come back with the result

A2A works the same way for agents:

1. **Discovery:** Your agent finds other agents through Agent Cards
2. **Capability check:** It reads the card to see what the agent can do
3. **Engagement:** It sends a task with a description of what needs to be done
4. **Delegation:** The remote agent works on it, sending status updates
5. **Delivery:** The remote agent returns the completed result

### Key concepts

#### Agent cards

An Agent Card is like a business card for an agent. It is a standardized JSON document that describes what the agent can do, how to communicate with it, and what authentication it requires.

```json
{
  "name": "Travel Booking Agent",
  "description": "Books flights and hotels based on travel requirements",
  "url": "https://travel-agent.example.com/a2a",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "book_flight",
      "name": "Book Flight",
      "description": "Search and book flights between cities"
    },
    {
      "id": "book_hotel",
      "name": "Book Hotel",
      "description": "Find and reserve hotel rooms"
    }
  ],
  "authentication": {
    "schemes": ["oauth2"]
  }
}
```

Agent Cards are hosted at a well-known URL (typically `/.well-known/agent.json`), making discovery straightforward. Your agent can check a known endpoint to see what another agent offers.

#### Tasks

A task is the fundamental unit of work in A2A. When one agent wants another agent to do something, it creates a task:

- **Task creation:** The calling agent sends a message describing what needs to be done
- **Task lifecycle:** The task moves through states - submitted, working, input-required, completed, or failed
- **Task updates:** The working agent can send progress updates so the caller knows what is happening
- **Task completion:** The working agent returns results as artifacts

#### Artifacts

Artifacts are the outputs of a task. They can be text, files, structured data, or any other content the working agent produces.

#### Event queues

A2A supports real-time communication through Server-Sent Events (SSE). This lets agents stream progress updates rather than waiting for the entire task to complete. This is especially important for long-running tasks where the calling agent (or a human) wants to see intermediate progress.

### How A2A compares to direct API calls

You might wonder: why not just call another agent's API directly? You could, but you would face the same N x M problem we discussed earlier. A2A gives you:

- **Standardized discovery** - Find agents without knowing their specific API
- **Common task lifecycle** - Every agent handles tasks the same way
- **Streaming by default** - Real-time updates without custom WebSocket code
- **Cross-framework compatibility** - Your ADK agent can work with a LangChain agent
- **Authentication standards** - Consistent security model across agents

### When to use A2A vs. MCP

This is one of the most important distinctions to understand:

| Aspect | MCP | A2A |
|---|---|---|
| **What talks** | Agent to tool | Agent to agent |
| **Communication style** | "Do this specific thing" | "Achieve this goal" |
| **Complexity of request** | Single function call | Open-ended task |
| **Intelligence on other side** | Tool (no reasoning) | Agent (has reasoning) |
| **Example** | "Query this database" | "Research this topic and write a report" |
| **Analogy** | Using a calculator | Hiring a consultant |

**MCP is for tools.** You know exactly what function you want to call and what parameters to pass. The tool executes and returns a result. There is no reasoning on the other side.

**A2A is for agents.** You describe a goal and let the other agent figure out how to accomplish it. The other agent has its own reasoning, its own tools, and its own approach.

**A practical example:** Suppose you are building a travel planning agent.

- You would use **MCP** to connect to a flight search API (a tool that takes departure city, arrival city, and date, and returns flights)
- You would use **A2A** to delegate to a hotel booking agent that can understand preferences like "somewhere quiet near the conference venue" and figure out the best options on its own

> **Learn more:** [A2A in ADK](https://google.github.io/adk-docs/a2a/) and [A2A Protocol Spec](https://developers.google.com/a2a)

---

## How MCP and A2A work together

MCP and A2A are not competing standards. They operate at different layers and complement each other.

```
+---------------------------------------------+
|              Your Agent                      |
|                                              |
|  "I need to book a trip to Tokyo"            |
|                                              |
|  +-------------------+  +----------------+   |
|  | MCP Client        |  | A2A Client     |   |
|  | (talks to tools)  |  | (talks to      |   |
|  |                   |  |  other agents)  |   |
|  +--------+----------+  +-------+--------+   |
+-----------|-----------------------|----------+
            |                       |
    +-------v--------+     +-------v--------+
    | MCP Servers     |     | Remote Agents  |
    |                 |     |                |
    | - Flight API    |     | - Hotel Agent  |
    | - Weather API   |     | - Budget Agent |
    | - Calendar      |     | - Review Agent |
    +--+---------+----+     +---+--------+---+
       |         |              |        |
       v         v              v        v
    [Flight   [Weather      [Hotel    [Budget
     Data]     Data]        Booking]   Analysis]
```

### The layered architecture

Think of it as layers:

1. **Tool layer (MCP):** Your agent connects to specific data sources and APIs through MCP servers. This gives it access to raw capabilities - search databases, call APIs, read files.

2. **Agent layer (A2A):** Your agent collaborates with other agents that have their own tools, reasoning, and expertise. This gives it access to higher-level capabilities - tasks that require judgment, planning, and multi-step execution.

3. **Orchestration layer (your agent):** Your agent decides when to use a tool directly (MCP) and when to delegate to another agent (A2A) based on the task at hand.

### A concrete scenario

A user asks your travel agent: "Plan a 3-day trip to Tokyo next month within a $3000 budget."

Your agent might:

1. **MCP call:** Check the user's calendar for available dates (calendar MCP server)
2. **MCP call:** Get current flight prices for those dates (flight API MCP server)
3. **A2A delegation:** Ask a hotel booking agent to find accommodations near Shibuya under $200/night
4. **A2A delegation:** Ask a local activities agent to suggest a 3-day itinerary
5. **MCP call:** Check weather forecasts for Tokyo during those dates (weather MCP server)
6. **Reasoning:** Combine all results, check against budget, and present a plan

Notice the pattern: MCP for specific data retrieval, A2A for tasks requiring another agent's expertise and judgment.

---

## ELI5: understanding MCP and A2A

### MCP is like a power adapter

You know how different countries have different electrical outlets? If you travel from the US to the UK, your laptop charger will not fit. You need an adapter.

MCP is that adapter for AI agents. Every tool used to have its own proprietary plug (custom API integration). MCP gives everyone a universal outlet. Plug any tool into MCP, and any agent can use it.

The tool itself does not get smarter. A power adapter does not make your laptop faster. But it makes your laptop usable in places it could not work before. Same with MCP - it makes tools accessible to agents that could not reach them before.

### A2A is like a phone call between coworkers

Now imagine you are working on a big project at a company. You handle the engineering, but you need marketing materials. You do not learn marketing yourself. You call your coworker in the marketing department.

You say: "We are launching the new API next Tuesday. Can you put together a launch blog post and social media plan?"

Your coworker says: "Sure, I will draft something and send you updates as I go."

A2A is that phone call. One agent (you) calls another agent (marketing) with a goal. The other agent uses their own skills and tools to accomplish it. They send updates along the way. And they deliver the finished work when it is done.

You did not need to know what tools marketing uses. You did not need to understand their process. You just needed to describe what you wanted and trust them to figure it out.

### Why we need both

Going back to the coworker analogy:

- **MCP** is like the tools on your desk - your keyboard, monitor, code editor. You use them directly.
- **A2A** is like your coworkers - you delegate work to them and they use their own tools.

You need both. Some things you do yourself with your tools. Other things you ask a specialist to handle.

---

## Security considerations for both protocols

### MCP Security

- **Server authentication:** Verify the identity of MCP servers before connecting. Use TLS for all communication.
- **Least privilege:** Only connect to the MCP servers your agent actually needs. Each additional server increases your attack surface.
- **Input validation:** MCP servers should validate all parameters they receive. Do not trust that the model will always send well-formed inputs.
- **Audit logging:** Log all MCP tool calls for debugging and security review.

### A2A Security

- **Agent verification:** Before delegating tasks, verify the remote agent's identity through its Agent Card and authentication scheme.
- **Data minimization:** Only share the information the remote agent needs to complete the task. Do not send your entire context.
- **Result validation:** Treat results from remote agents with appropriate skepticism. Verify critical outputs before acting on them.
- **Access control:** Define which agents can access which of your agent's capabilities.

### Defense in depth

Both protocols benefit from a layered security approach:

1. **Transport security:** TLS everywhere
2. **Authentication:** Verify identities on both sides
3. **Authorization:** Limit what each connection can do
4. **Monitoring:** Watch for unusual patterns
5. **Guardrails:** Validate inputs and outputs at every boundary

---

## The current state of the ecosystem

### MCP Ecosystem

MCP has seen rapid adoption since its introduction. The ecosystem includes:

- **Hundreds of MCP servers** for popular services (databases, cloud platforms, SaaS tools, development tools)
- **Support in major AI platforms** including Claude, ADK, VS Code, and many others
- **Growing community** of contributors building and maintaining servers

### A2A Ecosystem

A2A is newer and the ecosystem is still developing:

- **ADK support** for both creating A2A-compatible agents and connecting to remote A2A agents
- **Reference implementations** demonstrating common patterns
- **Growing interest** from organizations building multi-agent systems

### What to expect

Both protocols are actively evolving. Expect to see:

- More MCP servers for enterprise tools and services
- More agent frameworks adopting A2A support
- Better tooling for discovering, testing, and monitoring protocol connections
- Standardization of security patterns and best practices

---

## Practical tips

### When getting started with MCP

1. **Start with official servers.** Use well-maintained MCP servers from trusted sources before trying community ones.
2. **Test locally first.** Run MCP servers locally during development before pointing to remote ones.
3. **Monitor token usage.** Each connected MCP server adds tool definitions to your context. Keep track of how much context space your tools consume.
4. **Version pin your servers.** MCP servers are software dependencies. Pin versions to avoid surprises.

### When getting started with A2A

1. **Start with agents you control.** Build two agents yourself and practice A2A communication before connecting to external agents.
2. **Define clear contracts.** Be specific about what tasks you expect a remote agent to handle and what outputs you expect.
3. **Handle failures gracefully.** Remote agents can be slow, unavailable, or return unexpected results. Build retry and fallback logic.
4. **Log everything.** Multi-agent communication is hard to debug. Detailed logging is essential.

---

## Key takeaways

1. **Protocols solve the N x M integration problem.** Without standards, every agent-tool and agent-agent combination needs custom code. MCP and A2A replace that with universal interfaces.

2. **MCP connects agents to tools.** It is a universal adapter that lets any agent use any MCP-compatible tool. Think USB for AI.

3. **A2A connects agents to agents.** It enables agents to discover, communicate with, and delegate tasks to other agents across organizations and frameworks.

4. **MCP and A2A complement each other.** MCP operates at the tool layer (specific function calls). A2A operates at the agent layer (goal-oriented tasks). Use both together for maximum flexibility.

5. **Security requires attention at both layers.** Verify identities, minimize data sharing, validate results, and log everything.

---

## Where to learn more

- **MCP Tools in ADK:** [https://google.github.io/adk-docs/tools/mcp-tools/](https://google.github.io/adk-docs/tools/mcp-tools/)
- **A2A in ADK:** [https://google.github.io/adk-docs/a2a/](https://google.github.io/adk-docs/a2a/)
- **A2A Protocol Specification:** [https://developers.google.com/a2a](https://developers.google.com/a2a)
- **MCP Specification:** [https://modelcontextprotocol.io](https://modelcontextprotocol.io)

---

## What is next?

You have covered the fundamentals and the building blocks. In the final lesson, we will step back and look at the big picture - where to go from here, what resources to explore, and how to continue growing as an agent builder.

[Next: Lesson 15 - Where to Go From Here -->](../15-where-to-go-from-here/README.md)
