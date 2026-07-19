# Lesson 13: building your first agent with ADK

## Introduction

You have made it through the fundamentals. You understand what agents are, how they think, how they use tools, how they remember things, and how to evaluate whether they are doing a good job. Now it is time to build one.

In this lesson, we will walk through building a practical agent using Google's Agent Development Kit (ADK). By the end, you will understand how to set up a project, define an agent, give it tools, test it locally, and even build a small team of agents that work together.

We are going to keep this conceptual and link to the official quickstart for exact code samples - that way you always have up-to-date syntax, and we can focus on the ideas that matter.

---

## What we are building

Our goal is a practical agent that can:

- Answer questions about a topic using Google Search
- Call a custom tool you define (like looking up weather or product info)
- Run locally so you can test and iterate quickly

Think of this as a "hello world" for agents - simple enough to understand in one sitting, but real enough to show how everything connects.

<div id="agent-builder-viz" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 900px; margin: 2rem auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); padding: 24px; box-sizing: border-box;">
  <div style="text-align: center; margin-bottom: 20px;">
    <h3 style="margin: 0 0 4px 0; color: #1a1a2e; font-size: 1.3rem;">Interactive Agent Architecture Builder</h3>
    <p style="margin: 0; color: #666; font-size: 0.9rem;">Configure an agent by selecting components and see the architecture update live</p>
  </div>
  <div style="display: flex; gap: 20px; flex-wrap: wrap;">
    <!-- Left: Controls -->
    <div style="flex: 1; min-width: 280px;">
      <!-- Model Selection -->
      <div style="background: white; border-radius: 12px; padding: 16px; margin-bottom: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-weight: 600; color: #1a1a2e; margin-bottom: 10px; font-size: 0.95rem;">🧠 Model Selection</div>
        <div id="ab-models" style="display: flex; flex-direction: column; gap: 6px;"></div>
      </div>
      <!-- Agent Type -->
      <div style="background: white; border-radius: 12px; padding: 16px; margin-bottom: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-weight: 600; color: #1a1a2e; margin-bottom: 10px; font-size: 0.95rem;">🏗️ Agent Type</div>
        <div id="ab-types" style="display: flex; flex-direction: column; gap: 6px;"></div>
      </div>
      <!-- Tools -->
      <div style="background: white; border-radius: 12px; padding: 16px; margin-bottom: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-weight: 600; color: #1a1a2e; margin-bottom: 10px; font-size: 0.95rem;">🔧 Tools</div>
        <div id="ab-tools" style="display: flex; flex-wrap: wrap; gap: 6px;"></div>
      </div>
      <!-- Buttons -->
      <div style="display: flex; gap: 8px;">
        <button id="ab-reset" style="flex:1; padding: 10px; border: 2px solid #ea4335; background: white; color: #ea4335; border-radius: 8px; font-weight: 600; cursor: pointer; font-size: 0.85rem; transition: all 0.2s;">Reset</button>
        <button id="ab-example" style="flex:1; padding: 10px; border: 2px solid #4285f4; background: #4285f4; color: white; border-radius: 8px; font-weight: 600; cursor: pointer; font-size: 0.85rem; transition: all 0.2s;">Example Config</button>
      </div>
    </div>
    <!-- Right: Architecture Diagram + Metrics -->
    <div style="flex: 1; min-width: 280px;">
      <!-- Metrics -->
      <div style="display: flex; gap: 10px; margin-bottom: 12px;">
        <div style="flex:1; background: white; border-radius: 12px; padding: 12px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
          <div style="font-size: 0.75rem; color: #666; margin-bottom: 4px;">Est. Cost/Call</div>
          <div id="ab-cost" style="font-size: 1.3rem; font-weight: 700; color: #34a853;">$0.001</div>
        </div>
        <div style="flex:1; background: white; border-radius: 12px; padding: 12px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
          <div style="font-size: 0.75rem; color: #666; margin-bottom: 4px;">Est. Latency</div>
          <div id="ab-latency" style="font-size: 1.3rem; font-weight: 700; color: #4285f4;">~1s</div>
        </div>
        <div style="flex:1; background: white; border-radius: 12px; padding: 12px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
          <div style="font-size: 0.75rem; color: #666; margin-bottom: 4px;">Capability</div>
          <div id="ab-capability" style="font-size: 1.3rem; font-weight: 700; color: #9333ea;">★★★</div>
        </div>
      </div>
      <!-- Architecture Diagram -->
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04); margin-bottom: 12px;">
        <div style="font-weight: 600; color: #1a1a2e; margin-bottom: 10px; font-size: 0.95rem;">Architecture Diagram</div>
        <svg id="ab-diagram" viewBox="0 0 360 320" style="width: 100%; height: auto;"></svg>
      </div>
      <!-- System Prompt Preview -->
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-weight: 600; color: #1a1a2e; margin-bottom: 8px; font-size: 0.95rem;">Generated System Prompt</div>
        <pre id="ab-prompt" style="background: #1a1a2e; color: #a5d6a7; padding: 12px; border-radius: 8px; font-size: 0.75rem; line-height: 1.5; overflow-x: auto; white-space: pre-wrap; margin: 0; max-height: 150px; overflow-y: auto;"></pre>
      </div>
    </div>
  </div>
</div>

<script>
(function() {
  const models = [
    { id: 'pro', name: 'Gemini 3.1 Pro', cost: 0.007, latency: 3, cap: 5, color: '#9333ea', desc: 'Best reasoning, highest cost' },
    { id: 'flash', name: 'Gemini 3.5 Flash', cost: 0.002, latency: 1.2, cap: 4, color: '#4285f4', desc: 'Fast & capable, great default' },
    { id: 'lite', name: 'Gemini 3.1 Flash-Lite', cost: 0.0005, latency: 0.5, cap: 2, color: '#34a853', desc: 'Cheapest, simple tasks only' }
  ];
  const agentTypes = [
    { id: 'llm', name: 'LlmAgent', desc: 'Flexible reasoning, default choice', icon: '🤖' },
    { id: 'sequential', name: 'SequentialAgent', desc: 'Pipeline of ordered steps', icon: '➡️' },
    { id: 'parallel', name: 'ParallelAgent', desc: 'Concurrent independent tasks', icon: '⚡' },
    { id: 'loop', name: 'LoopAgent', desc: 'Iterative refinement cycle', icon: '🔄' }
  ];
  const tools = [
    { id: 'search', name: 'Google Search', cost: 0.001, icon: '🔍' },
    { id: 'code', name: 'Code Exec', cost: 0.0005, icon: '💻' },
    { id: 'database', name: 'Database', cost: 0.001, icon: '🗄️' },
    { id: 'api', name: 'API Call', cost: 0.0008, icon: '🌐' },
    { id: 'fileio', name: 'File I/O', cost: 0.0003, icon: '📁' }
  ];

  let state = { model: 'flash', type: 'llm', tools: [] };

  function el(id) { return document.getElementById(id); }

  function renderModels() {
    el('ab-models').innerHTML = models.map(m => `
      <label style="display:flex;align-items:center;gap:8px;padding:8px 12px;border-radius:8px;cursor:pointer;border:2px solid ${state.model===m.id?m.color:'#e0e0e0'};background:${state.model===m.id?m.color+'11':'white'};transition:all 0.2s;" onclick="document.dispatchEvent(new CustomEvent('ab-model',{detail:'${m.id}'}))">
        <div style="width:16px;height:16px;border-radius:50%;border:2px solid ${m.color};background:${state.model===m.id?m.color:'white'};transition:all 0.2s;flex-shrink:0;"></div>
        <div style="flex:1;">
          <div style="font-weight:600;font-size:0.85rem;color:#1a1a2e;">${m.name}</div>
          <div style="font-size:0.72rem;color:#888;">${m.desc}</div>
        </div>
        <div style="font-size:0.7rem;color:${m.color};font-weight:600;">$${m.cost.toFixed(4)}</div>
      </label>
    `).join('');
  }

  function renderTypes() {
    el('ab-types').innerHTML = agentTypes.map(t => `
      <label style="display:flex;align-items:center;gap:8px;padding:8px 12px;border-radius:8px;cursor:pointer;border:2px solid ${state.type===t.id?'#4285f4':'#e0e0e0'};background:${state.type===t.id?'#4285f411':'white'};transition:all 0.2s;" onclick="document.dispatchEvent(new CustomEvent('ab-type',{detail:'${t.id}'}))">
        <span style="font-size:1.1rem;">${t.icon}</span>
        <div style="flex:1;">
          <div style="font-weight:600;font-size:0.85rem;color:#1a1a2e;">${t.name}</div>
          <div style="font-size:0.72rem;color:#888;">${t.desc}</div>
        </div>
      </label>
    `).join('');
  }

  function renderTools() {
    el('ab-tools').innerHTML = tools.map(t => {
      const active = state.tools.includes(t.id);
      return `<button style="padding:6px 12px;border-radius:8px;border:2px solid ${active?'#34a853':'#e0e0e0'};background:${active?'#34a85311':'white'};cursor:pointer;font-size:0.8rem;font-weight:${active?'600':'400'};color:${active?'#34a853':'#666'};transition:all 0.2s;display:flex;align-items:center;gap:4px;" onclick="document.dispatchEvent(new CustomEvent('ab-tool',{detail:'${t.id}'}))">${t.icon} ${t.name}</button>`;
    }).join('');
  }

  function updateMetrics() {
    const m = models.find(x => x.id === state.model);
    const toolCost = state.tools.reduce((s, tid) => s + (tools.find(t=>t.id===tid)?.cost||0), 0);
    const totalCost = m.cost + toolCost;
    const latencyMult = state.type === 'parallel' ? 0.6 : state.type === 'sequential' ? 1.3 : state.type === 'loop' ? 2.0 : 1.0;
    const latency = (m.latency * latencyMult + state.tools.length * 0.3).toFixed(1);
    const stars = '★'.repeat(m.cap) + '☆'.repeat(5 - m.cap);
    el('ab-cost').textContent = '$' + totalCost.toFixed(4);
    el('ab-cost').style.color = totalCost > 0.005 ? '#ea4335' : totalCost > 0.002 ? '#fbbc04' : '#34a853';
    el('ab-latency').textContent = '~' + latency + 's';
    el('ab-latency').style.color = parseFloat(latency) > 3 ? '#ea4335' : parseFloat(latency) > 1.5 ? '#fbbc04' : '#4285f4';
    el('ab-capability').textContent = stars;
  }

  function drawDiagram() {
    const svg = el('ab-diagram');
    const m = models.find(x => x.id === state.model);
    const at = agentTypes.find(x => x.id === state.type);
    const selTools = state.tools.map(tid => tools.find(t=>t.id===tid));
    let html = '';

    // Model box at top
    html += `<rect x="110" y="10" width="140" height="44" rx="10" fill="${m.color}22" stroke="${m.color}" stroke-width="2"/>`;
    html += `<text x="180" y="28" text-anchor="middle" font-size="10" font-weight="600" fill="${m.color}">🧠 ${m.name}</text>`;
    html += `<text x="180" y="44" text-anchor="middle" font-size="9" fill="#888">Model</text>`;

    // Arrow down
    html += `<line x1="180" y1="54" x2="180" y2="80" stroke="#ccc" stroke-width="2" marker-end="url(#ab-arrow)"/>`;

    // Agent type box
    html += `<rect x="100" y="80" width="160" height="44" rx="10" fill="#4285f422" stroke="#4285f4" stroke-width="2"/>`;
    html += `<text x="180" y="98" text-anchor="middle" font-size="10" font-weight="600" fill="#4285f4">${at.icon} ${at.name}</text>`;
    html += `<text x="180" y="114" text-anchor="middle" font-size="9" fill="#888">Agent Type</text>`;

    // Instructions box
    html += `<line x1="180" y1="124" x2="180" y2="150" stroke="#ccc" stroke-width="2" marker-end="url(#ab-arrow)"/>`;
    html += `<rect x="80" y="150" width="200" height="34" rx="8" fill="#fbbc0422" stroke="#fbbc04" stroke-width="2"/>`;
    html += `<text x="180" y="172" text-anchor="middle" font-size="10" font-weight="600" fill="#e6a800">📋 System Instructions</text>`;

    // Tools
    if (selTools.length > 0) {
      html += `<line x1="180" y1="184" x2="180" y2="210" stroke="#ccc" stroke-width="2" marker-end="url(#ab-arrow)"/>`;
      const toolWidth = 64;
      const gap = 8;
      const totalW = selTools.length * toolWidth + (selTools.length - 1) * gap;
      const startX = 180 - totalW / 2;
      selTools.forEach((t, i) => {
        const tx = startX + i * (toolWidth + gap);
        html += `<rect x="${tx}" y="215" width="${toolWidth}" height="50" rx="8" fill="#34a85322" stroke="#34a853" stroke-width="1.5"/>`;
        html += `<text x="${tx + toolWidth/2}" y="237" text-anchor="middle" font-size="14">${t.icon}</text>`;
        html += `<text x="${tx + toolWidth/2}" y="254" text-anchor="middle" font-size="8" fill="#34a853" font-weight="600">${t.name}</text>`;
      });
    } else {
      html += `<text x="180" y="230" text-anchor="middle" font-size="11" fill="#ccc" font-style="italic">Select tools below</text>`;
    }

    // Sub-agents for non-LlmAgent
    if (state.type !== 'llm') {
      const labels = state.type === 'sequential' ? ['Step 1', 'Step 2', 'Step 3'] :
                     state.type === 'parallel' ? ['Worker A', 'Worker B', 'Worker C'] :
                     ['Draft', 'Evaluate', 'Revise'];
      html += `<text x="180" y="290" text-anchor="middle" font-size="9" fill="#888">Sub-agents:</text>`;
      const connector = state.type === 'sequential' ? '→' : state.type === 'parallel' ? '|' : '↻';
      labels.forEach((l, i) => {
        const bx = 60 + i * 100;
        html += `<rect x="${bx}" y="296" width="80" height="22" rx="6" fill="#9333ea18" stroke="#9333ea" stroke-width="1"/>`;
        html += `<text x="${bx+40}" y="311" text-anchor="middle" font-size="9" fill="#9333ea" font-weight="500">${l}</text>`;
        if (i < labels.length - 1) {
          html += `<text x="${bx+90}" y="311" text-anchor="middle" font-size="12" fill="#9333ea">${connector}</text>`;
        }
      });
    }

    // Arrow marker
    html = `<defs><marker id="ab-arrow" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="#ccc"/></marker></defs>` + html;
    svg.innerHTML = html;
  }

  function updatePrompt() {
    const m = models.find(x => x.id === state.model);
    const at = agentTypes.find(x => x.id === state.type);
    const selTools = state.tools.map(tid => tools.find(t=>t.id===tid));
    let prompt = `# Agent: my_${state.type}_agent\n# Model: ${m.name}\n\n`;
    prompt += `You are a helpful assistant`;
    if (selTools.length > 0) {
      prompt += ` with access to:\n`;
      selTools.forEach(t => { prompt += `- ${t.name}: Use for ${t.id}-related tasks\n`; });
    } else {
      prompt += `.\n`;
    }
    if (state.type === 'sequential') prompt += `\nProcess tasks in order. Each step builds on the previous.\n`;
    if (state.type === 'parallel') prompt += `\nDelegate independent subtasks to workers concurrently.\n`;
    if (state.type === 'loop') prompt += `\nIteratively refine output until quality threshold is met.\nMax iterations: 5\n`;
    prompt += `\nAlways be concise and accurate.\nAsk for clarification when the request is ambiguous.`;
    el('ab-prompt').textContent = prompt;
  }

  function render() { renderModels(); renderTypes(); renderTools(); updateMetrics(); drawDiagram(); updatePrompt(); }

  document.addEventListener('ab-model', e => { state.model = e.detail; render(); });
  document.addEventListener('ab-type', e => { state.type = e.detail; render(); });
  document.addEventListener('ab-tool', e => {
    const tid = e.detail;
    state.tools = state.tools.includes(tid) ? state.tools.filter(t=>t!==tid) : [...state.tools, tid];
    render();
  });
  el('ab-reset').addEventListener('click', () => { state = { model: 'flash', type: 'llm', tools: [] }; render(); });
  el('ab-example').addEventListener('click', () => { state = { model: 'pro', type: 'sequential', tools: ['search', 'code', 'database'] }; render(); });
  render();
})();
</script>

### Prerequisites

Before you start, make sure you have:

- **Python 3.9+** installed
- **ADK installed** (`pip install google-adk`)
- **A Google Cloud project** with the Gemini API enabled
- **An API key** or application default credentials configured

> **Get set up:** Follow the official setup guide at [ADK Getting Started](https://adk.dev/get-started/) for detailed installation instructions.

---

## ELI5: what are we actually doing?

Building your first agent is like assembling a Lego kit. You have a baseplate (the project structure), a minifigure brain (the language model), some tool accessories (search, custom functions), and an instruction manual (system instructions) that tells the minifigure how to behave.

Each piece snaps together in a specific way. The brain decides what to do. The tools let it do things. The instructions keep it focused. And the baseplate holds everything in place so it does not fall apart.

The beautiful thing is that once you understand how the pieces fit, you can swap them out, add more, or rearrange them to build something completely different.

---

## Step-by-Step Walkthrough

Let us walk through the process of building your first agent. We will cover the concepts at each step and point you to the official quickstart for the exact code.

### Step 1: set up the project structure

ADK expects a specific project layout. At its simplest, you need a folder for your agent with two files inside:

```
my_first_agent/
    __init__.py
    agent.py
```

The folder name becomes your agent's module name. The `__init__.py` file tells Python this is a package and exports your agent. The `agent.py` file is where you define the agent itself.

This structure matters because ADK tooling (like `adk web` and `adk eval`) discovers your agent by looking for this pattern. Keep it clean and consistent from the start.

> **Tip:** You can also use `adk create my_first_agent` to scaffold this structure automatically.

### Step 2: define your agent

Every ADK agent needs three things at minimum:

1. **A name** - A unique identifier for your agent
2. **A model** - Which Gemini model to use for reasoning
3. **System instructions** - The prompt that tells the agent who it is and how to behave

Here is what this looks like conceptually:

```python
from google.adk.agents import Agent

my_agent = Agent(
    name="my_first_agent",
    model="gemini-3.5-flash",
    instruction="You are a helpful assistant that answers questions clearly and concisely.",
)
```

The `model` parameter determines which Gemini model handles the reasoning. For learning and prototyping, `gemini-3.5-flash` is a great choice - it is fast, capable, and cost-effective. For more complex reasoning tasks, you might upgrade to `gemini-3.1-pro`. (You can also use a rolling alias like `gemini-flash-latest` to always get the newest Flash model.)

The `instruction` parameter is your agent's system prompt. This is where you define its personality, capabilities, and boundaries. We will cover how to write good instructions later in this lesson.

### Step 3: create a custom function tool

Tools are what separate an agent from a chatbot. Let us give our agent a custom function it can call.

A tool in ADK is simply a Python function with a clear docstring. The docstring matters because ADK uses it to tell the model what the tool does and when to use it.

```python
def get_weather(city: str) -> dict:
    """Get the current weather for a given city.

    Args:
        city: The name of the city to get weather for.

    Returns:
        A dictionary with weather information.
    """
    # In a real agent, this would call a weather API
    # For now, return mock data
    return {
        "city": city,
        "temperature": "72F",
        "conditions": "Sunny",
    }
```

Key things to notice:

- **Type hints** (`city: str`, `-> dict`) help the model understand what parameters to pass and what to expect back
- **The docstring** is how the model learns what this tool does - write it like you are explaining the function to a colleague
- **The function name** should be descriptive - the model uses it to decide when to call the tool

You then attach the tool to your agent:

```python
my_agent = Agent(
    name="my_first_agent",
    model="gemini-3.5-flash",
    instruction="You are a helpful assistant. Use the get_weather tool when asked about weather.",
    tools=[get_weather],
)
```

### Step 4: add a built-in tool (google search)

ADK comes with several built-in tools. One of the most useful is Google Search grounding, which lets your agent search the web for current information.

```python
from google.adk.tools import google_search

my_agent = Agent(
    name="my_first_agent",
    model="gemini-3.5-flash",
    instruction="You are a helpful assistant. Use search for current events and get_weather for weather.",
    tools=[get_weather, google_search],
)
```

Now your agent can both search the web and check the weather. The model decides which tool to use based on the user's question. Ask about the news and it searches. Ask about the weather and it calls your custom function.

### Step 5: run and test locally

ADK includes a local development server that gives you a web UI to interact with your agent:

```bash
adk web
```

This starts a local server (typically at `http://localhost:8000`) with a chat interface. You can:

- Send messages to your agent
- See which tools it calls and why
- Inspect the full conversation history
- Debug issues in real time

You can also test from the command line:

```bash
adk run my_first_agent
```

This is your fastest feedback loop. Make a change, refresh, test. Repeat until the agent behaves the way you want.

### Step 6: evaluate with ADK eval`

Once your agent is working, you want to make sure it keeps working as you make changes. ADK includes an evaluation framework for this:

```bash
adk eval my_first_agent eval_data.json
```

Evaluation lets you define test cases - pairs of inputs and expected behaviors - and automatically check whether your agent handles them correctly. This is the agent equivalent of unit testing.

We covered evaluation concepts in depth in Lesson 9. The key idea here is to start writing eval cases early, even for your first agent. It saves you from regressions later.

> **Full quickstart:** Follow along with the complete code at [ADK Quickstart](https://adk.dev/get-started/quickstart/)

---

## Agent types in ADK

ADK provides four agent types, each designed for a different kind of task. Picking the right type is like picking the right data structure - it shapes everything that follows.

### LlmAgent (The Default)

This is the agent type you will use most often. It uses a language model to make decisions about what to do next.

**When to use it:**
- The task requires reasoning and judgment
- The agent needs to decide which tools to call based on context
- User interaction is conversational

**How it works:** The model receives the user's message, the available tools, and the conversation history. It decides whether to call a tool, ask a clarifying question, or respond directly. This is the ReAct loop in action.

```python
from google.adk.agents import Agent

researcher = Agent(
    name="researcher",
    model="gemini-3.5-flash",
    instruction="You research topics thoroughly using search.",
    tools=[google_search],
)
```

### Sequentialagent (steps in order)

A SequentialAgent runs a fixed list of sub-agents one after another, like a pipeline.

**When to use it:**
- The task has clear, ordered stages
- Each stage depends on the output of the previous one
- You want predictable, repeatable execution

**Example:** A content creation pipeline where one agent researches, the next writes a draft, and the third edits for grammar and style.

```python
from google.adk.agents import SequentialAgent

pipeline = SequentialAgent(
    name="content_pipeline",
    sub_agents=[researcher, writer, editor],
)
```

**Analogy:** Think of a SequentialAgent like an assembly line in a factory. Each station does one job and passes the result to the next station.

### Parallelagent (steps at the same time)

A ParallelAgent runs multiple sub-agents concurrently and collects their results.

**When to use it:**
- You have independent subtasks that do not depend on each other
- Speed matters and you want to reduce wall-clock time
- You need multiple perspectives on the same input

**Example:** Evaluating a piece of code by running a security reviewer, a performance reviewer, and a style reviewer all at the same time.

```python
from google.adk.agents import ParallelAgent

review_team = ParallelAgent(
    name="code_review",
    sub_agents=[security_reviewer, perf_reviewer, style_reviewer],
)
```

**Analogy:** Think of a ParallelAgent like a team brainstorm where everyone works on their part simultaneously and then presents their findings.

### Loopagent (repeat until done)

A LoopAgent runs its sub-agents in a cycle until a termination condition is met.

**When to use it:**
- The task requires iterative refinement
- You want to keep improving output until it meets a quality threshold
- The number of iterations is not known in advance

**Example:** A writing agent that drafts content, evaluates it against criteria, and revises until the quality score exceeds a threshold.

```python
from google.adk.agents import LoopAgent

refiner = LoopAgent(
    name="iterative_writer",
    sub_agents=[drafter, evaluator, reviser],
    max_iterations=5,
)
```

**Analogy:** Think of a LoopAgent like a code review cycle - you write, get feedback, revise, and repeat until the reviewer approves.

### Choosing the right agent type

| Agent Type | Use When | Example |
|---|---|---|
| LlmAgent | Flexible reasoning needed | Chatbot, research assistant |
| SequentialAgent | Fixed pipeline of steps | ETL, content creation |
| ParallelAgent | Independent parallel tasks | Multi-reviewer systems |
| LoopAgent | Iterative refinement | Quality improvement loops |

You can also combine these types. A SequentialAgent might have an LlmAgent as one of its steps. A LoopAgent might contain a ParallelAgent inside it. This composability is one of ADK's strengths.

> **Learn more:** [ADK Agent Types](https://adk.dev/agents/)

---

## Building a multi-tool agent

Most real agents need more than one tool. Let us look at how to combine multiple capabilities.

### Combining search, code execution, and custom tools

```python
from google.adk.agents import Agent
from google.adk.tools import google_search

def look_up_product(product_id: str) -> dict:
    """Look up product details by ID.

    Args:
        product_id: The unique product identifier.

    Returns:
        Product details including name, price, and availability.
    """
    # Call your product database/API here
    return {"id": product_id, "name": "Widget Pro", "price": 29.99}

def calculate_discount(price: float, discount_percent: float) -> float:
    """Calculate the discounted price.

    Args:
        price: The original price.
        discount_percent: The discount percentage (e.g., 20 for 20%).

    Returns:
        The price after discount.
    """
    return price * (1 - discount_percent / 100)

shopping_agent = Agent(
    name="shopping_assistant",
    model="gemini-3.5-flash",
    instruction="""You are a shopping assistant. You can:
    - Search the web for product reviews and comparisons
    - Look up specific products in our catalog by ID
    - Calculate discounts on prices

    Always look up the product first before calculating discounts.""",
    tools=[google_search, look_up_product, calculate_discount],
)
```

The key to multi-tool agents is clear instruction about when to use each tool. Without guidance, the model might search the web when it should query your database, or vice versa.

### Tool design principles

When building tools for your agent, follow these guidelines:

1. **One tool, one job.** Each tool should do one thing well. Do not create a mega-tool that handles five different operations.

2. **Descriptive names and docstrings.** The model picks tools based on their names and descriptions. `get_order_status` is better than `fetch_data`.

3. **Clear parameter types.** Use type hints. The model needs to know whether to pass a string, number, or object.

4. **Graceful error handling.** Return helpful error messages instead of crashing. The model can often recover if it understands what went wrong.

5. **Deterministic when possible.** Tools that return consistent results for the same inputs are easier for the model to reason about.

---

## Building an agent team

Sometimes one agent cannot handle everything. You might need specialists - one agent for research, another for writing, a third for fact-checking. ADK lets you build agent teams with a root agent that delegates to sub-agents.

### The root agent pattern

```python
from google.adk.agents import Agent

# Specialist agents
researcher = Agent(
    name="researcher",
    model="gemini-3.5-flash",
    instruction="You research topics using web search. Return factual findings.",
    tools=[google_search],
)

writer = Agent(
    name="writer",
    model="gemini-3.5-flash",
    instruction="You write clear, engaging content based on research findings.",
)

fact_checker = Agent(
    name="fact_checker",
    model="gemini-3.5-flash",
    instruction="You verify claims by searching for supporting evidence.",
    tools=[google_search],
)

# Root agent that orchestrates
coordinator = Agent(
    name="content_team",
    model="gemini-3.1-pro",
    instruction="""You coordinate a content creation team. For any content request:
    1. Ask the researcher to gather information
    2. Ask the writer to create content based on the research
    3. Ask the fact_checker to verify key claims

    Delegate tasks to your team members and synthesize their work.""",
    sub_agents=[researcher, writer, fact_checker],
)
```

### When to use sub-agents vs. multiple tools

| Approach | Best For | Trade-offs |
|---|---|---|
| One agent, many tools | Tasks where a single reasoning chain works | Simpler, but the agent might struggle with too many tools |
| Multiple sub-agents | Tasks that benefit from specialization | More flexible, but adds coordination overhead |

**Rule of thumb:** If your single agent has more than 10-15 tools and starts getting confused about which to use, consider splitting into sub-agents with focused tool sets.

### Communication between agents

Sub-agents in ADK share a session state, which acts as a shared workspace. The root agent can pass context to sub-agents, and sub-agents can store results that other agents can access.

Think of it like a shared document in a team project. The coordinator writes the brief, the researcher adds findings, and the writer uses those findings to draft content.

---

## Tips for writing good system instructions

The system instruction (prompt) is the single most important factor in how your agent behaves. Here are practical guidelines.

### Be specific about the role

**Weak:**
```
You are a helpful assistant.
```

**Better:**
```
You are a customer support agent for Acme Corp. You help customers
with order tracking, returns, and product questions. You have access
to the order database and can look up orders by ID or email.
```

### Set clear boundaries

Tell the agent what it should and should not do:

```
You ONLY handle questions about orders, returns, and products.
For billing questions, tell the customer to contact billing@acme.com.
Never make promises about delivery dates unless the tracking system confirms them.
Never share internal pricing or cost information.
```

### Provide examples

Show the agent how you want it to behave:

```
When a customer asks about their order status, follow this pattern:
1. Ask for their order ID if they have not provided one
2. Look up the order using the get_order tool
3. Summarize the status in plain language
4. If the order is delayed, apologize and offer to escalate

Example:
Customer: "Where is my order?"
You: "I'd be happy to help track your order. Could you share your order ID? It should start with ORD- and you can find it in your confirmation email."
```

### Define the output format

If you want structured responses, say so:

```
Always respond in this format:
- Start with a one-sentence summary
- Provide details in bullet points
- End with a suggested next step
```

### Common instruction mistakes

| Mistake | Why It Is a Problem | Fix |
|---|---|---|
| Too vague ("be helpful") | The model has no specific guidance | Define the role, scope, and behavior |
| Too long (2000+ words) | Key instructions get lost in noise | Keep it focused, use structure |
| No tool guidance | The model guesses when to use tools | Explain when each tool is appropriate |
| No error handling | The agent does not know what to do when things fail | Add fallback instructions |
| No boundaries | The agent tries to handle everything | Define what is out of scope |

---

## Common mistakes beginners make

### 1. starting too complex

You do not need a multi-agent system with 10 tools on day one. Start with one agent and one tool. Get that working perfectly before adding complexity.

### 2. ignoring the system instruction

Many beginners focus on tools and code but write a one-line system instruction. The instruction is your primary lever for controlling agent behavior. Invest time in it.

### 3. not testing tool calls

Your agent is only as reliable as its tools. Test each tool function independently before wiring it into the agent. If `get_weather("London")` throws an exception, your agent will too.

### 4. forgetting to handle errors

Tools fail. APIs time out. Data is missing. Your agent needs instructions for what to do when things go wrong. Add error handling in both your tool code and your system instructions.

### 5. using the wrong model

Not every task needs the most powerful model. For simple routing and tool-calling, a lighter model like Flash works well and saves money. Reserve larger models for tasks that genuinely require complex reasoning.

### 6. skipping evaluation

If you do not have eval cases, you do not know whether your changes improve or break things. Write at least 5-10 test cases before you start iterating on prompts.

### 7. putting secrets in code

Never hardcode API keys or credentials in your agent code. Use environment variables or a secrets manager. This is standard software engineering practice, but it is easy to forget when prototyping.

---

## Putting it all together

Here is a mental model for the entire process of building an ADK agent:

```
1. Define the goal
   "What should this agent accomplish?"
        |
        v
2. Choose the agent type
   LlmAgent? Sequential? Parallel? Loop?
        |
        v
3. Write the system instruction
   Role, scope, boundaries, examples
        |
        v
4. Define the tools
   What actions does the agent need?
        |
        v
5. Wire it together
   Agent + model + instruction + tools
        |
        v
6. Test locally
   adk web / adk run
        |
        v
7. Write eval cases
   Input -> expected behavior
        |
        v
8. Iterate
   Improve instructions, add tools, refine
        |
        v
9. Deploy
   Agent Engine or your own infrastructure
```

Each step feeds the next. And steps 6-8 are a loop - you will go around multiple times before moving to step 9.

---

## Quick reference: ADK CLI commands

| Command | What It Does |
|---|---|
| `pip install google-adk` | Install ADK |
| `adk create <name>` | Scaffold a new agent project |
| `adk web` | Start the local dev server with web UI |
| `adk run <agent>` | Run an agent from the command line |
| `adk eval <agent> <data>` | Run evaluation cases against your agent |
| `adk deploy agent_engine` | Deploy your agent to Agent Engine |

---

## Where to learn more

- **Getting started with ADK:** [https://adk.dev/get-started/](https://adk.dev/get-started/)
- **Full quickstart tutorial:** [https://adk.dev/get-started/quickstart/](https://adk.dev/get-started/quickstart/)
- **Agent types in depth:** [https://adk.dev/agents/](https://adk.dev/agents/)
- **Tools reference:** [https://adk.dev/tools/](https://adk.dev/tools/)

---

## You have been building a harness

Step back and look at what this lesson actually produced. The model is one parameter. Everything else - the system instruction, the tool definitions and their docstrings, the eval cases, the agent type you chose - is scaffolding around the model. That scaffolding has a name: the **harness**. A useful equation to carry forward:

```
agent = model + harness
```

This matters because the harness, not the model, is usually what determines whether your agent works. A decent model with a great harness will beat a great model with a bad harness - benchmark teams have jumped dozens of ranks on agentic coding leaderboards by changing only the harness, with the model untouched. So when your agent misbehaves, resist the instinct to conclude "the model is not smart enough." Most agent failures are configuration failures: a vague tool description, a missing feedback signal, context stuffed with irrelevant material.

The practical habit this suggests is the **ratchet**: every time your agent fails, turn the failure into a permanent constraint in the harness so it cannot happen the same way twice. The agent used the wrong test runner? Add a line to your [AGENTS.md](../15-agents-md/). It skipped a required step? Tighten the system instruction or add a checkpoint to the workflow. It produced a bad output shape? Add the case to your evals from [Lesson 9](../09-evaluating-and-testing-agents/). Each fix ratchets the floor upward, and the harness compounds in value while prompts alone would not. Addy Osmani's [Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/) is a good deep dive on this mindset.

---

## Key takeaways

1. **An ADK agent needs three things:** a name, a model, and system instructions. Tools are optional but are what make agents genuinely useful.

2. **Start with LlmAgent.** It handles most use cases. Reach for SequentialAgent, ParallelAgent, or LoopAgent only when you have a clear structural reason.

3. **System instructions are your primary control lever.** Be specific about the role, set boundaries, provide examples, and include error handling guidance.

4. **Test early and often.** Use `adk web` for interactive testing and `adk eval` for automated regression checks.

5. **Build incrementally.** One agent, one tool, one eval case. Get each piece working before adding the next.

6. **The harness determines the agent.** Most agent failures are harness failures, not model failures. When something goes wrong, ratchet a fix into the harness - an instruction, a checkpoint, an eval - so it cannot recur.

---

## What is next?

Now that you can build an agent, you need to understand how agents communicate with the wider world. In the next lesson, we will explore two important protocols - MCP and A2A - that let agents talk to tools and to each other using open standards.

[Next: Lesson 14 - Agent Protocols: MCP and A2A -->](../14-agent-protocols-mcp-and-a2a/)
