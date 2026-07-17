# naqal

I saw the Kimi K3 launch video the day it came out. It was stunning — the pacing, the lighting, the topography, the way the audio locked into every cut. So I did what I always do: I pulled it apart.

I reverse-engineered the whole thing frame by frame — the scenes, the transitions, the color grade, the sound design, the rhythm, the typography, the lighting. I captured that entire process in a single prompt, then built another prompt that lets me generate images in the same style, and a third prompt that turns those images into video. The result is a workflow you can run on any video.

There is no lock-in to a specific model. Use whatever image generator or video generator you want — Midjourney, Flux, Kling, Runway, Luma, Pika, Kimi, or whatever ships next. The output quality depends on the model you choose, but the method stays the same: deconstruct a reference, encode the pattern, generate assets, assemble variants.

You can reverse-engineer any video and create as many variants of it as you want.

## What is here

- `image-to-video.md` — how to turn generated images into video clips
- `video-analysis.md` — the full frame-by-frame breakdown workflow
- `video-analysis-lite.md` — a lighter version of the analysis for faster runs
- `References/` — example analyses and prompt sets from videos I have already broken down

## How to use it

1. Pick a video you love.
2. Run it through the analysis prompts to capture the visual language, pacing, sound, and structure.
3. Use that analysis to generate image prompts in the same style.
4. Feed those images into the video-generation prompt to produce a variant.
5. Iterate until it feels right.
