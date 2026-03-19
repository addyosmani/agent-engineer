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

- **Billing**: Most APIs charge per token (input + output). Agent workflows use many more tokens than single prompts because every iteration of the loop sends the full context.
- **Speed**: More tokens = more time to generate. Keep your tool descriptions concise.
- **Context limits**: There is a maximum number of tokens the model can process in a single call. If your agent's accumulated context exceeds this, you lose information.

### Context windows: the model's working memory

The **context window** is the total number of tokens a model can consider at once. Think of it like the model's desk - everything it needs to reference must fit on this desk.

| Model | Context Window |
|---|---|
| Gemini 2.5 Pro | 1,000,000 tokens |
| Gemini 2.0 Flash | 1,000,000 tokens |
| GPT-4o | 128,000 tokens |
| Claude 3.5 Sonnet | 200,000 tokens |

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

As an agent executes multiple steps, the context grows with each tool call and result. A 5-step agent workflow might accumulate thousands of tokens of tool results. If you are not careful, you can exhaust the context window mid-task.

Strategies to manage this:
- **Summarize** intermediate results instead of keeping raw data
- **Truncate** long tool outputs to the relevant parts
- **Use models with large context windows** for complex multi-step tasks
- **Implement a sliding window** that drops older, less relevant context

### Attention: how the model focuses

The **attention mechanism** is what allows the model to figure out which parts of the context are relevant to the current decision. When deciding what token to generate next, the model assigns different weights to different parts of the input.

Think of it like reading a long document and highlighting the important parts. The model "highlights" the tokens most relevant to what it is trying to do right now.

**Why this matters for agents:**

- **Put important information where the model can find it.** Models tend to pay more attention to the beginning and end of the context. Critical instructions should go in the system prompt (beginning) or close to the user's query (end).
- **Be specific and clear.** Vague instructions force the model to guess what matters. Specific instructions make it easy for the attention mechanism to latch onto the right information.
- **Structure helps.** Clear headers, numbered lists, and consistent formatting help the model parse and attend to the right content.

---

## Reasoning strategies

How an agent "thinks" about a problem depends heavily on how you prompt the model. Different reasoning strategies produce very different results, especially for complex tasks.

### Chain-of-Thought (CoT)

**What it is:** Prompting the model to think through a problem step by step before giving a final answer.

**How it works:** Instead of jumping straight to an answer, the model generates intermediate reasoning steps. This dramatically improves accuracy on tasks that require logic, math, or multi-step analysis.

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
Gemini Flash          Gemini Pro          Gemini 2.5 Pro
(simple tasks)        (balanced)          (complex reasoning)
```

### When to use what

| Task Type | Recommended Tier | Why |
|---|---|---|
| Classification ("Is this spam?") | Light (Flash) | Simple decision, no complex reasoning needed |
| Data extraction ("Pull the date from this email") | Light (Flash) | Pattern matching, well-defined output |
| Summarization | Light to Medium | Depends on length and complexity of source |
| Multi-step reasoning | Medium to Heavy (Pro) | Requires sustained logical chains |
| Complex code generation | Heavy (2.5 Pro) | Needs deep understanding of patterns and edge cases |
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

### Google Cloud Model options

Google Cloud provides access to Gemini models through Vertex AI:

- **Gemini 2.0 Flash** - Fast and efficient for most agent tasks. Large context window (1M tokens). Good balance of capability and speed.
- **Gemini 2.5 Pro** - Top-tier reasoning for complex tasks. Use when the task requires deep analysis, complex multi-step logic, or nuanced understanding.
- **Gemini 2.0 Flash Lite** - Fastest and cheapest option for simple tasks like classification and extraction.

> **Learn more:** [Vertex AI Model Documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models)

> **Learn more:** [Gemini API Documentation](https://ai.google.dev/gemini-api/docs)

---

## The role of system instructions

System instructions are the agent's "job description." They tell the model who it is, what it can do, how it should behave, and what it should not do.

### What goes in system instructions

A well-written system instruction for an agent typically includes:

1. **Role definition**: What the agent is and who it serves
2. **Capabilities**: What tools are available and when to use them
3. **Constraints**: What the agent should not do
4. **Output format**: How to structure responses
5. **Error handling**: What to do when things go wrong
6. **Personality/tone**: How to communicate (if relevant)

### Example: customer support agent

```
You are a customer support agent for Acme Corp, an online electronics retailer.

Your role:
- Help customers with order inquiries, returns, and product questions
- Be friendly, professional, and concise

Available tools:
- lookup_order(order_id): Returns order status, items, and shipping info
- initiate_return(order_id, reason): Starts a return process
- search_products(query): Searches the product catalog
- escalate_to_human(reason): Transfers to a human agent

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
| Specify output format | Let the model choose its own format each time |
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

- **Temperature 0 (or very low)**: The model almost always picks the most likely token. Outputs are deterministic and focused.
- **Temperature 1**: The model samples proportionally to token probabilities. Outputs are more varied and creative.
- **Temperature > 1**: The model becomes increasingly random. Outputs become unpredictable.

**Visual analogy:**

```
Temperature 0:    "The capital of France is Paris."
                  (Always the same answer)

Temperature 0.7:  "The capital of France is Paris, a city known for..."
                  (Slight variation in elaboration)

Temperature 1.5:  "The capital of France is historically rooted in..."
                  (More creative, potentially off-track)
```

### What temperature to use for agents

| Agent Task | Recommended Temperature | Why |
|---|---|---|
| Tool selection | 0 - 0.2 | You want deterministic, correct tool calls |
| Data extraction | 0 | Exact answers, no creativity needed |
| Code generation | 0 - 0.3 | Correctness matters more than variety |
| Planning | 0.2 - 0.5 | Some flexibility helps explore options |
| Creative writing | 0.7 - 1.0 | Variety and originality are valued |
| Brainstorming | 0.8 - 1.0 | Want diverse ideas |

**For most agent use cases, keep temperature low (0 to 0.3).** Agents need to make reliable decisions about tool use, parameter extraction, and reasoning. High temperature introduces randomness where you want consistency.

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

**For agents:** Use conservative settings. Top-P around 0.9 and moderate Top-K values are reasonable defaults. The model's default settings are usually fine for agent work - temperature is the parameter you are most likely to want to adjust.

### How sampling affects the agent loop

Consider an agent that needs to decide which tool to call. With low temperature, it will consistently pick the same (usually correct) tool for a given situation. With high temperature, it might pick different tools on different runs, leading to inconsistent behavior.

```
User: "What is the weather in Tokyo?"

Low temperature (0):
  -> Agent thinks: "I need the weather tool"
  -> Calls: get_weather(city="Tokyo")
  -> Consistent, predictable

High temperature (1.2):
  -> Run 1: Calls get_weather(city="Tokyo")
  -> Run 2: Calls web_search("Tokyo weather forecast")
  -> Run 3: Tries to answer from training data (no tool call)
  -> Inconsistent, hard to debug
```

For production agents, deterministic behavior is almost always what you want.

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

**Temperature decisions:**

| Step | Temperature | Reasoning |
|---|---|---|
| Classify severity | 0 | Must be deterministic - same bug should always get the same severity |
| Assign to team | 0 | Must be consistent - same component should always route to the same team |
| Draft investigation plan | 0.3 | Slight flexibility helps generate more useful and varied investigation ideas |

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

This example shows how understanding the model's capabilities, setting appropriate parameters, and writing clear instructions all contribute to a reliable agent.

---

## Common mistakes when working with LLMs in agents

### 1. trusting the model for math

LLMs predict tokens, not compute equations. For any calculation that matters, use a code execution tool.

```
Bad:  "Calculate the total cost of 47 items at $23.99 each"
      -> Model might say $1,127.53 (the correct answer is $1,127.53,
         but it got lucky - it often gets these wrong)

Good: Have the agent call a calculator tool or code execution tool
      -> calculate("47 * 23.99") -> $1,127.53 (guaranteed correct)
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

3. **Reasoning strategies matter.** Chain-of-Thought, Tree-of-Thoughts, and step-by-step decomposition can dramatically improve agent performance on complex tasks.

4. **Pick the right model for each step.** Use lighter models for simple tasks and heavier models for complex reasoning. Model routing reduces cost and latency.

5. **System instructions are your main lever for controlling agent behavior.** Write them carefully, be specific, and iterate based on testing.

6. **Keep temperature low for agents.** Deterministic behavior is almost always better for production agent systems.

---

## What is next?

The brain is important, but an agent that can only think is just a chatbot. In the next lesson, we give our agent hands - tools that let it interact with the world, call APIs, search the web, and execute code.

[Next: Lesson 3 - Tools: Giving Agents Hands -->](../03-tools-giving-agents-hands/README.md)
