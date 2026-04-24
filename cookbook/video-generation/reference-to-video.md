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

Use the `polling_url` returned by the submit request until the job is `completed`:

```bash
POLLING_URL="https://openrouter.ai/api/v1/videos/{jobId}"

while true; do
  JOB_JSON=$(curl -s "$POLLING_URL" \
    -H "Authorization: Bearer $OPENROUTER_API_KEY")

  STATUS=$(printf '%s' "$JOB_JSON" | node -e 'let body=""; process.stdin.on("data", d => body += d); process.stdin.on("end", () => console.log(JSON.parse(body).status));')

  echo "Status: $STATUS"

  if [ "$STATUS" = "completed" ]; then
    break
  fi

  if [ "$STATUS" = "failed" ]; then
    echo "$JOB_JSON"
    exit 1
  fi

  sleep 30
done
```

Then download the content:

```bash
curl "https://openrouter.ai/api/v1/videos/{jobId}/content?index=0" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  --output reference-video.mp4
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
