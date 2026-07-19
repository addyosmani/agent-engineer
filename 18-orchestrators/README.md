# Lesson 18: orchestrators - managing agent control flow

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

See the [ADK SequentialAgent documentation](https://adk.dev/agents/workflow-agents/sequential-agents/) for implementation details.

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

See the [ADK ParallelAgent documentation](https://adk.dev/agents/workflow-agents/parallel-agents/) for implementation details.

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

See the [ADK LoopAgent documentation](https://adk.dev/agents/workflow-agents/loop-agent/) for implementation details.

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

<div id="orch-playground" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 900px; margin: 2rem auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 2px 12px rgba(0,0,0,0.08); padding: 24px; box-sizing: border-box;">
  <h3 style="margin: 0 0 4px 0; font-size: 1.2rem; color: #1a1a2e;">Orchestration Pattern Playground</h3>
  <p style="margin: 0 0 16px 0; font-size: 0.85rem; color: #666;">Click a pattern to see it animate. Select two patterns to see how they compose.</p>

  <!-- Pattern Tabs -->
  <div id="orch-tabs" style="display: flex; gap: 6px; flex-wrap: wrap; margin-bottom: 16px;"></div>

  <!-- Main Display Area -->
  <div style="display: grid; grid-template-columns: 1fr 260px; gap: 16px; align-items: start;">
    <!-- Canvas -->
    <div style="background: white; border-radius: 12px; border: 1px solid #e0e0e0; overflow: hidden;">
      <canvas id="orch-canvas" width="560" height="340" style="width: 100%; height: auto; display: block;"></canvas>
    </div>
    <!-- Info Panel -->
    <div id="orch-info" style="background: white; border-radius: 12px; border: 1px solid #e0e0e0; padding: 16px;">
      <div id="orch-title" style="font-weight: 700; font-size: 1rem; color: #1a1a2e; margin-bottom: 8px;">Select a Pattern</div>
      <div id="orch-desc" style="font-size: 0.8rem; color: #555; line-height: 1.5; margin-bottom: 12px;">Click any pattern above to see its flow diagram and details.</div>
      <div id="orch-meta" style="font-size: 0.78rem; color: #666;"></div>
    </div>
  </div>

  <!-- Compose Section -->
  <div id="orch-compose-section" style="margin-top: 16px; background: white; border-radius: 10px; border: 1px solid #e0e0e0; padding: 14px;">
    <div style="font-weight: 600; font-size: 0.85rem; color: #333; margin-bottom: 6px;">&#128268; Compose Patterns</div>
    <p style="font-size: 0.78rem; color: #666; margin: 0 0 8px 0;">Select two patterns above to see how they combine. <span id="orch-sel-count" style="color: #4285f4; font-weight: 600;">0 / 2 selected</span></p>
    <div id="orch-compose-result" style="display: none; background: #f0f7ff; border-radius: 8px; padding: 12px; font-size: 0.82rem; color: #333; line-height: 1.5;"></div>
  </div>
</div>

<style>
  @media (max-width: 700px) {
    #orch-playground > div:nth-child(3) {
      grid-template-columns: 1fr !important;
    }
  }
</style>

<script>
(function() {
  const patterns = [
    { id: 'sequential', name: 'Sequential', color: '#4285f4', icon: '&#10132;',
      desc: 'Agents execute one after another in a defined order. Each agent\'s output becomes the next agent\'s input.',
      when: 'Tasks with clear stages that build on each other, like draft-review-edit pipelines.',
      complexity: 1, type: 'Deterministic' },
    { id: 'parallel', name: 'Parallel', color: '#34a853', icon: '&#9881;',
      desc: 'Multiple agents execute simultaneously on the same input. Results are collected and combined.',
      when: 'Independent analysis from multiple perspectives, latency-sensitive tasks.',
      complexity: 2, type: 'Deterministic' },
    { id: 'loop', name: 'Loop', color: '#fbbc04', icon: '&#128260;',
      desc: 'An agent executes repeatedly until a condition is met. Includes generator-critic patterns.',
      when: 'Quality-sensitive tasks where iterative refinement is needed.',
      complexity: 2, type: 'Deterministic' },
    { id: 'router', name: 'Router', color: '#ea4335', icon: '&#128268;',
      desc: 'Input is classified and directed to a specialized agent. Only one agent handles each request.',
      when: 'Multi-domain systems where different inputs need different expertise.',
      complexity: 2, type: 'Hybrid' },
    { id: 'hierarchical', name: 'Hierarchical', color: '#9333ea', icon: '&#128101;',
      desc: 'A coordinator agent breaks goals into subtasks and delegates to specialist sub-agents.',
      when: 'Complex tasks requiring planning, delegation, and result synthesis.',
      complexity: 4, type: 'Dynamic' },
    { id: 'groupchat', name: 'Group Chat', color: '#e91e8c', icon: '&#128172;',
      desc: 'Multiple agents participate in a shared conversation, coordinated by a chat manager.',
      when: 'Consensus building, brainstorming, iterative multi-expert validation.',
      complexity: 5, type: 'Dynamic' }
  ];

  const compositions = {
    'sequential+parallel': { name: 'Pipeline with Parallel Stages', desc: 'A sequential pipeline where individual stages fan out to parallel agents. Example: research stage runs 3 search agents in parallel, then writing stage runs sequentially.' },
    'sequential+loop': { name: 'Pipeline with Refinement', desc: 'Sequential stages where one or more stages include a refinement loop. Example: draft stage, then a loop of edit-and-review until approved, then publish.' },
    'sequential+router': { name: 'Adaptive Pipeline', desc: 'A sequential pipeline that routes to different next-steps based on intermediate results. Example: classify input, then route to the appropriate processing pipeline.' },
    'sequential+hierarchical': { name: 'Orchestrated Pipeline', desc: 'A coordinator plans the pipeline stages dynamically rather than having them hardcoded. Stages still execute sequentially but the plan adapts.' },
    'sequential+groupchat': { name: 'Pipeline with Deliberation', desc: 'Sequential stages where one stage is a group discussion. Example: gather data, then debate findings, then write report.' },
    'parallel+loop': { name: 'Competitive Refinement', desc: 'Multiple agents work in parallel, each refining iteratively. Best result is selected. Example: 3 writers draft simultaneously with self-editing loops.' },
    'parallel+router': { name: 'Classified Fan-out', desc: 'Input is classified, then relevant parallel agents are dispatched based on classification. Not all agents run every time.' },
    'parallel+hierarchical': { name: 'Delegated Parallel Work', desc: 'A coordinator assigns independent subtasks to parallel workers. Example: research lead assigns 5 topics to 5 research agents simultaneously.' },
    'parallel+groupchat': { name: 'Parallel Brainstorm', desc: 'Multiple groups brainstorm in parallel, then results are merged. Useful for exploring a problem space from multiple angles simultaneously.' },
    'loop+router': { name: 'Iterative Routing', desc: 'Each iteration routes to a different specialist based on what needs improvement. Example: code review loop routes to security, perf, or style expert each round.' },
    'loop+hierarchical': { name: 'Managed Refinement', desc: 'A coordinator oversees the refinement loop, deciding when to delegate to specialists and when quality is sufficient.' },
    'loop+groupchat': { name: 'Debate Until Consensus', desc: 'Agents discuss in rounds until they reach agreement. Each round builds on the previous discussion. Has a maximum round limit.' },
    'router+hierarchical': { name: 'Hierarchical Dispatch', desc: 'Input is routed to a coordinator who then delegates to its own team of specialists. Multi-level routing.' },
    'router+groupchat': { name: 'Routed Collaboration', desc: 'Input type determines which group of agents collaborates. Different teams handle different request types.' },
    'hierarchical+groupchat': { name: 'Managed Discussion', desc: 'A coordinator sets the agenda and manages a group discussion, synthesizing results and directing conversation flow.' }
  };

  let selected = [];
  let activePattern = null;
  let animFrame = null;
  let animTime = 0;

  // Render tabs
  const tabsEl = document.getElementById('orch-tabs');
  patterns.forEach(p => {
    const tab = document.createElement('button');
    tab.id = 'orch-tab-' + p.id;
    tab.innerHTML = `${p.icon} ${p.name}`;
    tab.style.cssText = `padding: 8px 14px; border: 2px solid #e0e0e0; border-radius: 8px; background: white; cursor: pointer; font-size: 0.82rem; font-weight: 600; color: #555; transition: all 0.2s; white-space: nowrap;`;
    tab.addEventListener('mouseenter', () => { if (!selected.includes(p.id) && activePattern !== p.id) tab.style.borderColor = p.color; });
    tab.addEventListener('mouseleave', () => { if (!selected.includes(p.id) && activePattern !== p.id) tab.style.borderColor = '#e0e0e0'; });
    tab.addEventListener('click', () => handleTabClick(p));
    tabsEl.appendChild(tab);
  });

  function handleTabClick(p) {
    activePattern = p.id;
    // Toggle selection for compose
    const idx = selected.indexOf(p.id);
    if (idx >= 0) { selected.splice(idx, 1); }
    else { if (selected.length >= 2) selected.shift(); selected.push(p.id); }
    // Update tab styles
    patterns.forEach(pp => {
      const t = document.getElementById('orch-tab-' + pp.id);
      if (selected.includes(pp.id)) {
        t.style.borderColor = pp.color; t.style.background = pp.color + '15'; t.style.color = pp.color;
      } else {
        t.style.borderColor = '#e0e0e0'; t.style.background = 'white'; t.style.color = '#555';
      }
    });
    // Update info
    document.getElementById('orch-title').textContent = p.name + ' Pattern';
    document.getElementById('orch-desc').textContent = p.desc;
    const stars = '&#9733;'.repeat(p.complexity) + '&#9734;'.repeat(5 - p.complexity);
    const typeColor = p.type === 'Deterministic' ? '#34a853' : p.type === 'Dynamic' ? '#ea4335' : '#fbbc04';
    document.getElementById('orch-meta').innerHTML = `
      <div style="margin-bottom:8px;"><strong>When to use:</strong> ${p.when}</div>
      <div style="display:flex;gap:12px;flex-wrap:wrap;">
        <div><span style="font-size:0.7rem;color:#999;">Complexity</span><br><span style="color:#fbbc04;font-size:0.9rem;">${stars}</span></div>
        <div><span style="font-size:0.7rem;color:#999;">Type</span><br><span style="padding:2px 8px;border-radius:10px;font-size:0.7rem;font-weight:600;color:white;background:${typeColor};">${p.type}</span></div>
      </div>`;
    // Update compose
    document.getElementById('orch-sel-count').textContent = selected.length + ' / 2 selected';
    const compResult = document.getElementById('orch-compose-result');
    if (selected.length === 2) {
      const key = [selected[0], selected[1]].sort().join('+');
      const comp = compositions[key];
      if (comp) {
        compResult.style.display = 'block';
        compResult.innerHTML = `<strong>${comp.name}</strong><br>${comp.desc}`;
      }
    } else {
      compResult.style.display = 'none';
    }
    // Animate
    startAnimation(p.id);
  }

  const canvas = document.getElementById('orch-canvas');
  const ctx = canvas.getContext('2d');
  const W = 560, H = 340;

  function startAnimation(patternId) {
    if (animFrame) cancelAnimationFrame(animFrame);
    animTime = 0;
    function loop() {
      animTime += 0.02;
      ctx.clearRect(0, 0, W, H);
      drawPattern(patternId, animTime);
      animFrame = requestAnimationFrame(loop);
    }
    loop();
  }

  function drawNode(x, y, label, color, pulse) {
    const r = 24;
    ctx.save();
    if (pulse) {
      const s = 1 + Math.sin(animTime * 4) * 0.08;
      ctx.translate(x, y); ctx.scale(s, s); ctx.translate(-x, -y);
    }
    ctx.beginPath(); ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.fillStyle = color + '22'; ctx.fill();
    ctx.strokeStyle = color; ctx.lineWidth = 2.5; ctx.stroke();
    ctx.fillStyle = color; ctx.font = 'bold 10px sans-serif'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillText(label, x, y);
    ctx.restore();
  }

  function drawArrow(x1, y1, x2, y2, color) {
    const angle = Math.atan2(y2 - y1, x2 - x1);
    const r = 24;
    const sx = x1 + Math.cos(angle) * r, sy = y1 + Math.sin(angle) * r;
    const ex = x2 - Math.cos(angle) * r, ey = y2 - Math.sin(angle) * r;
    ctx.beginPath(); ctx.moveTo(sx, sy); ctx.lineTo(ex, ey);
    ctx.strokeStyle = color + '88'; ctx.lineWidth = 2; ctx.stroke();
    // arrowhead
    ctx.beginPath(); ctx.moveTo(ex, ey);
    ctx.lineTo(ex - 8 * Math.cos(angle - 0.4), ey - 8 * Math.sin(angle - 0.4));
    ctx.lineTo(ex - 8 * Math.cos(angle + 0.4), ey - 8 * Math.sin(angle + 0.4));
    ctx.closePath(); ctx.fillStyle = color + '88'; ctx.fill();
  }

  function drawDot(x1, y1, x2, y2, t, color) {
    const angle = Math.atan2(y2 - y1, x2 - x1);
    const r = 24;
    const sx = x1 + Math.cos(angle) * r, sy = y1 + Math.sin(angle) * r;
    const ex = x2 - Math.cos(angle) * r, ey = y2 - Math.sin(angle) * r;
    const p = (t % 1);
    const dx = sx + (ex - sx) * p, dy = sy + (ey - sy) * p;
    ctx.beginPath(); ctx.arc(dx, dy, 5, 0, Math.PI * 2);
    ctx.fillStyle = color; ctx.fill();
  }

  function drawLabel(x, y, text) {
    ctx.fillStyle = '#999'; ctx.font = '11px sans-serif'; ctx.textAlign = 'center';
    ctx.fillText(text, x, y);
  }

  function drawPattern(id, t) {
    const c = patterns.find(p => p.id === id).color;
    switch(id) {
      case 'sequential': {
        const nodes = [{x:80,y:170,l:'Agent A'},{x:230,y:170,l:'Agent B'},{x:380,y:170,l:'Agent C'},{x:500,y:170,l:'Output'}];
        drawLabel(80, 130, 'Step 1'); drawLabel(230, 130, 'Step 2'); drawLabel(380, 130, 'Step 3');
        for(let i=0;i<nodes.length-1;i++){drawArrow(nodes[i].x,nodes[i].y,nodes[i+1].x,nodes[i+1].y,c);}
        for(let i=0;i<nodes.length-1;i++){drawDot(nodes[i].x,nodes[i].y,nodes[i+1].x,nodes[i+1].y,t*0.6+i*0.33,c);}
        nodes.forEach((n,i)=>drawNode(n.x,n.y,n.l,i===3?'#666':c,i===Math.floor(t*0.6)%3));
        drawLabel(280, 260, 'Output of each agent feeds into the next');
        break;
      }
      case 'parallel': {
        const input={x:80,y:170}, agents=[{x:280,y:70,l:'Agent A'},{x:280,y:170,l:'Agent B'},{x:280,y:270,l:'Agent C'}], out={x:480,y:170};
        agents.forEach(a=>{drawArrow(input.x,input.y,a.x,a.y,c);drawArrow(a.x,a.y,out.x,out.y,c);});
        agents.forEach(a=>{drawDot(input.x,input.y,a.x,a.y,t*0.7,c);drawDot(a.x,a.y,out.x,out.y,t*0.7+0.5,c);});
        drawNode(input.x,input.y,'Input','#666',false);
        agents.forEach(a=>drawNode(a.x,a.y,a.l,c,true));
        drawNode(out.x,out.y,'Combine','#666',false);
        drawLabel(280, 320, 'Fan out, then gather results');
        break;
      }
      case 'loop': {
        const gen={x:180,y:170,l:'Generator'}, crit={x:380,y:170,l:'Critic'};
        drawArrow(gen.x,gen.y,crit.x,crit.y,c);
        // loop back arrow (curved)
        ctx.beginPath();ctx.moveTo(crit.x,crit.y-26);ctx.quadraticCurveTo(280,60,gen.x,gen.y-26);
        ctx.strokeStyle=c+'88';ctx.lineWidth=2;ctx.stroke();
        ctx.beginPath();ctx.moveTo(gen.x,gen.y-26);ctx.lineTo(gen.x+6,gen.y-34);ctx.lineTo(gen.x-6,gen.y-34);ctx.closePath();ctx.fillStyle=c+'88';ctx.fill();
        drawDot(gen.x,gen.y,crit.x,crit.y,t*0.5,c);
        // iteration counter
        const iter = Math.floor(t * 0.8) % 5 + 1;
        ctx.fillStyle=c;ctx.font='bold 13px sans-serif';ctx.textAlign='center';
        ctx.fillText('Iteration: ' + iter + ' / 5', 280, 100);
        drawNode(gen.x,gen.y,gen.l,c,true);drawNode(crit.x,crit.y,crit.l,c,false);
        // output arrow
        drawArrow(crit.x,crit.y,crit.x+100,crit.y,'#666');
        drawNode(crit.x+100,crit.y,'Output','#666',iter>=5);
        drawLabel(280, 280, iter >= 5 ? 'Quality approved - output ready' : 'Not good enough - refining...');
        break;
      }
      case 'router': {
        const router={x:160,y:170,l:'Router'};
        const agents=[{x:380,y:70,l:'Billing'},{x:380,y:170,l:'Tech Support'},{x:380,y:270,l:'General'}];
        const activeIdx = Math.floor(t*0.3) % 3;
        drawArrow(50,170,router.x,router.y,'#666');
        agents.forEach((a,i)=>{drawArrow(router.x,router.y,a.x,a.y,i===activeIdx?c:'#e0e0e0');});
        drawDot(50,170,router.x,router.y,t*0.5,'#666');
        drawDot(router.x,router.y,agents[activeIdx].x,agents[activeIdx].y,t*0.5+0.3,c);
        drawNode(50,170,'Input','#666',false);
        drawNode(router.x,router.y,router.l,c,true);
        agents.forEach((a,i)=>drawNode(a.x,a.y,a.l,i===activeIdx?c:'#ccc',i===activeIdx));
        drawLabel(280, 320, 'Routes to: ' + agents[activeIdx].l);
        break;
      }
      case 'hierarchical': {
        const coord={x:280,y:80,l:'Coordinator'};
        const workers=[{x:120,y:220,l:'Research'},{x:280,y:220,l:'Analysis'},{x:440,y:220,l:'Writing'}];
        const activeW = Math.floor(t*0.4) % 3;
        workers.forEach((w,i)=>{drawArrow(coord.x,coord.y,w.x,w.y,i===activeW?c:'#ddd');drawArrow(w.x,w.y,coord.x,coord.y,i===activeW?c+'66':'#ddd');});
        if(activeW>=0) drawDot(coord.x,coord.y,workers[activeW].x,workers[activeW].y,t*0.6,c);
        drawNode(coord.x,coord.y,coord.l,c,true);
        workers.forEach((w,i)=>drawNode(w.x,w.y,w.l,i===activeW?c:'#bbb',i===activeW));
        drawLabel(280, 310, 'Coordinator delegates to: ' + workers[activeW].l);
        break;
      }
      case 'groupchat': {
        const mgr={x:280,y:80,l:'Manager'};
        const agents=[{x:130,y:200,l:'Agent A'},{x:280,y:260,l:'Agent B'},{x:430,y:200,l:'Agent C'}];
        const speaker = Math.floor(t*0.4) % 3;
        // lines between all
        for(let i=0;i<agents.length;i++)for(let j=i+1;j<agents.length;j++){
          ctx.beginPath();ctx.moveTo(agents[i].x,agents[i].y);ctx.lineTo(agents[j].x,agents[j].y);
          ctx.strokeStyle='#e0e0e0';ctx.lineWidth=1;ctx.stroke();
        }
        agents.forEach(a=>{drawArrow(mgr.x,mgr.y,a.x,a.y,'#ddd');});
        // speech bubble
        const sp = agents[speaker];
        ctx.beginPath();ctx.roundRect(sp.x-40,sp.y-60,80,26,6);ctx.fillStyle=c+'22';ctx.fill();ctx.strokeStyle=c;ctx.lineWidth=1;ctx.stroke();
        ctx.fillStyle=c;ctx.font='10px sans-serif';ctx.textAlign='center';ctx.fillText('Speaking...',sp.x,sp.y-43);
        drawNode(mgr.x,mgr.y,mgr.l,'#666',false);
        agents.forEach((a,i)=>drawNode(a.x,a.y,a.l,c,i===speaker));
        const turns = ['A: "I think we should..."', 'B: "Building on that..."', 'C: "I disagree because..."'];
        drawLabel(280, 320, turns[speaker]);
        break;
      }
    }
  }

  // Initial state
  ctx.fillStyle = '#bbb'; ctx.font = '14px sans-serif'; ctx.textAlign = 'center';
  ctx.fillText('Select a pattern to see its animation', W/2, H/2);
})();
</script>

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
- [ADK Workflow Agents](https://adk.dev/agents/workflow-agents/)
- [ADK Multi-Agent Systems](https://adk.dev/agents/)
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

## Long-running agents: orchestrating across sessions

Everything above assumes the orchestration finishes within one run. But an increasingly common shape of work is the agent that makes forward progress on a goal across many sessions - hours, days, sometimes weeks. Think of a large migration, a recurring triage job, or a feature that takes an agent several context windows to complete. Orchestrating that well means designing around three walls every agent eventually hits:

1. **Finite context windows.** Even million-token windows fill up, and performance degrades from context rot well before the hard limit (see [Lesson 5](../05-memory-and-context/)).
2. **No persistent state.** A new session starts blank. Without deliberate design, every context window boundary is a cliff where the agent forgets everything - like a project staffed by engineers working in shifts, where each new engineer arrives with no memory of the previous shift.
3. **Self-verification failure.** Models over-report their own completion. An agent left alone will happily declare success without external evidence.

The design response to all three is the same: **move the state out of the model and the verification out of the agent.**

- **Externalize state to durable artifacts.** A task list file, a progress log, git commits. The agent stays amnesiac; the filesystem remembers. Each session starts by reading the current state from disk, not by being told it in a prompt.
- **Write explicit done-conditions before starting.** "Improve the test coverage" invites the agent to declare victory early. "All items in `tasks.json` marked done, with the test suite passing" is a stopping condition a fresh session can check.
- **Treat context resets as first-class events.** Do not fight the window - plan for full teardown and reconstruction from the handoff files. If the agent cannot resume from what is on disk, the state files are incomplete.
- **Keep the evaluator outside the generator.** The generator-critic pattern from earlier applies with more force here: verification must come from tests, CI, or a separate agent, never from the agent grading its own work.

The simplest working implementation of all this is a pattern practitioners call the **Ralph loop**: a dumb outer loop around a fresh agent session per task.

```
while tasks remain in tasks.json:
    task = next unfinished task
    prompt = task + relevant context + notes from progress.txt
    run agent session (fresh context)
    run verification (tests, lint)
    append outcome to progress.txt
    mark task done only if verification passed
```

No memory tricks, no giant context - continuity comes entirely from external files and git history. Framework persistence features (LangGraph checkpointing, ADK session state) are the built-in version of the same idea: state that survives the process, so a run can pause, be inspected, and resume.

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

See [ADK Tracing documentation](https://adk.dev/) and [Google Cloud Trace](https://cloud.google.com/trace) for implementation guidance.

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
- For work that outlives one session, externalize state to durable files, define explicit done-conditions, and keep verification outside the agent
- Set iteration limits, validate between steps, manage context aggressively, and design for failure

---

## Further reading

- [ADK Workflow Agents](https://adk.dev/agents/workflow-agents/)
- [Multi-Agent Patterns in ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
- [Anthropic - Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents)
- [Microsoft Azure - AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [CrewAI Documentation](https://docs.crewai.com/)
- [Long-running Agents - Addy Osmani](https://addyosmani.com/blog/long-running-agents/) - the three walls and production patterns for multi-session agents
- [Loop Engineering - Addy Osmani](https://addyosmani.com/blog/loop-engineering/) - designing the loop that prompts the agent so you do not have to

---

[Previous Lesson: Agent Skills](../17-agent-skills/) | [Next Lesson: Where to Go From Here ->](../19-where-to-go-from-here/)
