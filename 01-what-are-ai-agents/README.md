# Lesson 1: What are AI Agents?

## Introduction

You have probably used ChatGPT, Gemini, or Claude to answer a question or write some code. You typed something in, got a response, and moved on. That is a language model doing its thing - predicting useful text based on your input.

An AI agent is something different. An agent can **think**, **act**, and **remember**. It does not just answer your question - it figures out what steps to take, uses tools to carry those steps out, and adjusts its approach based on what happens along the way.

Think of it this way: if you hired a new engineer and only let them talk but never touch a keyboard, open a browser, or read documentation, they would be limited. That is an LLM on its own. Now give that engineer access to your codebase, a terminal, your company docs, and the ability to ask clarifying questions. That is an agent.

This lesson covers what AI agents are, how they differ from plain language models, what components make them work, and when you should (and should not) use them.

---

## What is an AI agent in plain terms?

An AI agent is a software system that uses a language model as its core reasoning engine, combined with the ability to take actions in the real world. Those actions might include:

- Searching the web
- Querying a database
- Calling an API
- Reading or writing files
- Sending an email
- Running code

The key distinction is **autonomy**. A plain LLM responds to a single prompt. An agent receives a goal and then independently decides what steps to take, executes those steps, observes the results, and continues until the goal is met (or it determines the goal cannot be met).

### The new hire analogy

Imagine you hire a new software engineer. On their first day, you would not expect them to know everything. But you would expect them to:

1. **Read documentation** to understand the codebase
2. **Use tools** like an IDE, terminal, and browser
3. **Ask questions** when something is unclear
4. **Break down tasks** into smaller steps
5. **Check their work** before saying they are done
6. **Learn from mistakes** and adjust their approach

An AI agent works the same way. It has a base of knowledge (the language model), access to tools, and an orchestration layer that manages the loop of thinking, acting, and observing.

---

## LLM vs. agent: what is the difference?

This is the most important distinction to internalize early.

| Aspect | LLM (alone) | AI Agent |
|---|---|---|
| **What it does** | Generates text based on a prompt | Pursues a goal through multiple steps |
| **Interaction** | Single turn (or multi-turn chat) | Autonomous loop of thought and action |
| **Tools** | None - text in, text out | Can call functions, APIs, search, etc. |
| **Memory** | Limited to context window | Can persist information across steps |
| **Decision-making** | Responds to what you ask | Decides what to do next on its own |
| **Error handling** | Gives you an answer (right or wrong) | Can observe errors and retry with a new approach |

A helpful mental model:

- **LLM** = Brain
- **Agent** = Brain + Hands + Memory

The brain (LLM) does the reasoning. The hands (tools) let it take action. The memory (state management) lets it keep track of what has happened and what still needs to be done.

### A concrete example

**LLM alone:** You ask "What is the current price of GOOG stock?" The model might say "As of my last training data, it was around $140" - which could be months out of date.

**Agent:** You ask the same question. The agent thinks "I need current stock data, I should use a finance API." It calls a stock price tool, gets the live price, and returns an accurate answer. If the API call fails, it might try a different data source.

That loop - think, act, observe, repeat - is what makes an agent an agent.

---

## Core components of an AI agent

Every agent system, regardless of framework, has three fundamental components:

### 1. the model (the brain)

This is the language model at the center of the agent. It handles:

- **Understanding** the user's goal
- **Reasoning** about what steps to take
- **Deciding** which tool to use (and with what parameters)
- **Interpreting** the results of tool calls
- **Generating** the final response

The model you choose matters. Harder tasks (multi-step reasoning, complex code generation, nuanced decision-making) benefit from frontier models like Gemini or Opus. Simpler tasks (classification, extraction, straightforward Q&A) can use lighter models like Gemini Flash to save cost and latency.

### 2. tools (the hands)

Tools are what let an agent interact with the world beyond text generation. Without tools, an agent is just a chatbot. With tools, it can:

- **Retrieve information**: Search the web, query a database, read a file
- **Take actions**: Send an email, create a ticket, deploy code
- **Compute**: Run calculations, execute code, transform data

Tools are typically defined as functions with clear names, descriptions, and parameter schemas. The model decides when and how to call them. We will cover tools in depth in Lesson 3.

### 3. the orchestration layer (the control loop)

This is the glue that connects the model and tools into a functioning system. The orchestration layer manages:

- **The agent loop**: Think -> Act -> Observe -> Repeat
- **State management**: What has happened so far, what context the model needs
- **Error handling**: What to do when a tool call fails
- **Termination conditions**: When to stop looping and return a result
- **Guardrails**: Safety checks, output validation, scope limits

The simplest orchestration pattern looks like this:

```
1. Receive user goal
2. Send goal + available tools to the model
3. Model returns either:
   a. A final answer -> Return to user
   b. A tool call -> Execute the tool, add result to context, go to step 2
```

This is often called a **ReAct loop** (Reasoning + Acting). More sophisticated patterns exist - we will explore them in later lessons.

### How the components work together

```
User Goal
    |
    v
+-------------------+
| Orchestration     |
| Layer             |
|                   |
|  +-------------+  |
|  |   Model     |  |    "I need to search for X"
|  |  (Brain)    |--+--->  Tool Call
|  +-------------+  |         |
|        ^          |         v
|        |          |  +-------------+
|        +----------+--+   Tools     |
|     Tool results  |  |  (Hands)   |
|                   |  +-------------+
+-------------------+
    |
    v
Final Response
```

---

## A taxonomy of agent systems

Not all agents are created equal. It is helpful to think about agent systems on a spectrum of autonomy and capability, from Level 0 through Level 4.

### Level 0: basic reasoning (simple LLM

**What it is:** A language model answering questions with no tools or memory.

**Example:** You ask Gemini "Explain the CAP theorem" and it gives you a clear explanation from its training data.

**Capabilities:**
- Text generation and comprehension
- Single-turn or multi-turn conversation
- No external data access
- No ability to take actions

**When it works well:** General knowledge questions, creative writing, brainstorming, summarization of provided text.

### Level 1: connected problem-solver (tool-using agent)

**What it is:** A model that can call tools to retrieve information or perform simple actions. This is where we cross the line from "chatbot" to "agent."

**Example:** A customer support bot that can look up order status by calling your order API, or a coding assistant that can search documentation.

**Capabilities:**
- Everything in Level 0
- Function calling (tools)
- Retrieval-Augmented Generation (RAG) for grounding in real data
- Simple single-step or few-step task completion

**When it works well:** Tasks that require current data, API integrations, straightforward workflows with a small number of steps.

### Level 2: strategic agent (autonomous with context)

**What it is:** An agent that can plan multi-step approaches, maintain context across a longer session, and adapt its strategy based on intermediate results.

**Example:** A research agent that takes a question like "Compare the top 3 cloud providers on serverless pricing," then searches for pricing pages, extracts data, builds a comparison table, and summarizes findings.

**Capabilities:**
- Everything in Level 1
- Multi-step planning and execution
- Dynamic replanning when things change
- Working memory across steps
- Self-evaluation ("Is this result good enough?")

**When it works well:** Research tasks, complex troubleshooting, multi-step workflows where the path depends on intermediate results.

### Level 3: collaborative multi-agent system

**What it is:** Multiple specialized agents working together, each handling a different aspect of a larger task. One agent might coordinate the others.

**Example:** A software development system where one agent writes code, another writes tests, a third reviews the code, and an orchestrator agent manages the workflow.

**Capabilities:**
- Everything in Level 2
- Agent-to-agent communication
- Specialized roles and delegation
- Parallel execution of subtasks
- Consensus or voting mechanisms for quality

**When it works well:** Complex projects that benefit from specialization, tasks requiring multiple perspectives or quality gates.

### Level 4: self-evolving agent

**What it is:** An agent that can reflect on its own performance, learn from past runs, update its strategies, and improve over time without manual intervention.

**Example:** A deployment agent that tracks which rollback strategies worked best historically and adjusts its approach for future deployments.

**Capabilities:**
- Everything in Level 3
- Long-term memory and learning
- Strategy optimization based on past outcomes
- Self-modification of prompts or tool selection
- Performance monitoring and self-correction

**When it works well:** Recurring tasks where patterns emerge over time, systems that benefit from continuous improvement.

### Summary table

| Level | Name | Key Feature | Example |
|---|---|---|---|
| 0 | Basic Reasoning | Text in, text out | Chatbot, Q&A |
| 1 | Connected Problem-Solver | Tool use | Order lookup bot |
| 2 | Strategic Agent | Multi-step planning | Research assistant |
| 3 | Collaborative Multi-Agent | Agent coordination | Dev team simulation |
| 4 | Self-Evolving | Learning from experience | Adaptive ops agent |

Most production agent systems today operate at Level 1 or Level 2. Levels 3 and 4 are active areas of research and are becoming more practical, but they add significant complexity. Start simple and move up only when you have a clear reason to.

<div id="agent-levels-explorer" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 900px; margin: 32px auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 2px 16px rgba(0,0,0,0.08); padding: 32px; box-sizing: border-box;">
  <h3 style="margin: 0 0 4px 0; font-size: 20px; color: #1a1a2e;">Interactive Agent Levels Explorer</h3>
  <p style="margin: 0 0 20px 0; font-size: 14px; color: #6b7280;">Click on a level to explore its capabilities, architecture, and examples.</p>

  <div id="ale-tabs" style="display: flex; gap: 8px; flex-wrap: wrap; margin-bottom: 24px;"></div>

  <div id="ale-detail" style="display: flex; gap: 24px; flex-wrap: wrap;">
    <div id="ale-diagram" style="flex: 1; min-width: 280px; background: white; border-radius: 12px; padding: 24px; box-shadow: 0 1px 4px rgba(0,0,0,0.06);"></div>
    <div id="ale-info" style="flex: 1; min-width: 280px;">
      <div id="ale-autonomy" style="background: white; border-radius: 12px; padding: 16px; margin-bottom: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06);">
        <div style="font-size: 13px; font-weight: 600; color: #6b7280; margin-bottom: 8px;">AUTONOMY LEVEL</div>
        <div style="background: #e5e7eb; border-radius: 8px; height: 24px; overflow: hidden; position: relative;">
          <div id="ale-bar" style="height: 100%; border-radius: 8px; transition: width 0.5s ease, background 0.5s ease; width: 10%;"></div>
          <span id="ale-bar-label" style="position: absolute; right: 8px; top: 50%; transform: translateY(-50%); font-size: 12px; font-weight: 600; color: #374151;"></span>
        </div>
      </div>
      <div id="ale-capabilities" style="background: white; border-radius: 12px; padding: 16px; margin-bottom: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06);"></div>
      <div id="ale-examples" style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06);"></div>
    </div>
  </div>

  <script>
  (function() {
    var levels = [
      {
        id: 0, name: "Basic Reasoning", color: "#6b7280", autonomy: 10,
        components: ["User Input", "LLM", "Text Output"],
        connections: [[0,1],[1,2]],
        capabilities: ["Text generation", "Question answering", "Summarization", "Translation"],
        examples: ["Simple chatbot", "Q&A assistant", "Text summarizer"],
        features: ["No tools or external access", "Single request/response", "Stateless interaction"]
      },
      {
        id: 1, name: "Connected Problem-Solver", color: "#4285f4", autonomy: 30,
        components: ["User Input", "LLM", "Tools / APIs", "Response"],
        connections: [[0,1],[1,2],[2,1],[1,3]],
        capabilities: ["API calls", "Database queries", "RAG retrieval", "Code execution"],
        examples: ["Order lookup bot", "RAG assistant", "Weather agent"],
        features: ["Single tool use per turn", "Function calling", "Grounded responses"]
      },
      {
        id: 2, name: "Strategic Agent", color: "#34a853", autonomy: 55,
        components: ["User Input", "Planner", "LLM", "Tools", "Memory", "Response"],
        connections: [[0,1],[1,2],[2,3],[3,2],[2,4],[4,2],[2,5]],
        capabilities: ["Multi-step planning", "Context management", "Error recovery", "Re-planning"],
        examples: ["Research assistant", "DevOps agent", "Data analyst"],
        features: ["Agentic loop (ReAct)", "Maintains context across steps", "Adapts plan on failure"]
      },
      {
        id: 3, name: "Collaborative Multi-Agent", color: "#fbbc04", autonomy: 78,
        components: ["Orchestrator", "Agent A", "Agent B", "Agent C", "Shared Memory", "Output"],
        connections: [[0,1],[0,2],[0,3],[1,4],[2,4],[3,4],[4,0],[0,5]],
        capabilities: ["Agent delegation", "Parallel execution", "Specialized roles", "Team coordination"],
        examples: ["Dev team simulation", "Research team", "Content pipeline"],
        features: ["Multiple specialized agents", "Inter-agent communication", "Parallel task execution"]
      },
      {
        id: 4, name: "Self-Evolving", color: "#ea4335", autonomy: 95,
        components: ["Goal", "Meta-Learner", "Agent Core", "Tools", "Long-term Memory", "Feedback Loop"],
        connections: [[0,1],[1,2],[2,3],[3,2],[2,4],[4,2],[2,5],[5,1]],
        capabilities: ["Self-improvement", "Strategy learning", "Long-term memory", "Goal decomposition"],
        examples: ["Adaptive ops agent", "Self-improving coder", "Autonomous researcher"],
        features: ["Learns from past runs", "Updates own strategies", "Persistent memory across sessions"]
      }
    ];

    var tabsEl = document.getElementById("ale-tabs");
    var diagramEl = document.getElementById("ale-diagram");
    var barEl = document.getElementById("ale-bar");
    var barLabelEl = document.getElementById("ale-bar-label");
    var capEl = document.getElementById("ale-capabilities");
    var exEl = document.getElementById("ale-examples");
    var selected = 0;

    function render(idx) {
      selected = idx;
      var lv = levels[idx];
      // Tabs
      tabsEl.innerHTML = "";
      levels.forEach(function(l, i) {
        var btn = document.createElement("button");
        btn.textContent = "L" + l.id + ": " + l.name;
        btn.style.cssText = "padding:10px 16px;border-radius:8px;border:2px solid " + (i === idx ? l.color : "#e5e7eb") + ";background:" + (i === idx ? l.color : "white") + ";color:" + (i === idx ? "white" : "#374151") + ";font-size:13px;font-weight:600;cursor:pointer;transition:all 0.2s;white-space:nowrap;";
        btn.onmouseover = function() { if (i !== selected) { btn.style.borderColor = l.color; btn.style.background = l.color + "18"; }};
        btn.onmouseout = function() { if (i !== selected) { btn.style.borderColor = "#e5e7eb"; btn.style.background = "white"; }};
        btn.onclick = function() { render(i); };
        tabsEl.appendChild(btn);
      });
      // Autonomy bar
      barEl.style.width = lv.autonomy + "%";
      barEl.style.background = "linear-gradient(90deg, " + lv.color + "88, " + lv.color + ")";
      barLabelEl.textContent = lv.autonomy + "%";
      // Diagram
      var svg = '<svg viewBox="0 0 400 220" style="width:100%;height:auto;">';
      var nc = lv.components.length;
      var positions = [];
      if (nc <= 3) {
        for (var i=0;i<nc;i++) positions.push({x: 60 + i * 140, y: 110});
      } else if (nc <= 4) {
        positions = [{x:200,y:40},{x:60,y:130},{x:200,y:130},{x:340,y:130}];
      } else if (nc <= 6) {
        positions = [{x:200,y:35},{x:70,y:100},{x:200,y:100},{x:330,y:100},{x:130,y:180},{x:270,y:180}];
      }
      // Draw connections
      lv.connections.forEach(function(c) {
        var from = positions[c[0]], to = positions[c[1]];
        if (from && to) {
          var dx = to.x - from.x, dy = to.y - from.y;
          var len = Math.sqrt(dx*dx + dy*dy);
          var ux = dx/len, uy = dy/len;
          var x1 = from.x + ux*32, y1 = from.y + uy*20;
          var x2 = to.x - ux*32, y2 = to.y - uy*20;
          svg += '<line x1="'+x1+'" y1="'+y1+'" x2="'+x2+'" y2="'+y2+'" stroke="'+lv.color+'" stroke-width="2" stroke-opacity="0.5" marker-end="url(#ah'+idx+')"/>';
        }
      });
      svg += '<defs><marker id="ah'+idx+'" markerWidth="8" markerHeight="8" refX="8" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8" fill="'+lv.color+'" opacity="0.5"/></marker></defs>';
      // Draw nodes
      positions.forEach(function(p, i) {
        if (i < nc) {
          svg += '<rect x="'+(p.x-45)+'" y="'+(p.y-22)+'" width="90" height="44" rx="10" fill="white" stroke="'+lv.color+'" stroke-width="2"/>';
          svg += '<text x="'+p.x+'" y="'+(p.y+1)+'" text-anchor="middle" dominant-baseline="middle" font-size="11" font-weight="600" fill="#374151">'+lv.components[i]+'</text>';
        }
      });
      svg += '</svg>';
      diagramEl.innerHTML = '<div style="font-size:13px;font-weight:600;color:#6b7280;margin-bottom:8px;">ARCHITECTURE</div>' + svg;
      // Capabilities
      capEl.innerHTML = '<div style="font-size:13px;font-weight:600;color:#6b7280;margin-bottom:8px;">KEY CAPABILITIES</div>' +
        lv.capabilities.map(function(c){ return '<div style="display:flex;align-items:center;gap:8px;padding:4px 0;font-size:14px;color:#374151;"><span style="width:8px;height:8px;border-radius:50%;background:'+lv.color+';flex-shrink:0;"></span>'+c+'</div>'; }).join("") +
        '<div style="margin-top:12px;font-size:13px;font-weight:600;color:#6b7280;margin-bottom:6px;">KEY FEATURES</div>' +
        lv.features.map(function(f){ return '<div style="font-size:13px;color:#6b7280;padding:2px 0;">- '+f+'</div>'; }).join("");
      // Examples
      exEl.innerHTML = '<div style="font-size:13px;font-weight:600;color:#6b7280;margin-bottom:8px;">EXAMPLES</div>' +
        lv.examples.map(function(e){ return '<span style="display:inline-block;background:'+lv.color+'18;color:'+lv.color+';border-radius:6px;padding:4px 10px;font-size:13px;font-weight:500;margin:2px 4px 2px 0;">'+e+'</span>'; }).join("");
    }
    render(0);
  })();
  </script>
</div>

---

## When to use agents vs. when a simple prompt is enough

Agents add power but also complexity, cost, and latency. Not every problem needs an agent. Here is a practical guide.

### Use a simple prompt when:

- The task can be completed in a single step
- No external data or actions are needed
- The answer exists within the model's training data
- Low latency is critical (agents add multiple round trips)
- The cost of multiple model calls is not justified

**Examples:**
- "Summarize this paragraph"
- "Convert this JSON to a Python dataclass"
- "Write a regex that matches email addresses"
- "Explain the difference between TCP and UDP"

### Use an agent when:

- The task requires multiple steps that depend on each other
- External data or tools are needed (APIs, databases, search)
- The task requires real-time or current information
- The approach may need to change based on intermediate results
- The task involves taking actions (not just generating text)

**Examples:**
- "Find the three most recent bugs in our issue tracker and draft a summary for the team standup"
- "Look up the customer's order, check the shipping status, and send them an update email"
- "Research competitors' pricing and build a comparison spreadsheet"
- "Review this pull request, run the tests, and suggest improvements"

### The decision flowchart

```
Can the model answer from the prompt and its own knowledge?
  |
  +-- Yes --> Use a simple prompt
  |
  +-- No  --> Does it only need to look up information (docs, records)?
               |
               +-- Yes --> Try RAG (retrieval) first
               +-- No  --> Does it need more than one tool call?
                            |
                            +-- No  --> A simple function call is enough
                            +-- Yes --> Do you know the steps in advance?
                                         |
                                         +-- Yes --> A fixed workflow (steps chained in code)
                                         +-- No  --> Use an agent
```

The last question is the one people most often skip. "Look up the order, check the shipping status, and send an update email" always runs the same three steps in the same order, so you can chain them in code. That is cheaper, faster, and more predictable than an agent. Reach for an agent when the next step depends on what the last one returned - for example, investigating a production incident, where each finding decides where to look next. Lesson 4 and Lesson 18 cover this spectrum from fixed workflows to dynamic agents in more depth.

### Cost and latency considerations

Every step in an agent loop involves a model call. A 5-step agent workflow means 5 or more calls to the model, plus tool execution time. This adds up:

- **Latency**: Each model call takes 1-10 seconds depending on the model and prompt size. A 5-step agent might take 15-30 seconds.
- **Cost**: Each model call costs tokens. Agent workflows can use 10-50x more tokens than a single prompt.
- **Reliability**: More steps means more chances for errors or hallucinations.

The engineering principle is the same as anywhere else: use the simplest approach that gets the job done.

---

## Real-World Examples

### Customer support agent

**Goal:** Handle customer inquiries end-to-end.

**How it works:**
1. Customer writes: "Where is my order #12345?"
2. Agent calls the order lookup tool with the order ID
3. Gets status: "Shipped, tracking number XYZ, estimated delivery March 20"
4. Agent formats a friendly response with the tracking link
5. If the customer asks to change the delivery address, the agent calls the address update tool

**Level:** 1-2 (tool use with some multi-step logic)

### Code assistant agent

**Goal:** Help developers write, debug, and improve code.

**How it works:**
1. Developer asks: "Why is this function returning null?"
2. Agent reads the relevant source files
3. Searches for related tests
4. Identifies the bug (missing null check on line 42)
5. Suggests a fix with code
6. Optionally runs the tests to verify the fix works

**Level:** 2 (multi-step reasoning with tool use)

### Research agent

**Goal:** Gather and synthesize information from multiple sources.

**How it works:**
1. User asks: "What are the pros and cons of server-side rendering in 2026?"
2. Agent searches for recent articles and benchmarks
3. Reads and extracts key points from multiple sources
4. Cross-references claims and checks for consistency
5. Produces a structured summary with citations

**Level:** 2 (search, read, synthesize across multiple steps)

### DevOps incident response agent

**Goal:** Help diagnose and resolve production incidents.

**How it works:**
1. Alert fires: "API latency spike on service-auth"
2. Agent queries monitoring dashboards for the last 30 minutes
3. Checks recent deployments for changes
4. Examines logs for error patterns
5. Correlates findings: "Latency spike started 5 minutes after deploy #789 which changed the auth token cache TTL"
6. Suggests rollback and drafts an incident report

**Level:** 2-3 (multi-step investigation, potentially coordinating with other agents)

---

## ELI5: what is an AI agent?

### Think of an agent like a really capable intern

Imagine you have a brand new intern on their first day. They are smart - they graduated top of their class - but they have never seen your codebase before.

**An LLM by itself is like this intern sitting in a room with no computer.** You can ask them questions and they will give you thoughtful answers based on what they learned in school. But they cannot look anything up, they cannot run any code, and they cannot send any emails. All they can do is talk.

**An agent is like this intern with a full desk setup.** They have a laptop, access to your internal tools, a browser, and your company Slack. Now when you ask them a question, they can:

- Look things up if they do not know the answer
- Try running code to test their ideas
- Check the documentation to make sure they are right
- Ask a colleague (another agent) for help
- Come back to you with a verified answer

The intern still makes mistakes sometimes - they are new, after all. But they can catch most of their errors because they can check their work. And if they get stuck, they know to ask for help rather than guessing.

**The key insight:** The intern's brain did not change. What changed was what they have access to and how they approach the work. That is exactly the difference between an LLM and an agent. Same brain, more capabilities, better process.

---

## How Google Cloud fits in

Google Cloud provides infrastructure for building and deploying agents through several services:

- **Vertex AI Agent Engine** (rebranded in 2026 as **Agent Runtime**, part of the Gemini Enterprise Agent Platform) - A managed platform for building, deploying, and managing AI agents in production. It handles orchestration, tool management, session state, and scaling so you can focus on agent logic rather than infrastructure.

- **Gemini Models** - The language models that serve as the "brain" of your agents, available in different sizes for different use cases.

- **Agent Development Kit (ADK)** - An open-source, code-first toolkit for building agents with features like multi-agent orchestration, built-in tool support, and easy deployment to Agent Engine.

We will use these tools throughout the course. For now, just know they exist.

> **Learn more:** [Vertex AI Agent Engine Overview](https://docs.cloud.google.com/agent-builder/agent-engine/overview)

---

## Key takeaways

1. **An AI agent is a system that uses a language model to reason, tools to act, and an orchestration layer to manage the loop between thinking and doing.**

2. **LLM = brain. Agent = brain + hands + memory.** The model provides reasoning. Tools provide action. The orchestration layer provides control flow.

3. **Agents exist on a spectrum** from simple tool-using assistants (Level 1) to self-evolving systems (Level 4). Start at the lowest level that solves your problem.

4. **Not everything needs an agent.** If a single prompt gets the job done, use a single prompt. Add agent capabilities only when the task genuinely requires tools, multi-step reasoning, or real-world actions.

5. **The core loop is simple:** Receive goal -> Think about what to do -> Use a tool -> Observe the result -> Repeat until done.

---

## What is next?

In the next lesson, we will look under the hood at the "brain" of the agent - the language model. You will learn how LLMs process information, how different reasoning strategies affect agent performance, and how to pick the right model for the job.

[Next: Lesson 2 - How Agents Think: LLMs as the Reasoning Engine -->](../02-how-agents-think/)
