# Part 1: Setting Up Pi and OpenRouter for Agentic Coding

If you are going to build AI apps seriously, your environment matters more than people admit.

You want a coding setup that is fast to start, easy to reason about, and flexible enough to survive the pace of model releases. You do not want to rebuild your workflow every time a new coding model gets good, or get locked into a harness that hides too much of what the agent is doing.

That is why this series starts with Pi and OpenRouter.

Pi gives us a minimal, agentic coding environment that stays close to the code and exposes what is happening. OpenRouter gives us a stable way to access and swap between models through one API. Together, they give us a setup that is simple enough to understand and flexible enough to grow with the project.

This first part is about getting to a productive setup fast.

By the end of this guide, you will have:
- Pi installed locally
- OpenRouter connected as your model provider
- a model selected inside Pi
- a working agentic coding environment ready for the rest of the series

In the next part, we’ll use this setup to build our Personal State Capitol Tracker with the OpenRouter Agent SDK.

## Why this setup is a good combo

Pi is a strong fit for this series because it was built around a simple idea: keep the coding harness minimal, observable, and extensible.

In Mario Zechner’s own writing about Pi, he explains that he built it because other coding-agent tools had become too complex, less predictable, and harder to reason about. He wanted a tool that stayed simple, gave him tighter control over context, and let him inspect what the agent was actually doing instead of hiding important behavior behind layers of abstraction.

That shows up clearly in Pi itself:
- a minimal default toolset
- a lightweight terminal workflow
- strong extensibility through skills, extensions, prompt templates, and packages
- better visibility into the agent’s actions and context than more opaque coding harnesses

OpenRouter is a strong match for that philosophy. Pi is built for a multi-model world, and OpenRouter gives you one API and one key across many models and providers. That means you can:
- try different coding models without rebuilding your workflow
- switch models as new releases appear
- keep your tooling stable while your model choice evolves
- optimize for price, latency, or throughput without rewriting your setup
- layer in OpenRouter features later in the series, including Broadcast-based observability

In other words, Pi gives us a coding environment we can actually reason about, and OpenRouter gives us a model layer we can swap and improve without tearing everything else apart.

For this series, we’ll use OpenRouter as the model layer and Pi as the coding environment.

## Prerequisites

You should have:
- Node.js and npm installed
- an OpenRouter account
- an OpenRouter API key
- a terminal you are comfortable working in

## Step 1: Install Pi

Install the Pi coding agent globally with npm:

```bash
npm install -g @mariozechner/pi-coding-agent
```

After installation, confirm it is available:

```bash
pi --help
```

If that prints Pi’s help output, you’re ready to continue.

## Step 2: Add your OpenRouter API key

Pi supports OpenRouter as a provider via the `OPENROUTER_API_KEY` environment variable.

For a quick session-only setup:

```bash
export OPENROUTER_API_KEY="your-openrouter-api-key"
```

If you want it to persist across sessions, add it to your shell profile instead:

```bash
echo 'export OPENROUTER_API_KEY="your-openrouter-api-key"' >> ~/.zshrc
source ~/.zshrc
```

If you use another shell, add the same export to the appropriate profile file.

## Step 3: Start Pi

Launch Pi in the directory where you want to work:

```bash
cd ~/code/openrouter-work
pi
```

When Pi opens, it will detect available providers based on your configured credentials.

## Step 4: Select OpenRouter and choose a model

Inside Pi, open the model picker with:

```text
/model
```

You can also use `Ctrl+L` to open model selection.

Choose OpenRouter as the provider, then pick a model for coding work.

Since this series is using OpenRouter as the gateway, this is where you can experiment with different coding-capable models without changing the rest of your setup. If you want to lean into a newer model release, this is the place to choose it.

For example, if Kimi 2.6 is available in your OpenRouter model list and you want to use it for this series, select it here.

## Step 5: Verify the setup with a simple task

Still inside Pi, run a quick prompt in your working directory:

```text
Create a file called hello.txt that explains what this project will do in 3 bullet points.
```

Pi should use its tools to create the file for you.

Then confirm the file exists:

```bash
ls
cat hello.txt
```

If that worked, your setup is ready.

## What you have now

At this point you have:
- Pi installed and working
- OpenRouter connected as your provider
- a model selected for agentic coding
- a working project directory at `~/code/openrouter-work`

That is enough to start building.

## Why this matters before we write any real app code

A lot of AI app workflows get brittle because the coding environment, model provider, and application stack are all tightly coupled from day one.

This setup avoids that.

Pi gives us a coding harness that is intentionally simple and visible. OpenRouter gives us a standardized, multi-model backend that we can change without rewriting the workflow around it.

That matters in practice because the model layer is moving quickly. A better coding model can show up next week, or a cheaper/faster provider can become the better choice for part of your workflow. If your setup depends too heavily on one provider’s tooling and assumptions, changing course gets expensive.

With Pi and OpenRouter, we can focus on building the app first, then improve model choice, routing, observability, and evaluation as the project grows.

That is exactly the workflow we’ll use in the rest of this series.

## Next up

In Part 2, we’ll use the OpenRouter Agent SDK to build the first version of our Personal State Capitol Tracker.
