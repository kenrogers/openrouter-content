# Part 2: Building Capitol Tracker with the OpenRouter Agent SDK

In Part 1 of this tutorial series, we got set up with Hermes and OpenRouter as our coding agent, trained Hermes in building with the OpenRouter SDK, and decided on a MVP architecture for our Capitol Tracker app.

In this part, we're going to actually build that using the setup we created in the last part.

By the end of this lesson, you will have:

- A working understanding of the OpenRouter Agent SDK and the `callModel` primitive
- Several typed tools built with the SDK's `tool()` function
- A digest agent that uses `callModel` to research and summarize bills
- A chat agent for follow-up questions about bills
- A tested CLI you can run locally

Before we do that, let's take a quick detour to talk about the OpenRouter SDK and specifically the Call Model primitive.

## Call Model

Before we dive into the SDK, let's make sure we're on the same page about what an agent loop actually is. If you've never built an agent before, this section is for you. If you have, skim it to confirm our vocabulary lines up.

### What is an agent loop?

A normal API call to an LLM looks like this: you send a prompt, the model sends back text, and you're done. An *agent* is different. An agent is a loop where the model can ask your code to do things on its behalf, then continue the conversation once it gets the answer.

Here is the simplest version of that loop:

1. **You send a request** with a task ("summarize these bills") and a list of available tools ("search_bills", "get_bill_details").
2. **The model responds.** It might answer directly, or it might say "I need more info — call `get_bill_details` for HB 1250."
3. **Your code runs the tool.** You execute the function the model asked for and get back a result.
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
    tools: [searchBillsTool, getBillDetailsTool],
    stopWhen: stepCountIs(5),
});
```

A "step" is one model response plus any tool calls it triggers. `stepCountIs(5)` caps the loop at 5 turns so a runaway model can't burn tokens forever.

Even though we are only calling the `callModel` function once here, that single call might result in 3 separate API requests under the hood, but you (or your agent) as the developer consume it as a single logical operation.

Two fields you will use constantly:

- **`instructions`** — the system prompt. Use it for personality, rules, and context that should apply to every turn.
- **`input`** — the user message or task. It can be a plain string or a message array. The docs show the shorthand `{ role: 'user', content: '...' }` form; we will use the explicit `Item` form (with `type: 'message'`) later when we add state persistence because it formats more reliably alongside loaded conversation history.

One of the biggest benefits of using the OpenRouter Agent SDK is that you get the `ModelResult` multi-consumption pattern.

Most SDKs make you choose what consumption model you want up front: streaming or final response? Text or structured output? Tool call details or not?

`callModel` returns a `ModelResult` that supports all of these concurrently on the same underlying stream. It maintains a shared buffer with independent read positions for each consumer. So you attach as many consumers as you want, even after streaming has started, and each gets every event from the beginning.

So we can do any/all of the following at the same time from our `result` from the call above:

```typescript
const response = await result.getResponse();            // full response + usage
for await (const d of result.getTextStream()) { }       // stream text deltas
for await (const d of result.getReasoningStream()) { }  // stream reasoning deltas
for await (const item of result.getItemsStream()) { }   // stream complete items (docs-recommended)
for await (const e of result.getToolStream()) { }       // tool execution events
for await (const c of result.getToolCallsStream()) { }  // structured tool calls
const calls = await result.getToolCalls();              // all tool calls after completion
```

`getItemsStream()` is the approach the docs recommend when you need structured access to everything the model produces — messages, tool calls, reasoning, and tool results — as complete items rather than raw text deltas.

This means you can add consumers to the same result without coordinating them or managing stream branches yourself. For Capitol Tracker, we can stream the human-readable summary to the terminal, log every tool execution event for debugging, and check the total cost across all turns — all from the same callModel result:

```typescript
// Stream the final analysis to the user
for await (const delta of result.getTextStream()) {
  process.stdout.write(delta);
}

// Independently capture tool calls for our report
const toolCalls = await result.getToolCalls();
const billData = toolCalls.find(t => t.name === 'get_bill_details')?.arguments;

// Check total cost after the full multi-turn loop completes
const response = await result.getResponse();
console.log(`Cost: $${response.usage?.cost?.toFixed(4) ?? "unknown"}`);
```

Most SDKs split streaming and non-streaming into separate entry points and rely on single-use stream branches that you can only consume once. `ModelResult` uses a reusable buffer with independent read positions, so every consumer gets full access to the same underlying data without needing to make a second API call.

This is exactly what the agent loop described above looks like under the hood — `callModel` runs the loop for you, and `ModelResult` gives you a recording of the whole conversation that you can consume however you need.

## Building Capitol Tracker

I'm going to start a new Hermes session with `/new` to start with fresh context, and I'll refer Hermes back to the previous session so it has context on what we built and we can get started scaffolding out our application.

```markdown
Check out our previous session where we brainstormed our capitol tracker app, now it's time to actually build it out. Take the architecture we planned and build out a minimal version of it using the openrouter sdk and openstates (https://docs.openstates.org/api-v3/) so we can start testing it.
```

Hermes is going to start by taking a look at the previous session to get some context and then build out the first version of the app.

Remember in Part 1 we created our custom OpenRouter skill. Hermes pulled that up and used it to create the first version of the project.

After working for a few minutes, here's the output I got from Hermes:

```markdown
Built out the minimal Capitol Tracker at ~/projects/capitol-tracker. Here's what we have:

    Architecture implemented
    - src/services/openstates.ts — Thin REST client for OpenStates API v3 (not GraphQL). Lists bills with updated_since filtering and fetches full bill details including actions, sponsors, and sources.
    - src/tools/bills.ts — Three typed tools: get_bill_details, search_bills, compare_bills. Factory pattern closes over the OpenStates client. Each tool catches errors and returns structured failure messages so the model can react.
    - src/agents/digest.ts — Single-loop digest agent using callModel with moonshotai/kimi-k2.6. The model selectively calls get_bill_details only for bills that look high-impact, then writes a prose digest when done. Stop condition: stepCountIs(10). Errors propagate to the CLI layer for clean reporting.
    - src/agents/chat.ts — Follow-up chat agent with file-based state persistence via the SDK's StateAccessor pattern. Uses async fs/promises for I/O, validates loaded state, and injects the latest digest as a one-time context message on the first turn. Conversation history is saved to ~/.capitol-tracker/state.json automatically by the SDK, so multi-turn context survives across CLI invocations.
    - src/cli/index.ts — Three commands: fetch [days], digest [days], chat. Handles async errors in the readline loop and uses the profile state in log messages.

    Profile-driven config
    - config/profile.yaml defines state, interests, priorities, impact signals, and digest style.
    - src/config/loader.ts validates with Zod using js-yaml for robust parsing and falls back to the project default if no user profile exists at ~/.capitol-tracker/profile.yaml.

    Bastiat lens
    - Baked into the digest instructions: the model is prompted to analyze seen/unseen effects, cost-shifting, incentives, and whether a bill expands or shrinks government scope.

    Build status
    - npx tsc emits successfully.
    - Our source files have zero type errors. The only errors shown are upstream noise from @openrouter/agent and zod declaration files in node_modules — benign at runtime.

    To test it
    You'll need both API keys set in your environment or a .env file:
    export OPENROUTER_API_KEY=***
    export OPENSTATES_API_KEY=***

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

We can set our API keys by creating a `.env` file in the project root:

```bash
# In your project directory
echo "OPENROUTER_API_KEY=sk-..." > .env
echo "OPENSTATES_API_KEY=..." >> .env
```

Make sure your CLI entry point loads it. You can install `dotenv` and import it at the top of `src/cli/index.ts`:

```typescript
import 'dotenv/config';
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
- Ran a targeted digest smoke test with 3 bills via the OpenRouter agent SDK — agent loop completes and returns analysis

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

This is great! We have a decently working first version of our Capitol Tracker CLI tool. Now of course your output will be different, but what we're going to do now is actually take a look under the hood and see what code Hermes generated for us, dig into how it works so you have a solid understanding, and see what improvements we can direct Hermes to make.

## What Hermes built (and what yours should look like)

At this point your agent has generated something. It might be close to what I got, or it might be structured differently. Both outcomes are fine. What matters is that you can verify it against the patterns we're about to discuss.

I maintain a complete reference implementation at [github.com/kenrogers/capitol-tracker](https://github.com/kenrogers/capitol-tracker) that matches what we're building in this tutorial. You can clone it if you want to compare against working code, or if your agent gets stuck and you need a reset point. But the goal of this section is not to make your files match mine — it's to teach you how to evaluate whatever your agent produced.

Here is the prompt I used to get from "first version" to "reviewable code":

```markdown
Take the capitol tracker we just built and refactor it for clarity. I need:
1. A clean separation between data access (OpenStates), tools (SDK tool() functions), and agents (callModel calls)
2. Each tool should have a clear name, description, Zod inputSchema, and execute function
3. The digest agent should take bill stubs, selectively call get_bill_details for high-impact bills, and return a prose summary
4. The chat agent should read from persistent state so follow-up questions have context
```

Your agent might have put everything in one file, or split it across six. It might have used classes instead of factory functions, or added error handling I didn't ask for. What we're going to do now is look at the architectural patterns I look for when reviewing agent output, using my implementation as an example.

### Checkpoint 1: Do you have an OpenRouter client?

Every agent you build needs a configured OpenRouter client. Verify your code has something equivalent to this:

```typescript
import { OpenRouter } from '@openrouter/sdk';

const client = new OpenRouter({ apiKey: process.env.OPENROUTER_API_KEY });
```

If your agent put this in `index.ts`, `config.ts`, or `client.ts`, that works. If it's missing or using a hardcoded string instead of an env var, fix that before moving on.

### Checkpoint 2: Do you have typed tools?

The Agent SDK's `tool()` function is what turns your code into something the model can invoke. Every tool your agent created should have these four parts:

1. A `name` the model uses to request it (use snake_case)
2. A `description` explaining when to use it
3. An `inputSchema` (Zod) defining the parameters
4. An `execute` function that runs when the model calls it

Here is what my agent produced for the bill-details tool. Use it to check whether your tool has the same structural properties, even if the names or formatting differ:

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
      actions: z.string().describe("Newline-delimited list of actions"),
      sponsors: z.string().describe("Newline-delimited list of sponsors"),
      sources: z.string().describe("Newline-delimited list of source URLs"),
    }),
    execute: async ({ jurisdiction, session, identifier }) => {
      const bill = await client.getBill({ jurisdiction, session, identifier });
      return {
        id: bill.id,
        identifier: bill.identifier,
        title: bill.title,
        abstract: bill.abstract ?? "",
        actions: bill.actions.map((a) => `${a.date}: ${a.description}`).join("\n"),
        sponsors: bill.sponsorships.map((s) => `${s.name} (${s.primary ? "primary" : "cosponsor"})`).join("\n"),
        sources: bill.sources.map((s) => s.url).join("\n"),
      };
    },
  });
}
```

**Pattern to notice:** `outputSchema` tells the SDK what shape your tool returns. This is optional but recommended — it gives the model a contract for the data it will receive back, which improves tool-call reliability. Without it, the model has to infer the return shape from your `execute` implementation alone.

**Pattern to notice:** `execute` flattens arrays into newline-delimited strings before returning them. The model reads tool results as text, so a dense JSON array is harder to scan than a simple list like `2026-04-20: Senate Second Reading Laid Over...`. We structure the output for the reader (the LLM), not a downstream parser. If your agent returned raw JSON arrays, that's fine — it will still work — but you might want to ask your agent to format tool output for readability.

**Pattern to notice:** `getBillDetailsTool(client)` is a factory function. It takes an `OpenStatesClient` and returns a configured tool. I used this because `OpenStatesClient` is a class instance with methods, not plain config data. The SDK has an official `context`/`contextSchema` system for passing serializable data into tools, but class instances don't play well with it. The factory is a standard closure pattern — it injects the dependency at creation time with no type gymnastics. If your agent used `context` instead and it compiles, that works too.

### Checkpoint 3: Do you have a search tool and a comparison tool?

The digest agent needs to find bills. The chat agent needs to answer questions like "how do SB 40 and HB 101 compare?" Verify your agent created tools that cover these two capabilities, even if the names differ.

Here is my search tool. Check yours for equivalent behavior:

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
    },
  });
}
```

**Critical detail:** OpenStates v3 rejects `per_page` above 20. If your agent defaulted to 50, change it to 20 or you'll get 400 errors at runtime. This is the kind of detail you validate during your smoke test, not during code review.

Here is my comparison tool. Yours might be named differently or structured as a single `get_bill_details` call with two identifiers. The pattern that matters is parallel fetching:

```typescript
export function compareBillsTool(client: OpenStatesClient) {
  return tool({
    name: "compare_bills",
    description: "Fetch and return side-by-side details for two bills so the model can compare them.",
    inputSchema: z.object({
      jurisdiction: z.string(),
      session: z.string(),
      identifierA: z.string(),
      identifierB: z.string(),
    }),
    execute: async ({ jurisdiction, session, identifierA, identifierB }) => {
      const [a, b] = await Promise.all([
        client.getBill({ jurisdiction, session, identifier: identifierA }),
        client.getBill({ jurisdiction, session, identifier: identifierB }),
      ]);
      return { billA: { ... }, billB: { ... } };
    },
  });
}
```

If your agent only created one tool, or created five, ask yourself: can the model (1) find bills, (2) get details on a specific bill, and (3) compare two bills? Those are the three capabilities Capitol Tracker needs.

**Error handling in tools:** Every tool that calls an external API should catch failures and return a structured error message. If the tool throws an uncaught exception, the SDK may crash the agent loop. If it returns a plain string like `"Error"`, the model has no context to recover. Return an object with an `error` field (and optionally a `suggestion`) so the model can explain what went wrong to the user or try a different approach:

```typescript
execute: async ({ jurisdiction, session, identifier }) => {
  try {
    const bill = await client.getBill({ jurisdiction, session, identifier });
    return { /* success data */ };
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    return {
      error: `Failed to fetch bill details: ${message}`,
      suggestion: "Verify the bill identifier and session are correct.",
    };
  }
},
```

**Error handling in agents:** Let errors from `callModel` propagate up to the CLI layer rather than swallowing them inside the agent. When an agent catches an error and returns a friendly string like `"Digest error: ..."`, the CLI has no way to know the operation failed — it just sees a string. By letting errors bubble, the CLI can print a clear message to stderr and exit with a non-zero code, which is what users (and CI systems) expect.

### Checkpoint 4: Do you have a digest agent that uses `callModel`?

This is the core of the application. Verify your agent created a function that calls `callModel` with at minimum:

- A model identifier
- Instructions that tell the model what kind of digest to write
- An `input` containing the bill stubs to review
- At least one tool the model can call to fetch more details
- A `stopWhen` condition so the loop can't run forever

Here is my digest agent. Use it to validate the shape of yours:

```typescript
const billTool = getBillDetailsTool(openStatesClient);

const result = callModel(client, {
  model: "moonshotai/kimi-k2.6",
  instructions: buildInstructions(profile),
  input:
    `Here are the recently updated bills for ${profile.state}:\n\n` +
    stubs.map((b) => `- ${b.identifier}: ${b.title} ...`).join("\n") +
    `\n\nReview these bills. For any that seem highly relevant, call get_bill_details ...`,
  tools: [billTool],  // readonly tuple — required for type narrowing
  stopWhen: stepCountIs(10),   // safety cap
});

const text = await result.getText();
return text;
```

**Pattern to notice:** `tools: [billTool]` creates a readonly tuple. The SDK infers tool types from the array, so `stopWhen` conditions and `getToolCalls()` results are precisely typed. If your agent omitted the tuple and you're seeing type errors around tool names, make sure the array is passed directly as a literal.

**Pattern to notice:** `const text = await result.getText()`. This blocks until the multi-turn agent loop completes and returns the final prose digest. The `await` is explicit so errors bubble cleanly into the caller. If your agent wrapped this in a `try/catch` that swallows the error and returns a string, change it — swallowed errors make it impossible for the CLI to distinguish success from failure programmatically.

**Better UX:** In a real CLI, streaming the digest as it is written is a nicer experience than a blank screen for 30 seconds. You can swap `getText()` for `getTextStream()` with a few extra lines:

```typescript
let text = "";
for await (const delta of result.getTextStream()) {
  text += delta;
  process.stdout.write(delta);  // print as it arrives
}
return text;
```

We use `getText()` in the tutorial to keep the code minimal, but in your own tool you should absolutely stream the output.

**Pattern to notice:** `stopWhen: stepCountIs(10)`. The `callModel` loop executes once per "turn" (one model response + any tool calls it triggers). A cap of 10 turns is plenty for a digest that reviews 10-20 bill stubs; 20 is overkill and risks burning tokens on a runaway loop.

**Performance tip:** Passing 20 bill stubs into `input` can make the digest slow — the model spends multiple turns inspecting each one. In the reference repo I capped the stub list to 10 before calling `callModel`. Your agent might have done the same, or it might have used a different strategy (like pre-filtering by subject match). If your digest takes longer than a minute, try reducing how many stubs you feed into the prompt.

If your agent's digest agent looks completely different — maybe it uses a while-loop instead of `callModel`, or maybe it batches all bill details upfront instead of letting the model choose which ones to inspect — that's okay. The question to ask is: does it produce a useful digest? Run it and see.

### Checkpoint 5: Do you have a chat agent with state persistence?

The chat agent is what makes Capitol Tracker interactive. The key pattern here is that `callModel` accepts a `state` parameter, which lets you resume a previous conversation. But `state` is not a plain JSON object — it's a `StateAccessor`, an interface with `load()` and `save()` methods. The SDK calls these automatically during the agent loop.

Here is the file-based `StateAccessor` from the reference repo:

```typescript
import type { StateAccessor, ConversationState } from "@openrouter/agent";
import { readFile, writeFile, mkdir, access } from "fs/promises";
import { dirname } from "path";

function createFileStateAccessor(path: string): StateAccessor {
  return {
    load: async () => {
      try {
        await access(path);
      } catch {
        return null;
      }
      try {
        const raw = await readFile(path, "utf-8");
        const parsed = JSON.parse(raw);
        // Minimal validation: if it doesn't have messages, treat as empty
        if (!parsed || typeof parsed !== "object" || !Array.isArray(parsed.messages)) {
          return null;
        }
        return parsed as ConversationState;
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

And here is how you pass it to `callModel`:

```typescript
import type { Item } from "@openrouter/agent";

const state = createFileStateAccessor("~/.capitol-tracker/state.json");

// Load existing state to check if this is the first turn
const existing = await state.load();
const isFresh = !existing || existing.messages.length === 0;

// Build the input array: on the first turn, prepend the digest as context
const input: Item[] = isFresh && lastDigest
  ? [
      { type: "message", role: "user", content: `Latest digest:\n\n${lastDigest}` },
      { type: "message", role: "user", content: userMessage },
    ]
  : [{ type: "message", role: "user", content: userMessage }];

const result = callModel(client, {
  model: "moonshotai/kimi-k2.6",
  instructions: buildInstructions(profile),  // system prompt only — no digest here
  input,                                      // Item[] — required when using state persistence
  tools,
  stopWhen: stepCountIs(10),
  state,  // <-- messages accumulate automatically across runs
});

const text = await result.getText();
return text;
```

The chat agent also works with `getText()`, but in a real interactive CLI you would stream the response with `getTextStream()` so the user sees text appear as the model generates it. See the "Better UX" note in Checkpoint 4 above for the swap.

**Critical detail:** When `state` is provided, the SDK appends each turn's input and response to the state's message history automatically via `state.save()`. The next time you run `callModel` with the same `state`, `state.load()` returns the full conversation including all prior tool calls and responses. This is how follow-up questions like "tell me more about SB 70" work — the model sees the previous digest and the user's prior questions.

**Why `input` is an array:** `callModel` accepts `input` as either a `string` or an `Item[]`. When using `state` with models that produce reasoning tokens (like `moonshotai/kimi-k2.6`), passing an explicit `Item[]` ensures the new user message is properly formatted alongside the loaded conversation history. A plain string can be misinterpreted when appended to an existing message array. The `Item` type is exported from `@openrouter/agent`.

**Why the digest is in `input`, not `instructions`:** The `instructions` field is functionally a system prompt — it gets sent on *every* turn. If you inject a 2,000-token digest into `instructions`, you pay for those tokens repeatedly across a multi-turn conversation. By loading the digest as a one-time user message on the first turn, it enters the persisted conversation history once and stays there for free on subsequent turns.

**Async I/O in StateAccessor:** The `load` and `save` methods are async, so use `fs/promises` rather than the sync `readFileSync` / `writeFileSync` variants. Sync file I/O blocks the event loop and can cause noticeable pauses in an interactive CLI.

If your agent used a custom `loadState()` / `saveState()` wrapper but never actually passed `state` into `callModel`, then persistence is not happening. The `StateAccessor` must be passed to `callModel` for the SDK to use it.

If your agent didn't implement state persistence at all, that's a good next prompt:

> "Add state persistence to the chat agent so follow-up questions remember previous context. Use the SDK's StateAccessor pattern with file-based storage."

### What to do if your agent diverged significantly

If your code doesn't match these checkpoints, you have three options:

1. **Prompt your agent to refactor.** Give it one checkpoint at a time. "Refactor the tools into factory functions." "Add a stopWhen condition to the digest agent."
2. **Check the reference repo.** Clone [github.com/kenrogers/capitol-tracker](https://github.com/kenrogers/capitol-tracker) and compare file-by-file.
3. **Keep your version and adapt the rest of the tutorial.** If your agent built something that works but looks different, the later sections on testing and observability will still apply. You just need to know where your equivalent files live.

The skill you're building here is not "type exactly what Kenny typed." It's "direct an agent to produce valid architecture, then verify it against known-good patterns."

### Next: walk through your own code

Before moving on to testing, spend five minutes with your agent asking it to explain what it built. I use a prompt like this:

```markdown
Walk me through the capitol tracker code you just created. For each file, tell me:
1. What it does
2. Why you structured it this way
3. What I should change if I want to add more features later
```

This is one of the most valuable habits you can build when coding with agents. The agent will surface design decisions you might not have noticed, and you'll catch architectural issues before they become bugs.

Once you're satisfied with the structure, we'll move on to testing.

## Testing Capitol Tracker

Let's finish up this lesson by running through a quick manual test of Capitol Tracker.

We'll change into our capitol tracker directory inside our Hermes agent and make sure our API keys are available. If you created a `.env` file earlier, verify it is in the project root and that `src/cli/index.ts` imports `dotenv/config` at the top. If you are using shell exports instead, make sure they are active in your current terminal session.

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

**Field mapping note:** OpenStates returns JSON with snake_case field names (`updated_at`, `jurisdiction.name`) while our TypeScript interfaces use camelCase (`updatedAt`, `jurisdiction`). If your `fetch` output shows `Updated: undefined`, you need to add explicit mapping in your `listBills` method. This is a common gotcha when building the REST client from scratch.

Now let's try running the digest.

```bash
npx tsx src/cli/index.ts digest 1
```

**Note on timing:** This command may take 30-60 seconds to produce output. The terminal will appear completely silent while the model internally calls `get_bill_details` for the bills it deems relevant. That silence is normal — the SDK's `getText()` blocks until the entire multi-turn loop completes. If you want to see real-time progress, you can add a `getToolStream()` consumer to log each tool execution.

Here's what I got:

```markdown
Your Colorado digest is ready. I focused on the five bills that actually move the needle for your priorities—privacy, civil liberties, property rights, and government growth.

**Bottom line:** The big privacy story is **SB 70**, a sweeping location-data ban with massive bipartisan momentum. On the spending side, keep an eye on **SB 172**, which would birth a brand-new rail district with taxing and eminent-domain authority. **HB 1226** is the sleeper cost-driver: emissions mandates on power plants that usually show up on your utility bill. Meanwhile, **HB 1283** and **SB 14** both touch civil liberties—one by limiting ID seizures, the other by tightening insanity-defense rules.

Everything else in the update was either ceremonial, too narrow, or already headed to the Governor with little drama, so I left it out. Let me know if you want me to track any of these through their next votes.
```

That's pretty interesting!

I don't have any context on SB 70 though, but I'd like to know more about what it is.

So I'll run `npx tsx src/cli/index.ts chat` and type "tell me more about sb70, what is this location data ban?"

That will kick off the `chat` agent we made and do some research on that individual bill. After running that I get:

```markdown
I've checked Colorado's 2023, 2024, and 2025 sessions, and I'm not finding a bill numbered **SB 70** (or the common variants like SB 24-70, SB 25-70, etc.) in the legislative database.

It's possible that:

- It's from a different year (e.g., 2022, 2021, or earlier).
- The bill number is slightly different than what you remember.
- It's from another state, even though you're asking about Colorado generally.

If you can tell me the year/session (for example, "2024" or "2025") or the formal bill title, I can pull the exact text and actions for you right away.

Also, if you're looking for Colorado legislation on location privacy or data bans in general, I can search for related bills—there have been a few privacy-related measures in recent years, though the numbering may not be SB 70. Just let me know which direction you want to go!
```

Well that doesn't seem right. The digest just told us that SB 70 was one to keep an eye on, what's going on here? The issue is that `chat` and `digest` are two completely different agents that have no awareness of each other. So when I start up the chat agent, it has no context of what I mean by SB 70, it also doesn't have an awareness of what day it is to be able to refer to.

Since the LLM, based on its training data, thinks it's 2025, it's looking there when the SB 70 I was referring to is from the 2026 session.

So how can we fix this?

One simple fix is to persist the digest prose and pass it as context to our chat agent. We save the digest output to `last-digest.txt` and inject it into the chat agent's instructions under a "LATEST DIGEST" heading. The chat agent sees the full prose, knows which bills were discussed and their identifiers, and can map user shorthand like "sb70" to the correct bill. We also drop the user straight into chat after generating the digest so the context is fresh.

```markdown
**SB 70** — *Ban Government Access Historical Location Information Database* — is moving right now. It just cleared the Senate Appropriations Committee today (April 21, 2026) and heads next to the full Senate floor.

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

We now have a solid Hermes OpenRouter developer agent set up and specifically trained on using the OpenRouter Agents SDK. It was also able to create a well-functioning CLI tool for helping us to stay more informed on our state's politics.

But we don't have much visibility into what's happening under the hood here.

How much did running this digest cost? How long did it take? How many steps did we go through? Exactly what tools did we call and what data did we get from each?

Luckily, OpenRouter makes it extremely easy to get access to all this data.

In the next lesson we'll explore the inner workings of our agent so we can observe, evaluate, and optimize it using OpenRouter's Broadcast feature paired with Langfuse.
