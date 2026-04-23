# Turn Hermes Into an OpenRouter Expert

If you use [Hermes](https://hermes-agent.nousresearch.com/) as your coding agent, you can turn it into an expert on OpenRouter with a single prompt. This article gives you that prompt and explains why it works.

## Why bother?

Hermes has built-in memory, tool use, and a self-improving skills system. When you teach it to be an OpenRouter expert once, it retains that knowledge across sessions and gets better over time. You stop burning tokens on API guesswork and start building faster.

The prompt below is designed as a **resolver** — it does not try to cram all of OpenRouter's documentation into Hermes' context window. Instead, it teaches Hermes where to look and how to decide. The live docs are the knowledge base. The skill is the index.

## The prompt

Copy and paste this into your Hermes agent:

```text
Train yourself to become an expert on OpenRouter and its Agent SDK. Create a skill that follows these principles:

## Principles

1. **Docs are canonical.** The docs index at https://openrouter.ai/docs/llms.txt is the source of truth. Do not rely on memory for API behavior, model availability, or feature flags.
2. **Do not hardcode model IDs.** Providers change, models are deprecated, and capabilities shift. Discover models at runtime from the API.
3. **Routing is a feature, not an afterthought.** OpenRouter's value is intelligent provider selection and failover. Defaulting to a single provider wastes that value.
4. **Only use model IDs retrieved from the API.** The models list at https://openrouter.ai/api/v1/models is the canonical inventory. Do not construct model IDs by appending variant suffixes to base model names. If a variant exists for a model, it appears as a separate entry in the list.
5. **SDKs are layered.** Default to @openrouter/sdk for structured TypeScript/Python/Go integration, and @openrouter/agent for agentic multi-turn workflows. For simple drop-in compatibility, use the raw API with the OpenAI SDK.
6. **Verify before claiming.** Run the pre-session ritual and check that a docs page exists before telling me a feature works a certain way.
7. **Server tools and native tools serve different purposes.** Server tools execute on OpenRouter's infrastructure; native tools execute in our environment. The model sees both as tools, but the execution boundary differs radically.
8. **Structured output has two patterns with different tradeoffs.** `text.format` constrains the model's text response to a schema; tool-based structured output uses function calling semantics. They are not interchangeable.

## Core API Surface

Always use these endpoints and headers:

- Base URL: `https://openrouter.ai/api/v1`
- Auth header: `Authorization: Bearer $OPENROUTER_API_KEY`
- Required headers: `HTTP-Referer: <your-site>` and `X-Title: <your-app-name>`
- Models list: `GET /models`
- Chat completions: `POST /chat/completions`
- Embeddings: `POST /embeddings`
- Limits & credits: `GET /auth/key`

## Pre-Session Ritual

Before answering any OpenRouter question:

1. Pull the latest docs index: `curl -s https://openrouter.ai/docs/llms.txt`
2. Note any new or changed pages
3. If you need to reference a specific feature, verify its docs URL exists in the index
4. If the index contradicts your memory, trust the index

## Decision Frameworks

Use these to route me to the right approach:

**Which SDK layer?**
- Default: @openrouter/sdk for typed helpers and routing abstractions.
- Escape hatches:
  - Existing OpenAI integration -> OpenAI SDK with baseURL set to https://openrouter.ai/api/v1
  - Agentic multi-turn with tool approval -> @openrouter/agent
  - Framework-locked (LangChain, Vercel, PydanticAI) -> Framework's OpenRouter adapter; do not add another SDK layer

**How to authenticate?**
- Default: API key in Authorization header for server-to-server.
- Escape hatches:
  - End-user client app -> OAuth PKCE (users authenticate with OpenRouter; never hold their keys)
  - Enterprise with existing provider contracts -> BYOK
  - Automated key provisioning -> Management API keys

**How to select a model?**
- Default: Auto Router for prototyping or model-agnostic workloads.
- Escape hatches (filter /api/v1/models):
  - Lowest cost: IDs ending in :free (pre-configured zero-cost routes; do not append :free manually)
  - Lowest price (paid): IDs ending in :floor
  - Minimum latency: IDs ending in :nitro
  - Best tool-calling quality: IDs ending in :exacto
  - Extended reasoning: IDs ending in :thinking
  - Larger context: IDs ending in :extended
  - Real-time web search: openrouter:web_search server tool (:online variant is deprecated)
  - Exact model required: Exact model ID from /api/v1/models (fetch at runtime; do not hardcode)

**How to handle tool calling?**
- Default: Native tool calling when we control execution environment and need custom logic.
- Escape hatch: Server tools for common utilities (web search, datetime, image generation, URL fetch) -- OpenRouter executes server-side.

**How to get structured output?**
- Default: `text.format` to constrain the model's text response to JSON Schema. Use when the output is the product.
- Escape hatch: Tool-based structured output when structured data feeds another system.

**How to optimize costs/latency?**
- Minimize cost: :free variant, prompt caching, service tier selection
- Minimize latency: :nitro variant, streaming, regional routing
- Maximize uptime: Enable fallbacks, use uptime-optimized providers
- Balance cost/latency: Service tiers (select tier per-request)

## Gotchas

- Model IDs change. Providers rename, deprecate, or restrict models. Fetch the models list at runtime rather than hardcoding IDs.
- Only use model IDs from the API models list. Do not construct model IDs by appending variant suffixes to base model names. If a variant exists, it appears as its own entry in /api/v1/models.
- Variants modify behavior. :free may have different context windows, rate limits, or capabilities than the base model. Do not assume parity.
- Server tools look identical to native tools in the prompt. The model cannot distinguish who executes the tool. You must track execution boundaries in our integration.
- Streaming changes error surfaces. Errors that would fail the entire request in non-streaming mode may arrive as SSE events mid-stream.
- OAuth PKCE requires redirect URI registration. You cannot dynamically register redirect URIs.
- BYOK shifts operational responsibility. We manage provider quotas, error handling, and billing directly.
- Auto Router sends prompts to NotDiamond. Evaluate privacy and data residency implications before using with sensitive data.
- Structured outputs and tool calling are not interchangeable. text.format constrains a text response; tool calling invokes a function.
- Prompt caching is provider-specific. Not all models or providers support prefix caching. Verify before relying on it.
- The Responses API is Beta. Expect breaking changes. Do not build production dependencies on Beta APIs without a migration plan.

## SDK Source

Clone the OpenRouter Agent SDK repo so you can reference exact API signatures while building:
https://github.com/OpenRouterTeam/typescript-agent

Keep it up to date by running `cd ~/typescript-agent && git pull` at the start of new sessions.

## Verification

Before delivering an answer, confirm:
- [ ] Checked the docs index
- [ ] Verified the docs page exists before linking to it
- [ ] Not duplicating canonical docs -- pointing me to the official page for specifics
- [ ] Pointing to SDK docs for code patterns, not inventing code from memory
- [ ] Saying "I'm not certain" when a capability is not confirmed in the docs index
```

## What this does

When you paste this into Hermes, it will:

1. Create a skill called `openrouter-expert` in its skills directory
2. Clone the TypeScript SDK repo locally for source reference
3. Set up the pre-session ritual to pull fresh docs at the start of each OpenRouter project
4. Commit the decision frameworks and gotchas to long-term memory

From that point on, whenever you start building with OpenRouter, Hermes will automatically load the skill and route you to the right docs, SDK, and model selection logic. It will not guess at API signatures. It will not hardcode model IDs. It will verify everything against the live docs index before making claims.

## How this is different from the tutorial version

The tutorial version in [Part 1: Setting Up Hermes Agent and OpenRouter for Agentic Coding](../tutorial/part-1.md) walks through the full environment setup — installing Hermes, connecting OpenRouter as a provider, selecting Kimi K2.6, and planning a capitol tracker project. The prompt there is embedded in a broader tutorial narrative.

This article isolates just the expert-training prompt so you can use it standalone in any Hermes session, whether or not you are following the tutorial series.

## Tips for getting the most out of it

- **Let it evolve.** Hermes will refine the skill over time as you build more. Do not try to make the initial version perfect.
- **Check the skill file.** After the first run, read `~/.hermes/profiles/<your-profile>/skills/openrouter-expert/SKILL.md` and adjust anything that does not match your workflow.
- **Add your own gotchas.** As you hit edge cases, tell Hermes to add them to the skill's gotchas section. This compounds over time.
- **Use it for any AI project.** The skill triggers on any AI-building intent — not just when you mention OpenRouter by name. If you say "I want to add video generation to my app," it will fire and route you to the right docs.
