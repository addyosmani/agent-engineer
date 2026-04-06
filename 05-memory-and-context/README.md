# Lesson 5: memory and context - how agents remember

## What you will learn

- How the context window works and why it matters
- Context engineering as the evolution from prompt engineering
- Types of agent memory: short-term, long-term, procedural, and declarative
- How sessions give agents conversational continuity
- The difference between memory and RAG
- What context rot is and how to fight it
- Practical strategies for managing context
- Memory storage options: vector databases, knowledge graphs, and hybrid approaches

## Prerequisites

- [Lesson 2: How Agents Think](../02-how-agents-think/README.md)
- [Lesson 4: Agentic Design Patterns](../04-agentic-design-patterns/README.md)

---

## ELI5: Think of context like a desk

Imagine you are working at a desk. The desk is your agent's context window - it is where you put all the stuff you are actively working with.

The desk has limited space. You can spread out notes, reference books, your laptop, and a cup of coffee. But if you keep piling things on, eventually the desk gets so cluttered that you cannot find anything. Important notes get buried under less relevant papers. You start losing track of what you were doing.

That is exactly what happens with an AI agent's context. The context window is the agent's working surface. Everything the agent needs to think about - the user's question, conversation history, tool results, instructions - has to fit on this desk. When the desk gets full, the agent has to decide what stays and what goes.

Good memory management is like being organized at your desk: keeping the important stuff visible and accessible while storing less critical information somewhere you can find it later.

---

## The context window explained

### What is a context window?

The context window is the total amount of text (measured in tokens) that an LLM can process in a single request. Think of it as the model's working memory - everything it can "see" at once.

```
+------------------------------------------------------------------+
|                        CONTEXT WINDOW                            |
|                                                                  |
|  [System instructions]                                           |
|  [Tool definitions]                                              |
|  [Conversation history - user and agent messages]                |
|  [Retrieved documents / RAG results]                             |
|  [Current user message]                                          |
|  [Agent's working thoughts]                                      |
|                                                                  |
|                    ... all must fit here ...                      |
+------------------------------------------------------------------+
```

### Context window sizes

Context windows have grown dramatically:

| Model | Context window |
|-------|---------------|
| GPT-3 (2020) | 4K tokens |
| GPT-4 (2023) | 128K tokens |
| Gemini 1.5 Pro | 1M tokens |
| Gemini 2.0 | 1M+ tokens |

A token is roughly 3/4 of a word in English. So 1 million tokens is approximately 750,000 words - that is about 10 novels worth of text.

### Bigger is not always better

You might think a bigger context window solves all problems. It helps, but there are trade-offs:

| Factor | Small context | Large context |
|--------|--------------|---------------|
| **Cost** | Less expensive per request | More expensive per request |
| **Latency** | Faster responses | Slower responses |
| **Accuracy** | Focused, less noise | May struggle to find relevant info in a sea of text |
| **Simplicity** | Forces you to be selective | Tempting to dump everything in |

Research has shown that LLMs can struggle with information placed in the middle of very long contexts - sometimes called the "lost in the middle" effect. Just because you *can* fit 1M tokens does not mean you *should*.

---

## From prompt engineering to context engineering

### Prompt engineering: the starting point

Prompt engineering is about crafting the right input text to get a good output from an LLM. You write a static prompt, maybe with a few examples, and send it off.

This works well for simple tasks, but it falls apart for agents. Why? Because agents deal with dynamic, changing information:

- Conversation history grows with every turn
- Tool results come back at runtime
- Retrieved documents vary by query
- The user's needs evolve mid-session

### Context engineering: the next step

Context engineering is the practice of dynamically assembling the right information into the context window at the right time. Instead of a static prompt, you are building a system that decides:

- **What goes in:** Which pieces of information are relevant right now?
- **What stays out:** What can be summarized, stored externally, or dropped?
- **In what order:** How should information be arranged for the model to process it best?
- **When to refresh:** When does old context need to be updated or replaced?

Think of prompt engineering as writing a good email. Context engineering is building the mail system.

### Why context engineering matters for agents

```
Prompt Engineering:            Context Engineering:
+------------------+          +---------------------------+
| Static prompt    |          | System instructions       |
| + user question  |          | + Relevant memory         |
|                  |          | + Session history (pruned) |
|                  |          | + Tool schemas (selected)  |
|                  |          | + Retrieved context        |
|                  |          | + Current task state       |
|                  |          |                           |
| "Answer this"    |          | All dynamically assembled |
+------------------+          +---------------------------+
```

An agent making a hotel booking needs different context than one writing code. Even the same agent needs different context at different stages of a task. Context engineering is what makes this dynamic assembly possible.

---

## Types of memory

Agents need different types of memory for different purposes. This mirrors how human memory works.

### Short-term memory (session/conversation)

**What it is:** The current conversation between the user and the agent. This lives directly in the context window.

**Human analogy:** Your working memory - what you are actively thinking about right now.

**Characteristics:**
- Exists for the duration of a session
- Grows with each interaction
- Limited by context window size
- Lost when the session ends (unless persisted)

**Example:**
```
User: "My name is Alex and I prefer dark mode."
Agent: "Got it, Alex. I will use dark mode settings."

[50 messages later]

User: "What theme am I using?"
Agent: "You are using dark mode, Alex."  <-- Only works if the earlier
                                             message is still in context
```

### Long-term memory (persisted across sessions)

**What it is:** Information stored outside the context window that persists between sessions. The agent retrieves it when relevant.

**Human analogy:** Your long-term memories - facts you have learned and experiences you remember, even if you are not thinking about them right now.

**Characteristics:**
- Survives across sessions
- Stored externally (database, file system, vector store)
- Must be explicitly retrieved and loaded into context
- Can grow without limit

**Example uses:**
- User preferences ("Alex likes dark mode and uses TypeScript")
- Past interactions ("Last week, Alex asked about deploying to Cloud Run")
- Learned facts ("The production database is on us-central1")

### Procedural memory (how to do things)

**What it is:** Knowledge about *how* to perform tasks - workflows, standard operating procedures, and step-by-step processes.

**Human analogy:** Knowing how to ride a bike or type on a keyboard. You do not think about each step - you just know how.

**Characteristics:**
- Often encoded in system instructions or tool definitions
- Relatively stable - does not change often
- Includes patterns, templates, and standard procedures

**Example:**
```
Procedural memory: "When a user reports a bug:
  1. Ask for reproduction steps
  2. Check the error logs
  3. Search for similar issues
  4. Propose a fix or workaround"
```

### Declarative memory (facts and knowledge)

**What it is:** Factual information the agent knows or has access to - data, documents, specifications, and reference material.

**Human analogy:** Knowing that Paris is the capital of France. A fact you can state, not a skill you perform.

**Characteristics:**
- Can be retrieved dynamically via RAG
- Includes documentation, databases, and knowledge bases
- May become stale and need updating

### Summary of memory types

| Memory type | Duration | Location | Updated how | Example |
|------------|----------|----------|-------------|---------|
| **Short-term** | One session | Context window | Automatically (conversation grows) | Current chat messages |
| **Long-term** | Across sessions | External storage | Explicitly by the agent or system | User preferences |
| **Procedural** | Permanent | System prompt / config | By developers | Workflow instructions |
| **Declarative** | Varies | Knowledge bases / RAG | By data pipelines | Product documentation |

---

## Sessions: containers for conversations

A session is the container that holds a conversation between a user and an agent. Sessions give agents continuity - the ability to remember what has happened so far in this interaction.

### What is in a session?

```
Session
+-----------------------------------------------+
| Session ID: abc-123                           |
| User ID: user-456                             |
| Created: 2025-01-15T10:30:00Z                |
|                                               |
| Events:                                       |
|   [User message: "Help me plan a trip"]       |
|   [Agent message: "Where would you like..."]  |
|   [User message: "Tokyo, 5 days"]             |
|   [Tool call: flight_search(dest="Tokyo")]    |
|   [Tool result: {flights: [...]}]             |
|   [Agent message: "I found several..."]       |
|                                               |
| State:                                        |
|   destination: "Tokyo"                        |
|   duration: "5 days"                          |
|   budget: null                                |
+-----------------------------------------------+
```

### Sessions vs. state

- **Session:** The full history of events (messages, tool calls, results) in a conversation.
- **State:** A structured summary of key information extracted from the session. Think of it as the agent's notepad.

State is useful because it gives the agent quick access to important facts without having to re-read the entire conversation history.

### Session management in Google Cloud

Google Cloud provides session management through the Agent Development Kit (ADK) and Vertex AI:

- **ADK Sessions:** The [ADK session system](https://google.github.io/adk-docs/sessions/) provides built-in session management with event tracking and state.
- **Vertex AI Sessions:** [Vertex AI Agent Engine sessions](https://cloud.google.com/agent-builder/agent-engine/sessions/overview) offer managed session storage with automatic scaling.

These handle the infrastructure of storing and retrieving sessions, so you can focus on the agent logic.

---

## Memory vs. RAG

Memory and Retrieval-Augmented Generation (RAG) are related but serve different purposes. Engineers often confuse them.

### The personal assistant vs. the library

**Memory** is like having a personal assistant who remembers your preferences, your schedule, and your past conversations. They know *you*.

**RAG** is like having access to a library. When you need a fact, you look it up. The library does not know you personally - it just has a lot of information available.

### Side-by-side comparison

| Aspect | Memory | RAG |
|--------|--------|-----|
| **What it stores** | User-specific, interaction-specific data | General knowledge, documents, data |
| **Who it is about** | This user, this agent, this context | Anyone - it is shared knowledge |
| **When it is written** | During agent interactions | During data ingestion (usually offline) |
| **When it is read** | At the start of or during a session | When the agent needs specific information |
| **Personalization** | High - unique to the user | Low - same for everyone |
| **Example** | "This user prefers concise answers" | "The API rate limit is 100 requests/minute" |

### How they work together

In practice, agents use both:

```
User: "Remind me, what was that deployment issue we had last month?"

Agent:
  1. Check MEMORY: "This user works on the payments team and
     had a Cloud Run deployment failure on Jan 3."
  2. Use RAG: Search incident reports for "Cloud Run deployment
     failure January" to get specific details.
  3. Combine: "Last month you had a Cloud Run deployment failure
     related to a misconfigured service account. Here are the
     details from the incident report..."
```

Memory tells the agent *what to look for*. RAG provides the *detailed information*.

We will cover RAG in depth in [Lesson 8: Agentic RAG](../08-agentic-rag/README.md).

---

## Context rot

### What is context rot?

Context rot happens when critical information gets lost, diluted, or buried as the context window fills up. The agent "forgets" important things - not because the information was deleted, but because it is no longer in the part of the context the model pays attention to.

### The cluttered desk analogy

Remember the desk analogy? Context rot is what happens when your desk gets so cluttered that the critical sticky note with the database password gets buried under a pile of meeting notes. The note is technically still on the desk, but you cannot find it when you need it.

### How context rot happens

1. **Long conversations.** Each turn adds messages to the context. After 50+ turns, early messages are far from where the model focuses.

2. **Verbose tool results.** A tool returns a large JSON blob. Most of it is irrelevant, but it takes up valuable context space.

3. **Accumulated instructions.** System instructions, few-shot examples, and guardrails take up space that could hold user-relevant information.

4. **Repetitive content.** Similar messages or results pile up without being consolidated.

### Signs of context rot

- The agent "forgets" things the user told it earlier in the conversation
- The agent contradicts earlier statements or decisions
- Tool results from early in the session are ignored
- The agent asks for information the user already provided

---

## Strategies for managing context

### 1. Sliding window

**How it works:** Keep only the most recent N messages in the context. Older messages are dropped.

```
Window size: 10 messages

Messages 1-5:   [dropped]
Messages 6-15:  [in context]
Message 16:     [new message arrives, message 6 gets dropped]
```

**Pros:** Simple to implement, predictable memory usage.

**Cons:** Loses important early context. The user might reference something from message 2 that is no longer in the window.

**Best for:** Casual conversational agents where recent context matters most.

### 2. Summarization

**How it works:** Periodically summarize older conversation turns and replace them with the summary. The summary takes up much less space than the original messages.

```
Before summarization:
  [20 detailed messages about trip planning]  -> 4,000 tokens

After summarization:
  [Summary: "User is planning a 5-day trip to Tokyo with a budget
   of $3,000. They prefer boutique hotels and want to visit temples
   and try local food. Flights are booked for March 15-20."]  -> 200 tokens
```

**Pros:** Preserves key information while saving space.

**Cons:** Summarization can lose nuance. The agent performing the summary might miss something important.

**Best for:** Long-running sessions where historical context matters.

### 3. Token-based truncation

**How it works:** Set a token budget for each section of the context (system instructions, conversation history, tool results) and truncate when any section exceeds its budget.

```
Total budget: 32,000 tokens
  System instructions:     4,000 tokens (fixed)
  Tool definitions:        2,000 tokens (fixed)
  Conversation history:   20,000 tokens (sliding)
  Current turn:            6,000 tokens (reserved)
```

**Pros:** Fine-grained control over context allocation. Ensures space is always available for the current task.

**Cons:** Requires careful tuning. Hard boundaries can cut off mid-message.

**Best for:** Production agents where you need predictable costs and latency.

### 4. Importance-based selection

**How it works:** Score each piece of context by relevance to the current task and keep only the most relevant items.

```
Current task: "Book a hotel in Tokyo"

High relevance (keep):
  - User's destination: Tokyo
  - User's dates: March 15-20
  - User's budget: $3,000
  - User's hotel preferences: boutique

Low relevance (drop):
  - Discussion about restaurant recommendations
  - Earlier conversation about flight options (already booked)
```

**Pros:** Maximizes the signal-to-noise ratio in context.

**Cons:** Relevance scoring is imperfect. The agent might drop something that turns out to be important later.

**Best for:** Complex agents with many types of context competing for space.

### 5. Externalize and retrieve

**How it works:** Store information in external memory (a database, vector store, or knowledge graph) and retrieve it only when needed.

```
Instead of keeping everything in context:

  User preferences -> stored in user profile database
  Past conversations -> stored in conversation archive
  Reference docs -> stored in vector database

When needed:
  Agent queries the relevant store and loads just the
  pieces it needs into the current context.
```

**Pros:** Virtually unlimited storage. Only relevant information enters the context.

**Cons:** Adds latency for retrieval. Retrieval quality depends on the search system.

**Best for:** Agents that need access to large amounts of information but only use a small fraction at any time.

### Comparison of strategies

| Strategy | Context savings | Information loss risk | Complexity | Latency impact |
|----------|----------------|----------------------|------------|---------------|
| Sliding window | High | High | Low | None |
| Summarization | High | Medium | Medium | Some (summarization step) |
| Token truncation | Medium | Medium | Medium | None |
| Importance selection | High | Medium | High | Some (scoring step) |
| Externalize + retrieve | Very high | Low | High | Higher (retrieval step) |

---

## Memory storage options

When you externalize memory, you need somewhere to put it. Here are the main options.

### Vector databases

**What they do:** Store information as mathematical vectors (embeddings) and retrieve it by similarity.

**How it works:**
1. Convert text into a vector using an embedding model
2. Store the vector alongside the original text
3. When searching, convert the query into a vector
4. Find the stored vectors most similar to the query vector

**Good for:** Finding semantically similar content. "What did we discuss about deployment?" will match past conversations about deploying, even if they used different words.

**Examples:** Vertex AI Vector Search, Pinecone, Weaviate, ChromaDB

**Trade-offs:**
- Great at semantic similarity search
- Less effective for exact matching or structured queries
- Embedding quality affects retrieval quality

### Knowledge graphs

**What they do:** Store information as entities and relationships in a graph structure.

**How it works:**
```
[User: Alex] --works_on--> [Project: Payments API]
[Project: Payments API] --deployed_on--> [Platform: Cloud Run]
[User: Alex] --prefers--> [Theme: Dark Mode]
```

**Good for:** Representing structured relationships between entities. "Who works on the Payments API?" or "What platform is the Payments API deployed on?"

**Examples:** Neo4j, Amazon Neptune, Google Cloud's Knowledge Graph

**Trade-offs:**
- Excellent for relationship queries
- Requires schema design and maintenance
- Less natural for unstructured text

### Hybrid approaches

In practice, many systems combine both:

- **Vector database** for unstructured memory (conversations, documents, notes)
- **Knowledge graph** for structured memory (relationships, facts, preferences)
- **Key-value store** for session state and quick lookups

```
"What does Alex prefer?"
  -> Key-value store: {theme: "dark", language: "TypeScript"}

"What did Alex and I discuss about deployments?"
  -> Vector search: [similar past conversations about deployments]

"What services does Alex's team own?"
  -> Knowledge graph: Alex -> team -> services -> dependencies
```

### Choosing a storage approach

| Need | Best approach |
|------|--------------|
| Semantic search over conversations | Vector database |
| User preferences and settings | Key-value store |
| Entity relationships | Knowledge graph |
| Recent session history | In-memory / session store |
| Document retrieval | Vector database + metadata filters |
| Complex multi-hop queries | Knowledge graph |

---

## Putting it all together

Here is how memory and context fit into an agent architecture:

```
                         +-------------------+
                         |   User Message    |
                         +--------+----------+
                                  |
                                  v
                    +-------------+-------------+
                    |   Context Engineering     |
                    |   Layer                   |
                    |                           |
                    |  1. Load system prompt    |
                    |  2. Retrieve relevant     |
                    |     memory                |
                    |  3. Load session history  |
                    |     (pruned/summarized)   |
                    |  4. Add tool definitions  |
                    |  5. Include current       |
                    |     message               |
                    +-------------+-------------+
                                  |
                         Assembled context
                                  |
                                  v
                         +--------+----------+
                         |       LLM         |
                         +--------+----------+
                                  |
                          Agent response
                                  |
                                  v
                    +-------------+-------------+
                    |   Memory Update Layer     |
                    |                           |
                    |  1. Save to session       |
                    |  2. Extract key facts     |
                    |     for long-term memory  |
                    |  3. Update user profile   |
                    +---------------------------+
```

The context engineering layer assembles the right context *before* the LLM sees it. The memory update layer extracts important information *after* the LLM responds. Together, they give the agent the ability to remember and stay focused.

---

## Key takeaways

1. **The context window is the agent's working memory.** Everything the agent considers has to fit in this space. Managing it well is critical.

2. **Context engineering goes beyond prompt engineering.** Agents need dynamic assembly of context, not just static prompts. What goes into the context window should change based on the situation.

3. **Agents need multiple types of memory.** Short-term for the current conversation, long-term for persistence across sessions, procedural for knowing how to do things, and declarative for facts.

4. **Sessions provide conversational continuity.** They track the history and state of an interaction, giving agents the ability to maintain coherent conversations.

5. **Memory and RAG serve different purposes.** Memory is personal and interaction-specific. RAG is about accessing general knowledge. Most agents need both.

6. **Context rot is a real problem.** As context grows, important information gets buried. Active management through summarization, pruning, and external storage keeps the agent effective.

7. **Choose your storage based on your access patterns.** Vector databases for semantic search, knowledge graphs for relationships, key-value stores for quick lookups. Hybrid approaches often work best.

---

## Further reading

- [ADK Sessions documentation](https://google.github.io/adk-docs/sessions/)
- [Vertex AI Agent Engine - Manage Sessions](https://cloud.google.com/agent-builder/agent-engine/sessions/overview)
- [Vertex AI Vector Search](https://cloud.google.com/vertex-ai/docs/vector-search/overview)
- [Google Cloud AI codelabs](https://codelabs.developers.google.com/?cat=AI)

---

**Next lesson:** [Planning and Reasoning - How Agents Tackle Complex Tasks](../06-planning-and-reasoning/README.md)
