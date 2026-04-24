# How to Choose a Video Generation Model

Use this guide when you want your app to pick an OpenRouter video model based on the clip you need to generate.

By the end, you will have a small model-selection helper that filters models by capability before you submit a video job.

## Before you start

You need an OpenRouter API key available as `OPENROUTER_API_KEY`.

## Step 1: Fetch the video model list

Call the dedicated video model endpoint:

```ts
type VideoModel = {
  id: string;
  supported_resolutions?: string[];
  supported_aspect_ratios?: string[];
  supported_durations?: number[];
  supported_frame_images?: string[] | null;
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

console.log(models.map((model) => model.id));
```

Each model includes the values you need for routing decisions, including supported resolutions, aspect ratios, sizes, durations, frame-image support, audio support, seed support, pricing SKUs, and allowed passthrough parameters.

## Step 2: Filter by the job you want to run

For example, this helper finds a model that can generate a 720p, vertical, image-to-video clip with first-frame support:

```ts
function findVideoModel(models: VideoModel[]) {
  return models.find((model) => {
    return (
      model.supported_resolutions?.includes("720p") &&
      model.supported_aspect_ratios?.includes("9:16") &&
      model.supported_durations?.includes(5) &&
      model.supported_frame_images?.includes("first_frame")
    );
  });
}

const model = findVideoModel(models);

if (!model) {
  throw new Error("No matching video model found.");
}

console.log(`Use ${model.id}`);
```

## Step 3: Use that model in your generation request

```ts
const generation = await fetch("https://openrouter.ai/api/v1/videos", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: model.id,
    prompt: "A handheld vertical product shot of a ceramic mug on a sunny kitchen counter",
    duration: 5,
    resolution: "720p",
    aspect_ratio: "9:16",
    frame_images: [
      {
        type: "image_url",
        image_url: {
          url: "https://example.com/first-frame.png",
        },
        frame_type: "first_frame",
      },
    ],
  }),
});

if (!generation.ok) {
  throw new Error(await generation.text());
}

console.log(await generation.json());
```

## Check your work

You should see a response with a video job `id`, a `polling_url`, and an initial status such as `pending`.

## Video tutorial

Title: Choose the right OpenRouter video model before generating

Run time: 60-90 seconds

1. Show the problem: video models support different durations, aspect ratios, image inputs, audio, and passthrough options.
2. Open the terminal and fetch `https://openrouter.ai/api/v1/videos/models`.
3. Highlight `supported_resolutions`, `supported_aspect_ratios`, `supported_durations`, `supported_frame_images`, `generate_audio`, and `allowed_passthrough_parameters`.
4. Paste the filtering helper and explain the target job: 720p, 9:16, 5 seconds, first-frame image support.
5. Submit a generation request using the selected `model.id`.
6. Close with the rule of thumb: discover capabilities first, then submit the job.
