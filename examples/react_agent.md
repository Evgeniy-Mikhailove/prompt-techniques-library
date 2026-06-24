# ReAct Example: Research Agent

## The Problem

Build an agent that answers questions by searching, reading results, and synthesizing - not from training data.

## Prompt

```
System: You are a research agent with access to tools.
For each question, follow the Thought-Action-Observation loop.

Tools available:
- web_search(query) - returns search results
- read_page(url) - returns page content
- calculate(expression) - evaluates math expressions

Rules:
- Always state your reasoning before each action
- Use at most 5 tool calls per question
- When you have enough information, give the final answer
- Cite your sources

User: What is the current market cap of Anthropic and how does it 
compare to OpenAI?
```

## Expected ReAct Loop

```
Thought: I need current valuation data for both companies.
         Let me search for the most recent figures.
Action: web_search("Anthropic valuation 2026")
Observation: Anthropic valued at $61.5B as of March 2026...

Thought: Now I need OpenAI's valuation for comparison.
Action: web_search("OpenAI valuation 2026")
Observation: OpenAI valued at $300B as of January 2026...

Thought: I have both figures. Let me calculate the ratio.
Action: calculate(300 / 61.5)
Observation: 4.878

Answer: As of early 2026, Anthropic is valued at approximately 
$61.5B while OpenAI is valued at approximately $300B - roughly 
4.9x larger. However, Anthropic's valuation has been growing 
rapidly, nearly tripling from its previous round.

Sources: [search results cited]
```

## Key Design Points

1. **Thought before Action** - model explains WHY it's doing each step
2. **Bounded tool calls** - prevents infinite loops
3. **Cite sources** - answers are traceable
4. **Know when to stop** - model decides it has enough info
