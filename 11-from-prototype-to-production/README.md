# Lesson 11: from prototype to production - shipping your agent

## Introduction

You have built a working agent. It handles your test cases, impresses your team in demos, and feels like magic. Now you need to ship it to real users. This is where things get hard.

The gap between "works on my laptop" and "works reliably for thousands of users" is enormous. In traditional software, production readiness mostly means handling edge cases, scaling, and monitoring. With agents, you have all of that plus the fundamental challenge that your system's behavior is non-deterministic and hard to fully predict.

### ELI5: Taking an agent to production is like opening a restaurant

You have been cooking great meals at home for your friends. Everyone loves your food. Now you want to open a restaurant. Suddenly you need to think about things that never mattered at home: health inspections, supply chains, consistent recipes so every dish tastes the same, training staff, handling complaints, managing costs, and making sure the kitchen does not catch fire on a busy Saturday night. The cooking skill is the same, but everything around it changes completely.

That is the prototype-to-production gap. Your agent's core logic might not change much, but everything around it - evaluation, deployment, monitoring, cost management, team processes - needs to be built from scratch.

> **Key takeaway:** The "last mile" from prototype to production is often 80% of the total effort. Plan for it from the start.

---

## The production gap

Here is what changes when you move from prototype to production:

| Dimension | Prototype | Production |
|-----------|-----------|------------|
| **Users** | You and your team | Hundreds or thousands of real users |
| **Inputs** | Curated test cases | Anything anyone types, including adversarial input |
| **Uptime** | Restart when it breaks | Must be available 24/7 with graceful degradation |
| **Latency** | "It takes a few seconds" is fine | Users expect sub-second responses for simple queries |
| **Cost** | Burn rate does not matter | Every token costs money at scale |
| **Quality** | "Usually works" is acceptable | Consistent quality is required; bad responses erode trust |
| **Safety** | Informal testing | Systematic guardrails, monitoring, and incident response |
| **Debugging** | Print statements | Structured logs, traces, and metrics |
| **Updates** | Edit and restart | CI/CD pipeline with evaluation gates |

### Why demos fool us

Demos work because the person giving the demo knows what inputs work well. They avoid edge cases. They retry failures off-screen. They pick the best example from multiple runs.

Production is the opposite. Real users will:
- Misspell things, use slang, write in unexpected languages
- Ask questions your agent was never designed to handle
- Provide extremely long or extremely short inputs
- Try the exact same query many times if they are unhappy with the result
- Discover failure modes you never imagined

This is why evaluation-gated deployment is so important. You should not ship an agent version that has not been tested against a comprehensive set of real-world scenarios.

---

## Team roles: who is involved

Productionizing an agent is not a solo effort. It typically involves several roles working together:

| Role | Responsibility |
|------|---------------|
| **AI Engineer** | Agent logic, prompt design, tool integration, eval creation |
| **Platform Engineer** | Infrastructure, deployment pipelines, service mesh, scaling |
| **Data Engineer** | Data pipelines for RAG, knowledge bases, training data management |
| **ML Ops / AI Ops** | Model serving, versioning, A/B testing, monitoring dashboards |
| **DevOps / SRE** | Reliability, incident response, alerting, cost tracking |
| **Product Manager** | User requirements, success metrics, prioritization |
| **Security / Trust & Safety** | Guardrails, red teaming, compliance, safety reviews |

In smaller teams, one person might wear multiple hats. But the responsibilities still exist regardless of how many people share them.

---

## Evaluation-Gated Deployment

This is the single most important practice for shipping agents safely. The principle is simple: **no agent version ships without passing evals.**

In Lesson 9, we covered how to build evals. Here is how they fit into the deployment process:

```
Code Change --> Evals Pass? --No--> Fix and retry
                    |
                   Yes
                    |
                    v
              Deploy to staging
                    |
                    v
            Staging evals pass? --No--> Fix and retry
                    |
                   Yes
                    |
                    v
            Deploy to production (canary)
                    |
                    v
          Production metrics OK? --No--> Rollback
                    |
                   Yes
                    |
                    v
            Full production rollout
```

### What makes a good eval gate?

Your eval suite for deployment should cover:

| Category | What to Test | Pass Criteria |
|----------|-------------|---------------|
| **Functional correctness** | Does the agent produce correct answers? | >= threshold on accuracy metrics |
| **Tool usage** | Does the agent call the right tools with correct arguments? | Tools called correctly in >= X% of cases |
| **Safety** | Does the agent resist prompt injection and follow policies? | 100% pass rate on safety-critical cases |
| **Latency** | Does the agent respond within acceptable time? | P95 latency < target |
| **Cost** | Does the agent stay within token budgets? | Average cost per interaction < budget |
| **Regression** | Do previously passing cases still pass? | No regressions on known-good cases |

Safety evals should have a hard gate - any failure blocks deployment. Other categories might have softer thresholds where you accept small regressions if overall quality improves.

---

## CI/CD for agents

Continuous integration and continuous deployment for agents follows the same principles as traditional CI/CD but with agent-specific steps. Think of it in three phases.

### Phase 1: Pre-merge (on every pull request)

These checks run quickly and catch obvious problems before code is merged.

```yaml
# Example: Pre-merge checks
pre_merge:
  - lint:
      - Check prompt formatting and syntax
      - Validate tool definitions match schemas
      - Static analysis of agent configuration

  - unit_tests:
      - Test individual tool functions
      - Test guardrail logic
      - Test input/output parsers

  - basic_evals:
      - Run a small, fast eval set (50-100 cases)
      - Focus on regression detection
      - Target: completes in < 5 minutes
```

### Phase 2: Post-merge (on every merge to main)

After code is merged, run more comprehensive checks before promoting to staging.

```yaml
# Example: Post-merge validation
post_merge:
  - staging_deployment:
      - Deploy to staging environment
      - Verify health checks pass

  - broad_evals:
      - Run full eval suite (500-1000+ cases)
      - Include safety evals
      - Include latency and cost benchmarks
      - Target: completes in < 30 minutes

  - integration_tests:
      - Test end-to-end flows with real tool connections
      - Verify external service integrations
```

### Phase 3: Production gate (before production deployment)

The final check before real users see the new version.

```yaml
# Example: Production gate
production_gate:
  - full_evals:
      - Complete eval suite including edge cases
      - Adversarial test cases
      - Cross-model consistency checks (if using multiple models)

  - safety_review:
      - Automated safety evals must pass 100%
      - Human review for significant prompt changes
      - Red team sign-off for major feature changes

  - approval:
      - Automated approval if all checks pass
      - Manual approval required if any check is marginal
```

### Managing prompts in CI/CD

Prompts deserve the same version control discipline as code:

- Store prompts in version control (not in a database or config service that is hard to diff)
- Review prompt changes in pull requests just like code changes
- Track which prompt version is deployed to which environment
- Make it easy to roll back to a previous prompt version

```
prompts/
  customer_support/
    system_prompt.txt      # The main system instructions
    tool_descriptions.txt  # Tool descriptions and schemas
    safety_rules.txt       # Safety-specific instructions
    version.txt            # Current version identifier
```

---

## Safe rollout strategies

Even with comprehensive evals, production can surprise you. Safe rollout strategies limit the blast radius when something goes wrong.

### Canary deployments

Route a small percentage of traffic to the new version. Monitor for problems before increasing the percentage.

```
Traffic ---> [Load Balancer]
                |         |
               95%       5%
                |         |
                v         v
         [Version 1]  [Version 2 - Canary]
         (current)     (new)
```

**How it works:**
1. Deploy the new version alongside the current one
2. Route 5% of traffic to the new version
3. Monitor key metrics (error rate, latency, user satisfaction, safety incidents)
4. If metrics are healthy after a set period, increase to 25%, then 50%, then 100%
5. If any metric degrades, route all traffic back to the current version

### Blue-green deployments

Maintain two identical production environments. Switch all traffic from one to the other.

```
Before:  Traffic --> [Blue - v1.2 ACTIVE]    [Green - idle]
During:  Traffic --> [Blue - v1.2]            [Green - v1.3 ACTIVE]
```

The advantage is a clean cutover and instant rollback (just switch back to Blue). The downside is you need double the infrastructure during the transition.

### A/B testing

Route traffic to different agent versions and compare their performance on real interactions.

| Version A | Version B | Metric | Winner |
|-----------|-----------|--------|--------|
| GPT-based, verbose prompts | Gemini-based, concise prompts | Task completion rate | Compare after N interactions |
| ReAct pattern | Plan-then-execute | User satisfaction | Compare after N interactions |
| Model A, 3 tool retries | Model A, 1 tool retry | Cost per interaction | Compare after N interactions |

A/B testing is especially valuable for agents because it lets you compare different models, prompts, and architectures on real traffic.

### Feature flags

Control agent capabilities with runtime flags that can be toggled without redeployment.

```python
# Example: Feature flags for agent capabilities
if feature_flags.is_enabled("new_refund_flow", user_id=user.id):
    agent.enable_tool("process_refund_v2")
else:
    agent.enable_tool("process_refund_v1")

if feature_flags.is_enabled("extended_context_window"):
    agent.set_max_context(128000)
else:
    agent.set_max_context(32000)
```

Feature flags let you gradually roll out new capabilities, quickly disable problematic features, and run experiments on subsets of users.

---

## Observability in production

Once your agent is running in production, you need to see what it is doing. Observability for agents has three pillars, just like traditional systems - but the specifics are different.

### Logs

Structured logs that capture every significant event in the agent's lifecycle:

```json
{
  "timestamp": "2025-06-15T10:23:45Z",
  "session_id": "sess_abc123",
  "event": "tool_call",
  "tool": "search_knowledge_base",
  "arguments": {"query": "return policy for electronics"},
  "result_status": "success",
  "latency_ms": 234,
  "tokens_used": {"input": 1250, "output": 380}
}
```

**What to log:**
- Every LLM call (model, input tokens, output tokens, latency)
- Every tool call (tool name, arguments, result status, latency)
- Agent decisions (which path was chosen and why)
- Guardrail activations (what was blocked and why)
- Escalation events
- Session start/end with summary metrics

### Traces

Traces show the full journey of a single request through your agent, including all the steps, tool calls, and decisions along the way.

```
[User Request] "Help me return my order"
    |
    +-- [LLM Call 1] Understand intent (150ms)
    |       Model: gemini-2.0-flash, Tokens: 800 in / 120 out
    |
    +-- [Tool Call] lookup_order(order_id="12345") (340ms)
    |       Status: success
    |
    +-- [Tool Call] check_return_eligibility(order_id="12345") (180ms)
    |       Status: success, eligible=true
    |
    +-- [LLM Call 2] Generate response (200ms)
    |       Model: gemini-2.0-flash, Tokens: 1200 in / 250 out
    |
    +-- [Output Guardrail] PII check (15ms)
    |       Status: pass
    |
    [Response] "Your order #12345 is eligible for return..."

    Total: 885ms, Cost: $0.003
```

[OpenTelemetry](https://opentelemetry.io/) is the industry standard for distributed tracing. Many agent frameworks support OpenTelemetry out of the box, and Google Cloud's operations suite (Cloud Logging, Cloud Trace, Cloud Monitoring) integrates with OpenTelemetry natively.

### Metrics

Aggregate metrics that tell you how your agent is performing overall:

| Metric | What It Tells You | Alert Threshold Example |
|--------|-------------------|------------------------|
| **Task completion rate** | How often the agent successfully completes user requests | Drop below 85% |
| **Average latency** | How long users wait for responses | P95 exceeds 5 seconds |
| **Cost per interaction** | How much each conversation costs | Average exceeds $0.10 |
| **Escalation rate** | How often the agent hands off to humans | Exceeds 20% |
| **Safety incident rate** | How often guardrails are triggered | Any increase above baseline |
| **Tool error rate** | How often tool calls fail | Exceeds 5% |
| **User satisfaction** | Thumbs up/down or CSAT scores | Drops below 4.0/5.0 |

### Building dashboards

A production agent dashboard should show at a glance:

```
+-------------------------------------------------------+
|  Agent Health Dashboard                                |
+-------------------------------------------------------+
|                                                        |
|  Status: HEALTHY          Active Sessions: 1,247       |
|                                                        |
|  +-------------------+  +-------------------+          |
|  | Completion Rate   |  | Avg Latency       |          |
|  | 92.3% (+0.5%)     |  | 1.2s (-0.1s)      |          |
|  +-------------------+  +-------------------+          |
|                                                        |
|  +-------------------+  +-------------------+          |
|  | Cost / Session    |  | Escalation Rate   |          |
|  | $0.042 (-$0.003)  |  | 8.1% (+0.2%)      |          |
|  +-------------------+  +-------------------+          |
|                                                        |
|  Recent Safety Incidents: 0 (last 24h)                 |
|  Recent Errors: 12 (last 24h, 0.04% of sessions)      |
|                                                        |
+-------------------------------------------------------+
```

---

## The Observe-Act-Evolve Loop

Production is not a destination. It is the beginning of a continuous improvement cycle.

```
    +----------+
    | Observe  |  <-- Collect metrics, logs, traces, user feedback
    +----+-----+
         |
         v
    +----+-----+
    |   Act    |  <-- Identify issues, prioritize improvements
    +----+-----+
         |
         v
    +----+-----+
    |  Evolve  |  <-- Update prompts, tools, evals, guardrails
    +----+-----+
         |
         +-------> Back to Observe
```

### Observe

Collect data about how your agent performs in production:

- **Quantitative:** Metrics dashboards, automated eval results on production traffic
- **Qualitative:** User feedback, support tickets, conversation reviews
- **Adversarial:** Ongoing red teaming, new attack pattern detection

### Act

Turn observations into concrete actions:

- Failing on a specific type of query? Add it to your eval set and improve the prompt.
- Tool errors spiking? Investigate the root cause and add better error handling.
- Users consistently confused by a response pattern? Revise the agent's instructions.
- New attack vector discovered? Add a guardrail and a safety eval.

### Evolve

Deploy improvements through your evaluation-gated CI/CD pipeline:

- Update prompts and re-run evals
- Add new tools or modify existing ones
- Expand the eval suite to cover newly discovered edge cases
- Adjust guardrails based on observed threats
- Retrain or swap models if better options become available

The key insight is that your eval suite grows over time. Every production incident, every user complaint, and every edge case becomes a new eval. This means your agent gets harder to break with each iteration.

---

## Cost management

LLM-based agents can be expensive at scale. A single conversation might involve multiple LLM calls, each consuming thousands of tokens. Multiply that by thousands of users and costs add up fast.

### Model routing

Use the cheapest model that can handle each task. Not every step requires your most powerful model.

```
User Query
    |
    v
[Router] --Simple query--> Gemini 2.0 Flash-Lite ($)
    |
    +-----Medium complexity--> Gemini 2.0 Flash ($$)
    |
    +-----Complex reasoning--> Gemini 2.0 Pro ($$$)
```

| Task Type | Recommended Model Tier | Rationale |
|-----------|----------------------|-----------|
| Intent classification | Small / Flash-Lite | Simple classification task |
| Information retrieval | Medium / Flash | Needs good comprehension, moderate generation |
| Complex reasoning | Large / Pro | Multi-step reasoning, nuanced judgment |
| Summarization | Medium / Flash | Good balance of quality and cost |
| Safety checks | Small / Flash-Lite | Pattern matching, classification |

### Caching

Cache responses for repeated or similar queries to avoid redundant LLM calls.

| Caching Strategy | When to Use |
|-----------------|-------------|
| **Exact match cache** | FAQ-style queries where many users ask the same thing |
| **Semantic cache** | Queries that are different in wording but identical in meaning |
| **Tool result cache** | Tool outputs that do not change frequently (e.g., product catalog lookups) |
| **Prompt cache** | Reuse cached prefixes for system prompts across calls (Vertex AI supports context caching) |

### Token budgets

Set hard limits on how many tokens an agent can consume per session.

```python
# Example: Token budget enforcement
class TokenBudget:
    def __init__(self, max_tokens: int):
        self.max_tokens = max_tokens
        self.used_tokens = 0

    def can_proceed(self, estimated_tokens: int) -> bool:
        return (self.used_tokens + estimated_tokens) <= self.max_tokens

    def record_usage(self, actual_tokens: int):
        self.used_tokens += actual_tokens

# Usage
budget = TokenBudget(max_tokens=50000)  # per session

while agent.has_next_step():
    estimated = agent.estimate_next_step_tokens()
    if not budget.can_proceed(estimated):
        agent.respond("I have reached my processing limit for this session. "
                      "Let me summarize what I have found so far.")
        break
    result = agent.execute_next_step()
    budget.record_usage(result.tokens_used)
```

### Cost monitoring

Track costs at multiple levels:

| Level | What to Track | Why |
|-------|--------------|-----|
| **Per request** | Tokens used, model tier, tool calls | Debug expensive individual requests |
| **Per session** | Total cost of a conversation | Set and enforce per-session budgets |
| **Per user** | Aggregate cost per user over time | Identify usage patterns and outliers |
| **Per feature** | Cost of specific agent capabilities | Decide which features are cost-effective |
| **Overall** | Daily/weekly/monthly spend | Budget planning and forecasting |

---

## A production readiness checklist

Before launching your agent to production users, walk through this checklist:

### Reliability
- [ ] Health checks and liveness probes configured
- [ ] Graceful degradation when dependencies fail (model API down, tool unavailable)
- [ ] Retry logic with exponential backoff for transient failures
- [ ] Circuit breakers for external service calls
- [ ] Timeout limits on all LLM and tool calls

### Deployment
- [ ] CI/CD pipeline with eval gates at each stage
- [ ] Rollback procedure tested and documented
- [ ] Canary or blue-green deployment configured
- [ ] Feature flags for new capabilities
- [ ] Prompt versioning and change tracking

### Observability
- [ ] Structured logging for all agent events
- [ ] Distributed tracing with OpenTelemetry
- [ ] Dashboards for key metrics (completion rate, latency, cost, safety)
- [ ] Alerting configured for critical thresholds
- [ ] On-call rotation and incident response runbook

### Cost
- [ ] Model routing configured (right model for each task)
- [ ] Caching strategy implemented
- [ ] Token budgets per session
- [ ] Cost monitoring and alerting
- [ ] Regular cost reviews and optimization

### Safety
- [ ] Guardrails from Lesson 10 implemented and tested
- [ ] Safety evals passing at 100%
- [ ] Red team review completed
- [ ] Incident response plan for safety failures
- [ ] User feedback channel for reporting problems

---

## Key takeaways

1. **The prototype-to-production gap is real and large.** Plan for production concerns from the beginning. The "last mile" is the majority of the work.

2. **Evaluation-gated deployment is non-negotiable.** No agent version should reach production without passing a comprehensive eval suite. Your eval suite is your quality guarantee.

3. **CI/CD for agents has three phases.** Pre-merge checks catch obvious issues fast. Post-merge validation runs broader evals. Production gates ensure safety and quality before real users are affected.

4. **Safe rollout strategies limit blast radius.** Canary deployments, feature flags, and A/B testing let you catch problems before they affect all users.

5. **Observability is essential.** You cannot improve what you cannot see. Invest in logs, traces, and metrics from day one.

6. **Cost management requires active attention.** Model routing, caching, and token budgets can reduce costs dramatically without sacrificing quality.

7. **Production is the beginning, not the end.** The Observe-Act-Evolve loop means your agent continuously improves based on real-world usage.

---

## Further reading

- [Deploy Agents on Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy) - Official guide for deploying agents on Google Cloud
- [Agent Starter Pack](https://github.com/GoogleCloudPlatform/agent-starter-pack) - Production-ready templates for deploying agents on Google Cloud with CI/CD, observability, and evaluation
- [Vertex AI Model Monitoring](https://cloud.google.com/vertex-ai/docs/model-monitoring/overview) - Monitor model performance in production
- [OpenTelemetry](https://opentelemetry.io/) - The industry standard for distributed tracing and observability

---

Next lesson: [Getting Started with Vertex AI and ADK](../12-getting-started-with-vertex-and-adk/README.md)
