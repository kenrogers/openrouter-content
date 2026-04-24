---
title: "Broadcast: Send OpenRouter Traces Anywhere"
date: "2026-04-23"
author: "OpenRouter"
---

**Broadcast** is automatic trace export from OpenRouter to external observability and analytics platforms. It has been live since November, and this post takes a closer look at what it enables for teams building with LLMs.

Turn it on in [Settings > Observability](https://openrouter.ai/settings/observability), add a destination, and OpenRouter will send traces for your API requests after they complete. No new SDK, no custom instrumentation, no extra code path in your application.

Broadcast now supports Langfuse, LangSmith, Braintrust, Datadog, PostHog, Snowflake, ClickHouse, S3, OpenTelemetry Collectors, webhooks, and more.

Broadcast builds on the work of [Untrace](https://www.untrace.dev/), which has joined OpenRouter. Untrace started with a simple idea: LLM traces should work more like product analytics events. Capture them once, route them to the tools your team already uses, and avoid rebuilding the same integration layer for every observability vendor.

We think that routing layer belongs directly in OpenRouter.

## The Observability Bottleneck

LLM traces are useful to more people than traditional application logs.

Developers need them to debug bad generations, tool calls, latency spikes, provider behavior, and cost regressions. Product managers need them to understand how real users interact with AI features. Data scientists need them for evals and model comparisons. Annotators need them for review workflows. Support teams need them when a customer asks why an agent did something. Finance and operations teams need them to understand spend by customer, feature, model, or environment.

The unusual thing about LLM traces is that many of these people can actually read the data. Inputs and outputs are natural language. A product manager does not need to decode a protobuf payload to understand that a support assistant gave the wrong refund policy. An annotator does not need backend access to judge whether a generated answer was good. A data scientist does not need to reconstruct the conversation from five internal systems before building an eval dataset.

But the data still usually depends on an engineer explicitly sending it somewhere.

Every destination has its own SDK, schema, authentication model, privacy controls, sampling behavior, and metadata conventions. If a team wants traces in Langfuse for debugging, PostHog for product analytics, Snowflake for analysis, and Datadog for operational monitoring, somebody has to wire all of that up and keep it working.

We built Broadcast to move that routing layer into OpenRouter.

## Segment for LLM Traces

The inspiration is familiar from analytics infrastructure: instrument once, route many places.

OpenRouter already sees the request, model, provider, timing, token usage, cost, tool usage, and response for every generation sent through the platform. Broadcast lets you use OpenRouter as the source of truth for that trace stream, then fan it out to the tools your team already uses.

That matters most when the people who need the data are not the same people writing the application code.

A product manager can enable a destination in OpenRouter and start seeing traces in the team's analytics stack. A developer can add custom metadata when they need more structure. An organization admin can configure shared destinations across API keys. Teams building agents on top of OpenRouter get observability for the LLM calls by default, even when the agent framework itself was not designed around a specific observability vendor.

The goal is not to replace your observability stack. The goal is to stop making every team rebuild the same trace plumbing.

## What Broadcast Sends

Each Broadcast trace can include:

- Request and response data
- Prompt, completion, and total token usage
- Request cost
- Start time, end time, and latency
- Model slug and provider
- Tool usage signals
- User and session identifiers
- Custom metadata

You can use Broadcast with no application changes and still get useful operational data. If you want richer traces, you can pass `user`, `session_id`, and a flexible `trace` object in your OpenRouter request.

For example, you can attach a feature name, environment, customer account, app version, experiment ID, workflow step, or eval run ID. The `trace` object accepts arbitrary JSON and is forwarded to your configured destinations.

Some metadata keys have special behavior across destinations. `trace_id` can group multiple requests into one workflow. `trace_name`, `span_name`, and `generation_name` can make traces easier to scan in tools like Langfuse. `parent_span_id` can connect an OpenRouter generation back to an existing application trace, including OpenTelemetry traces.

That means Broadcast works for both teams that want a no-code dashboard toggle and teams that need precise control over trace structure.

## Destinations

Broadcast supports 17 destinations today:

- Arize AI
- Braintrust
- ClickHouse
- Comet Opik
- Datadog
- Grafana Cloud
- Langfuse
- LangSmith
- New Relic
- OpenTelemetry Collector
- PostHog
- Ramp
- S3 / S3-compatible storage
- Sentry
- Snowflake
- W&B Weave
- Webhook

Each destination has its own setup guide in the [Broadcast docs](https://openrouter.ai/docs/guides/features/broadcast/overview). You provide the required credentials in OpenRouter, test the connection, and save the destination.

Credentials are encrypted before storage and decrypted only when sending traces.

## Controls

Broadcast is configured per destination.

API key filtering lets you decide which requests go to which tools. You might send traces from one production key to Datadog, another product-specific key to PostHog, and a staging key to Langfuse for debugging.

Sampling is also per destination. A sampling rate of `1.0` sends every trace. A lower rate sends a deterministic subset. When you provide `session_id`, sampling is session-aware, so complete conversations and agent runs stay together instead of being split into fragments.

Privacy Mode strips prompt and completion content before sending a trace. Token counts, cost, timing, model information, provider information, and metadata still get sent. This is useful when one destination needs full traces for debugging while another only needs cost and latency metrics.

Traces are sent asynchronously after requests complete, so enabling Broadcast does not add latency to your API responses.

## How to Enable It

Broadcast is available in [Settings > Observability](https://openrouter.ai/settings/observability).

1. Toggle **Enable Broadcast**.
2. Add a destination.
3. Enter the destination credentials.
4. Optionally configure API key filters, sampling, and Privacy Mode.

Organization admins can configure Broadcast at the organization level so shared destinations apply across team API keys.

Read or point your agent at the [Broadcast docs](https://openrouter.ai/docs/guides/features/broadcast/overview) for destination-specific walkthroughs and metadata examples.

Questions and feedback: we're on [Discord](https://discord.gg/openrouter).
