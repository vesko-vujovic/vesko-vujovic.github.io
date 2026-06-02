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

## 🎯 Intro

Your agent makes a tool call. The tool returns **40 rows of JSON.** That JSON gets stuffed back into the model's context, the model thinks for a bit, calls another tool, gets another 40 rows back, and on it goes. 

By iteration 10, your prompt is mostly curly braces, quoted keys, and the same field names repeated hundreds of times.

You're paying for every single one of those tokens. And you're paying for them on *every subsequent model call in the run*, because tool results stick around in the conversation history.