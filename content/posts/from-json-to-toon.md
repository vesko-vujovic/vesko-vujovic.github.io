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