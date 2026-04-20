# OpenRouter DevRel Content

This repo houses the content I'm creating for OpenRouter during my trial week.

There are three primary categories of content here: a tutorial series, a how-to guide, and two blog posts.

Below is a brief explanation of each and you can dig into the folder for each as well to check out the content itself.

Note this content is currently in-progress.

## Tutorial

The tutorial is a 3-part series called *Build a Personal State Capitol Tracker with Hermes Agent, OpenRouter, and Langfuse*.

The tutorial as a whole is designed to create a narrative, use-case driven learning experience showing how different aspects of OpenRouter work together to create AI-powered apps.

This is some of my favorite kind of content to create as it teaches the what, why, and how all at the same time while showing how to build something cool using specific features of OpenRouter.

This tutorial series is designed to showcase the Agents SDK and the Broadcast feature and, secondarily, how you can use Hermes + OpenRouter as a coding agent.

### [Intro - What We're Building](./tutorial/intro.md)

### [Part 1 - Setting Up Hermes Agent and OpenRouter for Agentic Coding](./tutorial/part-1.md)

### Part 2 - Building a State Capitol Tracker with the OpenRouter Agent SDK

### Part 3 - Adding Observability and Evals with Langfuse and OpenRouter Broadcast

## How-to Guide

[How to Turn an Image into a Video with OpenRouter](./how-to-guide/image-to-video.md)

The how-to guide is aimed at users who have a specific task they want to accomplish — in this case, turning an image into a video using the new video generation endpoints.

I'm a big fan of [Diataxis](https://diataxis.fr) as a framework for structuring content, and this example fits cleanly into the how-to guide category. Content like this is meant to complement pieces like the video announcement blog post (https://openrouter.ai/announcements/video-generation), which works as both marketing material and Explanation-style content in Diataxis.

## Blog Posts

- Agent SDK: Build Smarter AI Agents
- Broadcast: Automatically Send Your OpenRouter Traces to External Observability Platforms

The blog posts are meant to be relatively high-level explanatory pieces that introduce a compelling OpenRouter capability and then point readers toward more detailed implementation-style content.

## How I Think About Dev Content

Collectively these are a good example of how I think about content creation more broadly as well.

Let's frame it around Diataxis' four types of docs: Tutorials, How-to Guides, References, and Explanation.

OpenRouter already has solid reference content in place. The biggest opportunity for high-leverage work is in the other content types.

Tutorials are great foundational pieces for teaching people what they can build with OpenRouter, how, and why, from start to finish. Tutorials will often contain educational content on multiple different components and features of OpenRouter.

I like to organize these around product features, so let's take the video generation as an example.

You have the [feature page on the docs](https://openrouter.ai/docs/guides/overview/multimodal/video-generation), I would count that as a reference.

Then we have the blog post as a piece of Explanation content, then we can create several different how-to guides around the different things people can do with video generation and link them all to each other, creating them in different formats and posting them on different platforms.

With all these different formats and types of content we can create a consistent flywheel that just gets better over time.

I'm also a big believer in agent-forward content, so rather than showing people exactly what to type as we would have in the pre-AI era, I think content should lean into showing people how to effectively use agentic coding tools with OpenRouter.