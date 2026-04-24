# OpenRouter DevRel Content

This repo contains the content I created during my OpenRouter trial week, organized around the content system I would like to help build out from here.

**Content Caveat:** Each of these needs one final QA passthrough which I will be working on tonight/tomorrow.

I learned a lot during this week about what kind of content I think will perform well for OpenRouter and the strategy we should pursue. The opportunity is to create content that teaches agents and humans how to build with OpenRouter across every major product capability, across multiple content formats and channels.

## Recommended Content System

### Guides

Guides should live in a new dedicated top-level **Guides** section of the docs and follow the Diataxis [Tutorials](https://diataxis.fr/tutorials/) model.

These are learning-oriented, project-based, and designed to teach the what, why, and how of building with OpenRouter. The primary reader will often be a coding agent, or a developer steering a coding agent, so the content should be explicit, procedural, and easy to preserve as agent memory, skills, or project instructions.

The role of docs-native Guides is to be canonical and operational. They should be reliable enough that a developer can point an agent at them and say, "follow this pattern, then build me X." They should include constraints, validation steps, and reusable implementation patterns.

Current examples:

- [Build a Video Postcard Generator with OpenRouter](./guides/video-generation.md) — an agent-oriented docs Guide.
- [Build a Personal State Capitol Tracker with Hermes Agent, OpenRouter, and Langfuse](./guides/capitol-tracker/intro.md) — a longer project tutorial that could live in docs as a Guide, but also has the shape of distributed human-facing content.
  - [Part 1 - Setting Up Hermes Agent and OpenRouter for Agentic Coding](./guides/capitol-tracker/part-1.md)
  - [Part 2 - Building Capitol Tracker with the OpenRouter Agent SDK](./guides/capitol-tracker/part-2.md)
  - [Part 3 - Adding Observability and Evals with Langfuse and OpenRouter Broadcast](./guides/capitol-tracker/part-3.md)

### Cookbook

Like Guides, Cookbook content should live in a new dedicated top-level **Cookbook** section of the docs and follow the Diataxis [How-to Guides](https://diataxis.fr/how-to-guides/) model.

These are short, task-oriented, and focused on concrete outcomes. They should cover every practical use case for a capability, especially the tasks agents will need to perform repeatedly.

Current video generation cookbook:

- [How to choose a video generation model](./cookbook/video-generation/choose-video-model.md)
- [How to generate and download a video from text](./cookbook/video-generation/text-to-video.md)
- [How to turn an image into a video](./cookbook/video-generation/image-to-video.md)
- [How to guide a video with reference images](./cookbook/video-generation/reference-to-video.md)
- [How to get video results with webhooks](./cookbook/video-generation/video-generation-webhooks.md)
- [How to use provider-specific video options](./cookbook/video-generation/provider-specific-video-options.md)

### Skills

Skills should become a first-class OpenRouter developer asset, but the goal is not to publish static mini-docs inside `SKILL.md` files. A good OpenRouter skill should behave like an agent router: understand the task, inspect the repo, choose the right OpenRouter surface, read the right source material, implement, and verify.

The strongest implementations point to a concrete pattern:

- [Anthropic's `claude-api` skill](https://github.com/anthropics/skills/blob/main/skills/claude-api/SKILL.md) is a capability router. It defines trigger and skip conditions, checks for provider conflicts before editing, detects the project language, chooses SDK vs. raw HTTP, routes to feature references, and lists defaults, migrations, and common pitfalls.
- [Resend's send-email skill](https://github.com/resend/resend-skills/blob/main/send-email/SKILL.md) is built around a real product job. It makes decisions like single vs. batch, SDK vs. cURL, retry behavior, idempotency, webhook verification, safe testing addresses, and deliverability guardrails.
- [Netlify Skills](https://docs.netlify.com/build/build-with-ai/netlify-skills/) are split by platform primitive and paired with [`llms.txt`](https://docs.netlify.com/llms.txt), context files, CLI recipes, and marketplace/plugin installation paths. The skills are narrow, factual, and installable across many agents.
- [OpenAI's `openai-docs` skill](https://github.com/openai/skills/blob/main/skills/.curated/openai-docs/SKILL.md) acts as a source-of-truth resolver. It prioritizes official docs lookup, uses bundled references only as helper context, and defines fallback behavior when docs tooling is unavailable.

For OpenRouter, the default should be one strong general developer skill plus extensive docs, Guides, and Cookbook examples. Capability-specific skills should not be automatic. They should exist only when the general skill plus docs are not enough for reliable agent execution.

**General OpenRouter developer skill:** the front door for building with OpenRouter. It should inspect the task and repo, select the right surface (HTTP API, TypeScript SDK, Agent SDK, model routing, Broadcast, video generation, structured outputs, provider-specific options), then route to official docs, Guides, Cookbook pages, or capability skills. It should include trigger and skip rules, repo/language detection, source-of-truth rules using `https://openrouter.ai/docs/llms.txt`, a feature routing table, model and endpoint verification steps, auth/env setup, validation checks, and common mistakes. It should not copy the docs wholesale.

**Use-case skills:** opinionated build playbooks for workflows developers already want, with OpenRouter underneath. These are different from feature docs because the product capability is not the star; the build outcome is. Examples include building an agent harness, creating a coding agent, adding observability/evals, or building an AI search app. These should include the recommended architecture, likely files/modules to create, implementation sequence, agent-steering prompts, smoke tests, failure modes, and links back to the relevant OpenRouter capability docs.

**Capability skills:** create these only when a feature is complex enough that docs plus the general skill still leave agents making mistakes. A capability skill earns its place when it adds cross-page orchestration, decision logic, current-state verification, reusable scripts, or strong failure-mode handling. Video generation is a good candidate to watch because it is async, model-dependent, multimodal, provider-specific, and has real implementation pitfalls around job polling, webhooks, result download, and passthrough parameters. But the first move should still be: make the general skill route agents to the video generation docs, Guide, and Cookbook. Add a dedicated video generation skill only if repeated usage shows agents need a focused execution layer.

Every skill should be written and tested like an agent product:

- Keep `SKILL.md` short enough to route the task, with one-level-deep `references/` for detailed docs and examples.
- Include deterministic scripts only where they save time or reduce mistakes.
- Give agents defaults, not a menu of equal options.
- Tell agents when to stop and verify current docs instead of guessing API shapes, model IDs, prices, or parameters.
- Test each skill by giving a fresh agent realistic prompts and revising based on where it gets stuck.

This makes skills the reusable execution layer above docs: Guides teach the build path, Cookbook entries solve specific tasks, and skills let agents apply those patterns directly inside a user's agent.

### Announcements

Announcements are straightforward release content published alongside new feature launches. In Diataxis terms, they sit closest to [Explanation](https://diataxis.fr/explanation/): they explain what changed, why it matters, and how the feature fits into the product.

They should also indicate where developers or agents should go next: a reference page, a Guide, a skill, or the relevant Cookbook entries.

Current example:

- [Broadcast: Automatically Send Your OpenRouter Traces to External Observability Platforms](./announcements/broadcast-observability.md)

### Distributed Tutorials

Distributed tutorials are likely best as X Articles or similar off-docs channels.

These can cover many of the same projects as docs-native Guides, but they play a different role. The docs Guide is the canonical build path. The X Article is the story: why the project matters, what it felt like to build with an agent, what decisions came up, and why OpenRouter made the workflow better.

The audience is a human developer deciding what is worth building, how to think about the workflow, and whether OpenRouter is worth trying. These pieces can be more narrative, opinionated, and personal than docs-native Guides.

Current example:

- [OpenRouter on Hermes](./human-tutorials/openrouter-on-hermes.md)

In the current content set, the three tutorial-like pieces have different jobs:

- [Build a Video Postcard Generator with OpenRouter](./guides/video-generation.md): docs-native, agent-oriented Guide.
- [Build a Personal State Capitol Tracker](./guides/capitol-tracker/intro.md): longer project tutorial that can anchor a docs Guide series and also be adapted into distributed articles.
- [OpenRouter on Hermes](./human-tutorials/openrouter-on-hermes.md): distributed human tutorial/article.

### Video

Video content should primarily be human-to-human.

The best version is a real person walking through building something useful with a coding agent, covering the what, why, and how. This matters because humans still learn from watching other humans build, and because video creates useful surface area for AI search and recommendation systems.

Video will often pair with a written Guide or Cookbook sequence rather than stand alone.

### Agent Site

OpenRouter should also have a dedicated agent-facing web property, likely on `or.bot`, similar in spirit to `netlify.ai`.

This site should be optimized for agents and the humans steering them. It can point to the OpenRouter developer skill, capability-specific skills, `llms.txt`, agent-readable docs bundles, copyable prompts, example projects, and canonical build paths.

## What I Would Build Next

The next step is to scale this pattern across OpenRouter's major capabilities:

- One Guide per high-value build path.
- One Cookbook cluster per product capability.
- One maintained general OpenRouter developer skill.
- One use-case skill per repeated agent workflow/trending tools.
- One Announcement package per feature release.
- One human tutorial or video for the highest-leverage launches and workflows.
- One agent-facing `or.bot` site that ties the agent-native surface area together.
