# How to Generate and Download a Video from Text

Use this guide when you want to turn a text prompt into a finished video file with OpenRouter.

By the end, you will submit a video job, poll for completion, and save the generated video locally.

## Before you start

You need:

- An OpenRouter API key available as `OPENROUTER_API_KEY`
- Node.js 20 or newer
- A video model slug, such as `google/veo-3.1`, confirmed with `GET /api/v1/videos/models`

## Step 1: Submit the video job

Create `generate-video.ts`:

```ts
import { writeFile } from "node:fs/promises";

const apiKey = process.env.OPENROUTER_API_KEY;

if (!apiKey) {
  throw new Error("Set OPENROUTER_API_KEY first.");
}

async function openrouter(path: string, init: RequestInit = {}) {
  const response = await fetch(`https://openrouter.ai/api/v1${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });

  if (!response.ok) {
    throw new Error(await response.text());
  }

  return response;
}

const submitResponse = await openrouter("/videos", {
  method: "POST",
  body: JSON.stringify({
    model: "google/veo-3.1",
    prompt:
      "A cinematic 4-second shot of a glass greenhouse at sunrise, soft mist, slow dolly-in camera movement",
    duration: 4,
    resolution: "720p",
    aspect_ratio: "16:9",
    generate_audio: true,
  }),
});

const job = await submitResponse.json();
console.log(`Submitted video job: ${job.id}`);
```

## Step 2: Poll until the job finishes

Add this below the submit step:

```ts
let status = job;

for (let attempt = 1; attempt <= 60; attempt += 1) {
  if (status.status === "completed") {
    break;
  }

  if (status.status === "failed") {
    throw new Error(status.error ?? "Video generation failed.");
  }

  await new Promise((resolve) => setTimeout(resolve, 30_000));

  const pollResponse = await fetch(status.polling_url, {
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!pollResponse.ok) {
    throw new Error(await pollResponse.text());
  }

  status = await pollResponse.json();
  console.log(`Status: ${status.status}`);
}

if (status.status !== "completed") {
  throw new Error("Video generation did not complete after 60 attempts.");
}
```

## Step 3: Download the video

Add this final block:

```ts
const videoResponse = await fetch(
  `https://openrouter.ai/api/v1/videos/${job.id}/content?index=0`,
  {
    headers: { Authorization: `Bearer ${apiKey}` },
  },
);

if (!videoResponse.ok) {
  throw new Error(await videoResponse.text());
}

const videoBuffer = Buffer.from(await videoResponse.arrayBuffer());
await writeFile("greenhouse.mp4", videoBuffer);

console.log("Saved greenhouse.mp4");
```

You can also download from the first URL in the completed status. If that URL points to the OpenRouter API, include the bearer token:

```ts
const videoUrl = status.unsigned_urls?.[0];

if (!videoUrl) {
  throw new Error("Completed job did not include a downloadable video URL.");
}

const videoResponse = await fetch(videoUrl, {
  headers: videoUrl.startsWith("https://openrouter.ai/api/")
    ? { Authorization: `Bearer ${apiKey}` }
    : undefined,
});

if (!videoResponse.ok) {
  throw new Error(await videoResponse.text());
}

const videoBuffer = Buffer.from(await videoResponse.arrayBuffer());
await writeFile("greenhouse.mp4", videoBuffer);

console.log("Saved greenhouse.mp4");
```

## Complete script

```ts
import { writeFile } from "node:fs/promises";

const apiKey = process.env.OPENROUTER_API_KEY;

if (!apiKey) {
  throw new Error("Set OPENROUTER_API_KEY first.");
}

async function postJson(path: string, body: unknown) {
  const response = await fetch(`https://openrouter.ai/api/v1${path}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    throw new Error(await response.text());
  }

  return response.json();
}

const job = await postJson("/videos", {
  model: "google/veo-3.1",
  prompt:
    "A cinematic 4-second shot of a glass greenhouse at sunrise, soft mist, slow dolly-in camera movement",
  duration: 4,
  resolution: "720p",
  aspect_ratio: "16:9",
  generate_audio: true,
});

console.log(`Submitted video job: ${job.id}`);

let status = job;

for (let attempt = 1; attempt <= 60; attempt += 1) {
  if (status.status === "completed") {
    break;
  }

  if (status.status === "failed") {
    throw new Error(status.error ?? "Video generation failed.");
  }

  await new Promise((resolve) => setTimeout(resolve, 30_000));

  const pollResponse = await fetch(status.polling_url, {
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!pollResponse.ok) {
    throw new Error(await pollResponse.text());
  }

  status = await pollResponse.json();
  console.log(`Status: ${status.status}`);
}

if (status.status !== "completed") {
  throw new Error("Video generation did not complete after 60 attempts.");
}

const videoUrl = status.unsigned_urls?.[0];

if (!videoUrl) {
  throw new Error("Completed job did not include a downloadable video URL.");
}

const videoResponse = await fetch(videoUrl, {
  headers: videoUrl.startsWith("https://openrouter.ai/api/")
    ? { Authorization: `Bearer ${apiKey}` }
    : undefined,
});

if (!videoResponse.ok) {
  throw new Error(await videoResponse.text());
}

const videoBuffer = Buffer.from(await videoResponse.arrayBuffer());
await writeFile("greenhouse.mp4", videoBuffer);

console.log("Saved greenhouse.mp4");
```

## Step 4: Run it

```bash
npx tsx generate-video.ts
```

## Check your work

You should see the job move from `pending` or `in_progress` to `completed`, followed by a saved `greenhouse.mp4` file.

## Video tutorial

Title: Generate your first OpenRouter video from a text prompt

Run time: 2-3 minutes

1. Show the goal: a text prompt becomes a downloaded `.mp4`.
2. Briefly show the async workflow: submit, poll, download.
3. Paste the TypeScript setup and submit the `POST /api/v1/videos` request.
4. Point out the required fields: `model` and `prompt`.
5. Point out the optional controls: `duration`, `resolution`, `aspect_ratio`, and `generate_audio`.
6. Add the polling loop and explain that video jobs can take minutes.
7. Add the download step and run the script.
8. Open the resulting `.mp4` and close with the reusable pattern.
