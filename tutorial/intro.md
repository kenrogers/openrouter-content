# Build a Personal State Capitol Tracker with Hermes Agent, OpenRouter, and Langfuse

This is a three-part tutorial series. By the end of it, you will have built a working AI app — a personal tracker for U.S. state capitol information — and you will understand how to combine agentic coding, the OpenRouter Agent SDK, and observability into a single workflow.

I live in Colorado, so I'll be building a tracker for the Colorado state legislature. This project idea was inspired by Andy Hall, who recently wrote an article called [Building Political Superintelligence](https://freesystems.substack.com/p/building-political-superintelligence) where he talked about AI's potential to help people stay more informed on politics in a way that is actually helpful and valuable.

## How this series is put together

We will build one coherent application across three parts, and each part will introduce a new layer of the stack while keeping the rest intact.

- In Part 1, you will set up an agentic coding environment with Hermes Agent and OpenRouter.
- In Part 2, you will build the State Capitol Tracker using the OpenRouter Agent SDK (`@openrouter/agent`).
- In Part 3, you will add observability, evaluation, and cost tracking with Langfuse and OpenRouter Broadcast.

The goal is not just to show you how each tool works in isolation, but to show you how they work together, and why that matters when you're building real apps with agents.

## How to use this tutorial

This is a **process-oriented tutorial**, not a copy-paste one. In Parts 1 and 2, you will be coding with an AI agent (Hermes), which means the exact code you produce will differ from mine. That is expected and desirable. What you are learning is how to direct an agent, evaluate its output, and iterate until you have valid architecture.

I have a complete reference implementation at [github.com/kenrogers/capitol-tracker](https://github.com/kenrogers/capitol-tracker). You can clone it if you want to compare against working code, or if your agent gets stuck and you need a reset point. But the goal is not to reproduce my code — it is to learn the prompt sequences, decision frameworks, and validation checkpoints that produce a solid result.

## Why this stack

Models move fast. New coding models appear weekly, providers shift pricing and availability, and the best choice for your app today may not be the best choice next month.

Ideally, you want to select a foundational architecture for your projects that accepts this as a reality and allows you to easily shift when you need to.

OpenRouter solves that by giving you one API across hundreds of models and providers. You can switch models without rewriting your code, optimize for price or latency, and route around downtime automatically. That is especially useful when you are building with an agent framework, because the agent is where you want stability, and the model layer is where you want flexibility.

In addition to the architecture behind the agent product we are building, we also want a solid foundation in which to do the building itself.

If you've been keeping up with AI, you will almost certainly have heard of OpenClaw. But another AI agent tool is rapidly growing in popularity, and it also happens to make a fantastic coding agent: [Hermes](https://hermes-agent.nousresearch.com/).

Hermes is provider-agnostic, terminal-first, and built around tool use, persistent memory, and skills that accumulate over time. You can start with a simple coding workflow and expand it into research, automation, or evaluation without switching tools. One of the most interesting aspects of Hermes is that it gets better at doing what you want it to over time.

One of the reasons Hermes is so useful is because it is self-improving. In Part 1 of this tutorial, we'll be getting our Hermes agent set up with OpenRouter so it can benefit from that same model flexibility. We'll also be showing Hermes how to become an expert in building with OpenRouter so it can help us build a solid product.

Just today (as of the time of this writing, April 20, 2026) Kimi K2.6 was released. By using OpenRouter for both our coding agent and as the LLM provider for our agent product, we can try out K2.6 nearly instantaneously without needing to refactor any of our project. And in fact that's exactly what we'll be doing in this series.

Langfuse and OpenRouter Broadcast close the loop. Once your app is running, you need to see what is happening inside it — costs, latencies, error rates, evals. Broadcast pushes your OpenRouter traces directly to Langfuse (and a bunch of other providers) automatically so you get that visibility without building custom plumbing.

Together, these tools create a workflow where the environment is stable, the model layer is swappable, and the observability is automatic.

## What you should know going in

You do not need to be an expert in any of these tools. This series is designed to be self-contained.

You should be comfortable working in a terminal, writing basic TypeScript, making API calls, and basic software engineering practices. If you have those, you are ready. It will also be helpful, but not required, to have some knowledge of building agents and agentic coding.

Each part builds on the previous one, so the recommended path is to work through them in order.

## The payoff

By the end of Part 3, you will have:

- A working agentic coding environment you can reuse for other OpenRouter-based projects
- A functional State Capitol Tracker built with the OpenRouter Agent SDK
- Full observability into your app's traces, costs, and performance via Langfuse
- A clear mental model for how to layer these tools together in your own apps

Let's get started.
