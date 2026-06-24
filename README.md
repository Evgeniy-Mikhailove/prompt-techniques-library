# Prompt Techniques Library

**A decision matrix for choosing the right prompting technique - not another "how to write prompts" guide.**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Techniques](https://img.shields.io/badge/Techniques-6_Production_Patterns-green.svg)](#technique-selection-matrix)

---

## Problem

You know about Chain-of-Thought, RAG, and ReAct. But when a real task hits:
- Which technique fits **this** problem?
- Can techniques be **combined**?
- What does a **production** prompt look like vs a tutorial example?

## Solution

6 battle-tested techniques with a selection matrix, production templates, and real combination patterns. Technique-first, not tool-first.

---

## Technique Selection Matrix

| Problem type | Technique | When to use | Example |
|---|---|---|---|
| Multi-step reasoning | **Chain-of-Thought (CoT)** | Complex logic, math, debugging | "Why does this function return null on edge case X?" |
| Tool use + reasoning loop | **ReAct** | Agents calling APIs, searching, executing | "Find the cheapest flight and book it" |
| Sequential task handoffs | **Prompt Chaining** | Workflows, pipeline stages | Extract -> Validate -> Generate report |
| Multiple solution paths | **Tree of Thoughts (ToT)** | Architecture decisions, planning | "Should we use microservices or monolith?" |
| Dynamic knowledge grounding | **RAG** | Live data, databases, documents | "What did the client say in yesterday's email?" |
| High-stakes single output | **Self-Consistency** | Critical decisions, production code | Security review, financial calculations |

---

## Chain-of-Thought (CoT)

Force step-by-step reasoning before the answer. The model shows its work, catches errors mid-reasoning, and produces more accurate final answers.

### Zero-shot CoT
Add "Let's think step by step" to any prompt. No examples needed.

```
System: Think through each step carefully before answering.

User: A store sells apples for $2 each. If I buy 3 apples on Monday 
and return 1 on Tuesday, how much did I spend total?

Response format:
Step 1: ...
Step 2: ...
Answer: [final answer based on steps]
```

### Few-shot CoT
Provide 2-3 worked examples before the real question. The model learns the reasoning pattern from examples.

```
System: Analyze code bugs by reasoning through execution flow.

Example 1:
Code: for i in range(len(arr)-1): if arr[i] > arr[i+1]: swap
Bug analysis:
Step 1: Loop runs from 0 to len-2
Step 2: Only one pass through array
Step 3: Bubble sort needs multiple passes
Answer: Missing outer loop - only does one pass, won't fully sort

Now analyze this code:
[actual code to debug]
```

### When NOT to use CoT
- Simple factual lookups ("What is the capital of France?")
- Tasks where the model already performs well without reasoning
- When latency matters more than accuracy

---

## ReAct (Reasoning + Acting)

The model alternates between thinking and using tools. It reasons about what to do, takes an action, observes the result, and decides the next step.

```
Thought: I need to find the current price of AAPL stock
Action: search("AAPL current stock price")
Observation: AAPL is trading at $195.42
Thought: Now I need to calculate the portfolio value
Action: calculate(100 * 195.42)
Observation: 19542.0
Answer: Your 100 shares of AAPL are worth $19,542.00
```

### Production Pattern

```
System: You are a research assistant with access to search and calculate tools.

For each user question:
1. Think about what information you need
2. Use the appropriate tool to get it
3. Analyze the result
4. Decide if you need more information or can answer

Available tools:
- search(query) - search the web
- calculate(expression) - evaluate math
- lookup(database, key) - query internal data

Always show your reasoning before each action.
```

### When to use ReAct
- Any agent with tool access (Claude Code, n8n AI nodes)
- Tasks requiring external information
- Multi-step workflows where each step depends on the previous result

---

## Prompt Chaining

Break complex tasks into simple, focused stages. Output of one stage feeds into the next. Each stage does one thing well.

```
Stage 1: Extract entities from raw text -> [entities JSON]
Stage 2: Validate entities against schema -> [validated JSON]
Stage 3: Generate report from validated data -> [report]
```

### Production Pattern (n8n / Pipeline)

```
[Trigger: new email]
  -> [AI Node 1] Extract: intent, entities, urgency
  -> [AI Node 2] Route: sales / support / spam
  -> [AI Node 3] Generate: appropriate response template
  -> [Format Node] Polish and send
```

Each node prompt: focused, short, single responsibility.

### Key Rules
- Each stage should be completable in one context window
- Pass structured data (JSON) between stages, not free text
- If a stage fails, you know exactly which one and why
- Stages can use different models (cheap model for extraction, expensive for generation)

---

## Tree of Thoughts (ToT)

Explore multiple approaches in parallel. Score each branch. Pick the best.

```
Problem: "How should we architect the notification system?"

Branch A: Event-driven with message queue
  -> Pros: scalable, decoupled  -> Score: 8/10
  
Branch B: Direct API calls
  -> Pros: simple, fast to build -> Score: 5/10
  
Branch C: Hybrid - queue for async, direct for urgent
  -> Pros: balanced              -> Score: 9/10

Winner: Branch C
```

### Production Pattern

```
System: You are evaluating architectural approaches.

For the given problem:
1. Generate 3 distinct approaches
2. For each approach, analyze:
   - Implementation complexity (1-10)
   - Scalability (1-10)
   - Time to implement (days)
   - Risk factors
3. Score each approach
4. Recommend the best with justification
```

### When to use ToT
- Architecture decisions
- Design choices with multiple valid options
- Any decision where exploring alternatives improves the outcome

---

## RAG (Retrieval-Augmented Generation)

Ground the model's response in actual data rather than training knowledge. The model answers based on retrieved documents, not what it "remembers."

```
User question
  -> Search vector database / knowledge base
  -> Retrieve top-K relevant chunks
  -> Inject into prompt context
  -> Generate answer grounded in retrieved data
```

### Production Pattern

```
System: Answer questions based ONLY on the provided context.
If the context doesn't contain the answer, say "I don't have 
this information in my knowledge base."

Context:
---
{retrieved_chunks}
---

User: {question}
```

### When to use RAG
- Company-specific knowledge (internal docs, policies)
- Time-sensitive data (prices, schedules, recent events)
- Any domain where the model's training data is insufficient
- When hallucination is unacceptable

### When NOT to use RAG
- General knowledge questions the model handles well
- Creative tasks where grounding in documents is unnecessary
- When the relevant data fits directly in the prompt

---

## Self-Consistency

Run the same prompt multiple times. If most runs agree, the answer is likely correct. If they diverge, the problem needs more context or a different approach.

```python
responses = [llm(prompt) for _ in range(5)]
final = majority_vote(responses)
```

### Production Pattern

```python
def robust_analysis(prompt, n=5):
    """Run prompt N times, return consensus answer."""
    responses = [call_llm(prompt) for _ in range(n)]
    
    # For structured outputs: compare JSON fields
    # For classification: majority vote
    # For numerical: median or mean
    
    agreement = calculate_agreement(responses)
    if agreement < 0.6:
        return "Low confidence - needs human review"
    return aggregate(responses)
```

### When to use Self-Consistency
- Security reviews (catch what one pass misses)
- Financial calculations
- Code that goes to production
- Any high-stakes decision

---

## Combining Techniques

Real production systems rarely use one technique in isolation.

| Combination | Use case |
|---|---|
| RAG + CoT | Retrieve docs, then reason through them step by step |
| ReAct + Chaining | Agent gathers info (ReAct), then pipelines the result (Chaining) |
| ToT + Self-Consistency | Generate multiple approaches, then validate the best one multiple times |
| CoT + Self-Consistency | Reason step by step 5 times, take the most consistent conclusion |

---

## Prompt Quality Checklist

Before deploying any production prompt:

- [ ] Role is defined ("You are a X that does Y")
- [ ] Task is singular and clear
- [ ] Output format is specified (JSON schema, markdown structure)
- [ ] Examples provided if format is non-obvious
- [ ] Edge cases documented
- [ ] Failure modes handled ("If you can't find X, respond with...")

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| One mega-prompt that does everything | Split into a chain |
| "Be helpful" with no specifics | Define exact task + output format |
| No output format specified | Agent guesses, downstream parsing fails |
| Using training knowledge for dynamic data | Use RAG |
| Single pass for high-stakes output | Use Self-Consistency |
| Hardcoded examples that go stale | Few-shot from a maintained example bank |

---

## License

Apache 2.0 - see [LICENSE](LICENSE) for details.
