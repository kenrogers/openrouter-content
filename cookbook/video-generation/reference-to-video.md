# How to Guide a Video with Reference Images

Use this guide when you want images to influence a generated video without forcing them to be exact first or last frames.

By the end, you will submit a reference-to-video job with `input_references`.

## Before you start

You need:

- An OpenRouter API key available as `OPENROUTER_API_KEY`
- One or more public HTTPS image URLs
- A model that supports reference-to-video, confirmed from the current OpenRouter video docs or model description

Use `input_references` for visual guidance. Use `frame_images` only when you need exact frame control.

Use stable, directly downloadable image URLs. Some providers cannot fetch image URLs that require cookies, redirects through HTML pages, bot checks, or unusual headers.

## Step 1: Write a prompt that tells the model how to use the references

Reference images work best when the prompt explains what should stay consistent.

```text
Create a 4-second product video of the same backpack from the reference image.
Keep the shape, color, and logo placement consistent.
Place it on a wet city sidewalk at night with neon reflections.
Use a slow orbiting camera move and realistic lighting.
```

## Step 2: Submit the reference-to-video job

```ts
const response = await fetch("https://openrouter.ai/api/v1/videos", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: "bytedance/seedance-2.0-fast",
    prompt:
      "Create a 4-second product video of the same backpack from the reference image. Keep the shape, color, and logo placement consistent. Place it on a wet city sidewalk at night with neon reflections. Use a slow orbiting camera move and realistic lighting.",
    duration: 4,
    resolution: "720p",
    aspect_ratio: "16:9",
    input_references: [
      {
        type: "image_url",
        image_url: {
          url: "https://example.com/direct-backpack-reference.png",
        },
      },
    ],
  }),
});

if (!response.ok) {
  throw new Error(await response.text());
}

const job = await response.json();
console.log(job);
```

## Step 3: Add more references when consistency matters

Some models can use multiple reference images. Before doing this in production, check the current docs or model description for the selected model.

```ts
input_references: [
  {
    type: "image_url",
    image_url: { url: "https://example.com/character-front.png" },
  },
  {
    type: "image_url",
    image_url: { url: "https://example.com/character-side.png" },
  },
  {
    type: "image_url",
    image_url: { url: "https://example.com/style-reference.png" },
  },
];
```

## Step 4: Poll and download

In a real app, run this from a server route, worker, or job runner instead of the browser. This minimal helper keeps the foundational flow explicit: poll with a limit, stop on failure, then download the completed video.

```ts
type VideoJob = {
  id: string;
  polling_url: string;
  status: string;
  unsigned_urls?: string[];
  error?: string;
};

async function waitForVideo(job: VideoJob) {
  let current = job;

  for (let attempt = 1; attempt <= 60; attempt += 1) {
    if (current.status === "completed") {
      return current;
    }

    if (current.status === "failed") {
      throw new Error(current.error ?? "Video generation failed.");
    }

    await new Promise((resolve) => setTimeout(resolve, 30_000));

    const response = await fetch(current.polling_url, {
      headers: {
        Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
      },
    });

    if (!response.ok) {
      throw new Error(await response.text());
    }

    current = await response.json();
  }

  throw new Error("Video generation did not complete after 60 attempts.");
}

async function downloadVideo(job: Pick<VideoJob, "id" | "unsigned_urls">) {
  const videoUrl =
    job.unsigned_urls?.[0] ??
    `https://openrouter.ai/api/v1/videos/${job.id}/content?index=0`;

  const response = await fetch(videoUrl, {
    headers: videoUrl.startsWith("https://openrouter.ai/api/")
      ? { Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}` }
      : undefined,
  });

  if (!response.ok) {
    throw new Error(await response.text());
  }

  return Buffer.from(await response.arrayBuffer());
}

const completedJob = await waitForVideo(job);
const videoBuffer = await downloadVideo(completedJob);
```

## Check your work

The output should borrow subject, style, or identity cues from the reference images while still following the generated scene described in the prompt.

## Video tutorial

Title: Use reference images to guide an OpenRouter video

Run time: 2 minutes

1. Show the reference image or images.
2. Explain the distinction: `input_references` guide the video, `frame_images` control exact frames.
3. Write a prompt that names which details should remain consistent.
4. Add the `input_references` array to the request body.
5. Submit, poll, and download the result.
6. Review the output against the reference images.
