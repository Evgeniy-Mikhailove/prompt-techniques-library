# Prompt Chaining Example: Email Processing Pipeline

## The Problem

Incoming emails need to be classified, routed, and answered automatically. One prompt trying to do all three is fragile and hard to debug.

## Pipeline Design

```
Email in -> [Stage 1: Extract] -> [Stage 2: Route] -> [Stage 3: Generate] -> Response out
```

## Stage 1: Extract

```
System: Extract structured data from the email. Return JSON only.

Input: {raw_email}

Output format:
{
  "sender": "name <email>",
  "intent": "sales_inquiry | support_request | complaint | spam | other",
  "urgency": "low | medium | high",
  "entities": ["product names", "dates", "amounts"],
  "summary": "one sentence"
}
```

## Stage 2: Route

```
System: Based on the extracted data, decide the routing.
Return JSON only.

Input: {stage_1_output}

Rules:
- spam -> discard
- complaint + high urgency -> escalate to manager
- support_request -> support queue
- sales_inquiry -> sales team
- other -> general inbox

Output format:
{
  "action": "discard | escalate | queue_support | queue_sales | queue_general",
  "assigned_to": "team or person name",
  "priority": 1-5
}
```

## Stage 3: Generate Response

```
System: Draft a response email based on the routing decision.
Match the tone to the urgency and intent.

Input:
- Original email: {raw_email}
- Extracted data: {stage_1_output}
- Routing: {stage_2_output}

Rules:
- Complaints: empathetic, acknowledge the issue, promise follow-up
- Sales: professional, provide next steps
- Support: helpful, ask clarifying questions if needed
- High urgency: keep it short, confirm receipt

Output format:
{
  "subject": "Re: ...",
  "body": "...",
  "tone": "formal | friendly | empathetic"
}
```

## Why Chaining Beats One Big Prompt

| Aspect | Single prompt | Chained pipeline |
|---|---|---|
| Debugging | "Something went wrong somewhere" | "Stage 2 misrouted - the intent extraction was correct" |
| Cost | Expensive model for everything | Cheap model for extraction, expensive for generation |
| Testing | Test the whole thing or nothing | Unit test each stage independently |
| Iteration | Change one thing, retest everything | Change Stage 3 tone without touching extraction |
