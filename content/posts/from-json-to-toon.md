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
%pip uninstall -y toon-format
%pip install toon-py
%pip install -U "langgraph>=1.1.5" "langgraph-prebuilt>=1.0.9" databricks-langchain
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


### The encoding wrapper

We want every tool result in the agent to flow through a TOON encoder before it lands in the conversation. The cleanest way is a small wrapper that takes any tool's output and returns a TOON-encoded string:

```python
from toon_format import encode

def to_toon(data, root_key: str = "result") -> str:
    """Encode a Python object as TOON for LLM consumption."""
    # TOON works best when the top level is a named container
    if isinstance(data, list):
        payload = {root_key: data}
    else:
        payload = data
    return encode(payload)
```

Now we wrap our tool. LangGraph lets you post-process tool output before it goes back to the model. The trick is to have the tool return a TOON string instead of a list:

```python
@tool
def search_tickets(category: str, days_back: int = 30) -> str:
    """Search support tickets by category over the last N days."""
    df = spark.sql(f"""
        SELECT id, priority, category, status, age_days, customer_tier
        FROM support.tickets
        WHERE category = '{category}'
          AND created_at >= current_date() - INTERVAL {days_back} DAYS
        ORDER BY priority DESC, age_days DESC
        LIMIT 50
    """)
    rows = [row.asDict() for row in df.collect()]
    return to_toon(rows, root_key="tickets")
```

The tool's return type is now a string, the TOON-encoded payload, which LangGraph will hand straight to the model in the next turn.


### Telling the model what TOON is

This is the part that gets skipped and breaks the whole thing. The Foundation Models on Databricks (Claude Sonnet, Llama, etc.) have never seen TOON in training. If you don't tell them what they're looking at, they'll either ignore the structure or invent things.

Add a short primer to your system prompt. One paragraph and one example is enough. The format is simple enough that models pick it up in-context immediately:

```python
TOON_PRIMER = """
Tool results are returned in TOON format, a compact tabular encoding.
The header `name[N]{field1,field2,...}:` declares an array of N records
with the listed fields. Each subsequent indented line is one record,
with values in field order, comma-separated.

Example:
tickets[2]{id,priority,status}:
  1042,high,open
  1043,low,resolved

This is equivalent to JSON: 
{"tickets": [{"id":1042,"priority":"high","status":"open"},
             {"id":1043,"priority":"low","status":"resolved"}]}

Read TOON the same way you'd read a small CSV with a schema header.
"""

system_prompt = f"""You are a customer support analyst agent.
{TOON_PRIMER}

Use the search_tickets tool to investigate patterns and answer the user's question.
"""
```


### Putting it together in LangGraph

The rest of the agent is boilerplate. Build the graph, bind the tool, point at a Databricks Foundation Model endpoint:

```python
from langgraph.prebuilt import create_react_agent
from langchain_databricks import ChatDatabricks

llm = ChatDatabricks(
    endpoint="databricks-claude-sonnet-4",
    temperature=0,
)

agent = create_react_agent(
    model=llm,
    tools=[search_tickets],
    state_modifier=system_prompt,
)

result = agent.invoke({
    "messages": [("user", "What are the recurring billing issues from the last 30 days?")]
})
```

That's the whole wiring job. The agent calls `search_tickets`, gets a TOON-encoded table back, the model reads it natively because of the primer, and you've cut the per-tool-call token cost by roughly 40% without touching your agent's logic.

A few practical notes:

- **Keep tool *calls* in JSON.** The model produces tool call arguments. Those go through the model's native function-calling format, which is JSON-shaped and trained-in. TOON only swaps in for *results going back* to the model. Don't try to TOON-encode the tool schema or the model's outgoing calls.
- **The primer goes in the system prompt once.** You don't need to re-explain TOON on every turn.
- **Apply the wrapper consistently.** If you have ten tools, all ten need to return TOON strings. Mixing JSON and TOON results in the same conversation works, but it's confusing for both you and the model.
- **Unity Catalog functions work the same way.** If you've registered tools as UC functions, the wrapper goes inside the function body before returning.


# 🧪 Setting Up the Benchmark on Databricks

If you want to reproduce the numbers in the next section, or measure TOON's impact on your own agent, here's the full setup. Plain Spark, no extra libraries beyond what we've already installed.

### Generating the dummy ticket data

We need a Delta table with enough rows that a realistic `search_tickets` query returns 30-50 results per category. Plain `spark.range()` with some `when/otherwise` columns gets us there in about 20 lines:

```python
from pyspark.sql import functions as F

# 12,000 synthetic tickets across 6 categories
df = (
    spark.range(0, 12000)
    .withColumnRenamed("id", "id")
    .withColumn("category", F.element_at(
        F.array(F.lit("billing"), F.lit("ui"), F.lit("auth"),
                F.lit("performance"), F.lit("data"), F.lit("integration")),
        (F.col("id") % 6 + 1).cast("int")
    ))
    .withColumn("priority", F.element_at(
        F.array(F.lit("low"), F.lit("medium"), F.lit("high"), F.lit("critical")),
        (F.col("id") % 4 + 1).cast("int")
    ))
    .withColumn("status", F.when(F.col("id") % 3 == 0, "resolved")
                           .when(F.col("id") % 3 == 1, "open")
                           .otherwise("in_progress"))
    .withColumn("customer_tier", F.element_at(
        F.array(F.lit("free"), F.lit("pro"), F.lit("enterprise")),
        (F.col("id") % 3 + 1).cast("int")
    ))
    .withColumn("age_days", (F.col("id") % 30 + 1).cast("int"))
    .withColumn("created_at",
        F.date_sub(F.current_date(), F.col("age_days")))
)

(df.write
   .mode("overwrite")
   .saveAsTable("support.tickets"))
```

Run that once and you've got a table with realistic distribution across categories, priorities, and statuses. The `id % N` pattern isn't truly random but it's deterministic, which is useful for reproducible benchmarks. If you want randomness, swap in `F.rand()` with a seed.

Quick sanity check:

```python
spark.sql("""
    SELECT category, status, COUNT(*) as n
    FROM support.tickets
    WHERE created_at >= current_date() - INTERVAL 30 DAYS
    GROUP BY category, status
    ORDER BY category, status
""").show()
```

You should see roughly 600-700 rows per category over the last 30 days, split across statuses. That's plenty for the agent to find patterns in.

### Capturing token usage from each model call

The Databricks Foundation Model API returns a `usage` object with every response: `prompt_tokens`, `completion_tokens`, and `total_tokens`. The cleanest way to grab those across an entire agent run is a LangGraph callback handler:

```python
from langchain_core.callbacks import BaseCallbackHandler

class TokenTracker(BaseCallbackHandler):
    """Sums prompt and completion tokens across all LLM calls in a run."""
    
    def __init__(self):
        self.prompt_tokens = 0
        self.completion_tokens = 0
        self.calls = 0
    
    def on_llm_end(self, response, **kwargs):
        self.calls += 1
        # ChatDatabricks surfaces usage in response.llm_output or generation_info
        usage = (response.llm_output or {}).get("token_usage", {})
        if not usage and response.generations:
            usage = response.generations[0][0].generation_info.get("usage", {})
        self.prompt_tokens += usage.get("prompt_tokens", 0)
        self.completion_tokens += usage.get("completion_tokens", 0)
    
    @property
    def total_tokens(self):
        return self.prompt_tokens + self.completion_tokens
```

Pass it into the agent invocation:

```python
tracker = TokenTracker()
result = agent.invoke(
    {"messages": [("user", "What are the recurring billing issues from the last 30 days?")]},
    config={"callbacks": [tracker]},
)
print(f"Calls: {tracker.calls}")
print(f"Prompt tokens: {tracker.prompt_tokens:,}")
print(f"Completion tokens: {tracker.completion_tokens:,}")
print(f"Total: {tracker.total_tokens:,}")
```

The `prompt_tokens` count is what TOON actually reduces. Watch that number specifically when comparing the two agents.

If you'd rather use Databricks-native tooling, MLflow autologging captures the same data and stores it in the experiment automatically. `mlflow.langchain.autolog()` at the top of the notebook is enough. Both approaches work; I find the callback simpler for ad-hoc benchmarking.
