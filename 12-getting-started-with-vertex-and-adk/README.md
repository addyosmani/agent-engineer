# Lesson 12: getting started with Vertex AI and ADK

## Introduction

Over the first eleven lessons, we built a strong foundation in how agents work - reasoning, tools, memory, planning, multi-agent systems, RAG, evaluation, safety, and production operations. All of those concepts are platform-agnostic.

Now we shift to practice. This lesson maps those concepts to the specific tools and services available on Google Cloud. The goal is to give you a clear picture of what exists, how the pieces fit together, and which service to reach for when you are building a particular type of agent.

### ELI5: Think of the Google Cloud AI stack like a workshop

Imagine you are setting up a woodworking workshop. You need raw materials (wood, metal), power tools (saws, drills), a workbench (a stable surface to build on), safety equipment (goggles, gloves), and a space to display or sell your finished products.

The Google Cloud AI stack works similarly:

- **Gemini models** are your raw materials - the intelligence that powers everything
- **ADK (Agent Development Kit)** is your set of power tools - the framework you use to build agents
- **Vertex AI platform** is your workbench - model hosting, evaluation, and deployment infrastructure
- **Agent Engine** is a managed display room - it runs your agents in production without you managing servers
- **Model Armor** is your safety equipment - guardrails and content filtering
- **Vertex AI Search and RAG Engine** are your reference library - giving agents access to your data

You can use these pieces independently or together. Not every project needs every tool.

> **Key takeaway:** Google Cloud provides a full stack for building agents, from models to managed runtime. Understanding which piece does what helps you pick the right tool for each job.

---

## The Google Cloud AI stack for agents

Here is a map of the major components and how they relate to each other:

```
+------------------------------------------------------------------+
|                        Your Agent Application                     |
+------------------------------------------------------------------+
|                                                                    |
|  +--------------------+    +----------------------------------+   |
|  |  Agent Development |    |  Agent Engine                    |   |
|  |  Kit (ADK)         |    |  (Managed Runtime)               |   |
|  |                    |    |                                  |   |
|  |  - Build agents    |    |  - Deploy and run agents         |   |
|  |  - Define tools    |--->|  - Session management            |   |
|  |  - Orchestration   |    |  - Scaling and monitoring        |   |
|  +--------------------+    +----------------------------------+   |
|           |                            |                          |
|           v                            v                          |
|  +----------------------------------------------------+          |
|  |              Vertex AI Platform                     |          |
|  |                                                     |          |
|  |  - Model hosting (Gemini, partner models)           |          |
|  |  - Evaluation tools                                 |          |
|  |  - Context caching                                  |          |
|  |  - Grounding with Google Search                     |          |
|  +----------------------------------------------------+          |
|           |                            |                          |
|           v                            v                          |
|  +----------------------+    +------------------------+           |
|  | Vertex AI Search &   |    |  Model Armor           |           |
|  | RAG Engine           |    |  (Safety & Guardrails) |           |
|  +----------------------+    +------------------------+           |
|                                                                    |
+------------------------------------------------------------------+
|                     Gemini Models                                  |
|  Pro (complex reasoning) | Flash (balanced) | Flash-Lite (volume) |
+------------------------------------------------------------------+
```

Let's walk through each component.

---

## Gemini Models

Gemini is Google's family of multimodal AI models. For agent development, you will primarily work with three tiers:

| Model | Best For | Characteristics |
|-------|----------|----------------|
| **Gemini 2.0 Pro** | Complex reasoning, multi-step planning, nuanced decisions | Highest capability, higher latency, higher cost |
| **Gemini 2.0 Flash** | Balanced tasks - tool use, summarization, conversation | Good capability, fast, moderate cost |
| **Gemini 2.0 Flash-Lite** | High-volume, simpler tasks - classification, routing, extraction | Fast, lowest cost, good for high-throughput use cases |

> **Model Availability:** Gemini 2.0 models are currently GA (General Availability). Newer preview models may be available for experimentation but are recommended for development/testing rather than production use.

### Choosing the right model

Think of model selection like choosing the right vehicle for a trip:

- **Flash-Lite** is a bicycle - fast, cheap, great for short trips (simple classification, intent detection, basic extraction)
- **Flash** is a car - versatile, good for most journeys (general agent tasks, tool use, RAG, conversation)
- **Pro** is a truck - powerful, handles heavy loads (complex multi-step reasoning, long documents, difficult planning)

Most production agents use multiple model tiers through **model routing** (covered in Lesson 11). Use Flash-Lite for the simple steps, Flash for the core logic, and Pro only when the task genuinely requires it.

### Multimodal capabilities

Gemini models can process text, images, audio, and video. This means your agents can:

- Analyze images uploaded by users (product photos, screenshots, documents)
- Process audio inputs (voice commands, meeting recordings)
- Understand video content (tutorials, demonstrations)
- Work with PDFs and other document formats natively

This is a significant advantage over text-only models because it lets you build agents that interact with the real world in richer ways.

For model details and capabilities, see the [Gemini model documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models).

---

## Vertex AI Platform

Vertex AI is Google Cloud's machine learning platform. For agent developers, the most relevant capabilities are:

### Model hosting and API access

Vertex AI provides API access to Gemini models along with partner and open-source models. You get:

- **Managed endpoints** - no infrastructure to manage
- **Context caching** - cache long system prompts or documents to reduce cost and latency on repeated calls
- **Grounding with Google Search** - let the model verify and supplement its knowledge with web search results
- **Batch prediction** - process large volumes of requests efficiently

### Evaluation tools

Vertex AI includes evaluation capabilities that align with what we covered in Lesson 9:

- **AutoSxS (Auto Side-by-Side)** - compare two model versions automatically
- **Pointwise evaluation** - score individual responses on quality dimensions
- **Custom metrics** - define your own evaluation criteria
- **Bulk evaluation** - run evals across large datasets

These tools integrate with your CI/CD pipeline for evaluation-gated deployment (Lesson 11).

### Model garden

The Vertex AI Model Garden provides access to a wide range of models beyond Gemini - including open-source models and models from partner companies. This is useful when you need specialized models for particular tasks or want to compare different options.

For the full platform overview, see the [Vertex AI documentation](https://cloud.google.com/vertex-ai/docs).

---

## Agent Development Kit (ADK)

ADK is Google's open-source, code-first framework for building AI agents. If Vertex AI is the workbench, ADK is the set of power tools you use to actually build things.

### Key characteristics

| Feature | Detail |
|---------|--------|
| **Open source** | Available on GitHub, MIT licensed |
| **Multi-language** | Python, TypeScript/JavaScript, Go, Java |
| **Model-agnostic** | Works with Gemini, but also supports other LLMs |
| **Deployment-agnostic** | Run locally, on Agent Engine, on Cloud Run, on any container platform |
| **Opinionated but flexible** | Provides structure without locking you in |

### Why a framework?

You could build agents by calling the Gemini API directly - writing your own tool-calling loop, managing conversation state, and handling orchestration. ADK saves you from reinventing these common patterns:

- Tool registration and execution
- Conversation state management
- Multi-turn orchestration loops
- Multi-agent coordination
- Session and memory management
- Callback hooks for guardrails and logging

Think of it like the difference between writing raw HTTP handlers and using a web framework. You can do either, but the framework handles the boilerplate so you can focus on your agent's unique logic.

### ADK core concepts

ADK organizes agents into three categories:

#### 1. LLM agents

These are agents powered by a language model that can reason, plan, and decide which tools to call. This is the most common type and corresponds to the ReAct-style agents we covered in Lesson 4.

```python
from google.adk.agents import Agent
from google.adk.tools import FunctionTool

# Define a tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # In practice, this would call a weather API
    return f"The weather in {city} is sunny, 72F."

# Create an agent
weather_agent = Agent(
    name="weather_agent",
    model="gemini-2.0-flash",
    instruction="You are a helpful weather assistant. Use the get_weather "
                "tool to answer questions about weather conditions.",
    tools=[get_weather],
)
```

#### 2. workflow agents

These agents follow predefined orchestration patterns rather than relying on the LLM to decide the flow. ADK provides three built-in workflow types:

| Workflow Type | How It Works | Use When |
|--------------|-------------|----------|
| **SequentialAgent** | Runs sub-agents one after another in a fixed order | Steps must happen in sequence (e.g., validate -> process -> respond) |
| **ParallelAgent** | Runs sub-agents simultaneously | Steps are independent and can happen at the same time (e.g., search multiple sources) |
| **LoopAgent** | Runs a sub-agent repeatedly until a condition is met | Iterative refinement (e.g., generate -> evaluate -> improve) |

```python
from google.adk.agents import SequentialAgent

# A pipeline that validates input, processes it, and formats the response
pipeline = SequentialAgent(
    name="order_pipeline",
    sub_agents=[
        input_validator_agent,
        order_processor_agent,
        response_formatter_agent,
    ],
)
```

Workflow agents map directly to the agentic design patterns from Lesson 4. Sequential corresponds to pipeline patterns. Parallel corresponds to fan-out/fan-in. Loop corresponds to reflection and iterative refinement.

#### 3. custom agents

For orchestration patterns that do not fit the built-in types, you can create custom agents by subclassing the base agent class and implementing your own control flow.

### ADK tools ecosystem

One of ADK's biggest strengths is its tools ecosystem. Tools are how agents interact with the outside world (Lesson 3), and ADK provides several ways to define them:

| Tool Type | What It Is | Example |
|-----------|-----------|---------|
| **Function Tools** | Plain Python/JS functions decorated as tools | A function that queries your database |
| **MCP Tools** | Tools from Model Context Protocol servers | Connect to any MCP-compatible tool server |
| **OpenAPI Tools** | Auto-generated from OpenAPI/Swagger specs | Wrap any REST API as agent tools |
| **Built-in Tools** | Pre-built integrations provided by ADK | Google Search, code execution, RAG |

ADK includes 60+ pre-built tool integrations, covering common needs like:

- Google Search and web browsing
- Code execution (sandboxed)
- File operations
- Database queries
- API calls
- Google Workspace (Gmail, Calendar, Drive)

```python
from google.adk.tools import FunctionTool

# A simple function tool
def search_products(query: str, max_results: int = 5) -> list[dict]:
    """Search the product catalog.

    Args:
        query: The search query string.
        max_results: Maximum number of results to return.

    Returns:
        A list of matching products with name, price, and description.
    """
    # Your implementation here
    return product_database.search(query, limit=max_results)

# ADK automatically generates the tool schema from the function signature
# and docstring, so the LLM knows how to call it correctly.
```

### ADK Skills

Skills are a newer ADK concept that packages agent capabilities as self-contained, reusable units. Think of them as "plugins" for agents.

Skills have three levels of increasing complexity:

| Level | What It Includes | Example |
|-------|-----------------|---------|
| **L1 - Metadata** | Name, description, and tags that help the agent understand when to use the skill | "This skill handles flight booking" |
| **L2 - Instructions** | Detailed instructions for how the agent should use the skill | Step-by-step guide for the booking flow |
| **L3 - Resources** | Tools, data sources, and sub-agents that the skill needs | Flight search API tool, airline database |

Skills make it easier to share and compose agent capabilities across teams and projects.

For complete ADK documentation, see the [ADK docs site](https://google.github.io/adk-docs/).

---

## Agent engine

Agent Engine is a managed runtime service on Google Cloud for deploying and running agents. If ADK is how you build agents, Agent Engine is how you run them in production without managing infrastructure.

### What agent engine provides

| Capability | What It Does |
|-----------|-------------|
| **Managed hosting** | Run your agent without provisioning servers or containers |
| **Session management** | Built-in conversation state persistence |
| **Scaling** | Automatic scaling based on traffic |
| **Monitoring** | Integration with Cloud Monitoring and Cloud Logging |
| **Security** | IAM-based access control, VPC Service Controls support |

### When to use Agent Engine vs. other deployment options

| Deployment Option | Best For |
|-------------------|----------|
| **Agent Engine** | Production agents where you want managed infrastructure and do not want to handle scaling, session management, or deployment yourself |
| **Cloud Run** | Agents that need custom runtime environments, specific dependencies, or more control over the container |
| **GKE (Kubernetes)** | Agents that are part of a larger microservices architecture already running on Kubernetes |
| **Local / Self-hosted** | Development, testing, or when you cannot use cloud services |

ADK agents can be deployed to any of these targets. Agent Engine is the most managed option - you give it your agent code and it handles the rest.

For deployment details, see the [Agent Engine documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview).

---

## Vertex AI Search And RAG engine

In Lesson 8, we covered how agents can use retrieval-augmented generation (RAG) to access your data. Vertex AI provides managed services for this:

### Vertex AI Search

A managed search service that can index and search across:

- Websites
- Unstructured documents (PDFs, Word docs, HTML)
- Structured data (databases, spreadsheets)

It handles chunking, embedding, indexing, and retrieval - the full RAG pipeline we discussed in Lesson 8 - as a managed service.

### RAG Engine

RAG Engine provides a managed retrieval pipeline specifically designed for grounding LLM responses in your data:

- **Document ingestion** - upload and process documents automatically
- **Chunking strategies** - configurable approaches to splitting documents
- **Vector search** - managed embedding and similarity search
- **Integration with Gemini** - built-in grounding for Gemini model calls

The advantage of these managed services is that you do not have to run your own vector database, manage embeddings, or build retrieval pipelines. The tradeoff is less control over the details.

For RAG capabilities, see the [RAG Engine overview](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview).

---

## Model Armor

Model Armor is Google Cloud's managed guardrails service (we covered guardrails in depth in Lesson 10). It provides:

| Feature | What It Does |
|---------|-------------|
| **Prompt screening** | Detect and block harmful or adversarial prompts before they reach the model |
| **Response filtering** | Screen model outputs for harmful, toxic, or inappropriate content |
| **Prompt injection detection** | Identify attempts to override model instructions |
| **Configurable policies** | Set your own thresholds for different content categories |
| **Integration** | Works with Vertex AI endpoints and can be added to any generative AI application |

Model Armor gives you a production-ready Layer 2 defense (from the defense-in-depth model in Lesson 10) without building content filtering from scratch.

---

## Quick setup guide

Here is how to get started with ADK and Google Cloud for agent development.

### Prerequisites

- A Google Cloud account (you can start with the [free tier](https://cloud.google.com/free))
- Python 3.9+ (for the Python SDK)
- A Google Cloud project with billing enabled

### Step 1: install ADK

```bash
pip install google-adk
```

### Step 2: configure authentication

You have two options for authenticating:

**Option A: API Key (simplest for getting started)**
```bash
export GOOGLE_API_KEY="your-api-key-here"
```

You can get an API key from [Google AI Studio](https://aistudio.google.com/).

**Option B: Google Cloud project (for production and Vertex AI features)**
```bash
# Install the Google Cloud CLI
# https://cloud.google.com/sdk/docs/install

# Authenticate
gcloud auth application-default login

# Set your project
gcloud config set project YOUR_PROJECT_ID
```

### Step 3: Create your first agent

Create a file called `agent.py`:

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="greeting_agent",
    model="gemini-2.0-flash",
    instruction="You are a friendly assistant that greets users "
                "and answers basic questions.",
)
```

### Step 4: run it locally

```bash
adk web
```

This starts a local web interface where you can chat with your agent and inspect its behavior. The ADK dev UI shows you the agent's reasoning steps, tool calls, and state - which is invaluable for debugging.

### Step 5: Add tools and complexity

From here, you can add tools, create multi-agent systems, integrate RAG, and eventually deploy to Agent Engine or Cloud Run. The [ADK getting started guide](https://google.github.io/adk-docs/get-started/) walks through these steps in detail.

---

## Decision tree: which service do i use?

When building an agent, use this decision tree to pick the right Google Cloud services:

```
"I want to build..."
    |
    +-- "A simple chatbot with no tools"
    |       --> Gemini API directly (no framework needed)
    |
    +-- "An agent with tools and reasoning"
    |       --> ADK + Gemini Flash
    |       |
    |       +-- "...and I need it in production"
    |               --> Deploy to Agent Engine or Cloud Run
    |
    +-- "An agent that searches my documents"
    |       --> ADK + Vertex AI Search or RAG Engine
    |
    +-- "A multi-agent system"
    |       --> ADK (multi-agent orchestration built in)
    |
    +-- "An agent with strict safety requirements"
    |       --> ADK + Model Armor + custom guardrails
    |
    +-- "A high-volume, cost-sensitive application"
    |       --> Model routing (Flash-Lite for simple tasks,
    |           Flash for complex) + context caching
    |
    +-- "An agent that needs to use external APIs"
            --> ADK with OpenAPI tools or MCP tools
```

### Quick reference table

| I Need... | Use... |
|-----------|--------|
| An LLM to call | Gemini models via Vertex AI or AI Studio |
| A framework to build agents | Agent Development Kit (ADK) |
| Managed agent hosting | Agent Engine |
| Custom container hosting | Cloud Run or GKE |
| Document search / RAG | Vertex AI Search or RAG Engine |
| Content safety guardrails | Model Armor |
| Model evaluation | Vertex AI Evaluation tools |
| Prompt management | Vertex AI prompt management |
| Interop with external tools | MCP tools in ADK |
| Interop with other agents | A2A protocol (covered in Lesson 14) |

---

## How the pieces connect: a full example

Here is how a typical production agent uses multiple Google Cloud services together:

```
User asks: "What is the return policy for my recent order?"

1. [ADK Agent] receives the request
       |
2. [Model Armor] screens the input for safety
       |
3. [Gemini Flash] reasons about the request:
       "I need to look up the order and find the return policy"
       |
4. [ADK Tool: Order Lookup] calls your order database
       |
5. [RAG Engine] searches your policy documents
       for the relevant return policy
       |
6. [Gemini Flash] synthesizes a response from
       the order details and policy documents
       |
7. [Model Armor] screens the output for safety
       |
8. [Agent Engine] manages the session state
       and returns the response to the user
```

Each Google Cloud service handles one part of the puzzle. ADK orchestrates the flow. Gemini provides the reasoning. RAG Engine provides the knowledge. Model Armor provides the safety. Agent Engine provides the runtime.

---

## Where each lesson concept maps to Google Cloud

Here is a reference connecting the concepts from earlier lessons to specific Google Cloud services:

| Lesson | Concept | Google Cloud Service |
|--------|---------|---------------------|
| 2 - How Agents Think | LLM reasoning | Gemini models |
| 3 - Tools | Function calling | ADK Function Tools, MCP Tools, OpenAPI Tools |
| 4 - Design Patterns | Orchestration | ADK Sequential/Parallel/Loop Agents |
| 5 - Memory | Session state, long-term memory | ADK session management, Agent Engine |
| 6 - Planning | Multi-step reasoning | Gemini 2.0 Pro for complex planning |
| 7 - Multi-Agent | Agent coordination | ADK multi-agent support |
| 8 - RAG | Knowledge retrieval | Vertex AI Search, RAG Engine |
| 9 - Evaluation | Testing agents | Vertex AI Evaluation |
| 10 - Safety | Guardrails | Model Armor |
| 11 - Production | Deployment, CI/CD | Agent Engine, Cloud Run, Agent Starter Pack |

---

## Key takeaways

1. **Google Cloud provides a full stack for agents.** From Gemini models at the base to Agent Engine at the top, you can build and deploy complete agent systems.

2. **ADK is the code-first framework.** It is open-source, multi-language, model-agnostic, and deployment-agnostic. It handles the common patterns (tool calling, orchestration, state management) so you can focus on your agent's logic.

3. **Pick the right model tier.** Use Flash-Lite for simple tasks, Flash for most agent work, and Pro for complex reasoning. Model routing across tiers is a key cost optimization strategy.

4. **Managed services reduce operational burden.** Agent Engine, RAG Engine, and Model Armor handle infrastructure and operations so you can focus on building. The tradeoff is less control over implementation details.

5. **Everything is composable.** You can use ADK without Agent Engine, use Vertex AI without ADK, or use the full stack together. Start with what you need and add services as your requirements grow.

6. **ADK agents map directly to the concepts in this course.** LLM Agents for reasoning, Workflow Agents for orchestration patterns, tools for external interaction, and skills for reusable capabilities.

---

## Further reading

- [ADK Documentation](https://google.github.io/adk-docs/) - Complete guide to building agents with ADK
- [Vertex AI Documentation](https://cloud.google.com/vertex-ai/docs) - The full Vertex AI platform reference
- [Agent Engine Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) - Managed runtime for agents
- [RAG Engine Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview) - Managed retrieval-augmented generation
- [Agent Starter Pack](https://github.com/GoogleCloudPlatform/agent-starter-pack) - Production-ready templates with CI/CD and observability built in
- [Google AI Studio](https://aistudio.google.com/) - Get API keys and experiment with Gemini models

---

Next lesson: [Building Your First Agent](../13-building-your-first-agent/README.md)
