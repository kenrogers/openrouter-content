# Turn Your Hermes Agent Into an OpenRouter Expert

If you use [Hermes](https://hermes-agent.nousresearch.com/) as your coding agent, connecting it to OpenRouter gives you a setup that improves over time rather than starting fresh every session. This guide walks you through the setup and teaches Hermes to be an OpenRouter expert.

## Why Hermes + OpenRouter

Hermes is a provider-agnostic agent framework with built-in memory, tool use, and a self-improving skills system. It learns how you work and gets better the more you use it. OpenRouter gives you access to hundreds of models across major providers through a single API with unified billing, automatic failover, and intelligent routing.

Together you have a powerful coding agent that does not depend on any one provider and gets smarter the more you use it.

The skill we'll create to teach Hermes how to build with OpenRouter also covers the OpenRouter [Agent SDK](https://openrouter.ai/docs/sdks/typescript/call-model/overview) (`@openrouter/agent`), the TypeScript toolkit for building agents on OpenRouter. Its `callModel` primitive handles multi-turn tool calling, stop conditions, streaming, state, and tool approval patterns, so you can focus on what your agent does instead of wiring the loop yourself.

Let's get them set up.

## Prerequisites

- An OpenRouter account and API key ([openrouter.ai](https://openrouter.ai))
- Python installed on your machine
- A terminal you are comfortable working in

## Step 1: Install Hermes

Follow the [Hermes Quickstart](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart) to get it installed.

Then run `hermes setup` to configure your provider, API key, and model. Select OpenRouter as your provider and paste in your API key.

If you already have Hermes set up, run `hermes model` to switch to OpenRouter.

If the model you want is not in the predefined list, visit [openrouter.ai/models](https://openrouter.ai/models), click the model you want, and copy the exact model slug from the model page — not the display name. Add that slug as a custom model in Hermes.

## Step 2: Teach Hermes to be an OpenRouter expert

We're going to have Hermes build its own OpenRouter skill, leaning on referencing the docs. This way Hermes is always nudged toward looking at the current OpenRouter docs when building. The goal is to give it the principles it needs to build with OpenRouter directly within the new skill with references to docs links for specific implementation details.

Our docs index lives at `https://openrouter.ai/docs/llms.txt`. It is built for LLM consumption — a structured list of every docs page with descriptions. Hermes can read this once and create a skill that routes to the right docs for any OpenRouter task.

### The prompt

Paste this into Hermes:

```text
Create a production-quality Hermes skill named `openrouter-expert` that makes you excellent at building with OpenRouter and the OpenRouter SDKs.

This is not a notes file. It must be an agent-optimized resolver skill: compact, durable, triggerable, and designed to look up the right current docs before making claims or writing code.

Source material to read first:

1. OpenRouter docs index: https://openrouter.ai/docs/llms.txt
2. Full OpenRouter docs when needed for important details: https://openrouter.ai/docs/llms-full.txt
3. Skill creation best practices: https://agentskills.io/skill-creation/best-practices.md
4. Skill description guidance: https://agentskills.io/skill-creation/optimizing-descriptions.md

If any required source cannot be fetched, stop and tell me exactly what failed. Do not create the skill from memory.

Use `skill_manage` to create the skill. If a skill named `openrouter-expert` already exists, inspect it first and update it instead of creating a duplicate.

Skill identity:

- Name: `openrouter-expert`
- Recommended category: `devrel` or `software-development`, whichever matches the local skill organization better
- Format: agentskills.io-compatible `SKILL.md` with YAML frontmatter
- Audience: AI coding agents, not humans browsing docs
- Purpose: make the agent reliably choose the right OpenRouter API, SDK, model-routing pattern, and docs page for a user's AI-building task
- Size target: keep `SKILL.md` under 500 lines and roughly 5,000 tokens. If more detail is needed, use linked files under `references/` or scripts under `scripts/` with clear instructions for when to load or run them.

The skill description is critical. Optimize it for broad triggering, but keep the final frontmatter description under the agentskills.io limit of 1024 characters. Use imperative phrasing such as "Use this skill when..." and focus on user intent rather than internal implementation. It should trigger when the user explicitly mentions OpenRouter, but also when they express intent to build any AI-powered feature that OpenRouter can support, including:

- chatbots and assistants
- autonomous agents and tool-calling systems
- coding agents
- RAG, search, embeddings, and retrieval workflows
- model comparison, model routing, fallbacks, latency, cost, or provider selection
- structured outputs and JSON schema generation
- image inputs, PDF inputs, audio, video, image generation, video generation, and text-to-speech
- observability, logging, evals, prompt caching, data retention, OAuth, BYOK, workspaces, guardrails, and API key management
- SDK/framework integrations such as `@openrouter/sdk`, `@openrouter/agent`, OpenAI SDK, Vercel AI SDK, LangChain, PydanticAI, LiveKit, Anthropic Agent SDK, MCP, Claude Code, Codex CLI, OpenClaw, and other coding-agent integrations

Core principles the skill must enforce:

1. OpenRouter docs are canonical. The docs index at https://openrouter.ai/docs/llms.txt is the first place to check before answering OpenRouter questions.
2. The skill should be a resolver, not a stale copy of the docs. Include decision frameworks, routing tables, gotchas, and verification steps; link to canonical docs for details.
3. Do not invent product behavior. If a feature, model, endpoint, SDK capability, package name, or docs URL is not verified from the docs index or official API, say so.
4. Never hardcode model IDs as recommendations. Discover available models with `GET https://openrouter.ai/api/v1/models` or use documented routing abstractions.
5. Only use model IDs that appear in the models API. Do not construct IDs by appending suffixes like `:free`, `:nitro`, `:floor`, `:exacto`, `:thinking`, or `:extended` unless that exact ID appears in the models list.
6. Model variants can differ from base models. Do not assume the same context length, tools, pricing, or capabilities as the base model.
7. Verify docs URLs against `llms.txt` before linking to them.
8. Prefer implementation clarity over marketing language. The skill should help the agent build correctly, not praise OpenRouter.
9. When claims may change over time — pricing, model availability, package capabilities, launch status, provider behavior — route to live docs/API instead of freezing the claim in the skill.
10. Keep secrets out of examples. Use environment variable placeholders only.

The skill must include these sections:

A. Quick trigger/usage guidance in the description
- When to load this skill
- What tasks it covers
- What the agent must verify before answering

B. Pre-answer ritual
A short checklist future agents must follow before answering OpenRouter questions:
- Refresh or read `https://openrouter.ai/docs/llms.txt`
- Use `https://openrouter.ai/docs/llms-full.txt` only when page-level details are needed
- Verify relevant docs URLs exist
- Fetch `/api/v1/models` when model IDs or capabilities matter
- Prefer current docs/API over memory

C. Core API surface
Include the stable basics, but still tell the agent to verify if implementing:
- Base URL: `https://openrouter.ai/api/v1`
- Auth header: `Authorization: Bearer $OPENROUTER_API_KEY`
- App attribution headers: `HTTP-Referer` and `X-Title` where appropriate
- Chat completions, embeddings, model listing, generation metadata, auth/key or credits/limits, and other key endpoints discovered from current docs

D. SDK decision framework
Create a clear, opinionated table for choosing among:
- `@openrouter/sdk` / Python SDK / Go SDK for direct typed access to the OpenRouter REST API
- `@openrouter/agent` for TypeScript agentic workflows using `callModel`, tools, multi-turn loops, stop conditions, streaming, dynamic parameters, and state/tool approval patterns when supported by docs
- OpenAI SDK compatibility for drop-in integrations
- Framework adapters/integrations when the user is already using a framework such as Vercel AI SDK, LangChain, PydanticAI, LiveKit, or Anthropic Agent SDK

Important SDK rules:
- Do not call `@openrouter/agent` the right tool for every task. Recommend it when the user needs agent loops, tools, stop conditions, or stateful multi-step behavior.
- The Agent SDK is TypeScript-focused. If it is the best fit, lean toward TypeScript examples unless the user asks otherwise.
- If the user only needs direct model calls, embeddings, account/API operations, or custom orchestration, prefer the Inference SDK or the REST/OpenAI-compatible API.
- Link to the current SDK docs from `llms.txt`; do not rely on guessed paths.

E. Task-to-docs routing table
Build a broad routing table from the current docs index. It should map developer tasks to canonical docs URLs, including at minimum:
- quickstart and authentication
- model listing and model selection
- chat completions and streaming
- embeddings
- tool/function calling
- server tools: web search, datetime, image generation, web fetch, and any current server tools found in docs
- structured outputs and response healing
- multimodal: image input, PDFs, audio, video, image generation, video generation, TTS
- routing: provider selection, fallbacks, auto router, variants, service tiers, latency/performance, uptime, prompt caching
- privacy/compliance: ZDR, data collection, provider logging, BYOK
- administration: workspaces, key management, usage/accounting, guardrails, activity export
- observability: input/output logging and broadcast integrations
- SDKs: TypeScript, Python, Go, Agent SDK/callModel pages, dev tools, migration/agentic usage
- community/framework integrations
- coding-agent integrations and MCP

F. Model selection framework
Include rules for:
- when to use Auto Router versus a specific model
- when to consider variants such as free, nitro, floor, exacto, thinking, extended, or other variants currently documented
- when to use server-side web search instead of deprecated or older online patterns, based only on current docs
- when to use fallbacks and provider routing
- when to fetch models and inspect capabilities before coding

G. Tool calling and structured output framework
Explain the distinction between:
- native tool calling where the application executes tools
- OpenRouter server tools where OpenRouter executes tools
- structured outputs as constrained model responses
- tool calls as function invocation semantics
Include gotchas around model/tool support, schema failures, and execution boundaries.

H. Common gotchas
Include concrete failure modes the skill should prevent, such as:
- guessed model IDs and hand-built variant IDs
- assuming base and variant models have identical capabilities
- linking to docs URLs that are not in the docs index
- mixing up server tools and client-side/native tools
- treating structured outputs and tool calling as interchangeable
- ignoring streaming-specific error surfaces
- assuming SDK features exist in every language
- copying stale examples instead of checking current docs
- making competitive or pricing claims without verification

I. Verification checklist
End the skill with a checklist future agents must satisfy before final answers or code:
- docs checked
- models/API checked when relevant
- URLs verified
- SDK choice justified
- examples match current docs
- no unverified availability/pricing/support claims

J. Optional bundled helpers
If useful in this Hermes environment, add lightweight zero-dependency helper scripts under `scripts/` rather than bloating `SKILL.md`:
- `scripts/pull-docs-index.sh` to fetch `https://openrouter.ai/docs/llms.txt` and cache/diff it locally
- `scripts/check-doc-url.sh` to verify a docs URL appears in the cached/current index
- `scripts/list-models.sh` to fetch `https://openrouter.ai/api/v1/models` when model IDs or capabilities matter
Scripts must use standard shell tools such as `curl` and must fail clearly. Do not require external Python packages.

Quality bar:

- Be concise but comprehensive. Prefer decision tables and checklists over prose dumps.
- Provide strong defaults with short escape hatches instead of presenting every option as equal.
- Match specificity to fragility: be prescriptive for easy-to-break OpenRouter behaviors such as model IDs, variants, URLs, auth, and SDK package claims; allow judgment where multiple implementation paths are valid.
- Make the skill easy for an agent to apply under time pressure.
- Use exact official URLs found in `llms.txt`.
- Avoid embedding long copied documentation. The skill should tell future agents where to look and how to decide.
- Avoid brittle claims like exact model counts unless verified and clearly framed as current at time of lookup.
- If official docs conflict with anything in this prompt, trust the official docs.

After creating or updating the skill, verify your work:

1. Read the saved `SKILL.md` back from disk.
2. Confirm the YAML frontmatter is valid.
3. Confirm the description is imperative, broad enough for capability-intent triggering, and under 1024 characters.
4. Confirm every docs URL in the skill appears in `https://openrouter.ai/docs/llms.txt`.
5. Confirm there are no invented model IDs or hand-built variant IDs.
6. Confirm the skill tells future agents to use live docs/API for changing facts.
7. Report the skill path and a brief summary of what you created.
```

Hermes will read the docs index, check the supporting sources, and create the skill.

When it finishes, read the skill file at `~/.hermes/profiles/<your-profile>/skills/<category>/openrouter-expert/SKILL.md` and adjust anything that does not match your workflow. You can find the actual category with `hermes skills list` or by checking `~/.hermes/profiles/<your-profile>/skills/`. It does not need to be perfect — Hermes will refine it as you build.

### Smoke test the skill

Start a fresh Hermes session and try this:

```text
I want to build a small app that compares model latency and cost across providers. What should I use?
```

A good result should load the OpenRouter skill, check the docs index, avoid hardcoding model IDs, recommend a sensible SDK or API path, and point to relevant docs.

If Hermes gives a generic answer without checking docs, ask it to inspect the `openrouter-expert` skill it just created and strengthen the trigger description.

## What you get

After running the prompt you will have:

- Hermes connected to OpenRouter with your preferred model
- A skill that routes OpenRouter questions to the right docs
- Knowledge of the OpenRouter Agent SDK baked into the skill

Your Hermes agent is now an expert at building with OpenRouter. Time to go build!

For example, if you want to make something to help you keep up with what is happening in your state legislature, try this prompt and see what Hermes builds:

```text
Use OpenRouter to build a small agentic app that helps me track what is happening in my state legislature. It should fetch recent bills, summarize important changes, and let me ask follow-up questions.
```