# Lesson 19: orchestrators - managing agent control flow

## Introduction

In previous lessons, we covered what agents are, how they use tools, and how multiple agents can work together. But we have not yet looked closely at the layer that makes it all run - the orchestration layer.

The orchestrator is the control system that decides: What happens next? Which agent runs? What goes into the context? When do we stop? It is the part of the system that sits between the user's goal and the actual execution, coordinating everything.

If agents are the workers, the orchestrator is the project manager.

### ELI5: Think of an orchestrator like a film director

A film director does not act, operate the camera, or do lighting. Instead, they coordinate all the specialists: "Camera operator, get a close-up. Actor, deliver the line. Sound engineer, add the music." They decide the sequence, handle problems when a scene does not work, and keep everything moving toward the finished film.

An agent orchestrator does the same thing - it coordinates which agents run, in what order, with what inputs, and decides what to do when something goes wrong.

> **Key takeaway:** The orchestrator manages the control flow of your agent system. Choosing the right orchestration pattern is one of the most important architectural decisions you will make.

---

## What does an orchestrator actually do?

The orchestrator manages four core concerns:

### 1. Control flow

Deciding what happens next. Should the agent call a tool? Hand off to another agent? Ask the user for clarification? Stop because the goal is met?

### 2. Context assembly

Building the right context for each step. This means selecting which information goes into the LLM's context window - system instructions, relevant memory, tool results, conversation history - and keeping the window from overflowing.

### 3. State management

Tracking what has been done, what still needs to happen, what has succeeded, and what has failed. In multi-agent systems, this also means managing shared state between agents.

### 4. Error handling

Deciding what to do when things go wrong. Should the agent retry? Try a different approach? Fall back to a simpler method? Escalate to a human?

The orchestrator runs the core agent loop:

```
               +----> Assemble Context
               |            |
               |            v
  Receive Goal |      Invoke LLM (Reason)
               |            |
               |            v
               |      Execute Action (Act)
               |            |
               |            v
               +---- Observe Result
                            |
                   Goal met? ---> Return Result
```

Each iteration through this loop is one "step." The orchestrator decides when to loop and when to stop.

---

## Two types of orchestration

The most fundamental design decision is where your orchestrator falls on the spectrum between deterministic and dynamic control.

### Deterministic (workflow-based)

The control flow is predefined. The orchestrator follows a fixed blueprint - it does not consult an LLM to decide what happens next. Steps execute in a predetermined order with predetermined conditions.

**Strengths:**
- Predictable behavior - you know exactly what will happen
- Easy to debug - step through the workflow like regular code
- Fast - no LLM calls for orchestration decisions
- Reliable - no risk of the orchestrator going off-track

**Limitations:**
- Cannot handle novel situations the workflow was not designed for
- Requires upfront knowledge of all possible paths
- Changes to the workflow require code changes

**Example:** A document processing pipeline that always runs: extract text, classify document type, extract entities, validate, store.

### Dynamic (LLM-driven)

The orchestrator uses an LLM to decide what happens next. At each step, it reasons about the current state and chooses the next action. This is the classic ReAct loop.

**Strengths:**
- Handles novel and open-ended tasks
- Can adapt when plans fail
- Can work on tasks the developer did not anticipate

**Limitations:**
- Less predictable - the same input can produce different execution paths
- Harder to debug - "why did the agent do that?"
- More expensive - LLM calls for orchestration add up
- Can get stuck in loops or make poor routing decisions

**Example:** A research assistant that dynamically decides whether to search the web, query a database, read a document, or ask the user for clarification based on what it has learned so far.

### Hybrid (the practical choice)

Most production systems combine both approaches. They use deterministic orchestration for the overall structure while allowing LLM-driven flexibility within individual steps.

**Example:** A customer support system with a deterministic outer flow (receive ticket, classify, route to specialist, verify resolution, close) where each step internally uses an LLM agent that can reason freely about how to handle its specific task.

---

## Core orchestration patterns

Here are the most widely used patterns, with guidance on when each one fits:

### Sequential (pipeline)

Agents execute one after another in a defined order. Each agent's output becomes the next agent's input.

```
Input --> [Agent A] --> [Agent B] --> [Agent C] --> Output
```

**When to use:**
- Tasks with clear stages that build on each other
- Refinement workflows (draft, review, edit)
- Data processing pipelines (extract, transform, validate)

**When to avoid:**
- When stages are independent and could run in parallel
- When you need to backtrack (Agent C's failure requires re-running Agent A)

**Example:** Code generation pipeline: requirement analysis agent produces a spec, coding agent writes the implementation, testing agent writes tests, review agent checks for issues.

In ADK, this is the `SequentialAgent`:
```python
pipeline = SequentialAgent(
    name="code_pipeline",
    sub_agents=[analyzer, coder, tester, reviewer]
)
```

See the [ADK SequentialAgent documentation](https://google.github.io/adk-docs/agents/workflow-agents/sequential-agent/) for implementation details.

### Parallel (fan-out / gather)

Multiple agents execute at the same time on the same input. Results are collected and combined.

```
            +--> [Agent A] --+
            |                |
Input ------+--> [Agent B] --+--> Combine --> Output
            |                |
            +--> [Agent C] --+
```

**When to use:**
- Independent analysis from multiple perspectives
- Latency-sensitive tasks where parallel execution saves time
- Getting diverse viewpoints on the same input

**When to avoid:**
- When agents need each other's output to do their work
- When results might conflict and you have no resolution strategy

**Example:** Code review where a security agent, performance agent, and style agent all review the same PR simultaneously. Results are merged into a single review.

In ADK, this is the `ParallelAgent`:
```python
review = ParallelAgent(
    name="code_review",
    sub_agents=[security_reviewer, performance_reviewer, style_reviewer]
)
```

See the [ADK ParallelAgent documentation](https://google.github.io/adk-docs/agents/workflow-agents/parallel-agent/) for implementation details.

### Loop (iterative refinement)

An agent executes repeatedly until a condition is met. This includes two important sub-patterns:

**Generator-Critic (Maker-Checker):** One agent produces output, another evaluates it, and the loop continues until the evaluator approves.

```
+--> [Generator Agent] --> [Critic Agent] --+
|                              |            |
|         Not good enough -----+            |
|                                           |
+-------------------------------------------+
                    |
              Good enough --> Output
```

**Progressive Refinement:** A single agent improves its output through multiple passes, like an author revising a draft.

**When to use:**
- Quality-sensitive tasks where first attempts are rarely good enough
- Tasks with clear acceptance criteria
- Iterative improvement workflows

**When to avoid:**
- When you cannot define clear stopping criteria (risk of infinite loops)
- When the first attempt is usually good enough

**Important:** Always set a maximum iteration count. Without it, a loop can run forever if the critic never approves.

In ADK, this is the `LoopAgent`:
```python
refiner = LoopAgent(
    name="content_refiner",
    sub_agents=[writer, editor],
    max_iterations=5
)
```

See the [ADK LoopAgent documentation](https://google.github.io/adk-docs/agents/workflow-agents/loop-agent/) for implementation details.

### Routing (handoff / dispatch)

An input is classified and directed to a specialized agent. Only one agent handles each request.

```
            +--> [Billing Agent]
            |
Input --> [Router] +--> [Technical Support Agent]
            |
            +--> [General Inquiry Agent]
```

Routing can be:
- **Deterministic:** Rule-based classification (if the message contains "invoice," route to billing)
- **LLM-driven:** The router agent uses reasoning to pick the best specialist

**When to use:**
- Customer support with specialized departments
- Multi-domain systems where different inputs need different expertise
- When you want full control transfer (one active agent at a time)

**When to avoid:**
- When the request does not fit neatly into categories
- When multiple agents need to collaborate on the same request

### Hierarchical (Coordinator-Worker)

A lead agent coordinates the process while delegating tasks to specialized sub-agents.

```
                +---> [Research Agent]
                |
[Coordinator] --+---> [Analysis Agent]
                |
                +---> [Writing Agent]
```

The coordinator:
1. Breaks the overall goal into subtasks
2. Assigns subtasks to the right specialist
3. Monitors progress and handles dependencies
4. Combines results into a final output

**When to use:**
- Complex tasks that require multiple types of expertise
- Tasks where the plan is not known upfront and must be developed
- Research-style work where findings from one area inform what to explore next

**When to avoid:**
- Simple tasks that do not warrant the coordination overhead
- When a sequential pipeline would work just as well

In ADK, you can achieve this by wrapping sub-agents as tools using `AgentTool`, letting the coordinator call them like functions.

### Group chat (roundtable)

Multiple agents participate in a shared conversation, coordinated by a chat manager.

```
[Chat Manager]
      |
      +---> [Agent A]: "I think we should..."
      |
      +---> [Agent B]: "Building on that..."
      |
      +---> [Agent C]: "I disagree because..."
      |
      +---> [Agent A]: "Good point, let me revise..."
```

**When to use:**
- Consensus building
- Brainstorming where diverse perspectives improve the outcome
- Iterative validation (multiple experts review and refine)

**When to avoid:**
- When efficiency matters more than thoroughness (group chat is expensive in tokens)
- When more than three agents participate (conversations become chaotic)

---

## Choosing the right pattern

| Pattern | Predictability | Flexibility | Token Cost | Best For |
|---------|---------------|-------------|------------|----------|
| Sequential | High | Low | Low | Clear step-by-step processes |
| Parallel | High | Low | Medium (concurrent) | Independent analysis tasks |
| Loop | Medium | Medium | Variable | Quality refinement |
| Routing | High | Medium | Low | Multi-domain classification |
| Hierarchical | Medium | High | Higher | Complex multi-step research |
| Group Chat | Low | High | Highest | Consensus and brainstorming |

A decision flowchart:

```
Is the task a clear step-by-step process?
  Yes --> Sequential

Are there independent subtasks that can run simultaneously?
  Yes --> Parallel

Does the output need iterative improvement?
  Yes --> Loop

Does the task type determine which specialist handles it?
  Yes --> Routing

Is the task complex and requires planning and delegation?
  Yes --> Hierarchical

Does the task benefit from multiple perspectives and debate?
  Yes --> Group Chat
```

---

## Composing patterns

Real systems often nest patterns. Here is an example of a content creation system:

```
SequentialAgent("content_pipeline")
  |
  +-- ParallelAgent("research")
  |     +-- web_search_agent
  |     +-- database_query_agent
  |     +-- document_review_agent
  |
  +-- LlmAgent("writer")
  |     (uses research results to draft content)
  |
  +-- LoopAgent("refinement")
        +-- editor_agent
        +-- fact_checker_agent
        (loops until both approve)
```

This combines parallel research, sequential progression, and iterative refinement into one system. In ADK, each of these workflow agents can contain LLM agents, other workflow agents, or custom agents.

---

## Orchestration on Google Cloud with ADK

Google's Agent Development Kit provides three built-in workflow agent types for deterministic orchestration, plus LLM-driven coordination for dynamic scenarios.

### Built-in workflow agents

| Agent Type | Control Flow | ADK Class |
|-----------|-------------|-----------|
| Sequential | Run agents in order | `SequentialAgent` |
| Parallel | Run agents simultaneously | `ParallelAgent` |
| Loop | Repeat until condition met | `LoopAgent` |

These are deterministic - no LLM is involved in the orchestration decisions. The LLM is only used within the individual sub-agents for their specific tasks.

### LLM-driven coordination

For dynamic routing, use a parent `LlmAgent` (also called `Agent`) with sub-agents. The parent uses its LLM to decide which sub-agent to delegate to based on the conversation and current state. This is how you implement routing and hierarchical patterns.

### Custom agents

For orchestration logic that does not fit the built-in types, you can extend `BaseAgent` to create custom agents with arbitrary control flow.

### Agent-as-tool

ADK lets you wrap any agent as a tool using `AgentTool`. This allows a coordinator agent to call sub-agents as if they were function calls, receiving structured results back.

For full implementation details, see:
- [ADK Workflow Agents](https://google.github.io/adk-docs/agents/workflow-agents/)
- [ADK Multi-Agent Systems](https://google.github.io/adk-docs/agents/)
- [Multi-Agent Patterns in ADK - Google Developers Blog](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)

---

## Framework comparison

ADK is one of several frameworks that provide orchestration capabilities. Here is how the major options compare:

| Framework | Approach | Strengths | Considerations |
|-----------|----------|-----------|---------------|
| **Google ADK** | Three deterministic primitives (Sequential, Parallel, Loop) + LLM-driven coordination | Clean separation of workflow vs. reasoning. Deployment to Vertex AI Agent Engine. | Newer framework, smaller community than LangChain |
| **LangGraph** | Graph-based workflow with nodes and edges | Strongest support for complex branching and conditional logic. Mature observability. | Steeper learning curve |
| **CrewAI** | Role-based model where agents are defined like team members | Fastest time-to-value. Intuitive YAML-driven configuration. | May lack sophistication for complex enterprise scenarios |
| **AutoGen** (Microsoft) | Conversational architecture with dynamic role-playing | Good for human-in-the-loop and multi-party conversations. | Significant setup complexity for production |
| **Claude Agent SDK** | Orchestrator-worker with isolated context windows | Sub-agents use isolated context, sending only relevant info back. | Anthropic-specific |

The choice depends on your priorities: ADK if you want Vertex AI integration and clean workflow primitives, LangGraph if you need complex graph-based flows, CrewAI if you want fast setup with role-based teams.

---

## Best practices

### Start simple, add complexity when needed

A single agent with good tools often outperforms a multi-agent system with poor orchestration. Start with the simplest approach that works:

1. Single agent with tools
2. Sequential pipeline (if you need stages)
3. Parallel execution (if you need speed)
4. Full multi-agent coordination (if you need specialization)

Do not jump to a hierarchical multi-agent system because it sounds impressive. Add agents only when a single agent demonstrably cannot handle the task.

### Match the model to the task

Not every agent in your orchestration needs the same model. A classification router can use a fast, cheap model (Gemini Flash-Lite). A complex reasoning agent should use a capable model (Gemini Pro). This saves significant cost.

### Set iteration limits

Any loop or recursive orchestration must have a maximum iteration count. Without it, an agent that never satisfies its own criteria will run forever. A good default is 3-5 iterations for refinement loops.

### Validate between steps

In a sequential pipeline, validate each agent's output before passing it to the next. A malformed or off-topic result from Agent A will cascade through Agents B and C, wasting tokens and producing garbage.

### Manage context across agents

In multi-agent systems, context windows grow fast. Strategies to keep this under control:

- Summarize outputs before passing between agents
- Use external state stores for large shared data
- Give each agent only the context it needs, not everything
- Use context compaction (sliding windows, summarization) for long-running tasks

### Instrument for observability

Track performance per agent and per orchestration run:
- Latency per step
- Token usage per agent
- Success/failure rates per step
- End-to-end task completion rate

Use distributed tracing (e.g., OpenTelemetry) to follow a request through multiple agents. This is essential for debugging when things go wrong.

See [ADK Tracing documentation](https://google.github.io/adk-docs/) and [Google Cloud Trace](https://cloud.google.com/trace) for implementation guidance.

### Design for failure

Agents fail. Tools return errors. LLMs hallucinate. Your orchestrator needs to handle this gracefully:

- **Retry with backoff** for transient errors (API timeouts, rate limits)
- **Fallback strategies** for persistent failures (try a different tool, use a simpler model)
- **Circuit breakers** to prevent cascading failures
- **Human escalation** as the last resort for critical tasks

---

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Orchestration overkill | Using a multi-agent system for a task a single agent can handle | Start with one agent, add more only when needed |
| Agents without specialization | Multiple agents that all do roughly the same thing | Give each agent a clearly distinct role and expertise |
| Shared mutable state | Concurrent agents writing to the same state, causing race conditions | Use immutable messages or proper state locking |
| No iteration limits | Loops that run forever when the exit condition is never met | Always set max_iterations |
| Context window bloat | Passing full conversation history through every agent in a pipeline | Summarize and prune between steps |
| Deterministic when dynamic needed | Using a fixed pipeline for tasks that require adaptive reasoning | Use LLM-driven routing for unpredictable task types |
| Dynamic when deterministic works | Using LLM routing for tasks with a clear, known sequence | Use workflow agents to save cost and improve reliability |

---

## Key takeaways

- The orchestrator manages control flow, context assembly, state, and error handling
- Two fundamental types: deterministic (predictable, cheap, limited) and LLM-driven (flexible, expensive, less predictable)
- Most production systems use a hybrid - deterministic structure with LLM flexibility within steps
- Core patterns: sequential, parallel, loop, routing, hierarchical, group chat
- Patterns compose - nest them to build complex systems from simple pieces
- ADK provides SequentialAgent, ParallelAgent, and LoopAgent for deterministic orchestration, plus LLM-driven coordination for dynamic routing
- Start simple. A single well-equipped agent is better than a poorly orchestrated team.
- Set iteration limits, validate between steps, manage context aggressively, and design for failure

---

## Further reading

- [ADK Workflow Agents](https://google.github.io/adk-docs/agents/workflow-agents/)
- [Multi-Agent Patterns in ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
- [Anthropic - Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents)
- [Microsoft Azure - AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [CrewAI Documentation](https://docs.crewai.com/)

---

[Previous Lesson: Agent Skills](../18-agent-skills/README.md) | [Back to Course Overview](../README.md)
