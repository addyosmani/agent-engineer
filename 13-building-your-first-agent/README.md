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

### Prerequisites

Before you start, make sure you have:

- **Python 3.9+** installed
- **ADK installed** (`pip install google-adk`)
- **A Google Cloud project** with the Gemini API enabled
- **An API key** or application default credentials configured

> **Get set up:** Follow the official setup guide at [ADK Getting Started](https://google.github.io/adk-docs/get-started/) for detailed installation instructions.

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
    model="gemini-2.0-flash",
    instruction="You are a helpful assistant that answers questions clearly and concisely.",
)
```

The `model` parameter determines which Gemini model handles the reasoning. For learning and prototyping, `gemini-2.0-flash` is a great choice - it is fast and cost-effective. For more complex reasoning tasks, you might upgrade to `gemini-2.0-pro`.

> **Note:** Gemini 2.0 models are currently in General Availability (GA). Newer preview models may be available but are recommended for experimentation rather than production use.

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
    model="gemini-2.0-flash",
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
    model="gemini-2.0-flash",
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

> **Full quickstart:** Follow along with the complete code at [ADK Quickstart](https://google.github.io/adk-docs/get-started/quickstart/)

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
    model="gemini-2.0-flash",
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

> **Learn more:** [ADK Agent Types](https://google.github.io/adk-docs/agents/)

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
    model="gemini-2.0-flash",
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
    model="gemini-2.0-flash",
    instruction="You research topics using web search. Return factual findings.",
    tools=[google_search],
)

writer = Agent(
    name="writer",
    model="gemini-2.0-flash",
    instruction="You write clear, engaging content based on research findings.",
)

fact_checker = Agent(
    name="fact_checker",
    model="gemini-2.0-flash",
    instruction="You verify claims by searching for supporting evidence.",
    tools=[google_search],
)

# Root agent that orchestrates
coordinator = Agent(
    name="content_team",
    model="gemini-2.0-pro",
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
| `adk deploy` | Deploy your agent to Agent Engine |

---

## Where to learn more

- **Getting started with ADK:** [https://google.github.io/adk-docs/get-started/](https://google.github.io/adk-docs/get-started/)
- **Full quickstart tutorial:** [https://google.github.io/adk-docs/get-started/quickstart/](https://google.github.io/adk-docs/get-started/quickstart/)
- **Agent types in depth:** [https://google.github.io/adk-docs/agents/](https://google.github.io/adk-docs/agents/)
- **Tools reference:** [https://google.github.io/adk-docs/tools/](https://google.github.io/adk-docs/tools/)

---

## Key takeaways

1. **An ADK agent needs three things:** a name, a model, and system instructions. Tools are optional but are what make agents genuinely useful.

2. **Start with LlmAgent.** It handles most use cases. Reach for SequentialAgent, ParallelAgent, or LoopAgent only when you have a clear structural reason.

3. **System instructions are your primary control lever.** Be specific about the role, set boundaries, provide examples, and include error handling guidance.

4. **Test early and often.** Use `adk web` for interactive testing and `adk eval` for automated regression checks.

5. **Build incrementally.** One agent, one tool, one eval case. Get each piece working before adding the next.

---

## What is next?

Now that you can build an agent, you need to understand how agents communicate with the wider world. In the next lesson, we will explore two important protocols - MCP and A2A - that let agents talk to tools and to each other using open standards.

[Next: Lesson 14 - Agent Protocols: MCP and A2A -->](../14-agent-protocols-mcp-and-a2a/README.md)
