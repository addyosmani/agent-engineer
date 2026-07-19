# Lesson 7: multi-agent systems - when one agent is not enough

## Introduction

Up to this point in the course, we have focused on building single agents - one LLM with tools, memory, and a planning loop. That approach works remarkably well for many tasks. But as the scope of what you want to automate grows, a single agent starts to buckle under the weight. It gets slow. It gets confused. It tries to be an expert at everything and ends up being an expert at nothing.

Multi-agent systems solve this by splitting work across multiple specialized agents that coordinate to achieve a shared goal. Think of it like the difference between a solo freelancer and a well-organized team. The freelancer can do many things, but a team of specialists - each owning their domain - can tackle problems that no single person could handle alone.

In this lesson, you will learn when and why to use multiple agents, the main architectural patterns for organizing them, how agents communicate, and how to handle the coordination challenges that come with distributed systems.

## Why use multiple agents?

### The limits of a single agent

A single agent with access to twenty tools, a massive system prompt, and instructions covering five different domains is like a new hire who was given five different job descriptions on their first day. They might technically be capable of doing each task, but juggling all of them leads to mistakes, slowdowns, and confusion.

Here are the concrete reasons to consider splitting into multiple agents:

| Problem with Single Agent | How Multi-Agent Helps |
|---|---|
| Prompt becomes enormous and hard to maintain | Each agent gets a focused, manageable prompt |
| Too many tools cause selection errors | Each agent only sees the tools relevant to its role |
| One failure crashes the whole workflow | Failures are isolated to one agent; others keep working |
| Hard to scale bottleneck steps | You can scale individual agents independently |
| Difficult to test and debug | Each agent can be tested in isolation |
| Single point of failure for safety | Different agents can have different permission levels |

### ELI5: the hospital analogy

Multi-agent systems are like a hospital. When you walk in with a medical issue, you do not get one doctor who does everything - takes your vitals, runs bloodwork, reads the X-ray, performs surgery, and handles your billing. Instead, you move through a system of specialists: a triage nurse assesses you, a physician diagnoses you, a radiologist reads your scans, a surgeon operates if needed, and an administrator handles insurance. Each person is an expert in their role, and they coordinate through shared records and handoff protocols.

An AI multi-agent system works the same way. Each agent is a specialist with a focused role, its own set of tools, and clear boundaries for what it handles and what it passes along.

### When to stay single vs. go multi

Not every problem needs multiple agents. Here is a simple decision framework:

**Stay with a single agent when:**
- The task is well-defined and narrow (e.g., "summarize this document")
- You need fewer than 5-7 tools
- The workflow is linear and predictable
- Latency requirements are tight (multi-agent adds overhead)

**Consider multi-agent when:**
- The task spans multiple domains (e.g., customer service + payments + compliance)
- Different steps need different permission levels
- You want to develop, test, and deploy parts independently
- The system needs to scale different capabilities at different rates
- You need fault isolation - one part failing should not break everything

## Single agent vs. multi-agent: corner shop vs. department store

Imagine a corner shop run by one person. The owner greets you, finds your product, rings it up, handles returns, and restocks the shelves. For a small shop with a handful of customers, this works great. The owner knows everything and can do everything.

Now imagine that same person trying to run a department store alone. Thousands of customers, dozens of departments, complex inventory, specialized products. It would be chaos. Department stores work because they have specialized staff: someone in electronics, someone in clothing, cashiers, floor managers, a returns desk. Each person has a clear role and they coordinate through defined processes.

| Aspect | Single Agent (Corner Shop) | Multi-Agent (Department Store) |
|---|---|---|
| Complexity handling | Good for simple tasks | Built for complex workflows |
| Specialization | Jack of all trades | Expert in its domain |
| Failure impact | Everything goes down | Only affected agent is impacted |
| Development speed | Fast to prototype | Faster to iterate on individual parts |
| Coordination overhead | None | Requires explicit coordination logic |
| Cost per simple task | Lower | Higher (more LLM calls) |

## Multi-Agent Architectures

There are four primary patterns for organizing multiple agents. Each has different strengths, and you will often combine them in real systems.

### 1. sequential (assembly line)

In a sequential architecture, agents are arranged in a pipeline. Agent A completes its work and hands off to Agent B, who hands off to Agent C, and so on. Each agent transforms or enriches the output before passing it along.

**How it works:**
```
[Agent A] --> [Agent B] --> [Agent C] --> Final Output
 (Research)   (Draft)       (Review)
```

**Analogy:** Think of a car assembly line. One station welds the frame, the next installs the engine, the next paints the body. Each station does one thing well, and the car moves forward through the line.

**When to use it:**
- The task has clear, ordered stages
- Each stage depends on the output of the previous one
- You want simple, predictable flow

**Example:** A content pipeline where a Research Agent gathers sources, a Writing Agent produces a draft, and an Editing Agent reviews for quality and accuracy.

**Strengths:**
- Easy to understand and debug
- Clear data flow
- Simple to add or remove stages

**Weaknesses:**
- Only as fast as the slowest agent
- No parallelism
- If an early agent makes a mistake, it propagates forward

### 2. hierarchical (manager and workers)

In a hierarchical architecture, a supervisor agent (the "manager") receives the overall task, breaks it into subtasks, and delegates each subtask to a specialized worker agent. The manager collects results, checks quality, and may re-delegate if work is not satisfactory.

**How it works:**
```
              [Manager Agent]
             /       |       \
    [Worker A]  [Worker B]  [Worker C]
    (Search)    (Calculate)  (Draft)
```

**Analogy:** This is like a company org chart. The CEO does not write code or file taxes - they set direction and delegate to department heads, who in turn delegate to their teams. The CEO reviews the results and makes final decisions.

**When to use it:**
- The task requires dynamic decomposition (you cannot predict the subtasks in advance)
- Different subtasks need different capabilities
- You need a single point of coordination and quality control

**Example:** A project management agent that receives "Plan the Q3 product launch" and delegates research to one agent, timeline creation to another, and risk assessment to a third.

**Strengths:**
- Flexible - can handle varied tasks
- Central quality control
- Can parallelize subtasks

**Weaknesses:**
- Manager agent is a bottleneck and single point of failure
- Manager needs to be smart enough to decompose tasks well
- More complex to implement than sequential

In the [Google Agent Development Kit (ADK)](https://adk.dev/agents/), hierarchical patterns are well supported. You can define a parent agent that delegates to sub-agents, each with their own tools and instructions.

### 3. collaborative (peer network)

In a collaborative architecture, agents work together as equals. There is no boss. Agents share information, build on each other's work, and converge toward a solution through communication.

**How it works:**
```
[Agent A] <--> [Agent B]
    ^              ^
    |              |
    v              v
[Agent C] <--> [Agent D]
```

**Analogy:** This is like a brainstorming session with a group of colleagues. Everyone contributes ideas, reacts to what others have said, and the group converges on a plan. Nobody is in charge - the best ideas rise to the top through discussion.

**When to use it:**
- The problem benefits from diverse perspectives
- No single agent has all the information needed
- You want creative or exploratory outputs

**Example:** A code review system where a Security Agent, a Performance Agent, and a Readability Agent each review the same code and share their findings, then collaborate to produce a unified review.

**Strengths:**
- Good for complex problems with no clear decomposition
- Diverse perspectives improve quality
- No single point of failure

**Weaknesses:**
- Harder to predict behavior
- Risk of endless loops or circular discussion
- More difficult to debug

### 4. competitive (best answer wins)

In a competitive architecture, multiple agents independently tackle the same problem, and a judge (another agent or a scoring function) selects the best output.

**How it works:**
```
[Agent A] --\
[Agent B] ----> [Judge] --> Best Output
[Agent C] --/
```

**Analogy:** This is like a design competition. Three architecture firms each submit a proposal for a new building. A panel of judges reviews all three and selects the best one. The competition produces better results than any single firm would have produced alone.

**When to use it:**
- Quality matters more than cost
- The problem has multiple valid approaches
- You want to reduce the chance of a bad output

**Example:** Three different coding agents each write a solution to a programming problem. A Judge Agent runs tests, checks code quality, and selects the best implementation.

**Strengths:**
- Higher quality outputs through competition
- Naturally resilient - if one agent fails, others may succeed
- Good for critical tasks where errors are costly

**Weaknesses:**
- Expensive (N times the compute for N agents)
- Requires a good evaluation mechanism
- Wasteful if agents produce similar outputs

### Architecture comparison summary

| Architecture | Flow | Best For | Coordination Complexity |
|---|---|---|---|
| Sequential | Linear pipeline | Ordered multi-stage tasks | Low |
| Hierarchical | Tree (manager + workers) | Dynamic task decomposition | Medium |
| Collaborative | Mesh (peer-to-peer) | Complex problems needing diverse input | High |
| Competitive | Parallel with judge | High-stakes decisions | Medium |

<div id="multi-agent-viz" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 800px; margin: 2rem auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); padding: 24px; box-sizing: border-box;">
  <h3 style="margin: 0 0 4px 0; font-size: 1.3rem; color: #1a1a2e;">Multi-Agent Architecture Visualizer</h3>
  <p style="margin: 0 0 12px 0; font-size: 0.9rem; color: #666;">Click an architecture to see animated message passing between agents.</p>

  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 16px;" id="ma-tabs">
    <button onclick="setMATab('sequential')" id="ma-btn-sequential" class="ma-tab-btn" style="padding: 10px 12px; border-radius: 10px; border: 2px solid #4285f4; background: #4285f4; color: #fff; font-size: 0.82rem; font-weight: 600; cursor: pointer; text-align: left; transition: all 0.2s;">
      <div>Sequential (Pipeline)</div><div style="font-weight: 400; font-size: 0.75rem; opacity: 0.85; margin-top: 2px;">A &rarr; B &rarr; C linear chain</div>
    </button>
    <button onclick="setMATab('hierarchical')" id="ma-btn-hierarchical" class="ma-tab-btn" style="padding: 10px 12px; border-radius: 10px; border: 2px solid #34a853; background: #fff; color: #34a853; font-size: 0.82rem; font-weight: 600; cursor: pointer; text-align: left; transition: all 0.2s;">
      <div>Hierarchical</div><div style="font-weight: 400; font-size: 0.75rem; opacity: 0.7; margin-top: 2px;">Coordinator delegates to workers</div>
    </button>
    <button onclick="setMATab('collaborative')" id="ma-btn-collaborative" class="ma-tab-btn" style="padding: 10px 12px; border-radius: 10px; border: 2px solid #fbbc04; background: #fff; color: #997000; font-size: 0.82rem; font-weight: 600; cursor: pointer; text-align: left; transition: all 0.2s;">
      <div>Collaborative</div><div style="font-weight: 400; font-size: 0.75rem; opacity: 0.7; margin-top: 2px;">Peer-to-peer with shared blackboard</div>
    </button>
    <button onclick="setMATab('competitive')" id="ma-btn-competitive" class="ma-tab-btn" style="padding: 10px 12px; border-radius: 10px; border: 2px solid #ea4335; background: #fff; color: #ea4335; font-size: 0.82rem; font-weight: 600; cursor: pointer; text-align: left; transition: all 0.2s;">
      <div>Competitive</div><div style="font-weight: 400; font-size: 0.75rem; opacity: 0.7; margin-top: 2px;">Multiple solutions, best one wins</div>
    </button>
  </div>

  <div style="background: #fff; border-radius: 12px; padding: 16px; border: 2px solid #e0e0e0;">
    <canvas id="ma-canvas" width="760" height="320" style="width: 100%; border-radius: 8px; display: block;"></canvas>
    <div id="ma-metrics" style="display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 12px; margin-top: 16px;"></div>
    <div id="ma-description" style="margin-top: 12px; font-size: 0.82rem; color: #555; padding: 10px; background: #f8f9fa; border-radius: 8px;"></div>
  </div>
</div>

<script>
(function() {
  var architectures = {
    sequential: {
      color: '#4285f4',
      nodes: [
        { x: 100, y: 160, label: 'Research', color: '#4285f4' },
        { x: 300, y: 160, label: 'Draft', color: '#4285f4' },
        { x: 500, y: 160, label: 'Review', color: '#4285f4' },
        { x: 680, y: 160, label: 'Output', color: '#34a853' }
      ],
      edges: [[0,1],[1,2],[2,3]],
      metrics: { latency: 'High (serial)', cost: 'Low', complexity: 'Low' },
      desc: 'Each agent completes its work before passing to the next. Simple and predictable, but only as fast as the slowest agent.'
    },
    hierarchical: {
      color: '#34a853',
      nodes: [
        { x: 380, y: 55, label: 'Coordinator', color: '#34a853' },
        { x: 140, y: 200, label: 'Search', color: '#4285f4' },
        { x: 380, y: 200, label: 'Analyze', color: '#fbbc04' },
        { x: 620, y: 200, label: 'Draft', color: '#9333ea' },
        { x: 380, y: 290, label: 'Result', color: '#34a853' }
      ],
      edges: [[0,1],[0,2],[0,3],[1,4],[2,4],[3,4]],
      metrics: { latency: 'Medium (parallel workers)', cost: 'Medium', complexity: 'Medium' },
      desc: 'A coordinator breaks down the task and delegates to specialists. Workers can run in parallel. Central quality control.'
    },
    collaborative: {
      color: '#fbbc04',
      nodes: [
        { x: 200, y: 80, label: 'Security', color: '#ea4335' },
        { x: 560, y: 80, label: 'Perf', color: '#4285f4' },
        { x: 200, y: 260, label: 'Style', color: '#9333ea' },
        { x: 560, y: 260, label: 'Logic', color: '#34a853' },
        { x: 380, y: 170, label: 'Blackboard', color: '#fbbc04' }
      ],
      edges: [[0,4],[1,4],[2,4],[3,4],[0,1],[1,3],[3,2],[2,0]],
      metrics: { latency: 'Variable', cost: 'Medium-High', complexity: 'High' },
      desc: 'Agents communicate as peers through a shared blackboard. Good for problems needing diverse perspectives. Harder to predict.'
    },
    competitive: {
      color: '#ea4335',
      nodes: [
        { x: 120, y: 100, label: 'Agent A', color: '#4285f4' },
        { x: 120, y: 200, label: 'Agent B', color: '#34a853' },
        { x: 120, y: 290, label: 'Agent C', color: '#9333ea' },
        { x: 450, y: 195, label: 'Judge', color: '#ea4335' },
        { x: 670, y: 195, label: 'Best', color: '#34a853' }
      ],
      edges: [[0,3],[1,3],[2,3],[3,4]],
      metrics: { latency: 'Low (parallel)', cost: 'High (Nx)', complexity: 'Medium' },
      desc: 'Multiple agents independently solve the same problem. A judge evaluates all solutions and selects the best. High quality but expensive.'
    }
  };

  var currentArch = 'sequential';
  var animFrame = null;
  var particles = [];

  window.setMATab = function(tab) {
    currentArch = tab;
    var btnColors = { sequential: '#4285f4', hierarchical: '#34a853', collaborative: '#fbbc04', competitive: '#ea4335' };
    var textColors = { sequential: '#4285f4', hierarchical: '#34a853', collaborative: '#997000', competitive: '#ea4335' };
    ['sequential','hierarchical','collaborative','competitive'].forEach(function(t) {
      var btn = document.getElementById('ma-btn-' + t);
      btn.style.background = t === tab ? btnColors[t] : '#fff';
      btn.style.color = t === tab ? '#fff' : textColors[t];
    });
    particles = [];
    renderMA();
    startAnimation();
  };

  function renderMA() {
    var arch = architectures[currentArch];
    document.getElementById('ma-metrics').innerHTML =
      '<div style="text-align:center;padding:8px;background:#f0f7ff;border-radius:8px;"><div style="font-size:0.72rem;color:#666;font-weight:600;">Latency</div><div style="font-size:0.85rem;font-weight:700;color:#4285f4;margin-top:2px;">' + arch.metrics.latency + '</div></div>' +
      '<div style="text-align:center;padding:8px;background:#fff8f0;border-radius:8px;"><div style="font-size:0.72rem;color:#666;font-weight:600;">Cost</div><div style="font-size:0.85rem;font-weight:700;color:#fbbc04;margin-top:2px;">' + arch.metrics.cost + '</div></div>' +
      '<div style="text-align:center;padding:8px;background:#f8f0ff;border-radius:8px;"><div style="font-size:0.72rem;color:#666;font-weight:600;">Complexity</div><div style="font-size:0.85rem;font-weight:700;color:#9333ea;margin-top:2px;">' + arch.metrics.complexity + '</div></div>';
    document.getElementById('ma-description').textContent = arch.desc;
  }

  function startAnimation() {
    if (animFrame) cancelAnimationFrame(animFrame);
    var canvas = document.getElementById('ma-canvas');
    if (!canvas) return;
    var ctx = canvas.getContext('2d');
    var arch = architectures[currentArch];
    var tick = 0;

    function spawnParticle() {
      var edgeIdx = Math.floor(Math.random() * arch.edges.length);
      var e = arch.edges[edgeIdx];
      particles.push({ from: e[0], to: e[1], t: 0, speed: 0.008 + Math.random() * 0.008 });
    }

    function animate() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      tick++;
      if (tick % 30 === 0 && particles.length < 12) spawnParticle();

      // Draw edges
      arch.edges.forEach(function(e) {
        var f = arch.nodes[e[0]], t = arch.nodes[e[1]];
        ctx.beginPath(); ctx.moveTo(f.x, f.y); ctx.lineTo(t.x, t.y);
        ctx.strokeStyle = '#ddd'; ctx.lineWidth = 2; ctx.stroke();
        // Arrow head
        var angle = Math.atan2(t.y - f.y, t.x - f.x);
        var headLen = 10;
        var ax = t.x - 30 * Math.cos(angle), ay = t.y - 30 * Math.sin(angle);
        ctx.beginPath();
        ctx.moveTo(ax, ay);
        ctx.lineTo(ax - headLen * Math.cos(angle - 0.4), ay - headLen * Math.sin(angle - 0.4));
        ctx.moveTo(ax, ay);
        ctx.lineTo(ax - headLen * Math.cos(angle + 0.4), ay - headLen * Math.sin(angle + 0.4));
        ctx.strokeStyle = '#bbb'; ctx.lineWidth = 2; ctx.stroke();
      });

      // Draw particles
      particles.forEach(function(p) {
        p.t += p.speed;
        if (p.t > 1) p.t = 0;
        var f = arch.nodes[p.from], t = arch.nodes[p.to];
        var px = f.x + (t.x - f.x) * p.t;
        var py = f.y + (t.y - f.y) * p.t;
        ctx.beginPath(); ctx.arc(px, py, 5, 0, Math.PI * 2);
        ctx.fillStyle = arch.color; ctx.globalAlpha = 0.8; ctx.fill(); ctx.globalAlpha = 1;
        ctx.beginPath(); ctx.arc(px, py, 8, 0, Math.PI * 2);
        ctx.fillStyle = arch.color; ctx.globalAlpha = 0.2; ctx.fill(); ctx.globalAlpha = 1;
      });

      // Draw nodes
      arch.nodes.forEach(function(n) {
        ctx.beginPath(); ctx.arc(n.x, n.y, 26, 0, Math.PI * 2);
        ctx.fillStyle = '#fff'; ctx.fill();
        ctx.strokeStyle = n.color; ctx.lineWidth = 3; ctx.stroke();
        ctx.beginPath(); ctx.arc(n.x, n.y, 22, 0, Math.PI * 2);
        ctx.fillStyle = n.color; ctx.globalAlpha = 0.1; ctx.fill(); ctx.globalAlpha = 1;
        ctx.fillStyle = n.color; ctx.font = 'bold 11px sans-serif'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
        ctx.fillText(n.label, n.x, n.y);
      });

      animFrame = requestAnimationFrame(animate);
    }
    animate();
  }

  renderMA();
  // Start animation when visible
  var observer = new IntersectionObserver(function(entries) {
    entries.forEach(function(entry) {
      if (entry.isIntersecting) startAnimation();
      else if (animFrame) { cancelAnimationFrame(animFrame); animFrame = null; }
    });
  }, { threshold: 0.1 });
  var vizEl = document.getElementById('multi-agent-viz');
  if (vizEl) observer.observe(vizEl);
})();
</script>

## Communication patterns

Agents need to talk to each other. How they communicate shapes the system's behavior, debuggability, and performance. There are three main patterns:

### Direct messaging

Agents send messages directly to each other. Agent A knows about Agent B and sends it a request.

```
Agent A --"summarize this document"--> Agent B
Agent B --"here is the summary"--> Agent A
```

**Pros:** Simple, low latency, easy to trace.
**Cons:** Tight coupling - Agent A must know about Agent B. Adding new agents requires updating existing ones.

### Shared blackboard

All agents read from and write to a shared workspace (the "blackboard"). Agents check the blackboard for new information, do their work, and post results back.

```
[Blackboard / Shared State]
    ^       ^       ^
    |       |       |
Agent A  Agent B  Agent C
```

**Pros:** Loose coupling - agents do not need to know about each other. Easy to add new agents. Good for collaborative architectures.
**Cons:** Potential for conflicts when multiple agents write to the same area. Harder to trace causality (who changed what and why).

### Event-Based

Agents publish events to a message bus. Other agents subscribe to events they care about and react accordingly.

```
Agent A --publishes "order_refund_requested"--> [Event Bus]
[Event Bus] --notifies--> Agent B (Payment)
[Event Bus] --notifies--> Agent C (Compliance)
```

**Pros:** Highly decoupled. Scales well. Familiar to engineers who have worked with microservices.
**Cons:** More infrastructure to manage. Harder to debug. Eventual consistency challenges.

### Which pattern to choose

| Scenario | Recommended Pattern |
|---|---|
| Two agents with a clear request-response flow | Direct Messaging |
| Multiple agents building on shared context | Shared Blackboard |
| Microservice-style system with many agents | Event-Based |
| Simple prototype | Direct Messaging |
| Production system at scale | Event-Based |

## Agent roles

In well-designed multi-agent systems, each agent has a clear role. Here are four common roles you will see across many architectures:

### Planner

The Planner takes a high-level goal and breaks it down into a sequence of steps or subtasks. It decides what needs to happen and in what order.

**Responsibilities:**
- Interpret the user's goal
- Decompose it into subtasks
- Determine dependencies between subtasks
- Create an execution plan

**Example:** Given "Book a team offsite for next month," the Planner might produce: (1) check team calendars, (2) find available venues, (3) compare prices, (4) book the best option, (5) send calendar invites.

### Retriever

The Retriever finds information from external sources - databases, APIs, documents, the web. It knows where data lives and how to get it.

**Responsibilities:**
- Search knowledge bases and document stores
- Query APIs and databases
- Filter and rank results by relevance
- Return structured information to other agents

### Executor

The Executor takes actions in the real world. It calls APIs, writes files, sends emails, or makes database changes. It is the agent that "does things."

**Responsibilities:**
- Execute the steps identified by the Planner
- Call external APIs and tools
- Handle errors and retries
- Report results back

### Evaluator

The Evaluator checks the quality of work done by other agents. It verifies correctness, safety, and completeness.

**Responsibilities:**
- Validate outputs against requirements
- Check for errors, hallucinations, or policy violations
- Score quality and decide if work needs to be redone
- Provide feedback for improvement

## Real example walkthrough: customer refund system

Let us walk through a concrete multi-agent system for handling customer refunds at an e-commerce company. This example uses a hierarchical architecture with four specialized agents.

### The Agents

| Agent | Role | Tools | Permissions |
|---|---|---|---|
| Customer Agent | Planner + interface | Customer lookup, order history | Read customer data |
| Payment Agent | Executor | Refund API, payment gateway | Process refunds up to $500 |
| Compliance Agent | Evaluator | Policy database, fraud detection | Read-only access |
| Resolution Agent | Manager / Orchestrator | None (coordinates others) | Delegates to all agents |

### The Flow

**Scenario:** A customer writes in saying "I never received my order #12345 and I want a refund."

**Step 1: Resolution Agent receives the request**

The Resolution Agent is the entry point. It reads the customer's message and decides which agents need to be involved.

```
Resolution Agent thinks:
"This is a refund request for a missing order. I need to:
1. Verify the customer and order details
2. Check compliance with refund policy
3. Process the refund if approved"
```

**Step 2: Resolution Agent delegates to Customer Agent**

The Resolution Agent asks the Customer Agent to look up the customer and order.

```
Resolution Agent -> Customer Agent:
"Look up order #12345 and provide the order details,
delivery status, and customer history."
```

The Customer Agent queries the order database, finds that order #12345 was marked "shipped" but tracking shows it was never delivered. It returns this information to the Resolution Agent.

**Step 3: Resolution Agent delegates to Compliance Agent**

With the order details in hand, the Resolution Agent asks the Compliance Agent to check if a refund is appropriate.

```
Resolution Agent -> Compliance Agent:
"Order #12345, $89.99, shipped but never delivered.
Customer has 2 prior refund requests in the last year.
Is a refund appropriate per our policy?"
```

The Compliance Agent checks the refund policy, verifies this is not a pattern of fraud, and responds that the refund is approved - the customer is within policy limits and the delivery failure is confirmed by the carrier.

**Step 4: Resolution Agent delegates to Payment Agent**

With compliance approval, the Resolution Agent instructs the Payment Agent to process the refund.

```
Resolution Agent -> Payment Agent:
"Process a refund of $89.99 to the original payment
method for order #12345. Compliance approved."
```

The Payment Agent calls the payment gateway API, processes the refund, and returns a confirmation with the refund transaction ID.

**Step 5: Resolution Agent responds to the customer**

The Resolution Agent compiles the results and generates a customer-facing response confirming the refund.

### Why this design works

- **Separation of concerns:** Each agent handles one domain. The Payment Agent never touches customer data. The Compliance Agent never processes payments.
- **Security:** The Payment Agent has refund permissions, but only up to $500. Larger refunds require human approval. The Customer Agent can read data but cannot modify it.
- **Fault isolation:** If the Payment API is down, the Compliance and Customer agents still work. The system can queue the refund and retry later.
- **Testability:** You can test each agent independently. Does the Compliance Agent correctly reject a refund when the customer has too many recent claims? You can test that without involving payments at all.
- **Auditability:** Every delegation and response is logged. You have a clear trail of who decided what and why.

### Building this with Google ADK

The [Google Agent Development Kit (ADK)](https://adk.dev/agents/) provides built-in support for multi-agent patterns. You can define agents as classes with their own instructions, tools, and sub-agents. The ADK handles message passing between agents and provides tracing for debugging.

For workflow agents that follow predictable patterns (like our sequential compliance check), ADK offers [workflow agents](https://adk.dev/agents/workflow-agents/) with built-in sequential, parallel, and loop constructs.

## Coordination challenges

Multi-agent systems are distributed systems, and distributed systems have failure modes that single-agent systems do not. Here are the most common coordination challenges and how to handle them:

### Deadlocks

**What it is:** Two or more agents are waiting for each other to finish, so nothing progresses.

**Example:** Agent A waits for Agent B's output before proceeding. Agent B waits for Agent A's output before proceeding. Neither can move forward.

**How to prevent it:**
- Design unidirectional data flows where possible
- Add timeouts to all agent-to-agent communication
- Use a central orchestrator to detect and break cycles
- Implement circuit breakers that fail gracefully after a timeout

### Circular delegation

**What it is:** Agent A delegates to Agent B, which delegates back to Agent A, creating an infinite loop.

**Example:** A Planner Agent asks a Research Agent for information. The Research Agent decides it needs more context and asks the Planner Agent to clarify. The Planner Agent, not having new information, asks the Research Agent again. Loop forever.

**How to prevent it:**
- Set a maximum delegation depth (e.g., no more than 3 handoffs for any single task)
- Track delegation history and reject requests that create cycles
- Give agents clear boundaries for what they should handle vs. escalate to a human

### Conflicting actions

**What it is:** Two agents independently take contradictory actions on the same resource.

**Example:** A Pricing Agent sets a product price to $49.99 based on competitive analysis. Simultaneously, a Promotions Agent sets the same product price to $29.99 for a flash sale. The final price depends on which agent wrote last.

**How to prevent it:**
- Use locking mechanisms for shared resources
- Designate a single agent as the owner of each resource
- Implement a conflict resolution agent or policy
- Use event sourcing so all changes are tracked and reversible

### Resource contention

**What it is:** Multiple agents compete for limited resources (API rate limits, token budgets, database connections).

**Example:** Ten agents all try to call the same external API simultaneously, hitting the rate limit and causing failures for all of them.

**How to prevent it:**
- Implement rate limiting and queuing at the system level
- Use a shared resource pool with fair scheduling
- Give critical agents higher priority for shared resources
- Monitor resource usage and set per-agent budgets

### Inconsistent state

**What it is:** Agents have different views of the world because shared state updates have not propagated to everyone.

**Example:** The Customer Agent checks inventory and tells the customer an item is in stock. Meanwhile, the Fulfillment Agent just sold the last unit. The customer gets a confirmation for an item that is no longer available.

**How to prevent it:**
- Use a single source of truth for shared state
- Implement optimistic locking with version numbers
- Design agents to handle stale data gracefully (check before acting)
- Keep the window of inconsistency as small as possible

## The orchestration tax: your attention is the bottleneck

The coordination challenges above are agent-to-agent problems. There is one more constraint that dominates all of them in practice, and it is human: spawning agents is nearly free, but reviewing and integrating their work is not. Verification, merge conflict resolution, and architectural judgment are serial work that runs through one person - you. This gap between agent production capacity and human review capacity is sometimes called the **orchestration tax**.

Amdahl's law applies directly: the speedup from parallelization is capped by the fraction of work that stays serial. If human judgment is 30% of the job, no number of parallel agents gets you past roughly a 3x speedup - adding agents past that point does not increase decision velocity, it just deepens the queue of unreviewed work. And unreviewed output that merges anyway is worse than a queue: it is code nobody understands, a debt that surfaces as a production incident.

Practical consequences:

- **Scale agent count to your review rate, not your spawn capacity.** For most engineers that is a low single-digit number of concurrent agents, not twenty. Running twenty feels productive; shipping reviewed work is productive.
- **Automate the verifiable part.** Tests, linters, and screenshots can confirm the mechanical 80% so that your serial attention goes only to the 20% that needs genuine judgment.
- **Never accept a summary as evidence.** An agent's report that it finished is not proof that it finished - agents systematically over-report success. Require the diff, the test output, the logs. Accepting the summary instead is the "summary substitution" trap.
- **Batch your reviews.** Context-switching between agents costs minutes each time. Reviewing several results in one sitting beats polling each agent all day.

## Design principles for multi-agent systems

Based on the patterns and challenges above, here are the key principles to follow:

### 1. start simple

Begin with a single agent. Add more agents only when you hit clear limitations. Premature decomposition into multiple agents adds complexity without benefit.

### 2. define clear boundaries

Each agent should have a well-defined scope, its own tools, and clear rules for what it handles and what it passes to others. Vague boundaries lead to duplication and conflicts.

### 3. design for failure

Assume any agent can fail at any time. Use timeouts, retries, circuit breakers, and fallbacks. A well-designed multi-agent system degrades gracefully rather than collapsing entirely.

### 4. make communication observable

Log every message between agents. You will need these logs to debug issues. If you cannot trace the full path of a request through your system, you cannot fix it when it breaks.

### 5. limit agent autonomy

Each agent should have the minimum permissions it needs. The Payment Agent should not be able to read customer emails. The Customer Agent should not be able to process refunds. This limits the blast radius when an agent misbehaves.

### 6. use humans as circuit breakers

For high-stakes decisions, include a human-in-the-loop step. Agents are good at handling routine cases. Humans should handle exceptions, edge cases, and decisions with significant consequences.

## Hands-On Exercise

Design a multi-agent system for one of the following scenarios. For your chosen scenario, define:
1. The agents and their roles
2. The architecture pattern (sequential, hierarchical, collaborative, or competitive)
3. The communication pattern (direct, blackboard, or event-based)
4. At least two potential coordination challenges and your mitigation strategies

**Scenario A: Automated Code Review**
A system that reviews pull requests for code quality, security vulnerabilities, performance issues, and style compliance.

**Scenario B: Travel Booking Assistant**
A system that helps users plan and book travel - flights, hotels, car rentals, and activities - while staying within a budget.

**Scenario C: Content Moderation Pipeline**
A system that reviews user-generated content for policy violations, spam, misinformation, and harmful content before publishing.

## Key takeaways

- Multi-agent systems split work across specialized agents, each with focused responsibilities, tools, and permissions.
- There are four main architecture patterns: sequential (pipeline), hierarchical (manager-worker), collaborative (peer network), and competitive (best answer wins). Choose based on your task's structure.
- Communication patterns - direct messaging, shared blackboard, and event-based - determine how tightly coupled your agents are.
- Common agent roles (Planner, Retriever, Executor, Evaluator) provide a starting vocabulary for designing your system.
- Coordination challenges like deadlocks, circular delegation, and conflicting actions are the real engineering problems in multi-agent systems. Design for them from the start.
- Human review capacity is the real ceiling on parallelism. Scale the number of agents to your review rate, and demand evidence (diffs, test output) rather than the agent's own summary.
- Start with a single agent and add complexity only when you need it. The best multi-agent system is the simplest one that solves your problem.

## Further reading

- [Google ADK - Agents Overview](https://adk.dev/agents/) - How to build agents and multi-agent systems with the Agent Development Kit
- [Google ADK - Workflow Agents](https://adk.dev/agents/workflow-agents/) - Built-in sequential, parallel, and loop patterns for multi-agent workflows
- [The Orchestration Tax - Addy Osmani](https://addyosmani.com/blog/orchestration-tax/) - why human attention, not agent count, limits multi-agent throughput

---

**Next lesson:** [Agentic RAG](../08-agentic-rag/)
