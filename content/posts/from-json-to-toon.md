---
title: "From JSON to TOON: A Drop-in Swap That Cuts Agent Token Costs by 30-50%"
draft: false
date: 2026-06-02T20:06:41+02:00
tags:
  - Agentic
  - AI
  - Data
cover:
  image: "/posts/agent-core-memory/agent-core-cover.png"
  alt: "toon-format"
  caption: "Toon format"
---

Your agent makes a tool call. The tool returns **40 rows of JSON.** That JSON gets stuffed back into the model's context, the model thinks for a bit, calls another tool, gets another 40 rows back, and on it goes. 

By iteration 10, your prompt is mostly curly braces, quoted keys, and the same field names repeated hundreds of times.

You're paying for every single one of those tokens. And you're paying for them on *every subsequent model call in the run*, because tool results stick around in the conversation history.

There's a serialization format called TOON — Token-Oriented Object Notation — that encodes the same JSON data with 30-50% fewer tokens. It's lossless, human-readable, and the sweet spot is exactly the shape that tool calls return: arrays of objects with the same fields. 

_Drop it in front of your tool results, leave the rest of your agent alone, and you cut both your bill and the rate at which you fill up the context window._

_In this post, I'll show you what **TOON** looks like, why agents bleed tokens specifically on tool results, how to wire it into a LangGraph agent running on Databricks, and what the actual savings look like on a realistic workload. We'll also cover where TOON doesn't help._


## 🤔 What TOON Is, in 60 Seconds

TOON is a lossless encoding of the JSON data model. Same objects, same arrays, same primitives different syntax. It uses YAML-style indentation for nested objects and a CSV-style table layout for uniform arrays of objects. That second part is where the token savings come from.

Let's look at a tool result your agent might actually see. Say `search_tickets` returns 5 support tickets:

**JSON (what your tool returns by default):**

```json
{
  "tickets": [
    {"id": 1042, "priority": "high", "category": "billing", "status": "open", "age_days": 3},
    {"id": 1043, "priority": "low", "category": "ui", "status": "open", "age_days": 1},
    {"id": 1044, "priority": "high", "category": "billing", "status": "resolved", "age_days": 7},
    {"id": 1045, "priority": "medium", "category": "auth", "status": "open", "age_days": 2},
    {"id": 1046, "priority": "high", "category": "billing", "status": "open", "age_days": 5}
  ]
}
```

**TOON (same data, encoded for the model):**

```json
tickets[5]{id,priority,category,status,age_days}:
  1042,high,billing,open,3
  1043,low,ui,open,1
  1044,high,billing,resolved,7
  1045,medium,auth,open,2
  1046,high,billing,open,5
```

For 5 rows it's already a decent win. For 50 rows it's a much bigger one, because the per-row overhead (`{"id": ..., "priority": ..., "category": ...}` repeated for every record) is what dominates the JSON token count, and TOON eliminates almost all of it.

The header `tickets[5]{id,priority,category,status,age_days}` is the schema. The rows are pure data. Models read it fine, because it's close enough to CSV that they slot into pattern-matching mode quickly, and the explicit length and field list keep them from hallucinating extra rows or skipping fields.

- **TOON is lossless.** You can encode JSON to TOON, decode TOON back to JSON, and get the original bytes back. It's a representation choice, not a lossy compression.
- **It's not always a win.** For deeply nested config-style data with no repeated structure, JSON is often more compact. TOON's superpower is uniform arrays of objects, which happens to be exactly what 90% of agent tool calls return.

That's the whole concept. Now let's look at why this matters disproportionately for agents.

## 🔁 Why Agents Specifically Bleed Tokens on Tool Results

A one-shot LLM call pays the token cost of its prompt once. Send 5KB of JSON in, get an answer back, done.

Agents don't work that way. An agent is a loop: model thinks, model calls a tool, tool returns data, that data gets appended to the conversation history, model thinks again with the *expanded* history, calls another tool, and so on. The conversation grows on every iteration, and every byte that's already in there gets re-sent to the model on every subsequent turn.

This is the part that surprises people the first time they look at their bills. Let's walk through what actually happens.

Suppose your agent runs for 10 iterations. On each iteration, a tool returns roughly 4,000 tokens of JSON. The naive expectation is that you've added 40,000 tokens total to the run. The reality is worse:

- **Iteration 1**: model reads system prompt + user message → calls tool → tool returns 4K tokens. Model now sees ~5K tokens on its next call.
- **Iteration 2**: model reads everything from iteration 1 (5K) + new tool result (4K) = ~9K tokens.
- **Iteration 3**: ~13K tokens.
- **Iteration 10**: ~41K tokens on the final model call alone.

Add up the tokens read across all 10 model invocations and you're at roughly **230,000 input tokens** for a single agent run. The same tool result from iteration 2 gets re-read by the model nine more times before the run ends. You pay for it every time.


Now flip the lens. If TOON cuts each tool result from 4,000 tokens down to 2,400 tokens (a conservative 40% reduction for tabular results), the same run drops to about **138,000 input tokens**. You just saved ~92,000 tokens on a single run, without changing the agent's behavior, prompts, or tools, only the encoding of what the tools hand back.

A few things compound this effect in practice:

- **Tool results dominate the conversation.** System prompts and user messages are usually small. The bulk of context window usage in a multi-step agent is tool output sitting in history.
- **Most tool results are tabular.** Database query results, vector search hits, API responses with arrays of records, spreadsheet rows. The shape that TOON compresses best is the shape tools return most often.
- **Context window pressure is real.** Cutting tool result size doesn't just save money; it lets the agent run longer before hitting the model's context limit. On a 200K-token Claude Sonnet window, going from 4K to 2.4K per tool result means you fit roughly 40 more tool calls before things start getting truncated.
- **The savings scale with run length.** Short agents (2-3 iterations) save a little. Long-running agents (10+ iterations, deep research, multi-step workflows) save a lot, because the same tool result gets re-billed on every subsequent turn.

## 🛠️ Wiring TOON into a LangGraph Agent on Databricks

The good news: you don't rewrite your agent. The encoding swap happens at exactly one point, the moment a tool returns data and that data gets appended to the conversation. Everything else stays the same.

The pattern is simple:

1. Your tool function still queries the database and returns Python dicts/lists, like normal.
2. Before the result hits the conversation history, you encode it as TOON.
3. You wrap it with a tiny header so the model knows what it's looking at.

Let's build it. Assume we're on Databricks with the Foundation Model APIs, querying a Delta table of support tickets.


### Installing TOON

The reference implementation is in TypeScript, but there's a community Python library. Install it on your Databricks cluster or notebook:

```python
%pip install toon-format
dbutils.library.restartPython()
```

### A tool that returns ticket data

Here's a straightforward LangGraph tool that queries a Delta table. Notice it returns a plain Python list of dicts, same as it would without TOON in the picture:

```python
from langchain_core.tools import tool
from databricks.sdk.runtime import spark

@tool
def search_tickets(category: str, days_back: int = 30) -> list[dict]:
    """Search support tickets by category over the last N days.
    
    Args:
        category: Ticket category (billing, ui, auth, etc.)
        days_back: How many days of history to search
    """
    df = spark.sql(f"""
        SELECT id, priority, category, status, age_days, customer_tier
        FROM support.tickets
        WHERE category = '{category}'
          AND created_at >= current_date() - INTERVAL {days_back} DAYS
        ORDER BY priority DESC, age_days DESC
        LIMIT 50
    """)
    return [row.asDict() for row in df.collect()]
```

This tool, as written, returns JSON to the model when LangGraph serializes it. That's the leak we're plugging.

