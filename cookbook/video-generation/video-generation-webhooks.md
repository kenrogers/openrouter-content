# How to Get Video Results with Webhooks

Use this guide when you want OpenRouter to notify your app when a video job finishes instead of polling from a client or worker.

By the end, you will submit a video job with `callback_url` and verify the webhook signature.

## Before you start

You need:

- An OpenRouter API key available as `OPENROUTER_API_KEY`
- A public HTTPS endpoint for your webhook receiver
- A webhook signing secret configured in your OpenRouter workspace settings

## Step 1: Create a webhook receiver

This Express example preserves the raw request body, which is required for signature verification:

```ts
import crypto from "node:crypto";
import express from "express";

const app = express();
const signingSecret = process.env.OPENROUTER_WEBHOOK_SECRET;

function verifyOpenRouterSignature(rawBody: Buffer, header: string) {
  if (!signingSecret) return false;

  const parts = header.split(",").map((part) => part.trim());
  const timestamp = parts.find((part) => part.startsWith("t="))?.slice(2);
  const signature = parts.find((part) => part.startsWith("v1="))?.slice(3);

  if (!timestamp || !signature) return false;

  const age = Math.floor(Date.now() / 1000) - Number(timestamp);
  if (Number.isNaN(age) || age > 300) return false;

  const signedPayload = Buffer.concat([
    Buffer.from(`${timestamp},`, "utf8"),
    rawBody,
  ]);
  const expected = crypto
    .createHmac("sha256", signingSecret)
    .update(signedPayload)
    .digest("hex");

  if (expected.length !== signature.length) return false;

  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature),
  );
}

app.post(
  "/openrouter/video-webhook",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const signature = req.header("X-OpenRouter-Signature");

    if (!signature || !verifyOpenRouterSignature(req.body, signature)) {
      return res.sendStatus(401);
    }

    const event = JSON.parse(req.body.toString("utf8"));

    if (event.status === "completed") {
      console.log("Video ready:", event.unsigned_urls?.[0]);
    }

    if (event.status === "failed") {
      console.error("Video failed:", event.error);
    }

    res.sendStatus(204);
  },
);

app.listen(3000, () => {
  console.log("Listening on http://localhost:3000");
});
```

## Step 2: Submit a video job with `callback_url`

```ts
const response = await fetch("https://openrouter.ai/api/v1/videos", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: "google/veo-3.1",
    prompt: "A clean product reveal of a matte black desk lamp, slow camera slide, studio lighting",
    duration: 4,
    resolution: "720p",
    aspect_ratio: "16:9",
    callback_url: "https://your-app.example.com/openrouter/video-webhook",
  }),
});

if (!response.ok) {
  throw new Error(await response.text());
}

console.log(await response.json());
```

The per-request `callback_url` takes priority over a workspace-level default callback URL.

## Step 3: Handle the completed job

The webhook payload includes the job `id`, `status`, `generation_id`, `model`, `unsigned_urls`, and `usage`. Store the job state in your database, then download the video from the first `unsigned_urls` entry or from the content endpoint. If the URL points to the OpenRouter API, include the bearer token when downloading it.

## Check your work

Your receiver should return `204` for a valid OpenRouter webhook and `401` for a request with a missing or invalid signature.

## Video tutorial

Title: Receive OpenRouter video results with webhooks

Run time: 3 minutes

1. Show why polling is awkward for long-running video jobs.
2. Create the Express route with `express.raw`.
3. Add signature verification and highlight the raw-body requirement.
4. Submit a video job with `callback_url`.
5. Show the webhook payload for a completed job.
6. Store or download the finished video from `unsigned_urls[0]`, including auth when it points to the OpenRouter API.
