# Lesson 2: how agents think - LLMs as the reasoning engine

## Introduction

In Lesson 1, we said that an agent is made of three parts: a model (the brain), tools (the hands), and an orchestration layer (the control loop). In this lesson, we zoom in on the brain.

The language model is the most important component of an agent. It is the part that reads the user's goal, reasons about what to do, decides which tools to call, interprets results, and generates final answers. Everything else in the agent system exists to support or extend what the model can do.

Understanding how LLMs work - even at a high level - will make you a significantly better agent builder. You will know why some prompts work and others do not. You will understand why agents sometimes go off the rails. And you will be able to make informed decisions about model selection, prompt design, and system architecture.

---

## LLMs as the brain of the agent

A large language model is a neural network trained on vast amounts of text data. At its core, it does one thing: **predict the next token** (roughly, the next word or piece of a word) given everything that came before it.

That sounds simple, but the emergent capabilities from this training are remarkable:

- **Comprehension**: Understanding complex questions and instructions
- **Reasoning**: Working through multi-step logic problems
- **Planning**: Breaking a goal into subtasks
- **Code generation**: Writing and debugging software
- **Tool selection**: Deciding which function to call and with what parameters
- **Summarization**: Distilling long documents into key points
- **Translation**: Converting between languages, formats, and representations

### What LLMs can do well

| Capability | Example in Agent Context |
|---|---|
| Natural language understanding | Parsing a user's request: "Cancel my most recent order" |
| Reasoning and planning | Deciding: "First I need to find the order, then check if it is cancelable, then cancel it" |
| Tool selection | Choosing to call `lookup_order(user_id, sort="recent")` |
| Output formatting | Returning a clean JSON response or friendly message |
| Error interpretation | Reading an API error and deciding to retry with different parameters |
| Context synthesis | Combining results from multiple tool calls into a coherent answer |

### What LLMs cannot do (without help)

| Limitation | Why It Matters |
|---|---|
| No real-time data access | The model's knowledge has a training cutoff date |
| No computation guarantees | LLMs can get math wrong - they predict tokens, not calculate |
| No persistent memory | Each conversation starts fresh unless you build memory in |
| No ability to act | Without tools, the model can only generate text |
| Hallucination risk | Models can generate plausible but incorrect information |
| Context window limits | There is a maximum amount of text the model can process at once |

This is why agents exist. Tools compensate for the model's inability to act and access live data. Orchestration compensates for its lack of persistent memory and its tendency to go off track.

---

## How language models process information

You do not need to understand transformer architecture in detail to build agents, but understanding three key concepts will help you write better prompts and design better systems.

### Tokens: the unit of language

LLMs do not read characters or words. They read **tokens** - chunks of text that the model has learned to recognize during training. A token might be a whole word, part of a word, or a punctuation mark.

**Examples:**

| Text | Approximate Tokens |
|---|---|
| "Hello" | 1 token |
| "Hello, world!" | 3 tokens |
| "ChatGPT is amazing" | 4 tokens |
| A typical code function (20 lines) | 100-300 tokens |
| A full page of English text | ~500-700 tokens |

**Why this matters for agents:**

- **Billing**: Most APIs charge per token (input + output). Agent workflows use many more tokens than single prompts because every iteration of the loop sends the full context. **Prompt caching** softens this: when the start of the context (system instructions, tool definitions, earlier turns) is unchanged between calls, providers bill the cached part at a steep discount. Keep that stable prefix at the top and append new content at the end.
- **Thinking tokens**: Current models reason internally before answering, and those thinking tokens are billed as output. The thinking level you choose (covered below) is now a major cost and latency dial.
- **Speed**: More tokens = more time to generate. Keep your tool descriptions concise.
- **Context limits**: There is a maximum number of tokens the model can process in a single call. If your agent's accumulated context exceeds this, you lose information.

### Context windows: the model's working memory

The **context window** is the total number of tokens a model can consider at once. Think of it like the model's desk - everything it needs to reference must fit on this desk.

| Model | Context Window |
|---|---|
| Gemini 3.1 Pro | 1,000,000 tokens |
| Gemini 3.5 Flash | 1,000,000 tokens |
| GPT-5.6 | 1,050,000 tokens |
| Claude Sonnet 5.5 | 1,000,000 tokens |

**What goes into the context window during an agent call:**

```
+------------------------------------------+
| System instructions ("You are a...")     |  ~200-500 tokens
+------------------------------------------+
| Tool definitions (names, descriptions,   |  ~500-2000 tokens
| parameter schemas)                       |
+------------------------------------------+
| Conversation history                     |  Variable
+------------------------------------------+
| Previous tool calls and results          |  Variable (can grow fast)
+------------------------------------------+
| Current user message                     |  Variable
+------------------------------------------+
| = Total must fit within context window   |
+------------------------------------------+
```

**Why this matters for agents:**

As an agent executes multiple steps, the context grows with each tool call and result. A long-running agent can accumulate hundreds of thousands of tokens of tool results.

With frontier models now at around 1M tokens, running out of room is less often the problem. The bigger problem is **context rot**: answer quality and instruction-following degrade well before the window is full, as the model has more irrelevant material to sift through. A bigger window is not a substitute for a clean one.

Strategies to manage this:
- **Truncate** long tool outputs to the relevant parts before they enter the context
- **Compact** the history: when context grows large, replace older turns and stale tool results with a summary (many frameworks and APIs now do this for you)
- **Clear** tool results the agent no longer needs, such as a file it has already edited
- **Offload** to external memory (files, a notes tool, a database) and retrieve on demand instead of keeping everything in context
- **Keep the prefix stable** so prompt caching keeps working - edit or drop content near the end, not the start

<div id="context-window-explorer" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 900px; margin: 32px auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 2px 16px rgba(0,0,0,0.08); padding: 32px; box-sizing: border-box;">
  <h3 style="margin: 0 0 4px 0; font-size: 20px; color: #1a1a2e;">Context Window Explorer</h3>
  <p style="margin: 0 0 20px 0; font-size: 14px; color: #6b7280;">Adjust the sliders to see how different components fill up the context window.</p>

  <div style="display: flex; gap: 24px; flex-wrap: wrap;">
    <div style="flex: 1.2; min-width: 300px;">
      <div id="cwe-sliders"></div>
      <div id="cwe-model-select" style="margin-top: 16px; background: white; border-radius: 12px; padding: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06);">
        <div style="font-size: 13px; font-weight: 600; color: #6b7280; margin-bottom: 8px;">SELECT MODEL</div>
        <div id="cwe-models" style="display: flex; gap: 8px; flex-wrap: wrap;"></div>
      </div>
    </div>
    <div style="flex: 0.8; min-width: 250px;">
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06); margin-bottom: 16px;">
        <div style="font-size: 13px; font-weight: 600; color: #6b7280; margin-bottom: 8px;">CONTEXT WINDOW USAGE</div>
        <div id="cwe-container" style="width: 100%; height: 320px; background: #f3f4f6; border-radius: 10px; border: 2px solid #e5e7eb; position: relative; overflow: hidden;"></div>
        <div id="cwe-remaining" style="text-align: center; margin-top: 10px; font-size: 14px; font-weight: 600;"></div>
      </div>
      <div id="cwe-bar-chart" style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06);">
        <div style="font-size: 13px; font-weight: 600; color: #6b7280; margin-bottom: 8px;">TOKEN ALLOCATION</div>
        <div id="cwe-bars"></div>
      </div>
    </div>
  </div>

  <script>
  (function() {
    var components = [
      { name: "System Prompt", color: "#4285f4", min: 100, max: 2000, value: 400, icon: "S" },
      { name: "Tool Definitions", color: "#9333ea", min: 0, max: 5000, value: 1200, icon: "T" },
      { name: "Conversation History", color: "#34a853", min: 0, max: 50000, value: 8000, icon: "C" },
      { name: "Current Task", color: "#fbbc04", min: 50, max: 10000, value: 500, icon: "K" },
      { name: "Retrieved Context (RAG)", color: "#ea4335", min: 0, max: 30000, value: 4000, icon: "R" }
    ];
    var models = [
      { name: "GPT-5.6", tokens: 1050000 },
      { name: "Claude Sonnet 5.5", tokens: 1000000 },
      { name: "Gemini 3.5 Flash", tokens: 1000000 },
      { name: "Gemini 3.1 Pro", tokens: 1000000 }
    ];
    var selectedModel = 0;

    function getTotal() {
      return components.reduce(function(s, c) { return s + c.value; }, 0);
    }

    function formatK(n) {
      return n >= 1000 ? (n/1000).toFixed(n >= 10000 ? 0 : 1) + "K" : n.toString();
    }

    function renderSliders() {
      var el = document.getElementById("cwe-sliders");
      el.innerHTML = "";
      components.forEach(function(c, i) {
        var div = document.createElement("div");
        div.style.cssText = "background:white;border-radius:12px;padding:14px 16px;margin-bottom:10px;box-shadow:0 1px 4px rgba(0,0,0,0.06);";
        div.innerHTML = '<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:6px;">' +
          '<span style="font-size:14px;font-weight:600;color:#374151;display:flex;align-items:center;gap:8px;"><span style="width:12px;height:12px;border-radius:3px;background:'+c.color+';display:inline-block;"></span>'+c.name+'</span>' +
          '<span id="cwe-val-'+i+'" style="font-size:13px;font-weight:600;color:'+c.color+';">'+formatK(c.value)+' tokens</span></div>' +
          '<input type="range" id="cwe-slider-'+i+'" min="'+c.min+'" max="'+c.max+'" value="'+c.value+'" style="width:100%;accent-color:'+c.color+';height:6px;cursor:pointer;">';
        el.appendChild(div);
        document.getElementById("cwe-slider-" + i).addEventListener("input", function(e) {
          components[i].value = parseInt(e.target.value);
          document.getElementById("cwe-val-" + i).textContent = formatK(components[i].value) + " tokens";
          renderVisual();
        });
      });
    }

    function renderModels() {
      var el = document.getElementById("cwe-models");
      el.innerHTML = "";
      models.forEach(function(m, i) {
        var btn = document.createElement("button");
        btn.textContent = m.name + " (" + formatK(m.tokens) + ")";
        btn.style.cssText = "padding:6px 12px;border-radius:6px;border:2px solid " + (i === selectedModel ? "#4285f4" : "#e5e7eb") + ";background:" + (i === selectedModel ? "#4285f4" : "white") + ";color:" + (i === selectedModel ? "white" : "#374151") + ";font-size:12px;font-weight:500;cursor:pointer;transition:all 0.2s;";
        btn.onclick = function() { selectedModel = i; renderModels(); renderVisual(); };
        el.appendChild(btn);
      });
    }

    function renderVisual() {
      var total = getTotal();
      var maxTokens = models[selectedModel].tokens;
      var pct = Math.min(total / maxTokens, 1);
      var containerEl = document.getElementById("cwe-container");
      var html = "";
      var yPct = 0;
      components.forEach(function(c) {
        var h = (c.value / maxTokens) * 100;
        if (h > 0.1) {
          html += '<div style="position:absolute;left:0;right:0;top:'+yPct+'%;height:'+h+'%;background:'+c.color+';opacity:0.75;transition:all 0.3s;display:flex;align-items:center;justify-content:center;">' +
            (h > 3 ? '<span style="color:white;font-size:11px;font-weight:600;text-shadow:0 1px 2px rgba(0,0,0,0.3);">'+c.name+' ('+formatK(c.value)+')</span>' : '') + '</div>';
          yPct += h;
        }
      });
      if (yPct < 100) {
        html += '<div style="position:absolute;left:0;right:0;top:'+yPct+'%;bottom:0;background:repeating-linear-gradient(45deg,transparent,transparent 8px,#e5e7eb 8px,#e5e7eb 9px);display:flex;align-items:center;justify-content:center;">' +
          '<span style="font-size:12px;color:#9ca3af;font-weight:500;">Available: '+formatK(maxTokens - total)+'</span></div>';
      }
      containerEl.innerHTML = html;
      var remaining = maxTokens - total;
      var remEl = document.getElementById("cwe-remaining");
      if (remaining < 0) {
        remEl.innerHTML = '<span style="color:#ea4335;">Over limit by ' + formatK(Math.abs(remaining)) + ' tokens!</span>';
      } else if (remaining < maxTokens * 0.1) {
        remEl.innerHTML = '<span style="color:#fbbc04;">Warning: Only ' + formatK(remaining) + ' tokens remaining (' + ((remaining/maxTokens)*100).toFixed(1) + '%)</span>';
      } else {
        remEl.innerHTML = '<span style="color:#34a853;">' + formatK(remaining) + ' tokens remaining (' + ((remaining/maxTokens)*100).toFixed(1) + '%)</span>';
      }
      // Bar chart
      var barsEl = document.getElementById("cwe-bars");
      barsEl.innerHTML = "";
      components.forEach(function(c) {
        var barPct = Math.min((c.value / total) * 100, 100);
        barsEl.innerHTML += '<div style="display:flex;align-items:center;gap:8px;margin-bottom:6px;">' +
          '<span style="font-size:11px;color:#6b7280;width:80px;text-align:right;flex-shrink:0;">'+c.name.split(" ")[0]+'</span>' +
          '<div style="flex:1;background:#f3f4f6;border-radius:4px;height:16px;overflow:hidden;">' +
          '<div style="width:'+barPct+'%;height:100%;background:'+c.color+';border-radius:4px;transition:width 0.3s;"></div></div>' +
          '<span style="font-size:11px;color:#374151;font-weight:600;width:40px;">'+barPct.toFixed(0)+'%</span></div>';
      });
    }

    renderSliders();
    renderModels();
    renderVisual();
  })();
  </script>
</div>

### Attention: how the model focuses

The **attention mechanism** is what allows the model to figure out which parts of the context are relevant to the current decision. When deciding what token to generate next, the model assigns different weights to different parts of the input.

Think of it like reading a long document and highlighting the important parts. The model "highlights" the tokens most relevant to what it is trying to do right now.

**Why this matters for agents:**

- **Put important information where the model can find it.** Models tend to pay more attention to the beginning and end of the context. Critical instructions should go in the system prompt (beginning) or close to the user's query (end).
- **Be specific and clear.** Vague instructions force the model to guess what matters. Specific instructions make it easy for the attention mechanism to latch onto the right information.
- **Structure helps.** Clear headers, numbered lists, and consistent formatting help the model parse and attend to the right content.

---

## Reasoning strategies

How an agent "thinks" about a problem used to depend mostly on how you prompted the model. Today's frontier models (Gemini 3.x, Claude, GPT-5) are **thinking models**: they reason internally before they answer, and you control how much through an API parameter rather than prompt wording. The classic strategies below are still worth understanding - they explain what the model is doing under the hood, and they still apply to non-thinking models and to how you structure an agent's workflow.

### Built-in thinking and thinking levels

**What it is:** The model produces a hidden (or summarized) reasoning pass before its visible answer. This is Chain-of-Thought trained into the model, rather than requested in the prompt.

**How you control it:** Instead of writing "think step by step", you set how much reasoning to spend. In the Gemini API this is `thinking_level` (for example `low` or `high`; Gemini 3 models default to `high`). Other providers expose a similar setting, often called reasoning effort.

| Thinking level | Use for | Trade-off |
|---|---|---|
| Minimal / low | Classification, extraction, simple tool calls, chat | Fastest and cheapest |
| High | Multi-step planning, debugging, complex code, ambiguous tool choice | Slower, more output tokens billed |

**Why this matters for agents:**
- Adding "think step by step" to a thinking model's prompt is mostly redundant. Spend your prompt on the task, constraints, and context instead.
- The thinking level is a dial you can turn per step, which gives you much of the benefit of model routing without switching models (see below).
- In multi-turn tool use, some APIs return the model's reasoning state (for example, Gemini's thought signatures) that you must pass back on the next call. The official SDKs and frameworks like ADK handle this for you; hand-rolled loops must not drop it.

> **Learn more:** [Gemini thinking](https://ai.google.dev/gemini-api/docs/thinking)

### Chain-of-Thought (CoT)

**What it is:** Prompting the model to think through a problem step by step before giving a final answer. This is the technique that built-in thinking grew out of.

**How it works:** Instead of jumping straight to an answer, the model generates intermediate reasoning steps. This dramatically improves accuracy on tasks that require logic, math, or multi-step analysis. With thinking models this happens automatically; with non-thinking or lightweight models, you still prompt for it explicitly.

**Example without CoT:**
```
Prompt: "If a server handles 100 requests/second and we have 3 servers,
         with 40% of traffic going to server 1, how many requests/second
         does server 1 handle?"

Model response: "120 requests per second"  (wrong)
```

**Example with CoT:**
```
Prompt: "Think step by step. If a server handles 100 requests/second and
         we have 3 servers, with 40% of traffic going to server 1, how
         many requests/second does server 1 handle?"

Model response:
"Step 1: Total capacity is 3 servers x 100 req/s = 300 req/s
 Step 2: Server 1 receives 40% of total traffic
 Step 3: 40% of 300 = 120 req/s
 Step 4: Server 1 can handle 100 req/s but receives 120 req/s
 Answer: Server 1 receives 120 req/s but can only handle 100 req/s,
         so it is overloaded by 20 req/s"
```

The step-by-step approach caught the overload condition that the direct answer missed.

**When to use CoT for agents:**
- Complex tool selection decisions ("Given these 5 tools, which one helps here?")
- Multi-step planning ("What sequence of actions achieves this goal?")
- Error diagnosis ("The tool returned an error - what went wrong and what should I try next?")

### Tree-of-Thoughts (ToT)

**What it is:** An extension of Chain-of-Thought where the model explores multiple reasoning paths, evaluates them, and picks the best one.

**How it works:** Instead of one chain of reasoning, the model generates several possible approaches, scores or critiques each one, and proceeds with the most promising path.

```
Goal: "Optimize this slow database query"

Path A: "Add an index on the WHERE clause column"
  -> Evaluation: "Likely effective, low risk, easy to implement"

Path B: "Rewrite as a materialized view"
  -> Evaluation: "Might help, but adds complexity and maintenance"

Path C: "Denormalize the table structure"
  -> Evaluation: "Could work but high risk, affects other queries"

Decision: Proceed with Path A first, try Path B if A is insufficient
```

**When to use ToT for agents:**
- When there are multiple valid approaches and you want the model to consider trade-offs
- Complex debugging where the root cause is uncertain
- Architecture decisions that require evaluating alternatives

**Trade-off:** ToT uses more tokens and takes more time. Reserve it for decisions where the cost of choosing wrong is high.

**In practice today:** A thinking model already weighs alternatives internally, so ToT is rarely written as a single prompt anymore. When you need it explicitly, it lives in the orchestration layer: generate several candidate solutions in parallel calls, have a judge step (or tests) score them, and continue with the best. This is close kin to the reflection pattern in Lesson 4.

### Step-by-Step Decomposition

**What it is:** Breaking a complex goal into a sequence of simpler subtasks before executing any of them.

**How it works:** The agent first creates a plan, then executes each step of the plan, checking progress along the way.

```
User goal: "Set up monitoring for our new API endpoint"

Plan:
1. Check what monitoring tools are currently configured
2. Determine what metrics matter for this endpoint (latency, error rate, throughput)
3. Create the monitoring dashboard
4. Set up alerting thresholds
5. Test that alerts fire correctly
6. Document the monitoring setup

Execution: [proceeds step by step, with each step potentially using tools]
```

**When to use decomposition for agents:**
- Multi-step tasks where the order matters
- Tasks where you want the agent to be transparent about its approach
- Complex workflows that benefit from checkpoints

---

## Model selection: picking the right model for the job

Not all tasks need the most powerful model. Choosing the right model is an engineering decision that balances capability, cost, speed, and reliability.

### The model spectrum

```
Lighter / Faster / Cheaper                  Heavier / Smarter / More Expensive
|----------------------------------------------------------|
Gemini Flash-Lite     Gemini Flash        Gemini Pro
(simple tasks)        (balanced)          (complex reasoning)
```

### When to use what

| Task Type | Recommended Tier | Why |
|---|---|---|
| Classification ("Is this spam?") | Light (Flash) | Simple decision, no complex reasoning needed |
| Data extraction ("Pull the date from this email") | Light (Flash) | Pattern matching, well-defined output |
| Summarization | Light to Medium | Depends on length and complexity of source |
| Multi-step reasoning | Medium to Heavy (Pro) | Requires sustained logical chains |
| Complex code generation | Heavy (Pro) | Needs deep understanding of patterns and edge cases |
| Agentic tool use | Medium to Heavy | Tool selection and result interpretation need strong reasoning |
| Creative writing | Medium | Good results without the heaviest models |

### Model routing: using different models for different steps

A sophisticated agent system does not use the same model for every step. This is called **model routing** - directing different parts of the workflow to different models based on complexity.

**Example architecture:**

```
User query arrives
    |
    v
[Light model: Classify intent]  --> "order_status"
    |
    v
[Light model: Extract parameters]  --> order_id: 12345
    |
    v
[Tool call: Look up order]  --> status data
    |
    v
[Light model: Format response]  --> "Your order #12345 shipped on March 15"
```

In this flow, every step uses a fast, cheap model because none of the individual steps require heavy reasoning. The total cost and latency are much lower than using a frontier model for the entire interaction.

**Compare with a harder task:**

```
User: "Review this pull request and suggest improvements"
    |
    v
[Heavy model: Analyze code changes, reason about patterns,
 identify bugs, suggest improvements]
    |
    v
[Return detailed review]
```

This task needs deep reasoning, so it warrants a more capable model.

**Routing by thinking level:** You do not always need separate models. A single capable model with a low thinking level for simple steps and a high thinking level for hard ones covers much of the same ground, with one model to evaluate and maintain. Route across models when the cost or latency difference is large enough to justify the extra moving parts.

### Google Cloud Model options

Google Cloud provides access to Gemini models through Vertex AI:

- **Gemini 3.5 Flash** - Google's current default workhorse. Fast and efficient for most agent tasks, with strong agentic and coding performance and a large context window (1M tokens).
- **Gemini 3.1 Pro** - Top-tier reasoning for complex tasks. Use when the task requires deep analysis, complex multi-step logic, or nuanced understanding.
- **Gemini 3.1 Flash-Lite** - Fastest and cheapest option for simple tasks like classification and extraction.

> **Note:** Model names move quickly. The Gemini 2.5 family (2.5 Flash, 2.5 Pro, 2.5 Flash-Lite) reached its retirement date on October 16, 2026, and Gemini 2.0 models were shut down on June 1, 2026. If you have code pinned to those IDs, migrate it. Always check the docs for the current model IDs.

> **Note:** Use the Google Gen AI SDK (`google-genai`) to call Gemini. Releases of the older Vertex AI SDK after June 2026 do not support Gemini, and Gemini 3.5 Flash and later need `google-genai` 2.0.0 or newer. See the [migration guide](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/migrate).

> **Learn more:** [Vertex AI Model Documentation](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/models)

> **Learn more:** [Gemini API Documentation](https://ai.google.dev/gemini-api/docs)

---

## The role of system instructions

System instructions are the agent's "job description." They tell the model who it is, what it can do, how it should behave, and what it should not do.

### What goes in system instructions

A well-written system instruction for an agent typically includes:

1. **Role definition**: What the agent is and who it serves
2. **Tool usage policy**: When to use which tool, and in what order (the tools themselves are defined separately - see below)
3. **Constraints**: What the agent should not do
4. **Output format**: How to structure responses
5. **Error handling**: What to do when things go wrong
6. **Personality/tone**: How to communicate (if relevant)

Tools are passed to the model as structured function declarations (name, description, parameter schema) through the API, not written into the prompt. Re-listing their signatures in the system instructions wastes tokens and can drift out of sync with the real definitions. Put *what a tool does* in its description, and *when to use it* in the system instructions. Lesson 3 covers tool definitions in detail.

### Example: customer support agent

```
You are a customer support agent for Acme Corp, an online electronics retailer.

Your role:
- Help customers with order inquiries, returns, and product questions
- Be friendly, professional, and concise

Tool usage:
- Use lookup_order before answering any question about an order
- Only call initiate_return after the customer confirms they want a return
- Use escalate_to_human when the customer asks for a person

Guidelines:
- Always verify the customer's identity before accessing order information
- If you cannot resolve an issue in 3 attempts, escalate to a human agent
- Never make up order information - always use the lookup tool
- Do not offer discounts or refunds beyond standard policy
- Keep responses under 3 paragraphs

When you encounter an error from a tool:
- If it is a temporary error (timeout, 500), retry once
- If it is a permanent error (not found, unauthorized), explain the issue to the customer
- If you are unsure, escalate to a human agent
```

### Tips for writing effective system instructions

| Do | Do Not |
|---|---|
| Be specific about tool usage | Leave tool selection ambiguous |
| Define clear boundaries | Assume the model knows your business rules |
| Include error handling guidance | Hope the model figures out errors on its own |
| Enforce machine-read output with structured output (a JSON schema in the API config) | Rely on prompt wording alone to get valid JSON |
| Use concrete examples | Write abstract, vague instructions |
| Keep instructions concise | Write a 10-page essay (wastes context) |

### Ordering matters

Models pay more attention to instructions at the beginning and end of the system prompt. Structure your system instructions like this:

```
1. Most critical rules (identity, safety constraints)     <-- Start
2. Tool usage guidelines
3. Output format
4. Examples
5. Edge case handling
6. Reminder of most critical rules                        <-- End
```

This takes advantage of the "primacy and recency" effects in attention.

---

## Temperature, sampling, and agent behavior

When a language model generates text, it does not just pick the single most likely next token. It samples from a probability distribution over all possible tokens. The parameters that control this sampling have a big impact on agent behavior.

### Temperature

**Temperature** controls how random the model's outputs are.

- **Temperature 0 (or very low)**: The model almost always picks the most likely token. Outputs are focused and far more repeatable, though not perfectly deterministic - serving infrastructure (batching, hardware) can still introduce small run-to-run differences.
- **Temperature 1**: The model samples proportionally to token probabilities. Outputs are more varied and creative.
- **Temperature > 1**: The model becomes increasingly random. Outputs become unpredictable.

**Visual analogy:**

```
Temperature 0:    "The capital of France is Paris."
                  (Nearly always the same answer)

Temperature 0.7:  "The capital of France is Paris, a city known for..."
                  (Slight variation in elaboration)

Temperature 1.5:  "The capital of France is historically rooted in..."
                  (More creative, potentially off-track)
```

### What temperature to use for agents

The long-standing advice was "keep temperature low (0 to 0.3) for agents." **That no longer holds for thinking models.**

- **Gemini 3.x:** Google recommends leaving temperature at its default of **1.0**. The models' reasoning is tuned for that setting, and lowering it can cause looping or degraded performance, especially on complex reasoning tasks.
- **Other providers' reasoning models** similarly restrict or ignore temperature when thinking is enabled.

| Model type | Temperature guidance |
|---|---|
| Thinking models (Gemini 3.x and peers) | Leave at the default. Tune the thinking level instead. |
| Non-thinking or older models | Low (0 - 0.3) for tool calls, extraction, and code; higher (0.7 - 1.0) for creative or brainstorming tasks |

**For thinking models, reliability comes from other levers:** clear tool descriptions, a well-structured system prompt, structured output for anything a program parses, an appropriate thinking level, and evals that catch regressions (Lesson 9). Always check the current docs for the model you use.

> **Learn more:** [Gemini 3 developer guide](https://ai.google.dev/gemini-api/docs/gemini-3)

### Top-K and Top-P Sampling

These are additional controls on how the model selects tokens.

**Top-K:** Only consider the K most likely tokens. If K=50, the model ignores every token outside the top 50 candidates.

**Top-P (nucleus sampling):** Only consider tokens whose cumulative probability reaches P. If P=0.9, the model considers the smallest set of tokens that together have a 90% probability.

```
Token probabilities: [0.4, 0.25, 0.15, 0.08, 0.05, 0.03, 0.02, ...]

Top-K=3:  Consider only [0.4, 0.25, 0.15]
Top-P=0.8: Consider only [0.4, 0.25, 0.15] (cumulative = 0.8)
Top-P=0.9: Consider only [0.4, 0.25, 0.15, 0.08] (cumulative = 0.88... round up)
```

**For agents:** Leave Top-K and Top-P at the model's defaults. With current models, the sampling defaults are the tested configuration; the settings you are more likely to adjust are the thinking level and, for older models, temperature.

### How sampling affects the agent loop

Consider an agent that needs to decide which tool to call. You want it to pick the same (correct) tool for the same situation, every time. Inconsistency here makes agents hard to debug.

```
User: "What is the weather in Tokyo?"

Reliable:
  -> Agent thinks: "I need the weather tool"
  -> Calls: get_weather(city="Tokyo")
  -> Same choice on every run

Unreliable:
  -> Run 1: Calls get_weather(city="Tokyo")
  -> Run 2: Calls web_search("Tokyo weather forecast")
  -> Run 3: Tries to answer from training data (no tool call)
  -> Inconsistent, hard to debug
```

With older models, the main cause of the unreliable pattern was high temperature. With thinking models at default settings, it is far more often an unclear tool description, overlapping tools, or vague instructions. Fix those first, and measure consistency with repeated eval runs rather than assuming a sampling setting guarantees it.

---

## ELI5: how the LLM brain works

### Think of the LLM like a chef

Imagine you are running a restaurant kitchen, and the LLM is your head chef.

**The chef's training (model training):**
The chef has spent years studying thousands of cookbooks, watching cooking shows, and practicing recipes. They have not memorized every recipe word for word, but they have developed deep intuitions about what flavors go together, what techniques work for what ingredients, and how to improvise when something is missing.

**Tokens are like ingredients:**
The chef does not think in terms of complete dishes all at once. They think in terms of individual ingredients and steps. "First the onion, then the garlic, then the tomatoes..." Each ingredient choice informs the next one. That is how token prediction works - each token is chosen based on all the tokens before it.

**The context window is like the counter space:**
The chef can only work with what fits on the kitchen counter. If the counter is huge (1 million tokens), they can have lots of ingredients, recipes, and prep work visible at once. If the counter is small, they have to put things away to make room, and might forget what they were doing.

**Temperature is like the chef's mood:**
- Low temperature: The chef is focused and methodical. They follow the recipe exactly. Every time you order the same dish, it tastes the same.
- High temperature: The chef is feeling creative. They improvise, substitute ingredients, try new things. Sometimes the result is amazing, sometimes it is weird.

**The thinking level is like prep time:**
For a simple order, the chef starts cooking right away. For a tricky one, they stop and plan the whole dish first. More planning time means better results on hard dishes, but slower service - so you only ask for it when the dish needs it.

**System instructions are like the restaurant concept:**
"You are a French bistro. You use traditional techniques. You do not serve sushi." This shapes every decision the chef makes without having to be repeated for each dish.

**Tools are like kitchen equipment:**
The chef's knowledge alone does not cook food. They need an oven, a stove, knives, and measuring tools. Similarly, the LLM's reasoning alone does not look up data or call APIs. It needs tools.

**The agent loop is like a cooking show challenge:**
The chef gets a challenge ("Make a three-course meal for someone who is gluten-free"). They plan their approach, start cooking, taste as they go, adjust seasoning, plate the food, and evaluate the result. If the sauce breaks, they troubleshoot and adapt. That plan-act-observe-adjust loop is exactly what an agent does.

---

## Putting it together: how model choice affects agent quality

Here is a practical example of how these concepts combine in a real agent scenario.

### Scenario: a bug triage agent

Your team wants an agent that reads new bug reports, categorizes them by severity, assigns them to the right team, and drafts an initial investigation plan.

**Model selection decision:**

| Step | Model Choice | Reasoning |
|---|---|---|
| Classify severity (P0-P3) | Flash (light) | Simple classification with clear criteria |
| Assign to team | Flash (light) | Lookup-style decision based on component |
| Draft investigation plan | Pro (heavy) | Requires understanding the bug, related systems, and suggesting diagnostic steps |

**Thinking level and output decisions:**

| Step | Thinking level | Output | Reasoning |
|---|---|---|---|
| Classify severity | Low | Structured output: enum `P0`-`P3` | Clear criteria; the schema guarantees a valid label every time |
| Assign to team | Low | Structured output: enum of team names | Lookup-style decision; the schema rules out made-up team names |
| Draft investigation plan | High | Free text | Needs real reasoning about the bug, related systems, and diagnostics |

Temperature stays at the model default throughout. Consistency comes from the schemas, the clear criteria in the system instructions, and an eval set of past bugs with known severities and owners.

**System instruction excerpt:**

```
You are a bug triage agent for the Platform Engineering team.

Severity classification:
- P0: Service is down or data loss is occurring
- P1: Major feature is broken, no workaround
- P2: Feature is impaired but a workaround exists
- P3: Minor issue, cosmetic, or improvement request

Team routing:
- Auth/login issues -> Identity team
- API errors -> Platform team
- UI issues -> Frontend team
- Database/performance -> Infrastructure team

When drafting an investigation plan:
- Start with the most likely root cause
- List 3-5 diagnostic steps in order of priority
- Include relevant log queries or dashboard links
- Note any recent deployments that might be related
```

This example shows how understanding the model's capabilities, choosing the right thinking level and output format, and writing clear instructions all contribute to a reliable agent.

---

## Common mistakes when working with LLMs in agents

### 1. trusting the model for math

LLMs predict tokens, not compute equations. Thinking models get simple arithmetic right far more often than older models did, but "usually right" is not good enough for an invoice. For any calculation that matters, use a code execution tool.

```
Bad:  "Sum the line items in these 300 invoices and apply each
       customer's discount tier"
      -> Model produces a plausible total that may be silently off
         by a few rows or a rounding step

Good: Have the agent write and run code over the data
      -> Exact, repeatable, and you can inspect the code it ran
```

### 2. assuming perfect memory

The model only "remembers" what is in its current context window. If information from step 1 gets truncated by step 10, the model will not remember it.

### 3. over-relying on a single model

Using a frontier model for every step wastes money and adds latency. Use model routing to match model capability to task complexity.

### 4. ignoring the system prompt

A well-crafted system prompt can be the difference between an agent that works 50% of the time and one that works 95% of the time. Invest time in writing and iterating on your system instructions.

### 5. not accounting for hallucination

LLMs will confidently generate plausible but incorrect information. For any fact that matters, ground the agent's response in tool results rather than the model's training data.

---

## Key takeaways

1. **The LLM is the reasoning engine** of your agent. Understanding its capabilities and limitations is foundational to building good agents.

2. **Tokens, context windows, and attention** are the three key concepts. Tokens determine cost and speed. Context windows determine how much information the model can work with. Attention determines what the model focuses on.

3. **Reasoning is now built in.** Thinking models do Chain-of-Thought internally; you control how much with a thinking level rather than prompt tricks. Tree-of-Thoughts and decomposition still matter, but increasingly live in your orchestration layer.

4. **Pick the right model and thinking level for each step.** Use lighter models or lower thinking levels for simple tasks, and heavier models or higher thinking levels for complex reasoning. This reduces cost and latency.

5. **System instructions are your main lever for controlling agent behavior.** Write them carefully, be specific, and iterate based on testing. Define tools through the API, and use structured output for anything a program reads.

6. **Do not reflexively lower temperature.** For thinking models like Gemini 3.x, leave sampling at the defaults. Get consistency from clear instructions, good tool descriptions, structured output, and evals.

7. **Manage context, do not just fill it.** Large windows still suffer from context rot. Truncate, compact, and keep a stable prefix so prompt caching keeps costs down.

---

## What is next?

The brain is important, but an agent that can only think is just a chatbot. In the next lesson, we give our agent hands - tools that let it interact with the world, call APIs, search the web, and execute code.

[Next: Lesson 3 - Tools: Giving Agents Hands -->](../03-tools-giving-agents-hands/)
