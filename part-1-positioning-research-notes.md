# Part 1 Positioning Research Notes

## Pi: why Mario built it

Sources reviewed:
- Pi coding agent README: https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent
- Mario Zechner blog post: https://mariozechner.at/posts/2025-11-30-pi-coding-agent/

Grounded takeaways:

- Mario built Pi because he wanted a simpler, more predictable coding agent than tools that had grown into "spaceships" full of features he did not use.
- In his blog post, he says Claude Code used to fit his workflow better when it was simpler, but over time it accumulated features and changing prompts/tools that broke workflows.
- He explicitly emphasizes control over context engineering: he says exactly controlling what goes into model context yields better outputs, especially for code, and that existing harnesses often inject hidden context.
- He also says he wants to inspect every aspect of model interactions, have a cleanly documented session format, and make it easy to build alternate UIs on top of the core agent runtime.
- Pi's README and philosophy section make the product philosophy explicit: Pi is "aggressively extensible" so it does not have to dictate the user's workflow.
- The core idea is: keep the harness minimal, then let people extend it with extensions, skills, prompt templates, themes, and packages.
- Mario's philosophy statement in the blog is very clear: "if I don't need it, it won't be built."

## Pi: why people seem to like it

Grounded signals from repo/docs:

- The GitHub repo snapshot showed ~37.8k stars and ~4.4k forks, which is a strong adoption signal.
- Pi's appeal is clearly tied to minimalism and visibility:
  - minimal system prompt
  - minimal toolset (read, write, edit, bash)
  - no hidden complexity by default
  - full observability of what the agent is doing
- Mario repeatedly contrasts Pi with more opaque harnesses where users cannot see sub-agents, planning behavior, or hidden context assembly.
- The README emphasizes that Pi adapts to your workflow instead of forcing you into its built-in abstractions.
- Pi is also attractive to power users because it supports multiple modes: interactive, print/JSON, RPC, and SDK embedding.
- The customization surface is large without forcing complexity into the core: extensions, skills, prompt templates, themes, and installable Pi packages.

## Best concise framing for Pi in Part 1

Suggested positioning:

Pi is a good fit for this series because it stays close to the metal. Mario built it as a minimal, extensible coding harness for people who want simpler workflows, tighter control over context, and better visibility into what the agent is actually doing.

## OpenRouter: why it fits this setup

Sources reviewed:
- OpenRouter Principles: https://openrouter.ai/docs/guides/overview/principles
- OpenRouter Models overview: https://openrouter.ai/docs/guides/overview/models
- OpenRouter Provider Selection: https://openrouter.ai/docs/guides/routing/provider-selection

Grounded takeaways:

- OpenRouter's own principles page says the future is multi-model and multi-provider.
- The same page highlights the main reasons to use it:
  - price and performance optimization across dozens of providers
  - standardized API so you do not need to change code when switching models/providers
  - real-world insights into model usage
  - consolidated billing
  - higher availability via fallbacks and smart routing
  - higher rate limits / throughput
- The models page frames OpenRouter as "one API for hundreds of models" and currently advertises 300+ models/providers.
- The provider selection docs show the practical side of that abstraction:
  - default price-based load balancing
  - explicit sorting by throughput or latency
  - fallbacks
  - allow/ignore specific providers
  - requiring support for specific parameters
  - data policy controls and zero data retention routing

## Why OpenRouter is especially sensible with Pi

- Mario explicitly says "we live in a multi-model world" in his Pi blog post.
- He also says Pi's model registry is built from both OpenRouter and models.dev data, including token costs and model capabilities.
- That means OpenRouter is philosophically aligned with Pi's design rather than being an awkward bolt-on.
- For this series, OpenRouter is useful not just because it gives access to Kimi or other hot models, but because it lets the coding workflow stay stable while model choice changes.
- That is especially valuable in fast-moving coding-model markets where a newly strong model can appear mid-week.

## Best concise framing for OpenRouter in Part 1

Suggested positioning:

OpenRouter is a strong match for Pi because Pi is built for a multi-model world, and OpenRouter gives you one consistent API across many models and providers. That makes it easy to try new coding models, optimize for price or speed, and keep the rest of your workflow stable while the model layer evolves.

## Suggested combined reasoning paragraph

Pi and OpenRouter fit together unusually well. Pi was built around a minimal, observable, extensible coding workflow, and Mario's own writing makes clear that he values simple tools, explicit context control, and visibility into what the agent is doing. OpenRouter complements that by giving the same workflow access to a multi-model, multi-provider backend through one API. In practice, that means you can keep your coding environment stable while switching models, testing new releases, and later layering in features like routing and broadcast-based observability.
