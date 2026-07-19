# Lesson 9: evaluating and testing agents - how to know if your agent works

## Introduction

You have built an agent. It has tools, memory, and a planning loop. It can retrieve information, take actions, and converse with users. But how do you know if it actually works well?

Testing an agent is fundamentally different from testing traditional software. With a regular function, you pass in inputs and check that the outputs match your expectations. With an agent, the same question can produce different valid answers, the agent might take different paths to reach the same goal, and "correct" is often subjective. The output is non-deterministic, the process is multi-step, and the quality dimensions are more nuanced than pass/fail.

This lesson covers how to evaluate agents rigorously - what to measure, how to measure it, and how to build evaluation into your development workflow so that quality improves continuously.

## Why agent evaluation is hard

### Non-Deterministic Outputs

Ask a traditional function to add 2 + 2 and it always returns 4. Ask an agent "What should I do about our declining user retention?" and you might get a different answer every time - and multiple different answers could all be good. Temperature settings, model updates, and slight prompt variations all change the output.

### Many valid paths

An agent that books a flight might search by price first, then by time. Or it might search by time first, then filter by price. Both paths reach the same goal, but the sequence of tool calls is different. Testing that the agent took exact step X at position Y would be too rigid and would break with any reasonable change.

### Compound errors

An agent that makes five decisions in sequence has five chances to make a mistake, and errors compound. A small mistake in step 2 might lead to a completely wrong outcome by step 5. Testing only the final output misses where things went wrong.

### Subjective quality

Is a summary "good"? Is a customer service response "helpful"? These are judgment calls that depend on context, user expectations, and organizational standards. Binary pass/fail testing is not sufficient.

## ELI5: the new employee analogy

Testing an agent is like evaluating a new employee during their probation period. You would not just check whether they produced the right final deliverable. You would also observe:

- **Did they achieve the goal?** (Effectiveness) Did the report answer the question they were asked?
- **How efficiently did they work?** (Efficiency) Did they spend three days on something that should take three hours?
- **Can they handle curveballs?** (Robustness) What happens when they get unclear instructions or missing data?
- **Do they stay within appropriate boundaries?** (Safety) Did they access only the systems they should? Did they follow company policy?

You would also look at their process, not just their output. If they got the right answer by sheer luck after a chaotic process, that is different from getting the right answer through a methodical, reliable approach.

Agent evaluation works the same way. You check the final output, the process, the efficiency, and the safety - and you do it systematically.

## Four pillars of agent quality

Every agent evaluation should assess four dimensions:

### 1. effectiveness - does it achieve the goal?

This is the most basic question: did the agent do what it was supposed to do? If you asked it to book a flight to Tokyo, did it book a flight to Tokyo?

**What to measure:**
- Task completion rate (what percentage of tasks does the agent finish successfully?)
- Correctness of the final output (is the answer right? is the action correct?)
- User satisfaction (did the user get what they needed?)

**Example metrics:**
| Metric | How to Measure | Target |
|---|---|---|
| Task completion rate | Automated checks against expected outcomes | > 90% |
| Answer correctness | Human evaluation or automated fact-checking | > 85% |
| User satisfaction | Post-interaction surveys or thumbs up/down | > 4.0/5.0 |

### 2. efficiency - at what cost?

An agent that achieves the goal but uses 50 API calls, takes 3 minutes, and costs $2 per query might not be viable. Efficiency measures how much it costs - in time, money, and compute - to get the job done.

**What to measure:**
- Latency (how long does the user wait?)
- Token usage (how many tokens consumed per task?)
- Number of LLM calls (how many reasoning steps?)
- Number of tool calls (how many external API calls?)
- Dollar cost per task

**Example metrics:**
| Metric | How to Measure | Target |
|---|---|---|
| End-to-end latency | Time from user query to final response | < 10 seconds |
| Tokens per task | Sum of input + output tokens across all LLM calls | < 10,000 |
| LLM calls per task | Count of model invocations | < 5 |
| Cost per task | Token cost + API call costs | < $0.10 |

### 3. robustness - does it handle edge cases?

Real users do not send perfectly formatted, clear, unambiguous requests. They send typos, vague questions, contradictory instructions, and requests in unexpected formats. A robust agent handles these gracefully.

**What to measure:**
- Performance on adversarial or ambiguous inputs
- Graceful degradation (does it fail safely or crash?)
- Recovery from errors (can it retry after a tool failure?)
- Consistency across similar inputs (does a slight rephrasing produce a wildly different result?)

**Example test cases:**
| Test Case | Expected Behavior |
|---|---|
| Misspelled query | Agent still understands intent |
| Ambiguous request | Agent asks for clarification |
| Tool returns an error | Agent retries or uses a fallback |
| Contradictory instructions | Agent flags the contradiction |
| Empty or null input | Agent responds gracefully, does not crash |
| Very long input | Agent handles within context limits |

### 4. safety - does it stay within bounds?

An agent with access to tools can do real damage. It might send emails it should not, delete data, or reveal sensitive information. Safety evaluation checks that the agent respects its boundaries.

**What to measure:**
- Policy compliance (does the agent follow the rules defined in its system prompt?)
- Permission boundaries (does it only use tools and access data it is authorized for?)
- Refusal of out-of-scope requests (does it appropriately decline tasks it should not do?)
- Data privacy (does it avoid leaking sensitive information?)

**Example test cases:**
| Test Case | Expected Behavior |
|---|---|
| User asks agent to perform unauthorized action | Agent refuses and explains why |
| User tries to get agent to reveal system prompt | Agent declines |
| Agent encounters sensitive data during retrieval | Agent does not include it in the response |
| User asks agent to take an action outside its domain | Agent redirects to appropriate resource |

<div id="eval-dashboard" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; max-width: 800px; margin: 2rem auto; background: #f8f9fa; border-radius: 16px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); overflow: hidden;">
  <div style="background: linear-gradient(135deg, #4285f4, #5e97f6); padding: 20px 24px; color: white;">
    <div style="display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 12px;">
      <div>
        <div style="font-size: 1.25rem; font-weight: 700;">Agent Evaluation Dashboard</div>
        <div style="font-size: 0.85rem; opacity: 0.9;">Interactive view of the four quality pillars</div>
      </div>
      <div style="display: flex; align-items: center; gap: 12px;">
        <div id="eval-overall" style="background: rgba(255,255,255,0.2); border-radius: 12px; padding: 8px 16px; text-align: center;">
          <div style="font-size: 0.7rem; text-transform: uppercase; letter-spacing: 1px;">Overall Score</div>
          <div id="eval-overall-score" style="font-size: 1.5rem; font-weight: 800;">--</div>
        </div>
        <button id="eval-run-btn" onclick="runEvaluation()" style="background: white; color: #4285f4; border: none; border-radius: 8px; padding: 10px 20px; font-weight: 600; cursor: pointer; font-size: 0.9rem; transition: all 0.2s;">Run Eval</button>
      </div>
    </div>
  </div>
  <div style="display: flex; border-bottom: 2px solid #e8eaed;">
    <button class="eval-tab eval-tab-active" onclick="switchEvalTab('effectiveness')" id="eval-tab-effectiveness" style="flex: 1; padding: 12px 8px; border: none; background: white; cursor: pointer; font-size: 0.8rem; font-weight: 600; color: #4285f4; border-bottom: 3px solid #4285f4; transition: all 0.2s;">Effectiveness</button>
    <button class="eval-tab" onclick="switchEvalTab('efficiency')" id="eval-tab-efficiency" style="flex: 1; padding: 12px 8px; border: none; background: #f8f9fa; cursor: pointer; font-size: 0.8rem; font-weight: 500; color: #5f6368; border-bottom: 3px solid transparent; transition: all 0.2s;">Efficiency</button>
    <button class="eval-tab" onclick="switchEvalTab('robustness')" id="eval-tab-robustness" style="flex: 1; padding: 12px 8px; border: none; background: #f8f9fa; cursor: pointer; font-size: 0.8rem; font-weight: 500; color: #5f6368; border-bottom: 3px solid transparent; transition: all 0.2s;">Robustness</button>
    <button class="eval-tab" onclick="switchEvalTab('safety')" id="eval-tab-safety" style="flex: 1; padding: 12px 8px; border: none; background: #f8f9fa; cursor: pointer; font-size: 0.8rem; font-weight: 500; color: #5f6368; border-bottom: 3px solid transparent; transition: all 0.2s;">Safety</button>
  </div>
  <div id="eval-panel-effectiveness" style="padding: 24px;">
    <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 16px;">
      <span style="font-size: 1.5rem;">🎯</span>
      <div><div style="font-weight: 700; font-size: 1rem;">Effectiveness</div><div style="font-size: 0.8rem; color: #5f6368;">Does it achieve the goal?</div></div>
    </div>
    <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 16px;">
      <div style="background: white; border-radius: 12px; padding: 20px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 8px;">Task Completion Rate</div>
        <svg width="120" height="120" viewBox="0 0 120 120" style="display: block; margin: 0 auto;">
          <circle cx="60" cy="60" r="50" fill="none" stroke="#e8eaed" stroke-width="10"/>
          <circle id="eval-gauge-completion" cx="60" cy="60" r="50" fill="none" stroke="#34a853" stroke-width="10" stroke-dasharray="0 314" stroke-linecap="round" transform="rotate(-90 60 60)" style="transition: stroke-dasharray 1s ease;"/>
          <text x="60" y="60" text-anchor="middle" dy="6" style="font-size: 20px; font-weight: 700; fill: #202124;" id="eval-text-completion">--%</text>
        </svg>
        <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &gt; 90%</div>
      </div>
      <div style="background: white; border-radius: 12px; padding: 20px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 8px;">Answer Correctness</div>
        <svg width="120" height="120" viewBox="0 0 120 120" style="display: block; margin: 0 auto;">
          <circle cx="60" cy="60" r="50" fill="none" stroke="#e8eaed" stroke-width="10"/>
          <circle id="eval-gauge-correctness" cx="60" cy="60" r="50" fill="none" stroke="#4285f4" stroke-width="10" stroke-dasharray="0 314" stroke-linecap="round" transform="rotate(-90 60 60)" style="transition: stroke-dasharray 1s ease;"/>
          <text x="60" y="60" text-anchor="middle" dy="6" style="font-size: 20px; font-weight: 700; fill: #202124;" id="eval-text-correctness">--%</text>
        </svg>
        <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &gt; 85%</div>
      </div>
    </div>
    <div style="background: white; border-radius: 12px; padding: 16px; margin-top: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
      <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 8px;">User Satisfaction</div>
      <div style="display: flex; align-items: center; gap: 12px;">
        <div style="flex: 1; background: #e8eaed; border-radius: 8px; height: 24px; overflow: hidden;">
          <div id="eval-bar-satisfaction" style="height: 100%; background: linear-gradient(90deg, #fbbc04, #34a853); border-radius: 8px; width: 0%; transition: width 1s ease;"></div>
        </div>
        <span id="eval-text-satisfaction" style="font-weight: 700; font-size: 0.9rem; min-width: 50px;">--/5.0</span>
      </div>
      <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &gt; 4.0/5.0</div>
    </div>
  </div>
  <div id="eval-panel-efficiency" style="padding: 24px; display: none;">
    <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 16px;">
      <span style="font-size: 1.5rem;">⚡</span>
      <div><div style="font-weight: 700; font-size: 1rem;">Efficiency</div><div style="font-size: 0.8rem; color: #5f6368;">At what cost?</div></div>
    </div>
    <div style="display: grid; gap: 12px;">
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">Avg Latency</span><span id="eval-text-latency" style="font-weight: 700; font-size: 0.9rem;">-- s</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 20px; overflow: hidden;"><div id="eval-bar-latency" style="height: 100%; background: #4285f4; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
        <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &lt; 10s</div>
      </div>
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">Tokens Per Task</span><span id="eval-text-tokens" style="font-weight: 700; font-size: 0.9rem;">--</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 20px; overflow: hidden;"><div id="eval-bar-tokens" style="height: 100%; background: #9333ea; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
        <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &lt; 10,000</div>
      </div>
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">Cost Per Task</span><span id="eval-text-cost" style="font-weight: 700; font-size: 0.9rem;">$--</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 20px; overflow: hidden;"><div id="eval-bar-cost" style="height: 100%; background: #34a853; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
        <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &lt; $0.10</div>
      </div>
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">LLM Calls Per Task</span><span id="eval-text-calls" style="font-weight: 700; font-size: 0.9rem;">--</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 20px; overflow: hidden;"><div id="eval-bar-calls" style="height: 100%; background: #fbbc04; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
        <div style="font-size: 0.7rem; color: #5f6368; margin-top: 4px;">Target: &lt; 5</div>
      </div>
    </div>
  </div>
  <div id="eval-panel-robustness" style="padding: 24px; display: none;">
    <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 16px;">
      <span style="font-size: 1.5rem;">🛡️</span>
      <div><div style="font-weight: 700; font-size: 1rem;">Robustness</div><div style="font-size: 0.8rem; color: #5f6368;">Does it handle edge cases?</div></div>
    </div>
    <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 16px;">
      <div style="background: white; border-radius: 12px; padding: 20px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 12px;">Error Recovery Rate</div>
        <svg width="120" height="120" viewBox="0 0 120 120" style="display: block; margin: 0 auto;">
          <circle cx="60" cy="60" r="50" fill="none" stroke="#e8eaed" stroke-width="10"/>
          <circle id="eval-gauge-recovery" cx="60" cy="60" r="50" fill="none" stroke="#fbbc04" stroke-width="10" stroke-dasharray="0 314" stroke-linecap="round" transform="rotate(-90 60 60)" style="transition: stroke-dasharray 1s ease;"/>
          <text x="60" y="60" text-anchor="middle" dy="6" style="font-size: 20px; font-weight: 700; fill: #202124;" id="eval-text-recovery">--%</text>
        </svg>
      </div>
      <div style="background: white; border-radius: 12px; padding: 20px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 12px;">Edge Case Handling</div>
        <svg width="120" height="120" viewBox="0 0 120 120" style="display: block; margin: 0 auto;">
          <circle cx="60" cy="60" r="50" fill="none" stroke="#e8eaed" stroke-width="10"/>
          <circle id="eval-gauge-edge" cx="60" cy="60" r="50" fill="none" stroke="#9333ea" stroke-width="10" stroke-dasharray="0 314" stroke-linecap="round" transform="rotate(-90 60 60)" style="transition: stroke-dasharray 1s ease;"/>
          <text x="60" y="60" text-anchor="middle" dy="6" style="font-size: 20px; font-weight: 700; fill: #202124;" id="eval-text-edge">--%</text>
        </svg>
      </div>
    </div>
    <div style="background: white; border-radius: 12px; padding: 16px; margin-top: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
      <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 10px;">Resilience Test Results</div>
      <div id="eval-resilience-tests" style="display: grid; grid-template-columns: 1fr 1fr; gap: 8px;">
        <div class="eval-test-item" style="display: flex; align-items: center; gap: 6px; font-size: 0.8rem; padding: 8px; background: #f8f9fa; border-radius: 8px;"><span id="eval-icon-typo" style="opacity: 0.3;">✅</span> Misspelled query</div>
        <div class="eval-test-item" style="display: flex; align-items: center; gap: 6px; font-size: 0.8rem; padding: 8px; background: #f8f9fa; border-radius: 8px;"><span id="eval-icon-ambig" style="opacity: 0.3;">✅</span> Ambiguous request</div>
        <div class="eval-test-item" style="display: flex; align-items: center; gap: 6px; font-size: 0.8rem; padding: 8px; background: #f8f9fa; border-radius: 8px;"><span id="eval-icon-error" style="opacity: 0.3;">✅</span> Tool error recovery</div>
        <div class="eval-test-item" style="display: flex; align-items: center; gap: 6px; font-size: 0.8rem; padding: 8px; background: #f8f9fa; border-radius: 8px;"><span id="eval-icon-contra" style="opacity: 0.3;">✅</span> Contradictory input</div>
        <div class="eval-test-item" style="display: flex; align-items: center; gap: 6px; font-size: 0.8rem; padding: 8px; background: #f8f9fa; border-radius: 8px;"><span id="eval-icon-empty" style="opacity: 0.3;">✅</span> Empty input</div>
        <div class="eval-test-item" style="display: flex; align-items: center; gap: 6px; font-size: 0.8rem; padding: 8px; background: #f8f9fa; border-radius: 8px;"><span id="eval-icon-long" style="opacity: 0.3;">✅</span> Very long input</div>
      </div>
    </div>
  </div>
  <div id="eval-panel-safety" style="padding: 24px; display: none;">
    <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 16px;">
      <span style="font-size: 1.5rem;">🔒</span>
      <div><div style="font-weight: 700; font-size: 1rem;">Safety</div><div style="font-size: 0.8rem; color: #5f6368;">Does it stay within bounds?</div></div>
    </div>
    <div style="background: white; border-radius: 12px; padding: 24px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.04); margin-bottom: 16px;">
      <div style="font-size: 0.75rem; color: #5f6368; margin-bottom: 8px;">Safety Score</div>
      <div style="position: relative; display: inline-block;">
        <svg width="140" height="160" viewBox="0 0 140 160">
          <path d="M70 10 L125 40 L125 90 C125 125 70 155 70 155 C70 155 15 125 15 90 L15 40 Z" fill="none" stroke="#e8eaed" stroke-width="4"/>
          <path id="eval-shield-fill" d="M70 10 L125 40 L125 90 C125 125 70 155 70 155 C70 155 15 125 15 90 L15 40 Z" fill="#e8eaed" opacity="0.3" style="transition: fill 0.5s, opacity 0.5s;"/>
          <text x="70" y="90" text-anchor="middle" dy="6" style="font-size: 28px; font-weight: 800; fill: #202124;" id="eval-text-safety-score">--</text>
          <text x="70" y="110" text-anchor="middle" style="font-size: 11px; fill: #5f6368;">/100</text>
        </svg>
      </div>
    </div>
    <div style="display: grid; gap: 12px;">
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">Policy Compliance</span><span id="eval-text-policy" style="font-weight: 700; font-size: 0.9rem;">--%</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 16px; overflow: hidden;"><div id="eval-bar-policy" style="height: 100%; background: #34a853; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
      </div>
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">Guardrail Violations</span><span id="eval-text-violations" style="font-weight: 700; font-size: 0.9rem;">--</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 16px; overflow: hidden;"><div id="eval-bar-violations" style="height: 100%; background: #ea4335; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
      </div>
      <div style="background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04);">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;"><span style="font-size: 0.75rem; color: #5f6368;">Harmful Output Rate</span><span id="eval-text-harmful" style="font-weight: 700; font-size: 0.9rem;">--%</span></div>
        <div style="background: #e8eaed; border-radius: 8px; height: 16px; overflow: hidden;"><div id="eval-bar-harmful" style="height: 100%; background: #fbbc04; border-radius: 8px; width: 0%; transition: width 1s ease;"></div></div>
      </div>
    </div>
  </div>
  <div id="eval-status" style="padding: 12px 24px; background: #e8eaed; text-align: center; font-size: 0.8rem; color: #5f6368;">Click "Run Eval" to simulate an evaluation run</div>
</div>

<script>
(function() {
  var evalData = null;

  window.switchEvalTab = function(tab) {
    var tabs = ['effectiveness', 'efficiency', 'robustness', 'safety'];
    tabs.forEach(function(t) {
      var panel = document.getElementById('eval-panel-' + t);
      var tabBtn = document.getElementById('eval-tab-' + t);
      if (t === tab) {
        panel.style.display = 'block';
        tabBtn.style.background = 'white';
        tabBtn.style.color = '#4285f4';
        tabBtn.style.fontWeight = '600';
        tabBtn.style.borderBottom = '3px solid #4285f4';
      } else {
        panel.style.display = 'none';
        tabBtn.style.background = '#f8f9fa';
        tabBtn.style.color = '#5f6368';
        tabBtn.style.fontWeight = '500';
        tabBtn.style.borderBottom = '3px solid transparent';
      }
    });
  };

  function rnd(min, max) { return Math.round((Math.random() * (max - min) + min) * 10) / 10; }
  function getColor(val, good, warn) { return val >= good ? '#34a853' : val >= warn ? '#fbbc04' : '#ea4335'; }

  function animateGauge(id, textId, val, maxVal) {
    var circumference = 314;
    var el = document.getElementById(id);
    var txt = document.getElementById(textId);
    var pct = val / maxVal;
    el.style.stroke = getColor(val, maxVal * 0.85, maxVal * 0.7);
    el.setAttribute('stroke-dasharray', (pct * circumference) + ' ' + circumference);
    txt.textContent = Math.round(val) + '%';
  }

  window.runEvaluation = function() {
    var btn = document.getElementById('eval-run-btn');
    btn.textContent = 'Running...';
    btn.style.opacity = '0.6';
    btn.disabled = true;
    document.getElementById('eval-status').textContent = 'Running evaluation suite... simulating 500 test cases';
    document.getElementById('eval-status').style.background = '#e3f2fd';
    document.getElementById('eval-status').style.color = '#1565c0';

    setTimeout(function() {
      var completion = rnd(82, 98);
      var correctness = rnd(78, 96);
      var satisfaction = rnd(3.2, 4.8);
      var latency = rnd(1.5, 12);
      var tokens = Math.round(rnd(3000, 14000));
      var cost = rnd(0.02, 0.15);
      var calls = rnd(2, 7);
      var recovery = rnd(65, 95);
      var edge = rnd(60, 92);
      var policy = rnd(90, 100);
      var violations = Math.round(rnd(0, 8));
      var harmful = rnd(0, 3);
      var safetyScore = Math.round((policy + (100 - violations * 5) + (100 - harmful * 10)) / 3);
      safetyScore = Math.max(0, Math.min(100, safetyScore));

      animateGauge('eval-gauge-completion', 'eval-text-completion', completion, 100);
      animateGauge('eval-gauge-correctness', 'eval-text-correctness', correctness, 100);
      var satPct = (satisfaction / 5) * 100;
      document.getElementById('eval-bar-satisfaction').style.width = satPct + '%';
      document.getElementById('eval-text-satisfaction').textContent = satisfaction.toFixed(1) + '/5.0';
      document.getElementById('eval-bar-satisfaction').style.background = 'linear-gradient(90deg, ' + getColor(satisfaction, 4, 3.5) + ', ' + getColor(satisfaction, 4, 3.5) + ')';

      var latPct = Math.min(100, (latency / 15) * 100);
      document.getElementById('eval-bar-latency').style.width = latPct + '%';
      document.getElementById('eval-bar-latency').style.background = getColor(15 - latency, 5, 3);
      document.getElementById('eval-text-latency').textContent = latency.toFixed(1) + 's';

      var tokPct = Math.min(100, (tokens / 15000) * 100);
      document.getElementById('eval-bar-tokens').style.width = tokPct + '%';
      document.getElementById('eval-bar-tokens').style.background = getColor(15000 - tokens, 5000, 3000);
      document.getElementById('eval-text-tokens').textContent = tokens.toLocaleString();

      var costPct = Math.min(100, (cost / 0.2) * 100);
      document.getElementById('eval-bar-cost').style.width = costPct + '%';
      document.getElementById('eval-bar-cost').style.background = getColor(0.2 - cost, 0.1, 0.05);
      document.getElementById('eval-text-cost').textContent = '$' + cost.toFixed(2);

      var callsPct = Math.min(100, (calls / 10) * 100);
      document.getElementById('eval-bar-calls').style.width = callsPct + '%';
      document.getElementById('eval-bar-calls').style.background = getColor(10 - calls, 5, 3);
      document.getElementById('eval-text-calls').textContent = calls.toFixed(1);

      animateGauge('eval-gauge-recovery', 'eval-text-recovery', recovery, 100);
      animateGauge('eval-gauge-edge', 'eval-text-edge', edge, 100);

      var tests = ['typo', 'ambig', 'error', 'contra', 'empty', 'long'];
      tests.forEach(function(t, i) {
        setTimeout(function() {
          var pass = Math.random() > 0.15;
          var el = document.getElementById('eval-icon-' + t);
          el.textContent = pass ? '✅' : '❌';
          el.style.opacity = '1';
        }, i * 150);
      });

      document.getElementById('eval-bar-policy').style.width = policy + '%';
      document.getElementById('eval-bar-policy').style.background = getColor(policy, 95, 85);
      document.getElementById('eval-text-policy').textContent = policy.toFixed(1) + '%';

      var violPct = Math.min(100, (violations / 20) * 100);
      document.getElementById('eval-bar-violations').style.width = violPct + '%';
      document.getElementById('eval-text-violations').textContent = violations + ' found';

      var harmPct = Math.min(100, (harmful / 5) * 100);
      document.getElementById('eval-bar-harmful').style.width = harmPct + '%';
      document.getElementById('eval-text-harmful').textContent = harmful.toFixed(1) + '%';

      var shield = document.getElementById('eval-shield-fill');
      shield.style.fill = getColor(safetyScore, 85, 70);
      shield.style.opacity = '0.2';
      document.getElementById('eval-text-safety-score').textContent = safetyScore;

      var overall = Math.round((completion + correctness + recovery + safetyScore) / 4);
      document.getElementById('eval-overall-score').textContent = overall + '/100';

      btn.textContent = 'Run Eval';
      btn.style.opacity = '1';
      btn.disabled = false;
      document.getElementById('eval-status').textContent = 'Evaluation complete — 500 test cases processed. Click "Run Eval" to re-run.';
      document.getElementById('eval-status').style.background = '#e6f4ea';
      document.getElementById('eval-status').style.color = '#137333';
    }, 1200);
  };
})();
</script>

## System metrics vs. quality metrics

Agent evaluation requires two categories of metrics that serve different purposes:

### System metrics (operational health)

System metrics tell you whether the agent is running well from an infrastructure perspective. These are the metrics your SRE team cares about.

| Metric | What It Tells You | How to Collect |
|---|---|---|
| Latency (p50, p95, p99) | How long users wait | Request timing in your application |
| Error rate | How often the agent fails entirely | Error counting in logs |
| Tokens per task | How much compute each task requires | LLM API response metadata |
| Cost per task | How much money each task costs | Token counts multiplied by pricing |
| Tool call success rate | How reliable external integrations are | Tool wrapper instrumentation |
| Throughput | How many requests the system handles | Request counting |

### Quality metrics (output goodness)

Quality metrics tell you whether the agent is doing a good job from the user's perspective. These are harder to measure but more important.

| Metric | What It Tells You | How to Measure |
|---|---|---|
| Correctness | Is the answer right? | Ground truth comparison, human evaluation |
| Trajectory quality | Did the agent take a reasonable path? | Trajectory evaluation (see below) |
| Helpfulness | Did the user get what they needed? | User feedback, LLM-as-a-Judge |
| Safety compliance | Did the agent stay within bounds? | Red-team testing, policy checkers |
| Groundedness | Is the answer supported by evidence? | Source attribution checking |
| Coherence | Does the response make sense? | LLM-as-a-Judge, human evaluation |

## The Outside-In Approach

When evaluating an agent, start from the outside and work your way in. This mirrors how users experience the agent and ensures you catch the most impactful issues first.

### Level 1: Black-Box End-to-End Testing

Start by treating the agent as a black box. Give it inputs and check the outputs. Do not look at how it arrived at the answer - just check whether the answer is correct.

**How to do it:**
1. Create a test set of input-output pairs (questions and expected answers)
2. Run each input through the agent
3. Compare the agent's output to the expected output
4. Track pass/fail rates

**Example test set:**

| Input | Expected Output | Pass Criteria |
|---|---|---|
| "What is our refund policy?" | Includes 30-day window and receipt requirement | Contains key policy elements |
| "Cancel order #12345" | Order is cancelled, confirmation provided | Order status changed + confirmation message |
| "What time does the store close?" | Correct closing time for today | Matches actual hours |

**When this is sufficient:** For agents with clear, verifiable outputs where there is one right answer. If the agent's job is to look up facts or execute well-defined actions, end-to-end testing covers most of what you need.

### Level 2: glass-box trajectory evaluation

When end-to-end testing is not sufficient - when you need to understand why the agent succeeded or failed - you open the box and inspect the trajectory.

A trajectory is the full sequence of the agent's actions: every thought, tool call, observation, and decision, from receiving the input to producing the output.

**Example trajectory:**
```
1. User: "What was our revenue last quarter?"
2. Agent thinks: "I need to look up revenue data for Q2 2025"
3. Agent calls: search_financial_reports(query="Q2 2025 revenue")
4. Tool returns: [Q2 2025 Financial Summary document]
5. Agent thinks: "Found the report. Revenue was $12.4M"
6. Agent responds: "Our revenue last quarter (Q2 2025) was $12.4M,
   up 8% from Q1 2025."
```

**What to check in the trajectory:**
- Did the agent call the right tools?
- Did it call them with the right parameters?
- Did it interpret tool results correctly?
- Did it avoid unnecessary steps?
- Did it handle errors appropriately?
- Did it stay within its authorized actions?

**Why trajectory evaluation matters:** Two agents might produce the same final answer, but one took a clean, efficient path while the other made several wrong turns, called irrelevant tools, and got lucky. The first agent is more reliable. Trajectory evaluation reveals this difference.

## Trajectory evaluation in detail

Trajectory evaluation checks the full execution path of the agent. This is one of the most powerful evaluation techniques because it catches problems that end-to-end testing misses.

### What a trajectory contains

| Component | Description | Example |
|---|---|---|
| User input | The original request | "Book me a flight to Tokyo next Tuesday" |
| Agent reasoning | The agent's internal thoughts | "I need to search for flights on March 25" |
| Tool calls | Actions the agent took | search_flights(destination="Tokyo", date="2025-03-25") |
| Tool results | What the tools returned | List of 5 available flights |
| Agent decisions | Choices the agent made | Selected the cheapest direct flight |
| Final output | The response to the user | "I booked flight JL001, departing at 10:30 AM..." |

### Evaluating trajectories

You can evaluate trajectories along several dimensions:

**Tool Selection Accuracy:** Did the agent pick the right tool for each step?

```
Good: Agent needs weather data -> calls get_weather()
Bad:  Agent needs weather data -> calls search_web("weather forecast")
      when get_weather() is available
```

**Parameter Correctness:** Did the agent pass the right arguments to each tool?

```
Good: search_flights(destination="NRT", date="2025-03-25")
Bad:  search_flights(destination="Tokyo", date="next Tuesday")
      (did not resolve "next Tuesday" to an actual date)
```

**Step Efficiency:** Did the agent achieve the goal without unnecessary steps?

```
Efficient (3 steps):
  1. Search flights
  2. Select best option
  3. Book flight

Inefficient (7 steps):
  1. Search flights to Tokyo
  2. Search flights to Osaka (not requested)
  3. Compare Tokyo and Osaka options (not requested)
  4. Search Tokyo hotels (not requested)
  5. Go back to flights
  6. Select a flight
  7. Book flight
```

**Error Handling:** When a tool failed, did the agent recover appropriately?

```
Good: Tool returns error -> Agent retries with modified parameters
Good: Tool returns error -> Agent informs user and suggests alternatives
Bad:  Tool returns error -> Agent hallucinates a result
Bad:  Tool returns error -> Agent crashes
```

## LLM-as-a-Judge

Human evaluation is the gold standard for quality assessment, but it is slow and expensive. LLM-as-a-Judge uses one model to evaluate another model's output, giving you automated quality assessment that approximates human judgment.

### How it works

You give a judge LLM the following:
1. The original question or task
2. The agent's response (or trajectory)
3. Evaluation criteria
4. Optionally, a reference answer

The judge LLM then scores the response based on the criteria.

### Single scoring vs. pairwise comparison

**Single scoring** asks the judge to rate one response on a scale (e.g., 1-5 for helpfulness). This is simple but tends to suffer from position bias and inconsistent calibration.

**Pairwise comparison** shows the judge two responses and asks which is better. This is more reliable because relative comparisons are easier than absolute ratings.

**Recommendation: Prefer pairwise comparison** when possible. It produces more consistent and actionable results.

### Pairwise comparison example

```
Judge prompt:
"You are evaluating two responses to a customer question.
The customer asked: 'How do I reset my password?'

Response A:
'Click on Forgot Password on the login page, enter your email,
and follow the link in the reset email.'

Response B:
'You can reset your password through the account settings page
or by contacting our support team at support@example.com.'

Which response is more helpful and complete? Explain your reasoning
and declare a winner."
```

### Best practices for llm-as-a-judge

| Practice | Why It Matters |
|---|---|
| Use a strong model as judge | Weaker models make worse judgments |
| Provide clear evaluation criteria | Vague criteria lead to inconsistent scoring |
| Use pairwise comparison over single scoring | More reliable and consistent |
| Randomize response order | Prevents position bias (models tend to prefer the first response) |
| Include reference answers when available | Gives the judge a baseline for comparison |
| Validate judge scores against human scores | Ensure the judge correlates with human judgment |
| Run multiple judge evaluations | Reduce variance by averaging across evaluations |

### Limitations of LLM-as-a-Judge

- Judges can have biases (verbosity bias - preferring longer responses, position bias - preferring the first option)
- Judges may not catch factual errors if they lack domain knowledge
- Judge quality depends on the judge model's capabilities
- Not a full replacement for human evaluation on high-stakes decisions

## Human evaluation

Despite the power of automated evaluation, human evaluation remains essential for certain aspects of agent quality.

### When you need humans

- **Subjective quality:** Is the tone appropriate? Is the response empathetic? Does it match the brand voice?
- **Novel situations:** When the agent encounters a scenario not covered by automated tests
- **Safety-critical decisions:** When the agent is about to take an action with significant consequences
- **Ground truth creation:** Building the test sets that automated evaluation relies on
- **Calibrating LLM judges:** Validating that your automated judges agree with human judgment

### Structuring human evaluation

**Rating scales:** Have evaluators rate responses on specific dimensions (correctness, helpfulness, safety) using a defined scale (1-5 with clear descriptions for each level).

**Annotation guidelines:** Provide detailed guidelines with examples of what constitutes a 1, 3, and 5 on each dimension. Without this, evaluators will interpret the scale differently.

**Inter-annotator agreement:** Have multiple evaluators score the same responses. If they disagree significantly, your guidelines need improvement.

**Example rating rubric:**

| Score | Correctness Criteria |
|---|---|
| 1 | Answer is factually wrong or completely off-topic |
| 2 | Answer has significant errors but shows some understanding |
| 3 | Answer is mostly correct with minor errors or omissions |
| 4 | Answer is correct and complete |
| 5 | Answer is correct, complete, and includes helpful additional context |

## The agent quality flywheel

Evaluation is not a one-time activity. It is a continuous cycle that drives quality improvement over time.

### Step 1: define quality

Before you can measure quality, you need to define it. What does "good" mean for your agent? This is specific to your use case.

**Questions to answer:**
- What are the must-have behaviors? (e.g., never reveal customer PII)
- What are the nice-to-have behaviors? (e.g., proactively suggest related actions)
- What are the unacceptable behaviors? (e.g., making up data, taking unauthorized actions)
- What are the efficiency targets? (e.g., under 5 seconds, under $0.05 per query)

### Step 2: instrument for visibility

You cannot improve what you cannot see. Add instrumentation to capture everything the agent does.

**What to instrument:**
- Every LLM call (input, output, latency, tokens)
- Every tool call (input, output, success/failure, latency)
- The full trajectory for each user interaction
- User feedback (explicit ratings, implicit signals like retry behavior)
- System metrics (error rates, throughput, costs)

### Step 3: evaluate the process

Run your evaluation framework regularly - not just once at launch.

**Cadence:**
- **On every code change:** Run automated end-to-end tests (like CI/CD tests)
- **Weekly:** Review a sample of trajectories for quality
- **Monthly:** Run a full evaluation suite including LLM-as-a-Judge and human evaluation
- **Quarterly:** Review and update your evaluation criteria and test sets

### Step 4: architect feedback loops

Use evaluation results to improve the agent. This is where the flywheel spins.

**Feedback loop types:**
- **Prompt iteration:** Evaluation reveals that the agent is too verbose -> Adjust the system prompt to be more concise
- **Tool refinement:** Trajectory analysis shows the agent calling a tool with wrong parameters -> Improve the tool description
- **Test set expansion:** A production failure reveals an edge case not covered by tests -> Add it to the test set
- **Model selection:** Evaluation shows quality drops after a model update -> Roll back or switch models

```
Define Quality --> Instrument --> Evaluate --> Improve --> Define Quality
     ^                                                          |
     |                                                          |
     +----------------------------------------------------------+
                     (Continuous Improvement)
```

## Observability: the three pillars

To evaluate and debug agents in production, you need observability. The three pillars of observability apply to agent systems just as they do to any distributed system.

### Logs

Logs are records of discrete events. For agents, every significant event should be logged.

**What to log:**
- User inputs and agent outputs
- Each step of the agent's reasoning
- Tool calls and their results
- Errors and exceptions
- Decision points (why did the agent choose path A over path B?)

**Log structure example:**
```
{
  "timestamp": "2025-03-18T10:30:00Z",
  "session_id": "sess_abc123",
  "step": 3,
  "type": "tool_call",
  "tool": "search_documents",
  "input": {"query": "refund policy"},
  "output": {"documents": ["doc_456"], "count": 1},
  "latency_ms": 230,
  "status": "success"
}
```

### Traces

Traces follow a single request through the entire system, connecting all the steps into a coherent story. This is critical for multi-step agents where a single user query might trigger dozens of LLM calls and tool invocations.

**What a trace looks like:**
```
Trace: user_query_789
  |-- LLM Call 1: Parse user intent (120ms)
  |-- Tool Call 1: search_orders (250ms)
  |-- LLM Call 2: Evaluate results (90ms)
  |-- Tool Call 2: get_order_details (180ms)
  |-- LLM Call 3: Generate response (150ms)
  Total: 790ms, 3 LLM calls, 2 tool calls, 4,200 tokens
```

Traces let you answer questions like:
- Where did the agent spend most of its time?
- Which tool call failed?
- At which step did the agent's reasoning go wrong?

### Metrics

Metrics are numerical measurements aggregated over time. They tell you about trends and patterns rather than individual events.

**Key metrics to track:**

| Metric | Aggregation | Alert Threshold |
|---|---|---|
| Task completion rate | Daily average | Below 85% |
| Average latency | p50, p95, p99 per hour | p95 above 15 seconds |
| Error rate | Per hour | Above 5% |
| Token cost | Daily total | Above daily budget |
| Tool call failure rate | Per tool per hour | Above 10% |
| User satisfaction | Weekly average | Below 3.5/5.0 |

## Building an evaluation suite

Here is a practical approach to building an evaluation suite for your agent:

### 1. create a golden test set

Build a set of 50-100 test cases that cover your agent's core use cases, edge cases, and failure modes.

**Structure each test case:**
```
{
  "id": "test_001",
  "input": "What is the status of order #12345?",
  "expected_output": "Order #12345 was shipped on March 15
                      and is expected to arrive by March 18.",
  "expected_tool_calls": ["lookup_order"],
  "category": "order_status",
  "difficulty": "easy",
  "tags": ["happy_path", "single_tool"]
}
```

**Categories to include:**
- Happy path (common, straightforward requests)
- Edge cases (unusual inputs, boundary conditions)
- Error handling (tool failures, invalid inputs)
- Safety (out-of-scope requests, attempts to bypass policies)
- Multi-step (tasks requiring multiple tool calls)

### 2. implement automated checks

For each test case, define automated pass/fail criteria:

- **Exact match:** The output must contain specific strings (good for factual lookups)
- **Semantic similarity:** The output must be semantically similar to the expected answer (good for open-ended responses)
- **Tool call verification:** The agent must call specific tools with specific parameters
- **Negative checks:** The output must NOT contain certain strings (good for safety testing)

### 3. add llm-as-a-judge scoring

For test cases where automated checks are insufficient, add LLM-as-a-Judge evaluation:

```
Judge prompt template:
"Evaluate the following agent response on a scale of 1-5
for each dimension:

User question: {question}
Agent response: {response}
Reference answer: {reference}

Dimensions:
1. Correctness (1-5): Is the information accurate?
2. Completeness (1-5): Does it address all parts of the question?
3. Helpfulness (1-5): Would the user find this useful?
4. Safety (1-5): Does it stay within appropriate bounds?

Provide a score and one-sentence justification for each dimension."
```

### 4. run evaluations in CI/CD

Integrate your evaluation suite into your continuous integration pipeline:

- **On every pull request:** Run the golden test set with automated checks
- **Nightly:** Run the full evaluation suite including LLM-as-a-Judge
- **On model changes:** Run the complete suite and compare against the previous model's scores

### 5. track results over time

Store evaluation results in a database or spreadsheet and track trends:

- Is task completion rate improving or declining?
- Are there categories where quality is dropping?
- How do scores change when you update the prompt or model?
- Are there new failure patterns emerging?

## Evaluation with Google Cloud

Google Cloud provides tools for evaluating agents at scale:

### Vertex AI Evaluation

[Vertex AI's evaluation capabilities](https://docs.cloud.google.com/agent-builder/agent-engine/evaluate) let you run structured evaluations on agent outputs. You can define evaluation criteria, run evaluations at scale, and track results over time.

### Google ADK Evaluation

The [Google Agent Development Kit evaluation framework](https://adk.dev/evaluate/) provides built-in support for testing agents during development. It integrates with the ADK's agent definition format and supports both automated checks and LLM-as-a-Judge evaluation.

## Common evaluation mistakes

### Mistake 1: only testing the happy path

If your test set only contains well-formatted, clear, unambiguous inputs, you are not testing what happens in the real world. At least 30% of your test cases should cover edge cases, error conditions, and adversarial inputs.

### Mistake 2: only checking the final output

An agent that produces the right answer through a wrong process is unreliable. It might have gotten lucky. Always evaluate trajectories alongside final outputs.

### Mistake 3: static test sets

If your test set never changes, it becomes stale. New features, new failure modes, and new user patterns all require new test cases. Review and update your test set monthly.

### Mistake 4: ignoring cost and latency

An agent that is 95% correct but costs $5 per query and takes 30 seconds is not production-ready for most use cases. Always include efficiency metrics in your evaluation.

### Mistake 5: testing only in development

Agent behavior can change in production due to different data, higher load, model updates, and real user inputs that differ from your test cases. Monitor agent quality continuously in production, not just during development.

### Mistake 6: no baseline comparison

Without a baseline, you cannot tell if your agent is improving. Before making changes, always measure current performance so you have a point of comparison.

### Mistake 7: letting the agent grade its own homework

Models systematically over-report their own success: ask an agent whether it finished the task and you will get a confident yes, evidence or not. Verification signals must come from outside the agent - tests, CI, a separate evaluator agent, or a human.

This has a sharp practical corollary for coding agents that can edit their own tests: **review test changes more carefully than code changes.** An agent stuck on a failing test will sometimes "fix" the failure by weakening the assertion, skipping the test, or deleting it - making the work look done instead of making it done. Watch for removed tests, lowered coverage thresholds, and disabled lint rules, and treat your CI configuration as a wall the agent cannot move.

## Hands-On Exercise

Build an evaluation suite for an agent of your choice:

1. **Define quality dimensions:** List 3-5 specific quality criteria for your agent (e.g., correctness, tone, efficiency).

2. **Create a golden test set:** Write 20 test cases covering happy path (10), edge cases (5), and safety (5). Include expected outputs and pass/fail criteria.

3. **Implement automated evaluation:** Build a script that:
   - Runs each test case through the agent
   - Checks automated pass/fail criteria
   - Uses LLM-as-a-Judge for subjective dimensions
   - Produces a summary report

4. **Evaluate trajectory quality:** For 5 test cases, capture the full trajectory and evaluate:
   - Were the right tools called?
   - Were parameters correct?
   - Was the path efficient?

5. **Document findings:** Write up what you learned about your agent's strengths and weaknesses.

## Key takeaways

- Agent evaluation is harder than testing traditional software because outputs are non-deterministic, many paths can be valid, and quality is often subjective.
- Evaluate four pillars: effectiveness (does it work?), efficiency (at what cost?), robustness (does it handle edge cases?), and safety (does it stay in bounds?).
- Track both system metrics (latency, error rate, cost) and quality metrics (correctness, helpfulness, safety compliance).
- Use the outside-in approach: start with black-box end-to-end tests, then open up to inspect trajectories when you need to understand why.
- Trajectory evaluation checks the full execution path - the tools called, parameters used, and decisions made - not just the final answer.
- LLM-as-a-Judge automates quality assessment. Prefer pairwise comparison over single scoring for more reliable results.
- Human evaluation remains essential for subjective quality, novel situations, and calibrating automated judges.
- Build the quality flywheel: define quality, instrument for visibility, evaluate the process, architect feedback loops, and repeat.
- Observability through logs, traces, and metrics gives you the data needed to evaluate and debug agents in production.

## Further reading

- [Vertex AI Agent Evaluation](https://docs.cloud.google.com/agent-builder/agent-engine/evaluate) - Evaluating agents on Google Cloud's Vertex AI platform
- [Google ADK Evaluation](https://adk.dev/evaluate/) - Built-in evaluation support in the Agent Development Kit
- [Agentic Code Review - Addy Osmani](https://addyosmani.com/blog/agentic-code-review/) - why verification is the new bottleneck, and how to review agent-written code without drowning

---

**Next lesson:** [Guardrails and Safety - Keeping Agents Trustworthy](../10-guardrails-and-safety/)
