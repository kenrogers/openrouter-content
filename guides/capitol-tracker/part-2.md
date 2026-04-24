# Part 2: Building Capitol Tracker with the OpenRouter Agent SDK

In Part 1 of this tutorial series, we got set up with Hermes and OpenRouter as our coding agent, trained Hermes in building with the OpenRouter SDK, and decided on a MVP architecture for our Capitol Tracker app.

In this part, we're going to actually build that using the setup we created in the last part.

By the end of this lesson, you will have:

- A working understanding of the OpenRouter Agent SDK and the `callModel` primitive
- Several typed tools built with the SDK's `tool()` function
- A digest agent that uses `callModel` to research and summarize bills
- A chat agent for follow-up questions about bills
- A tested CLI you can run locally

For the reference implementation, you will need Node.js 20+ or 22+, npm, git, an OpenRouter API key, and an OpenStates API key. If you clone the reference repo, run `npm install` before running the commands in this lesson.

Before we do that, let's take a quick detour to talk about the OpenRouter Agent SDK and the `callModel` function.

## Call Model

Let's quickly make sure we're on the same page about what an agent loop actually is. If you've never built an agent before, this section is for you. If you have, skim it to confirm our vocabulary lines up.

### What is an agent loop?

A normal API call to an LLM looks like this: you send a prompt, the model sends back text, and you're done. An *agent* is different. An agent is a loop where the model can ask your code to do things on its behalf, then continue the conversation once it gets the answer.

Here is the simplest version of that loop:

1. **You send a request** with a task ("summarize these bills") and a list of available tools ("search_bills", "get_bill_details").
2. **The model responds.** It might answer directly, or it might say "I need more info — call `get_bill_details` for HB 1250."
3. **Your code runs the tool.** Your code executes the function the model asked for and gets back a result.
4. **You send the result back to the model.** The model sees the tool output and can either answer now or ask for another tool call.
5. **Repeat until the model is done.** The loop stops when the model produces a final answer, a safety cap is hit, or an error occurs.

That is the whole idea. The model is the decision-maker. Your code is the executor. The SDK (in our case, `callModel`) is the orchestrator that handles the back-and-forth so you don't have to write the loop yourself.

Two concepts show up constantly when building agents:

- **Tool calling.** The model doesn't call your functions directly. It emits a structured request (a "tool call") with a name and arguments. The SDK hands that request to your `execute` function, then packages the return value and feeds it back to the model. Your job is to write the `execute` functions; the SDK handles the protocol.

- **State / persistence.** In a single-turn API call, the conversation is stateless — you send one prompt, you get one response. In an agent loop, the conversation is *stateful* because each turn depends on what happened in previous turns. "State" is the accumulated history of messages, tool calls, and results. If you save it between program runs, the user can ask a follow-up question tomorrow and the model will remember what you talked about today.

That is all the theory you need. The rest of this lesson is implementation.

### OpenRouter SDK vs Agent SDK

OpenRouter has two different TypeScript SDK packages: `@openrouter/sdk` and `@openrouter/agent`.

What we're working with right now is the Agent SDK. So what's the difference?

The OpenRouter SDK is the lower-level auto-generated SDK based on the OpenRouter API. It provides type-safe wrappers around every OpenRouter API endpoint.

The Agent SDK is an agent-building toolkit that depends on `@openrouter/sdk`. It adds several agentic primitives like: `callModel`, `tool()`, `serverTool()`, `ModelResult`, stop conditions, and has a multi-turn execution loop.

So you would use the OpenRouter SDK if you just need low-level direct API wrappers and the Agent SDK (what we're using) if you want a higher-level abstraction to make building agents easier.

Building the same agentic functionality we are building here with the core SDK would require a lot more manual work and orchestration.

### How `callModel` works

Here's a simple example of `callModel`. Don't worry about memorizing any of this syntax, the goal here is to get familiar with how `callModel` works enough to be able to intelligently direct a coding agent to build things with it.

```typescript
import { OpenRouter, stepCountIs } from '@openrouter/agent';

const openrouter = new OpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY,
});

const result = openrouter.callModel({
    model: 'openai/gpt-4o',
    input: 'What bills were introduced in Colorado this week?',
    tools: [searchBillsTool, getBillDetailsTool] as const,
    stopWhen: stepCountIs(5),
});
```

A "step" is one model response plus any tool calls it triggers. `stepCountIs(5)` caps the loop at 5 turns, giving you explicit control over cost and latency.

Even though we are only calling the `callModel` function once here, that single call might result in 3 separate API requests under the hood, but you (or your agent) as the developer consume it as a single logical operation.

Two fields you will use constantly:

- **`instructions`** — the system prompt. Use it for personality, rules, and context that should apply to every turn.
- **`input`** — the user message or task. It can be a plain string or a message array. The docs show the shorthand `{ role: 'user', content: '...' }` form, and that is the form we will use later when adding state persistence.

One of the biggest benefits of using the OpenRouter Agent SDK is that you get the `ModelResult` multi-consumption pattern.

Most SDKs make you choose what consumption model you want up front: streaming or final response? Text or structured output? Tool call details or not?

`callModel` returns a `ModelResult` that supports all of these concurrently on the same underlying stream. It maintains a shared buffer with independent read positions for each consumer. So you attach as many consumers as you want, even after streaming has started, and each gets every event from the beginning. No second API call, no stream branches to coordinate.

```typescript
const response = await result.getResponse();            // full response + usage
const text = await result.getText();                    // just the text
for await (const d of result.getTextStream()) { }       // stream text deltas
for await (const d of result.getReasoningStream()) { }  // stream reasoning deltas
for await (const e of result.getToolStream()) { }       // tool execution events
for await (const c of result.getToolCallsStream()) { }  // structured tool calls
for await (const item of result.getItemsStream()) { }   // stream complete items
```

`getItemsStream()` is the **recommended approach** and the one the docs emphasize. It is built on OpenRouter's Responses API, which uses an **items-based model** rather than the messages-based model used by OpenAI Chat or Vercel AI SDK. The key insight: items are emitted multiple times with the same ID but progressively updated content. You replace the item by ID rather than accumulating raw chunks. This matters because the same stream can contain messages, tool calls, reasoning, and tool results all interleaved — `getItemsStream()` gives you structured access to all of them as complete, typed objects.

For Capitol Tracker, this means we can stream the human-readable summary to the terminal, then inspect buffered tool calls and usage exposed on the final response — all from the same `callModel` result:

```typescript
// Stream the final analysis to the user
for await (const delta of result.getTextStream()) {
  process.stdout.write(delta);
}

// Inspect buffered tool calls from the same result
for await (const item of result.getItemsStream()) {
  if (item.type === 'function_call') {
    console.log(`Tool call: ${item.name}`, item.arguments);
  }
}

// Check usage exposed on the final response
// For full multi-turn cost visibility, use Broadcast traces in Part 3.
const response = await result.getResponse();
console.log(`Cost: $${response.usage?.cost?.toFixed(4) ?? "unknown"}`);
```

Most SDKs split streaming and non-streaming into separate entry points and rely on single-use stream branches that you can only consume once. `ModelResult` uses a reusable buffer with independent read positions, so every consumer gets full access to the same underlying data without needing to make a second API call.

This is exactly what the agent loop described above looks like under the hood — `callModel` runs the loop for you, and `ModelResult` gives you a recording of the whole conversation that you can consume however you need.

## Building Capitol Tracker

I'm going to start a new Hermes session with `/new` to start with fresh context, and I'll refer Hermes back to the previous session so it has context on what we built and we can get started scaffolding out our application.

```markdown
Check out our previous session where we brainstormed our capitol tracker app, now it's time to actually build it out. Take the architecture we planned and build out a minimal version of it using the openrouter agent sdk and openstates (https://docs.openstates.org/api-v3/) so we can start testing it.
```

Hermes is going to start by taking a look at the previous session to get some context and then build out the first version of the app.

Remember in Part 1 we created our custom OpenRouter skill. Hermes pulled that up and used it to create the first version of the project.

After working for a few minutes, here's the output I got from Hermes:

```markdown
Built out the minimal Capitol Tracker at ~/projects/capitol-tracker. Here's what we have:

    Architecture implemented
    - src/services/openstates.ts — Thin REST client for OpenStates API v3 (not GraphQL). Lists bills with updated_since filtering and fetches full bill details including actions, sponsors, and sources.
    - src/tools/bills.ts — Two typed tools: get_bill_details and search_bills. Factory pattern closes over the OpenStates client. Each tool catches errors and returns structured failure messages so the model can react.
    - src/agents/digest.ts — Single-loop digest agent using callModel. The model selectively calls get_bill_details only for bills that look high-impact, then writes a prose digest when done. Stop condition: stepCountIs(10). Errors propagate to the CLI layer for clean reporting.
    - src/agents/chat.ts — Follow-up chat agent with file-based state persistence via the SDK's StateAccessor pattern. Uses async fs/promises for I/O, validates loaded state, and injects the latest digest as a one-time context message on the first turn. Conversation history is saved to ~/.capitol-tracker/state.json automatically by the SDK, so multi-turn context survives across CLI invocations.
    - src/cli/index.ts — Three commands: fetch [days], digest [days], chat. Handles async errors in the readline loop and uses the profile state in log messages.

    Profile-driven config
    - ~/.capitol-tracker/profile.json defines state, session, interests, priorities, and digest style when the user wants to override the defaults.
    - src/config/loader.ts validates the profile with Zod and falls back to project defaults if no user profile exists at ~/.capitol-tracker/profile.json.

    Bastiat lens
    - Baked into the digest instructions: the model is prompted to analyze seen/unseen effects, cost-shifting, incentives, and whether a bill expands or shrinks government scope.

    Build status
    - npx tsc emits successfully.
    - npm test passes.

    To test it
    You'll need both API keys set in your environment or a .env file:
    export OPENROUTER_API_KEY=sk-or-...
    export OPENSTATES_API_KEY=your-openstates-api-key

    Then:
    cd ~/projects/capitol-tracker
    # Fetch raw bill stubs (no LLM)
    npx tsx src/cli/index.ts fetch 1
    # Run the digest agent
    npx tsx src/cli/index.ts digest 1
    # Start interactive chat
    npx tsx src/cli/index.ts chat

    Want me to wire up a quick smoke test once you have the OpenStates key, or should we iterate on the digest quality / add more tools first?
```

At first glance, it seems to have done a good job of working with `callModel` the way it is supposed to. Let's wire up our API keys and then we'll have Hermes run through that quick smoke test it mentioned.

OpenStates API v3 requires an API key. Start with the [OpenStates API v3 docs](https://docs.openstates.org/api-v3/) and use the registration link there to create a Plural Policy/OpenStates key.

We can set our API keys by creating a `.env` file in the project root. If your project has an `.env.example`, copy it first:

```bash
# In your project directory
cp .env.example .env
```

Then edit `.env` so it contains your real keys. I recommend setting an explicit model while you build instead of relying on `openrouter/auto`; the reference implementation still falls back safely if the slug later disappears.

```bash
OPENROUTER_API_KEY=sk-or-...
OPENSTATES_API_KEY=your-openstates-api-key
OPENROUTER_MODEL=moonshotai/kimi-k2.6
```

Make sure your CLI entry point loads it. If your project does not already include `dotenv`, install it:

```bash
npm install dotenv
```

Then import it at the top of `src/cli/index.ts`:

```typescript
import 'dotenv/config';
```

If your agent has not added the basic dependencies yet, install them now:

```bash
npm install @openrouter/agent dotenv zod
npm install --save-dev tsx typescript @types/node
```

Also ask Hermes to add a small model resolver before the smoke test. This prevents a stale `OPENROUTER_MODEL` value from turning into a confusing runtime failure:

```markdown
Add a model resolver for Capitol Tracker.

Create `src/services/models.ts`. It should read `OPENROUTER_MODEL`, default to `openrouter/auto`, fetch `https://openrouter.ai/api/v1/models` with the OpenRouter API key, and return the preferred model only if it appears in the live model list. If the preferred model is missing, fall back to `openrouter/auto`. Use this resolver in the CLI before calling the digest or chat agents.
```

Then I'll jump into a new Hermes session and run:

```markdown
in our last session we built out the first version of capitol tracker, i've added the environment variables we need via the .env file and now need to run a smoke test to make sure everything is working
```

Alright after running that we have the following response:

```markdown
Smoke test complete. Here's what happened:

What I ran
- Verified env vars are present
- Built the project — compiles clean
- Ran fetch — pulls 20 recent Colorado bills from OpenStates
- Ran a targeted digest smoke test with 3 bills via the OpenRouter agent SDK — agent loop completes and streams the analysis

Bugs found & fixed during the test

1. per_page limit
OpenStates v3 rejects per_page above 20. The client and CLI were defaulting to 50. Changed defaults to 20 in src/services/openstates.ts and src/cli/index.ts.

2. Missing bill details
getBill was returning 0 actions and 0 sponsors because OpenStates v3 requires explicit include query params (include=actions&include=sponsorships&include=sources). Updated the request helper to support array params and added the includes. Now fetches full bill detail correctly (e.g. SB 134 has 23 actions and 16 sponsors).

Next steps

The CLI works end-to-end. If you want, I can:
- Tune the digest prompt to guarantee the model calls a finalize tool
- Add more robust error handling
- Set up a quick Git repo and commit the fixes

What do you want to tackle first?
```

We now have a working first version of the Capitol Tracker CLI. Your output will probably be different, but the next step is the same: look under the hood, inspect what Hermes generated, understand how it works, and identify the improvements worth making.

## What Hermes built (and what yours should look like)

At this point your agent has generated something. It might be close to what I got, or it might be structured differently. Both outcomes are fine. What matters is that you can verify it against the patterns we're about to discuss.

I have a complete reference implementation at [github.com/kenrogers/capitol-tracker](https://github.com/kenrogers/capitol-tracker) that matches what we're building in this tutorial. You can clone it if you want to compare against working code, or if your agent gets stuck and you need a reset point. But the goal of this section is not to make your files match mine — it's to teach you how to evaluate whatever your agent produced.

At each of these checkpoints, take a look at the code your agent produced and see if it fits the patterns and has the requirements.

If you aren't sure, ask your agent! Learning how to collaborate with your coding agent is an excellent practice to get into. If you notice areas where the agent didn't get it quite right, direct it to improve by pointing it at this tutorial or at the OpenRouter docs and telling it how it needs to improve.

### Checkpoint 1: Do you have an OpenRouter client?

Every agent you build needs a configured OpenRouter client. Verify your code has something equivalent to this excerpt:

```typescript
import { OpenRouter } from '@openrouter/agent';

const client = new OpenRouter({ apiKey: process.env.OPENROUTER_API_KEY });
```

If your agent put this in `index.ts`, `config.ts`, or `client.ts`, that works. If it's missing or using a hardcoded string instead of an env var, fix that before moving on.

### Checkpoint 2: Do you have typed tools?

The Agent SDK's `tool()` function is what turns your code into something the model can invoke. Every tool your agent created should have these four parts:

1. A `name` the model uses to request it (use snake_case)
2. A `description` explaining when to use it
3. An `inputSchema` (Zod) defining the parameters
4. An `execute` function that runs when the model calls it

Here is what my agent produced for the bill-details tool. This is an excerpt, so your file also needs imports for `tool`, `OpenStatesClient`, and any local bill types. Use it to check whether your tool has the same structural properties, even if the names or formatting differ:

```typescript
import { z } from "zod";

export function getBillDetailsTool(client: OpenStatesClient) {
  return tool({
    name: "get_bill_details",
    description: "Fetch full details for a specific bill including actions, sponsors, and sources.",
    inputSchema: z.object({
      jurisdiction: z.string().describe("State name, e.g. Colorado"),
      session: z.string().describe("Legislative session, e.g. 2026A"),
      identifier: z.string().describe("Bill identifier, e.g. HB 24-1001"),
    }),
    outputSchema: z.object({
      id: z.string(),
      identifier: z.string(),
      title: z.string(),
      abstract: z.string(),
      session: z.string(),
      jurisdiction: z.string(),
      actions: z.string().describe("Newline-delimited list of actions"),
      sponsors: z.string().describe("Newline-delimited list of sponsors"),
      sources: z.string().describe("Newline-delimited list of source URLs"),
    }),
    execute: async ({ jurisdiction, session, identifier }) => {
      try {
        const bill = await client.getBill({ jurisdiction, session, identifier });
        return {
          id: bill.id,
          identifier: bill.identifier,
          title: bill.title,
          abstract: bill.abstract ?? "",
          session: bill.session,
          jurisdiction: bill.jurisdiction,
          actions: bill.actions.map((a) => `${a.date}: ${a.description}`).join("\n"),
          sponsors: bill.sponsorships.map((s) => `${s.name} (${s.primary ? "primary" : "cosponsor"})`).join("\n"),
          sources: bill.sources.map((s) => s.url).join("\n"),
        };
      } catch (err) {
        const message = err instanceof Error ? err.message : String(err);
        return {
          id: "",
          identifier: "",
          title: "",
          abstract: "",
          session: "",
          jurisdiction: "",
          actions: `Failed to fetch bill details: ${message}`,
          sponsors: "",
          sources: "",
        };
      }
    },
  });
}
```

Let's walk through what this tool is doing so you can check whether your agent produced something equivalent. The `tool()` function creates a contract between your code and the model. The SDK uses that contract to automatically handle tool calling: it validates the model's arguments against `inputSchema`, runs your `execute` function, validates the return value against `outputSchema`, and feeds the result back to the model. You do not write the loop — you just define the tool.

That contract has four parts you should verify in every tool in your codebase.

First, `name` and `description` tell the model when and how to call the tool. The description is specific enough that the model knows this fetches full details rather than just stubs. Check your own tools: do they have clear names and descriptions that distinguish them from each other?

Second, `inputSchema` defines exactly what parameters the model must provide. Each field should have a `.describe()` call. The model reads these descriptions to decide what values to pass, so "State name, e.g. Colorado" is doing real work. The SDK validates every tool call against this schema before your `execute` function runs, which means malformed calls never reach your code. Descriptions are not enforced by the validator, but they are essential for reliable tool use. Verify that your tool schemas have descriptions on every field.

Third, `outputSchema` gives the model a contract for what it will receive back. Without it, the SDK skips runtime validation of your return value, so type mismatches reach the model undetected. The SDK validates your `execute` return value against `outputSchema`, catching type mismatches at runtime. If your tool is missing `outputSchema`, your code will still compile, but you lose that safety net.

Fourth, `execute` runs when the model requests the tool. The critical requirement here is error handling. This implementation catches exceptions and returns a structured failure message rather than throwing. If your tool throws an unhandled exception, the SDK actually catches it automatically and reports the error back to the model as JSON, so the agent loop does not break. Catching errors yourself gives you control over what the model sees — you can return a structured failure message with context and suggestions, rather than relying on the default error JSON. Check your own tools: do they catch errors and return something the model can work with?

Inside `execute`, the function flattens arrays into newline-delimited strings. The model reads tool results as text, so a dense JSON array is harder to scan than a simple list. If your agent returned raw JSON arrays, that will still work, but readability matters because the model's next decision depends on parsing this output correctly.

The `getBillDetailsTool(client)` wrapper is a factory function. It takes an `OpenStatesClient` and returns a configured tool, keeping the tool definition decoupled from the client instance. If your agent hardcoded the client inside the tool or used a global import, consider refactoring it to accept the client as an argument so multiple agents can share the same service layer.

One typing detail to verify in your own code: when you pass tools to `callModel`, use `tools: [myTool] as const`. The `as const` tells TypeScript to treat the array as a literal tuple, which lets the SDK narrow tool-call types so `getToolCallsStream()` and `getItemsStream()` know exactly which tools are available. Without it, tool names and arguments fall back to `string` and `unknown`, and you lose autocomplete and type checking on tool results. If your `callModel` call has type errors around tool names, `as const` is likely missing.

### Checkpoint 3: Do you have a search tool and a details tool?

The app needs two bill-data capabilities: listing/searching bill stubs, and fetching full bill details. In the reference implementation, the digest pipeline calls `OpenStatesClient.listBills()` before `callModel` and gives the digest model only `get_bill_details`; the chat agent gets both `search_bills` and `get_bill_details` so it can answer queries like "tell me more about how SB 70 would work in practice." Verify that your agent covers both capabilities, even if the exact names or placement differ.

Here is my search tool excerpt. Check yours for equivalent behavior:

```typescript
export function searchBillsTool(client: OpenStatesClient) {
  return tool({
    name: "search_bills",
    description: "Search for bills in a jurisdiction/session. Returns stubs with titles and identifiers.",
    inputSchema: z.object({
      jurisdiction: z.string().describe("State name, e.g. Colorado"),
      session: z.string().describe("Legislative session, e.g. 2026A"),
      updatedSince: z.string().optional().describe("ISO date to filter bills updated since, e.g. 2026-04-01"),
      perPage: z.number().default(20).describe("Number of results to return, max 20"),
    }),
    outputSchema: z.object({
      count: z.number(),
      bills: z.array(
        z.object({
          identifier: z.string(),
          title: z.string(),
          updatedAt: z.string(),
          subject: z.string(),
        })
      ),
    }),
    execute: async ({ jurisdiction, session, updatedSince, perPage }) => {
      try {
        const { results } = await client.listBills({
          jurisdiction,
          session,
          updatedSince,
          perPage: Math.min(perPage ?? 20, 20),
        });
        return {
          count: results.length,
          bills: results.map((b) => ({
            identifier: b.identifier,
            title: b.title,
            updatedAt: b.updatedAt,
            subject: b.subject.join(", ") || "none",
          })),
        };
      } catch (err) {
        const message = err instanceof Error ? err.message : String(err);
        return {
          count: 0,
          bills: [],
        };
      }
    },
  });
}
```

Here is the details tool excerpt.

```typescript
export function getBillDetailsTool(client: OpenStatesClient) {
  return tool({
    name: "get_bill_details",
    description: "Fetch full details for a specific bill including actions, sponsors, and sources.",
    inputSchema: z.object({
      jurisdiction: z.string().describe("State name, e.g. Colorado"),
      session: z.string().describe("Legislative session, e.g. 2026A"),
      identifier: z.string().describe("Bill identifier, e.g. HB 24-1001"),
    }),
    outputSchema: z.object({
      id: z.string(),
      identifier: z.string(),
      title: z.string(),
      abstract: z.string(),
      session: z.string(),
      jurisdiction: z.string(),
      actions: z.string().describe("Newline-delimited list of actions"),
      sponsors: z.string().describe("Newline-delimited list of sponsors"),
      sources: z.string().describe("Newline-delimited list of source URLs"),
    }),
    execute: async ({ jurisdiction, session, identifier }) => {
      try {
        const bill = await client.getBill({ jurisdiction, session, identifier });
        return {
          id: bill.id,
          identifier: bill.identifier,
          title: bill.title,
          abstract: bill.abstract ?? "",
          session: bill.session,
          jurisdiction: bill.jurisdiction,
          actions: bill.actions.map((a) => `${a.date}: ${a.description}`).join("\n"),
          sponsors: bill.sponsorships.map((s) => `${s.name} (${s.primary ? "primary" : "cosponsor"})`).join("\n"),
          sources: bill.sources.map((s) => s.url).join("\n"),
        };
      } catch (err) {
        const message = err instanceof Error ? err.message : String(err);
        return {
          id: "",
          identifier: "",
          title: "",
          abstract: "",
          session: "",
          jurisdiction: "",
          actions: `Failed to fetch bill details: ${message}`,
          sponsors: "",
          sources: "",
        };
      }
    },
  });
}
```

At first Hermes created three tools, one for searching, one for getting details, and one for comparing.

A `compare_bills` tool is narrow: it only handles exactly two bills.

The model is already capable of parallel tool calls. When you ask "how do SB 40 and HB 101 compare?" the model can call `get_bill_details` for both bills in the same turn, receive both results, and synthesize the comparison in its response. You get the same outcome with less code and more flexibility. This is also just not a very realistic or common scenario in a tool like this.

Keep an eye out for overengineering like this in your agent's output and seek to simplify where you can.

### Checkpoint 4: Do you have a digest agent that uses `callModel`?

This is the core of the application. Verify your agent created a function that calls `callModel` with at minimum:

- A model identifier
- Instructions that tell the model what kind of digest to write
- An `input` containing the bill stubs to review
- At least one tool the model can call to fetch more details
- A `stopWhen` condition so the loop can't run forever

Here is my digest agent excerpt. Use it to validate the shape of yours. The reference implementation resolves this model through `src/services/models.ts`, which checks the live OpenRouter model list and falls back to `openrouter/auto`; the snippet below keeps that part compact.

```typescript
const billTool = getBillDetailsTool(openStatesClient);
const model = process.env.OPENROUTER_MODEL ?? "openrouter/auto";

const result = client.callModel({
  model,
  instructions: buildInstructions(profile),
  input:
    `Here are the recently updated bills for ${profile.state}:\n\n` +
    stubs
      .map(
        (b) =>
          `- ${b.identifier}: ${b.title} [session: ${b.session}, subjects: ${b.subject.join(", ") || "none"}, updated: ${b.updatedAt}]`
      )
      .join("\n") +
    `\n\nReview these bills. For any that seem highly relevant, call get_bill_details to inspect sponsors, actions, and sources. ` +
    `Then write a concise digest highlighting only the bills that genuinely matter. Max ${profile.digest.max_items} entries.`,
  tools: [billTool] as const,
  stopWhen: stepCountIs(10),   // safety cap
});

let text = "";
for await (const delta of result.getTextStream()) {
  text += delta;
  process.stdout.write(delta);
}
process.stdout.write("\n");
return text;
```

Let's walk through what the digest agent is doing so you can verify your own implementation. This is the core of the application, and it demonstrates the two things `callModel` does for you: it runs the multi-turn agent loop automatically, and it gives you a `ModelResult` you can consume however you need.

The five parameters work together to research bills and produce a summary. `model` selects which LLM runs the loop. `instructions` is the system prompt — it defines the agent's persona, rules, and constraints. In the reference implementation this includes information from the user's profile. `input` is the user message for this turn, which here contains the bill stubs the model should review. The model sees this list and decides which bills are worth investigating further. `tools` gives the model access to `get_bill_details`, and `stopWhen` sets a hard ceiling on how many turns the loop can run.

Check your own code for these five elements. If any are missing, the agent will not behave correctly. If you omit `stopWhen`, the SDK defaults to `stepCountIs(5)`, which may be too low for a digest that needs to inspect several bills. Setting an explicit cap gives you predictable control over cost and latency.

The `tools: [billTool] as const` line creates a readonly tuple. The SDK infers tool types from the array literal, so `stopWhen` conditions and `getToolCalls()` results are precisely typed. If your agent omitted `as const` and you're seeing type errors around tool names, add it back.

`stopWhen: stepCountIs(10)` is one of several built-in stop conditions. The docs also provide `maxCost()` for budget caps and `maxTokensUsed()` for token limits. You can combine them by passing an array — execution stops when any condition is met. If you want the model to decide when it is done, define a `finish` tool and use `hasToolCall('finish')` as the stop condition. For a digest that reviews 10-20 bill stubs, 10 turns is likely plenty. If your agent set this much higher, consider lowering it.

After calling `callModel`, the code consumes the result with `getTextStream()`. This yields text deltas as the model generates them. We accumulate them into `text` so the caller still gets the full string, and we write each delta to `process.stdout` so the user sees output in real time instead of staring at a blank screen for 30 seconds. The trailing `process.stdout.write("\n")` ensures the terminal prompt lands on a fresh line after the stream ends. Verify that your own digest agent streams output to the terminal rather than buffering everything until the end.

### Checkpoint 5: Do you have a chat agent with state persistence?

The chat agent is what makes Capitol Tracker interactive. The key pattern here is that `callModel` accepts a `state` parameter, which lets you resume a previous conversation. But `state` is not a plain JSON object — it is a `StateAccessor`, an interface with `load()` and `save()` methods that the SDK calls automatically during the agent loop. The SDK tracks conversation status (`in_progress`, `complete`, `awaiting_approval`, `interrupted`) and message history through this interface, so you can stop and resume at any point.

Here is the file-based `StateAccessor` from the reference repo:

```typescript
import type { StateAccessor, ConversationState } from "@openrouter/agent";
import { readFile, writeFile, mkdir } from "fs/promises";
import { existsSync } from "fs";
import { dirname } from "path";

function isValidConversationState(value: unknown): value is ConversationState {
  if (!value || typeof value !== "object") return false;
  const obj = value as Record<string, unknown>;
  return (
    typeof obj.id === "string" &&
    Array.isArray(obj.messages) &&
    typeof obj.status === "string"
  );
}

function createFileStateAccessor(path: string): StateAccessor {
  return {
    load: async () => {
      if (!existsSync(path)) return null;
      try {
        const raw = await readFile(path, "utf-8");
        const parsed: unknown = JSON.parse(raw);
        if (!isValidConversationState(parsed)) {
          console.warn("State file is malformed. Starting fresh conversation.");
          return null;
        }
        return parsed;
      } catch {
        return null;
      }
    },
    save: async (state) => {
      await mkdir(dirname(path), { recursive: true });
      await writeFile(path, JSON.stringify(state, null, 2));
    },
  };
}
```

And here is the relevant excerpt for passing it to `callModel`:

```typescript
const STATE_DIR = join(homedir(), ".capitol-tracker");
const STATE_PATH = join(STATE_DIR, "state.json");
const model = process.env.OPENROUTER_MODEL ?? "openrouter/auto";

const state = createFileStateAccessor(STATE_PATH);

// Load existing state to check if this is the first turn
const loaded = await state.load();
const isFirstTurn = !loaded || loaded.messages.length === 0;

// Build the input: on the first turn, prepend the digest as context
const input = [];
if (isFirstTurn) {
  input.push({
    role: "user" as const,
    content: `Here is the latest legislative digest for context. You may refer to bills mentioned below when answering questions:\n\n${lastDigest}`,
  });
}
input.push({
  role: "user" as const,
  content: userMessage,
});

const result = client.callModel({
  model,
  instructions: buildInstructions(profile),  // system prompt only — no digest here
  input,
  tools,
  stopWhen: stepCountIs(15),
  state,  // <-- messages accumulate automatically across runs
});

let text = "";
for await (const delta of result.getTextStream()) {
  text += delta;
  process.stdout.write(delta);
}
process.stdout.write("\n");
return text;
```

Let's walk through what this code is doing so you can verify your own state persistence implementation. The `StateAccessor` interface has two methods: `load()` returns the previous conversation state, and `save()` writes the current state after each turn. During the agent loop itself, the SDK calls these automatically — you do not need to call `save()` yourself. The example above still calls `load()` manually before `callModel` to check whether this is the first turn, which is fine: the SDK handles loop-level persistence, and your application logic handles when to inject context.

The `isValidConversationState` guard is important. If the state file gets corrupted or the schema changes between versions, a raw `JSON.parse` could return malformed data that crashes the SDK. This validator checks that the loaded object has the required fields — `id`, `messages`, and `status` — before returning it. If validation fails, the function returns `null`, which tells the SDK to start a fresh conversation. Check your own implementation: does it handle malformed state files gracefully, or does it assume the file is always valid?

The `load` and `save` methods are async, which is why the reference uses `fs/promises`. Sync file I/O blocks the event loop and can cause noticeable pauses in an interactive CLI. If your agent used `readFileSync` or `writeFileSync`, consider switching to the promise-based versions.

When you pass `state` to `callModel`, the SDK appends each turn's input and response to the state's message history automatically via `state.save()`. The next time you run `callModel` with the same `state`, `state.load()` returns the full conversation including all prior tool calls and responses. This is how follow-up questions like "tell me more about SB 70" work — the model sees the previous digest and the user's prior questions.

Notice that the digest is loaded into `input`, not `instructions`. The `instructions` field is functionally a system prompt: it gets sent on every turn. If you injected a 2,000-token digest into `instructions`, you would force that digest into every request. By loading the digest as a one-time user message on the first turn, the app avoids re-injecting it into `instructions` every time. It is still part of the persisted conversation history, so it can still contribute to token usage on future turns. For a longer-lived app, add pruning or summarization once the state grows. Verify that your own chat agent follows the same pattern.

If your agent used a custom `loadState()` / `saveState()` wrapper but never actually passed `state` into `callModel`, then persistence is not happening. The `StateAccessor` must be passed to `callModel` for the SDK to use it. Check your code: is `state` actually present in the `callModel` call?

If your agent didn't implement state persistence at all, that's a good next prompt:

> "Add state persistence to the chat agent so follow-up questions remember previous context. Use the SDK's StateAccessor pattern with file-based storage."

### What to do if your agent diverged significantly

If your code doesn't match these checkpoints at all, try to debug why. Is Hermes not using the docs or loading the skill file we created? Something else? Point it at this tutorial or the reference implementation to help steer it in the right direction.

### Next: walk through your own code

Before moving on to testing, spend five minutes with your agent asking it to explain what it built. I like to start with a basic code walkthrough, asking Hermes to walk me through the entire application file by file and explain what each is doing.

This is one of the most valuable habits you can build when coding with agents. The agent will surface design decisions you might not have noticed, and you'll catch architectural issues before they become bugs.

Once you're satisfied with the structure, we'll move on to testing.

## Testing Capitol Tracker

Let's finish up this lesson by running through a quick manual test of Capitol Tracker.

We'll change into our capitol tracker directory inside our Hermes agent and make sure our API keys are available. If you created a `.env` file earlier, verify it is in the project root and that `src/cli/index.ts` imports `dotenv/config` at the top. If you are using shell exports instead, make sure they are active in your current terminal session.

The reference profile defaults to Colorado's `2026A` session because that was active while this tutorial was written. If you want a different state, or if Colorado's session changes after publication, create `~/.capitol-tracker/profile.json`:

```json
{
  "state": "Colorado",
  "session": "2026A",
  "priorities": ["privacy", "civil liberties", "property rights"],
  "interests": [],
  "digest": {
    "max_items": 5,
    "style": "concise"
  }
}
```

You can find current sessions in the OpenStates API docs and jurisdiction endpoints. If `fetch 1` returns no bills, first check that the `state`, `session`, and date window match current legislative data.

To make sure that works we'll do a quick fetch of the OpenStates API (note I am doing this based on what my agent generated, your method might be different. I recommend working with your Hermes agent to see how best to test out your version of what we built).

```bash
npx tsx src/cli/index.ts fetch 1
```

That should return a list of bills something like this:

```bash
SB 117: Permissible Methods of Selling Lottery Tickets
  Subjects: Gaming, Lottery, & Racing
  Updated: 2026-04-21T17:48:27.663604+00:00

HB 1344: Sunset Podiatry Board
  Subjects: Health Care & Health Insurance
  Updated: 2026-04-21T17:48:23.869925+00:00

SB 9: Charitable Organization State Sales & Use Tax
  Subjects: Fiscal Policy & Taxes
  Updated: 2026-04-21T17:48:22.546852+00:00

HB 1052: Rights for Victims of Certain Crimes
  Subjects: Crimes, Corrections, & Enforcement
  Updated: 2026-04-21T17:48:21.293592+00:00

SB 70: Ban Government Access Historical Location Information Database
  Subjects: Courts & Judicial, Crimes, Corrections, & Enforcement
  Updated: 2026-04-21T17:48:14.747001+00:00
```

Now let's try running the digest.

```bash
npx tsx src/cli/index.ts digest 1
```

This command may take 30-60 seconds to complete. You will see the digest stream to the terminal in real time as the model generates it. The model may internally call `get_bill_details` for bills it deems relevant — those tool calls happen behind the scenes and do not produce visible output unless you add a `getToolStream()` consumer. If you want to see real-time progress of tool executions, you can add a `getToolStream()` consumer to log each tool call.

Here's what I got:

```markdown
Your Colorado digest is ready. I focused on the five bills that actually move the needle for your priorities—privacy, civil liberties, property rights, and government growth.

**Bottom line:** The big privacy story is **SB 70**, a sweeping location-data ban with massive bipartisan momentum. On the spending side, keep an eye on **SB 172**, which would birth a brand-new rail district with taxing and eminent-domain authority. **HB 1226** is the sleeper cost-driver: emissions mandates on power plants that usually show up on your utility bill. Meanwhile, **HB 1283** and **SB 14** both touch civil liberties—one by limiting ID seizures, the other by tightening insanity-defense rules.

Everything else in the update was either ceremonial, too narrow, or already headed to the Governor with little drama, so I left it out. Let me know if you want me to track any of these through their next votes.
```

That gives us a useful lead, but I don't have much context on SB 70 yet. I'd like to know more about what it is.

So I'll start the chat command:

```bash
npx tsx src/cli/index.ts chat
```

Then inside chat, I'll respond with:

```text
tell me more about sb70, what is this location data ban?
```

That will kick off the `chat` agent we made and do some research on that individual bill. After running that I get:

```markdown
**SB 70** — *Ban Government Access Historical Location Information Database* — was moving when this sample was captured. It had just cleared the Senate Appropriations Committee on April 21, 2026 and was headed next to the full Senate floor.

### What “location data” means here

When you carry a smartphone, use apps, or drive a connected car, your device constantly generates **geolocation signals**. Private companies—often called **data brokers**—collect, package, and sell this information. The result is a commercial database that can reconstruct where a specific person (or device) has been over days, weeks, or years.

Right now, Colorado state and local agencies can **buy access to those databases** without getting a warrant. SB 70 would **prohibit that practice**.

### What the bill actually does

- **Bans state and local government agencies** from accessing commercial databases that store historical location information on individuals.
- **Closes the “data-broker loophole.”** Currently, law enforcement can skip the traditional warrant process and simply purchase location histories from private vendors. This bill says that shortcut is off-limits for Colorado government actors.

### The seen and unseen effects

- **Seen:** A direct privacy win. Your historical whereabouts—doctor visits, religious sites, political meetings, relationships—would no longer be available to government agencies via a credit-card transaction to a data broker. It's a rare instance of the legislature explicitly restraining surveillance power.
- **Unseen:** Investigators lose a cheap, fast intelligence tool. The risk is that agencies may redirect resources toward other surveillance methods (more subpoenas, broader device warrants, or expanded use of real-time tracking tools), potentially shifting costs and privacy threats rather than eliminating them.

### Where it stands

- **Sponsors:** 17 Democratic primary sponsors; no Republican cosponsors. Unified Democratic caucus support.
- **Status:** Introduced January 28 → Judiciary Committee (amended) → Appropriations Committee (amended) → **referred to the Senate Committee of the Whole today.** That means it's ready for floor debate and votes in the full Senate.

If you want, I can compare this with any other surveillance or privacy bills floating around this session, or dig into the specific amendments Judiciary and Appropriations added.
```

## Next Steps

We now have a Hermes OpenRouter developer agent set up and specifically trained on using the OpenRouter Agent SDK. It also created a working CLI tool for helping us stay more informed on our state's politics.

But we don't have much visibility into what's happening under the hood here.

How much did running this digest cost? How long did it take? How many steps did we go through? Exactly what tools did we call and what data did we get from each?

Luckily, OpenRouter makes it extremely easy to get access to all this data.

In the next lesson we'll explore the inner workings of our agent so we can observe, evaluate, and optimize it using OpenRouter's Broadcast feature paired with Langfuse.
