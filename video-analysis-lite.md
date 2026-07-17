# Video analysis Lite — reference-to-generation workflow

Use this workflow when the goal is to make a new video inspired by a reference: generate the still images, animate them into clips, and reproduce the reference's pacing and editing grammar.

This is not a forensic frame-by-frame audit. It recovers the creative decisions that matter for production while keeping extraction, image inspection, and report length small.

Use `video-analysis.md` instead when the user needs exact cut timestamps, every-frame coverage, literal transcription, measured camera motion, legal/compliance evidence, or a defensible archival analysis.

## What Lite must recover

A Lite analysis is complete when it can answer five questions:

1. What is the narrative arc?
2. What visual ingredients make the reference recognizable?
3. What still images need to be generated?
4. How should each still move when converted into a clip?
5. How should the clips be timed and joined?

Everything else is optional.

## Default budget

Unless the user asks for more detail:

- Inspect no more than **24 representative frames** with an image-capable model.
- Reduce the film to **6–12 production beats**.
- Produce **6–12 image briefs** and matching motion briefs.
- Use **1 fps overview sampling** for videos up to three minutes.
- Use scene detection to catch short inserts, but keep only one useful representative per scene.
- Do not generate a literal per-frame report.
- Do not inspect every detected cut with neighboring frames.
- Do not analyze audio beyond captions and broad timing unless sound design is part of the requested recreation.

This usually captures the transferable creative grammar with a fraction of the image inspection required by the full workflow.

## Core rules

1. **Analyze mechanisms, not pixels.** Recover composition, palette, subject type, motion, pacing, and transition logic instead of cataloging every visible detail.
2. **Cover the whole video cheaply.** Sparse overview frames plus cut representatives are enough for inspiration work.
3. **Cap the keyframe deck.** Select the frames that explain the film; do not send hundreds of similar frames to an image model.
4. **Separate observation from invention.** State what the reference does, then write a new generation brief that uses the mechanism without copying a specific person, logo, or frame.
5. **Design backward from production.** Every observation should change an image prompt, motion prompt, duration, transition, or sound cue.
6. **Use exact timing only where it matters.** Approximate shot durations to the nearest 0.25 or 0.5 seconds unless a beat, title, or transition needs tighter placement.
7. **Choose continuity deliberately.** Do not chain every clip through matching end/start frames when the reference uses hard cuts between unrelated scenes.

## Required tools

Core:

- `yt-dlp` for supported URLs
- FFmpeg and FFprobe
- An image-capable model or analysis tool

Optional:

- Creator captions or Whisper when spoken structure matters
- Tesseract for a few title cards
- A waveform only when edits visibly depend on music or sound hits

Preflight:

```bash
command -v ffmpeg
command -v ffprobe
python3 -m yt_dlp --version 2>/dev/null || command -v yt-dlp
```

## Workspace

```text
video-analysis-lite/
├── source.mp4
├── metadata.json
├── ffprobe.json
├── overview/
├── cut-reps/
├── keyframes/
├── contact-sheets/
├── captions.vtt
├── analysis-lite.md
└── generation-plan.md
```

Keep the source unchanged. Derived images belong in the other directories.

## Phase 1: acquire a usable source

Download a clean 720p or 1080p source. Lite does not require the highest-bitrate codec if a normal 1080p rendition is available and readable.

```bash
URL='https://example.com/video'
WORK='tmp/video-analysis-lite'
mkdir -p "$WORK"/{overview,cut-reps,keyframes,contact-sheets}

python3 -m yt_dlp --dump-single-json --skip-download "$URL" \
  > "$WORK/metadata.json"

python3 -m yt_dlp \
  -f 'bestvideo[height<=1080]+bestaudio/best[height<=1080]/best' \
  --merge-output-format mp4 \
  --write-subs --write-auto-subs --sub-langs 'en.*' --sub-format vtt \
  -o "$WORK/source.%(ext)s" \
  "$URL"
```

Use the actual downloaded extension instead of assuming MP4.

Probe only the fields needed for planning:

```bash
SOURCE="$WORK/source.mp4"
ffprobe -v error \
  -show_entries \
format=duration:stream=codec_type,width,height,r_frame_rate,avg_frame_rate,sample_rate,channels \
  -of json "$SOURCE" > "$WORK/ffprobe.json"
```

Record duration, resolution, aspect ratio, frame rate, and whether audio exists. Stop there unless a technical issue appears.

## Phase 2: cover the timeline cheaply

### Default sampling

For a reference up to three minutes:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf 'fps=1,scale=960:-2' \
  -q:v 2 \
  "$WORK/overview/frame-%04d.jpg"
```

For three to fifteen minutes, use `fps=0.25`. For longer videos, sample chapter or scene representatives rather than the whole film at a fixed rate.

### Catch short inserts

Use one permissive scene pass. This still evaluates the full timeline, but it exports only likely changes:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf "select='gt(scene,0.25)',scale=960:-2" \
  -fps_mode vfr -q:v 2 \
  "$WORK/cut-reps/cut-%04d.jpg"
```

Do not treat every output as a real cut. Remove:

- full-black duplicates
- repeated frames from the same scene
- exposure or flash variations
- frames that add no new production information

If the film has many cuts to black, keep the black pattern as an editing note rather than filling the keyframe deck with black images.

### Build two contact sheets

Make one overview sheet and one scene-representative sheet. Split them only when the tiles become unreadable.

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf 'fps=1,scale=384:-2,tile=5x4:padding=4:margin=4' \
  "$WORK/contact-sheets/overview-%02d.jpg"
```

A custom sheet from the selected `cut-reps/` is optional. The important limit is the final keyframe deck, not the number of temporary detector outputs.

## Phase 3: select the keyframe deck

Choose **12–24 frames** that explain the reference. Prefer frames that establish:

- opening hook
- each narrative turn
- each distinct composition family
- palette changes
- repeated motifs
- the fastest montage section
- title or brand treatment
- final image

For each selected frame, record only:

- timestamp, approximate to 0.25 seconds
- subject and action
- framing and composition
- palette and lighting
- texture or medium
- why the frame matters to the sequence

Do not spend tokens describing nearly identical frames. Group them as a visual family.

## Phase 4: extract the creative DNA

Summarize the reference in one page.

### Narrative arc

Reduce the film to 3–5 phases, such as:

```text
Hook → tension → human turn → acceleration → resolution
```

For each phase, state what changes visually and why the change matters.

### Visual system

Capture the repeatable design choices:

- aspect ratio
- photographic, illustrated, rendered, or mixed medium
- subject families
- wide/medium/close-up balance
- symmetry, negative space, layering, or centered framing
- dominant and accent colors
- hard/soft light and contrast
- grain, haze, bloom, motion blur, or clean digital texture
- typography placement and scale

### Motion system

Describe only motion that should enter a generation prompt:

- locked still with subtle environmental movement
- slow push-in or pull-back
- pan, orbit, or parallax
- subject gesture
- particles, smoke, water, fabric, or light movement
- speed ramp or freeze
- no camera movement

Do not invent camera moves merely to make a still “more cinematic.” A held image may be the reference's defining choice.

### Editing system

Record the reusable edit grammar:

- approximate shot-duration range
- hard cuts, black gaps, dissolves, or match cuts
- where pacing accelerates or relaxes
- whether shots cut on speech, music, gesture, or visual contrast
- title-card duration and placement
- final hold length

Use rounded durations unless exact rhythm is the point.

## Phase 5: choose the image-to-video continuity mode

This decision connects the analysis to `image-to-video.md`.

### Mode A: independent montage

Use this when the reference cuts between unrelated images or locations.

1. Generate each still independently.
2. Animate each still into its own short clip.
3. Preserve composition within the clip.
4. Join clips with the reference's hard cuts, black gaps, or other transitions.
5. Do not force the last frame of one scene to match the first frame of a different scene.

### Mode B: seamless continuation

Use this when multiple clips form one continuous shot or transformation.

1. Generate the first still.
2. Animate clip 1.
3. Extract its exact final frame.
4. Use that frame as clip 2's start frame.
5. Repeat and concatenate with no added transition.

Core rule from `image-to-video.md`:

```text
End frame of clip N = start frame of clip N+1
```

### Mode C: hybrid

Most generated films should use this.

- Chain clips within one scene when continuity matters.
- Start from a new independently generated still at each intended hard cut.
- Use the edit to hide model drift instead of trying to maintain impossible continuity across unrelated scenes.

Assign every shot a `continuity_group`. Shots in the same group use end/start-frame chaining. A new group starts from a new still.

## Phase 6: produce the generation plan

Create one row per planned shot. Six to twelve shots are enough by default.

| Shot | Duration | Still-image brief | Motion-only brief | Edit in/out | Continuity group |
| --- | ---: | --- | --- | --- | --- |
| S01 | 1.5 s | Subject, composition, lens feel, palette, light, texture, safe margins | Only movement, camera behavior, stability, and end state | Open / hard cut to black | A |

### Still-image brief

Each brief should specify:

- original subject and setting
- shot size and viewpoint
- subject placement and negative space
- lighting direction and contrast
- palette
- surface texture or image medium
- target aspect ratio
- safe margins for animation and reframing
- elements that must remain stable
- elements to avoid

Do not name a living photographer or request a copy of a recognizable reference frame. Translate the reference into visual properties.

### Motion-only brief

Assume the still is already correct. Prompt only for:

- subject motion
- environmental motion
- camera motion, or explicitly locked camera
- motion intensity and speed
- stable elements
- desired final state
- no cuts, no new subjects, and no composition replacement

Example:

```text
Locked camera. Hold the composition. Only the distant smoke drifts slowly left and the reflected light flickers. Preserve the building, horizon, and framing. No cuts, no new objects, no zoom. End on a stable frame.
```

### Editing brief

For every boundary, choose one:

- hard cut
- cut to black with a stated hold duration
- matched-frame continuation
- dissolve or fade
- title card

Do not add transitions just because the editor offers them.

## Phase 7: use audio only as far as production needs it

If narration drives the structure:

- use creator captions or a quick transcript
- divide the spoken material into 3–8 semantic beats
- align image families to those beats

If music drives the cuts:

- create one waveform
- note major peaks, dropouts, and the final release
- time edits approximately to those events

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -filter_complex 'showwavespic=s=1600x400:colors=white' \
  -frames:v 1 \
  "$WORK/contact-sheets/waveform.png"
```

Skip loudness, spectrogram, and silence analysis unless the new film needs a sound-design specification.

## Phase 8: write the Lite report

Use this structure:

```markdown
# [Reference title] — production-oriented video analysis

Source:
Duration / aspect ratio:
Continuity mode:

## One-paragraph creative reading

## Narrative and pacing map

| Phase | Approx. time | Visual change | Editing change | Production lesson |

## Visual DNA

## Motion DNA

## Editing recipe

## Keyframe deck

## Still-image generation plan

## Clip motion plan

## Final assembly order

## Text and audio notes

## What to borrow / what not to copy

## Escalation notes
```

The report should read like a production brief, not a forensic record.

## Quality gates

### Coverage

- [ ] The opening, every major turn, fastest section, title treatment, and ending are represented.
- [ ] At least one frame from every distinct visual family was inspected.
- [ ] Short inserts were checked through scene representatives, not only fixed-rate sampling.

### Generation readiness

- [ ] Every planned shot has a still-image brief.
- [ ] Every planned shot has a motion-only brief.
- [ ] Every boundary has an explicit edit instruction.
- [ ] Every shot has a continuity group.
- [ ] Safe margins and stable elements are stated for images that will be animated.

### Restraint

- [ ] No more than 24 keyframes were sent for detailed visual analysis unless the cap was explicitly raised.
- [ ] Similar shots were grouped instead of described repeatedly.
- [ ] Exact frame numbers, OCR, and audio metrics were omitted unless they affect production.
- [ ] Observed reference traits are separated from the new creative proposal.

## Escalate to the full workflow only when needed

Switch to `video-analysis.md`, or run only the relevant full phase, when:

- a transition cannot be classified from the contact sheets
- a one-frame insert or exact cut must be reproduced
- small on-screen text must be transcribed exactly
- the source is variable-frame-rate and frame identity matters
- camera motion must be measured rather than described
- sound effects must land on exact frames
- the user asks for complete timeline coverage or evidence-grade reporting

Do not restart the whole analysis automatically. Escalate only the uncertain section.

## Minimal reproducible Lite command set

```bash
URL='https://example.com/video'
WORK='tmp/video-analysis-lite'
mkdir -p "$WORK"/{overview,cut-reps,keyframes,contact-sheets}

python3 -m yt_dlp \
  -f 'bestvideo[height<=1080]+bestaudio/best[height<=1080]/best' \
  --merge-output-format mp4 \
  -o "$WORK/source.%(ext)s" \
  "$URL"

SOURCE="$WORK/source.mp4"

ffprobe -v error \
  -show_entries format=duration:stream=codec_type,width,height,r_frame_rate,avg_frame_rate \
  -of json "$SOURCE" > "$WORK/ffprobe.json"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -vf 'fps=1,scale=960:-2' -q:v 2 \
  "$WORK/overview/frame-%04d.jpg"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -vf "select='gt(scene,0.25)',scale=960:-2" \
  -fps_mode vfr -q:v 2 \
  "$WORK/cut-reps/cut-%04d.jpg"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -vf 'fps=1,scale=384:-2,tile=5x4:padding=4:margin=4' \
  "$WORK/contact-sheets/overview-%02d.jpg"
```

The commands create evidence. The useful deliverable is the compact generation plan: original still briefs, motion-only clip prompts, continuity groups, and an explicit edit map.
