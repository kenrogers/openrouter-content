# How to Turn an Image into a Video

Use this guide when you have an image that should become the first or last frame of a generated video.

By the end, you will submit an image-to-video job with `frame_images` and download the finished clip.

## Before you start

You need:

- An OpenRouter API key available as `OPENROUTER_API_KEY`
- A public HTTPS URL for your image
- A model that supports `frame_images`, confirmed with `GET /api/v1/videos/models`

`frame_images` is for exact frame control. If you provide both `frame_images` and `input_references`, OpenRouter treats the request as image-to-video.

Use a stable, directly downloadable image URL. Some providers cannot fetch image URLs that require cookies, redirects through HTML pages, bot checks, or unusual headers.

## Step 1: Choose a model with frame-image support

Fetch the model list and choose a model whose `supported_frame_images` includes the frame type you want:

```bash
curl https://openrouter.ai/api/v1/videos/models \
  -H "Authorization: Bearer $OPENROUTER_API_KEY"
```

For first-frame and last-frame control, look for `supported_frame_images` containing `first_frame` and `last_frame`.

## Step 2: Submit the image-to-video job

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
      "The camera slowly pushes in as the subject turns toward warm window light, cinematic, realistic motion",
    duration: 4,
    resolution: "720p",
    aspect_ratio: "16:9",
    generate_audio: false,
    frame_images: [
      {
        type: "image_url",
        image_url: {
          url: "https://example.com/direct-first-frame.png",
        },
        frame_type: "first_frame",
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

## Step 3: Use a last frame when you need a transition

If the selected model supports `last_frame`, include both frames:

```ts
frame_images: [
  {
    type: "image_url",
    image_url: { url: "https://example.com/first-frame.png" },
    frame_type: "first_frame",
  },
  {
    type: "image_url",
    image_url: { url: "https://example.com/last-frame.png" },
    frame_type: "last_frame",
  },
];
```

This is useful when you want the video to move from a known starting composition to a known ending composition.

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

The first frame of the resulting video should closely match the image you provided as `first_frame`. If you also supplied `last_frame`, the clip should resolve toward that image.

## Video tutorial

Title: Turn a still image into a video with OpenRouter

Run time: 2-3 minutes

1. Show the starting image and the target: animate it into a short clip.
2. Open the model discovery response and point to `supported_frame_images`.
3. Explain `first_frame` versus `last_frame`.
4. Build the `POST /api/v1/videos` payload with `frame_images`.
5. Submit the job and show the returned `id` and `polling_url`.
6. Poll the job and download the completed `.mp4`.
7. Compare the starting image to the first frame of the output.
