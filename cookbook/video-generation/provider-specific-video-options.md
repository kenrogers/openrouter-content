# How to Use Provider-Specific Video Options

Use this guide when a video model exposes controls that are not part of OpenRouter's normalized video schema.

By the end, you will inspect a model's allowed passthrough parameters and send one through `provider.options`.

## Before you start

You need an OpenRouter API key available as `OPENROUTER_API_KEY`.

Provider-specific options can change by model and provider. Always check `allowed_passthrough_parameters` before relying on one.

## Step 1: Inspect allowed passthrough parameters

```ts
type VideoModel = {
  id: string;
  allowed_passthrough_parameters?: string[];
};

const response = await fetch("https://openrouter.ai/api/v1/videos/models", {
  headers: {
    Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
  },
});

if (!response.ok) {
  throw new Error(await response.text());
}

const { data } = await response.json();
const models = data as VideoModel[];
const veo = models.find((model: VideoModel) => model.id === "google/veo-3.1-lite");

if (!veo) {
  throw new Error("google/veo-3.1-lite was not found in the video model list.");
}

console.log(veo.allowed_passthrough_parameters);
```

For example, `google/veo-3.1-lite` may expose passthrough controls such as `personGeneration`, `negativePrompt`, `conditioningScale`, or `enhancePrompt`.

## Step 2: Add provider options to the video request

Provider options are keyed by provider slug. Only the options for the matched provider are forwarded.

```ts
const response = await fetch("https://openrouter.ai/api/v1/videos", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: "google/veo-3.1-lite",
    prompt:
      "A 4-second time-lapse of a white orchid blooming on a dark tabletop, macro lens, gentle studio light",
    duration: 4,
    resolution: "720p",
    aspect_ratio: "16:9",
    generate_audio: false,
    provider: {
      options: {
        "google-vertex": {
          parameters: {
            personGeneration: "allow",
            negativePrompt: "blurry, low quality, distorted petals",
            enhancePrompt: true,
          },
        },
      },
    },
  }),
});

if (!response.ok) {
  throw new Error(await response.text());
}

console.log(await response.json());
```

## Step 3: Keep a fallback without passthrough options

If your app can route across multiple video models, keep the normalized request separate from model-specific options:

```ts
const baseRequest = {
  prompt: "A short cinematic product shot of a white orchid blooming",
  duration: 4,
  resolution: "720p",
  aspect_ratio: "16:9",
};

const selectedModel = {
  id: "google/veo-3.1-lite",
};

const requestBody =
  selectedModel.id === "google/veo-3.1-lite"
    ? {
        ...baseRequest,
        model: selectedModel.id,
        provider: {
          options: {
            "google-vertex": {
              parameters: {
                negativePrompt: "blurry, low quality",
              },
            },
          },
        },
      }
    : {
        ...baseRequest,
        model: selectedModel.id,
      };
```

## Check your work

Your request should return a video job. If the provider option is invalid for the selected model, remove it or re-check the current `allowed_passthrough_parameters` list.

## Video tutorial

Title: Tune OpenRouter video models with provider-specific options

Run time: 2 minutes

1. Show a normalized video request first.
2. Fetch `GET /api/v1/videos/models`.
3. Highlight `allowed_passthrough_parameters`.
4. Add `provider.options` for `google/veo-3.1-lite`.
5. Submit the job and show the response.
6. Close with the production pattern: use normalized parameters by default, passthrough only after checking support.
