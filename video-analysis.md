# Full video extraction and analysis pipeline

Use this workflow when the user wants a deep visual analysis of a video: frame-by-frame description, shot breakdown, edit and transition analysis, camera motion, pacing, on-screen text, audio structure, tone, motifs, or narrative flow.

The goal is not to dump isolated screenshots. Recover the best available source, account for the complete timeline, inspect transitions at frame-level precision, and produce a report in which every frame belongs to a described range.

## Core rules

1. **Analyze the source file, not the browser player.** Browser playback is useful for locating the media and preserving an authenticated session, but the player may show a lower rendition, hide frames during seeking, or expose only a `blob:` URL.
2. **Download the highest-quality non-upscaled source available.** Record which rendition was selected and verify it with `ffprobe`.
3. **Cover the entire timeline.** Overview sampling finds the structure; dense sampling and per-frame scene scores find short inserts and transitions.
4. **Treat cut detection as evidence, not truth.** Particle motion, flashes, zooms, and lighting changes can look like cuts. Dissolves and subtle match cuts may score weakly. Inspect candidate boundaries manually.
5. **Group unchanged frames into ranges.** A human-readable “frame-by-frame” report should describe every frame through contiguous frame ranges. If the user literally needs one record per frame, also export CSV or JSON.
6. **Separate observation from interpretation.** Write “a blue numeral 3 appears” as observation. Write “this likely teases version 3” as an explicitly labeled inference.
7. **Do not claim to have heard, read, or identified something without evidence.** Metadata does not prove that audio contains no speech. OCR output does not prove exact wording until the source frame is checked.
8. **Preserve the original media.** Generate derived frames, contact sheets, and audio artifacts in a separate analysis directory.

## Required tools

Use the tools that are available. The core path requires:

- **yt-dlp** for media extraction from supported URLs
- **FFmpeg** for decoding, frame extraction, scene scores, contact sheets, and audio analysis
- **FFprobe** for source metadata
- An image-capable analysis tool or model for visual inspection

Optional tools:

- **Chrome CDP + a browser extension relay** when the page must be opened in the user’s existing authenticated Chrome session
- **Tesseract** for OCR
- **PySceneDetect** as a secondary scene detector
- **Whisper** or another speech recognizer for dialogue and voice-over transcription
- **ImageMagick** for labeled contact sheets when FFmpeg lacks `drawtext`
- **OpenCV** or another optical-flow tool for quantitative motion analysis

Run a preflight check:

```bash
command -v ffmpeg
command -v ffprobe
python3 -m yt_dlp --version 2>/dev/null || command -v yt-dlp
command -v tesseract || true
ffmpeg -hide_banner -filters 2>/dev/null | grep -E 'drawtext|tile|select|showinfo' || true
```

Install only what is missing. Prefer `python3 -m yt_dlp` after a user-level Python installation so PATH differences do not break later commands.

## Workspace

Use one directory per video and keep the original separate from generated artifacts:

```text
video-analysis/
├── source.mp4
├── metadata.json
├── ffprobe.json
├── frames-overview/
├── frames-dense/
├── frames-cuts/
├── contact-sheets/
├── audio/
│   ├── track.wav
│   ├── spectrum.png
│   └── waveform.png
├── scene-scores.txt
├── cut-candidates.tsv
├── ocr/
└── analysis.md
```

Example setup:

```bash
WORK="tmp/video-analysis"
SOURCE="$WORK/source.mp4"
mkdir -p "$WORK"/{frames-overview,frames-dense,frames-cuts,contact-sheets,audio,ocr}
```

Use quoted variables throughout. Video URLs and local paths commonly contain characters that the shell interprets.

## Phase 1: acquire the source

### Public or directly supported URL

Capture metadata before downloading:

```bash
URL='https://example.com/video-page'
python3 -m yt_dlp --dump-single-json --skip-download "$URL" > "$WORK/metadata.json"
python3 -m yt_dlp -F "$URL"
```

Download the best separate video and audio streams, then merge without unnecessary re-encoding:

```bash
python3 -m yt_dlp \
  -f 'bestvideo+bestaudio/best' \
  --merge-output-format mp4 \
  -o "$WORK/source.%(ext)s" \
  "$URL"
```

The resulting extension may not be MP4 if the best streams cannot be merged into that container. Locate the actual file instead of assuming its name:

```bash
find "$WORK" -maxdepth 1 -type f \( -name 'source.mp4' -o -name 'source.mkv' -o -name 'source.webm' \) -print
```

If analysis tooling requires a normalized MP4, remux first and re-encode only when necessary:

```bash
ffmpeg -i "$WORK/source.mkv" -map 0 -c copy "$WORK/source.mp4"
```

### Local file

Copy or link the original into the workspace without transcoding:

```bash
cp '/path/to/input.mov' "$WORK/source.mov"
SOURCE="$WORK/source.mov"
```

### Existing authenticated Chrome session

Use this only when the user explicitly wants their running Chrome session or the page is accessible only while logged in.

- Connect through the extension relay or raw CDP endpoint.
- **Do not call a browser-launch command when the requirement is to use an existing browser.** A launcher can create a second Chrome process and temporary profile even when given a port.
- Verify the relay before acting:

```bash
curl -sS http://127.0.0.1:9222/
curl -sS http://127.0.0.1:9222/json/version
```

The relay response should report that the extension is connected. Open the URL through `Target.createTarget` only when a new tab inside the existing Chrome profile is acceptable.

Inspect the page’s video element through CDP:

```js
(() =>
  [...document.querySelectorAll("video")].map((video, index) => ({
    index,
    src: video.src,
    currentSrc: video.currentSrc,
    poster: video.poster,
    duration: video.duration,
    currentTime: video.currentTime,
    paused: video.paused,
    readyState: video.readyState,
    width: video.videoWidth,
    height: video.videoHeight,
    playbackRate: video.playbackRate,
    sourceTags: [...video.querySelectorAll("source")].map((source) => ({
      src: source.src,
      type: source.type,
    })),
  })))();
```

A `blob:` URL is not the downloadable source. Use the page URL with `yt-dlp`, inspect network requests for the HLS/DASH manifest, or export cookies for a downloader where permitted. Do not attempt to bypass DRM or access controls.

### X/Twitter-specific notes

For an X post:

1. Record the post URL, author, text, timestamp, and engagement context separately from the media analysis.
2. Use the post URL with `yt-dlp`; do not use the player’s `blob:` URL.
3. X commonly serves HLS fragments. Let `yt-dlp` download and merge the selected video and audio formats.
4. Verify the downloaded dimensions. The in-browser player may expose a 720p rendition while the source has a 4K stream.
5. If context matters, inspect the post/thread separately. Do not let social replies change the factual visual description.

## Phase 2: verify source fidelity

Generate a complete probe report:

```bash
ffprobe -v error \
  -show_format \
  -show_streams \
  -count_frames \
  -of json \
  "$SOURCE" > "$WORK/ffprobe.json"
```

Print the fields needed in the final report:

```bash
ffprobe -v error \
  -show_entries \
format=duration,size,bit_rate,format_name:\
stream=index,codec_name,codec_type,width,height,pix_fmt,color_space,color_transfer,color_primaries,r_frame_rate,avg_frame_rate,nb_frames,nb_read_frames,sample_rate,channels \
  -of json \
  "$SOURCE"
```

Record:

- Exact duration
- Width and height
- Display aspect ratio and rotation metadata
- Video and audio codecs
- Average and nominal frame rates
- Decoded frame count when available
- Audio sample rate and channel count
- HDR/color-transfer metadata
- File size and overall bitrate

### Constant versus variable frame rate

Do not calculate frame numbers as `time × fps` until the source is confirmed constant-frame-rate. `r_frame_rate` and `avg_frame_rate` can differ.

- For constant frame rate, a timestamp-to-frame estimate is acceptable: `frame ≈ floor(time × fps)`.
- For variable frame rate, use presentation timestamps from `ffprobe` or extracted frame filenames. Report timestamps as authoritative and frame numbers as source indices only.

To inspect frame timestamps:

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries frame=best_effort_timestamp_time,pkt_duration_time,key_frame,pict_type \
  -of csv=p=0 \
  "$SOURCE" > "$WORK/frame-timestamps.csv"
```

### Normalize only when necessary

Do not transcode merely for convenience. Normalize when the source is corrupted, interlaced, unusually encoded, or incompatible with analysis tools.

Example CFR analysis proxy:

```bash
ffmpeg -i "$SOURCE" \
  -map 0:v:0 -map 0:a? \
  -vf 'fps=24,scale=1920:-2' \
  -c:v libx264 -crf 16 -preset slow \
  -c:a aac -b:a 192k \
  "$WORK/analysis-proxy.mp4"
```

Keep the original and state clearly if timing analysis uses a normalized proxy.

## Phase 3: choose a sampling plan

Sampling should scale with duration and edit density.

| Video duration   |                   Overview sampling |                                  Dense sampling |                        Boundary sampling |
| ---------------- | ----------------------------------: | ----------------------------------------------: | ---------------------------------------: |
| Up to 60 seconds |                             1–2 fps |                                           4 fps | Every frame within ±0.25 s of candidates |
| 1–5 minutes      |                           0.5–1 fps |                  2–4 fps around active sections |               Every frame within ±0.25 s |
| 5–30 minutes     |                         0.1–0.5 fps |                      1–2 fps per detected scene |                Every frame within ±0.5 s |
| Over 30 minutes  | Chapter/scene representative frames | Dense only in requested or high-change sections |       Every frame at selected boundaries |

For a short, fast teaser, use **4 fps**, which means one frame every 0.25 seconds, across the entire timeline. Add all cut-adjacent frames. For a slow single-take clip, lower the base sampling rate but inspect motion continuously through smaller ranges where the camera or subject changes.

Sampling alone cannot prove that no one-frame insert exists. Per-frame scene scoring and boundary inspection provide that coverage.

## Phase 4: extract overview and dense frames

Overview frames at 2 fps:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf 'fps=2,scale=960:-2' \
  -q:v 2 \
  "$WORK/frames-overview/frame-%05d.jpg"
```

Dense frames at 4 fps:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf 'fps=4,scale=1280:-2' \
  -q:v 2 \
  "$WORK/frames-dense/frame-%05d.jpg"
```

Keep enough resolution for small text. Use 1920-pixel-wide frames or exact 4K crops for OCR-heavy material.

### Extract an exact source frame

Placing `-ss` before `-i` can seek to a nearby keyframe. For exact frame-level work, use a frame-selection filter:

```bash
FRAME=314
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf "select='eq(n,$FRAME)'" \
  -vsync 0 \
  "$WORK/frames-cuts/frame-$FRAME.png"
```

For exact timestamp extraction, place `-ss` after `-i`:

```bash
TIME='00:13.083'
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" -ss "$TIME" \
  -frames:v 1 \
  "$WORK/frames-cuts/time-13.083.png"
```

Use PNG for text, UI, diagrams, or compression-sensitive comparisons. JPEG is acceptable for broad contact sheets.

## Phase 5: build contact sheets

Contact sheets expose rhythm, repeated motifs, color progression, and missed inserts more efficiently than opening frames one at a time.

Example: 20 frames per sheet, arranged 5 × 4:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf 'fps=2,scale=480:-2,tile=5x4:padding=4:margin=4' \
  "$WORK/contact-sheets/overview-%03d.jpg"
```

Example: 16 dense frames per sheet, arranged 4 × 4:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf 'fps=4,scale=640:-2,tile=4x4:padding=4:margin=4' \
  "$WORK/contact-sheets/dense-%03d.jpg"
```

Some FFmpeg builds do not include `drawtext`. Check before relying on timestamp labels:

```bash
ffmpeg -hide_banner -filters 2>/dev/null | grep drawtext
```

If `drawtext` is unavailable:

- Generate unlabeled sheets and document the timing formula.
- For 4 fps, frame tiles advance by 0.25 seconds in row-major order.
- Use ImageMagick or a small image script to add labels afterward when exact timestamps must appear on the sheet.

Do not shrink tiles so far that text, faces, or UI become unreadable. Split into more sheets instead.

## Phase 6: detect cuts and transitions

### FFmpeg per-frame scene scores

Write a scene score for each decoded frame:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf "select='gt(scene,0)',metadata=print:file=$WORK/scene-scores.txt" \
  -an -f null -
```

Useful exploratory thresholds:

- `0.10`: sensitive; catches flashes, rapid object motion, and many false positives
- `0.15–0.25`: probable cuts in typical footage
- `0.30+`: strong discontinuity

Thresholds are source-dependent. Animation, particles, strobing, and fast zooms can produce high scores without a cut.

Extract candidate frames with a permissive threshold:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vf "select='gt(scene,0.15)',showinfo,scale=1280:-2" \
  -fps_mode vfr -q:v 2 \
  "$WORK/frames-cuts/candidate-%04d.jpg" \
  2> "$WORK/scene-detect.log"

grep -oE 'pts_time:[0-9.]+' "$WORK/scene-detect.log" \
  | cut -d: -f2 \
  > "$WORK/cut-candidate-times.txt"
```

Collapse clusters of adjacent high-score frames into local maxima. A dissolve, flash, or rapid morph may generate several candidates within 0.2 seconds.

### Secondary detector

PySceneDetect can provide another opinion:

```bash
python3 -m scenedetect \
  -i "$SOURCE" \
  detect-content -t 22 \
  list-scenes \
  -o "$WORK/scenes.csv"
```

Do not make PySceneDetect a hard dependency. It can fail on malformed final timestamps, variable-frame-rate files, or decoder edge cases. If it crashes near the final frame, keep FFmpeg’s complete per-frame results and document the failure.

### Manual boundary verification

For every candidate:

1. Inspect at least the previous two frames, candidate frame, and next two frames.
2. Decide whether it is a hard cut, dissolve, wipe, flash, morph, match cut, or continuous motion.
3. Record the exact boundary timestamp or frame.
4. Note whether audio changes at the same point.
5. Remove false positives caused by subject motion, exposure shifts, or particle effects.

Also inspect low-score visual transitions discovered in contact sheets. A clean graphic match can have a smaller score than a violent movement within one shot.

## Phase 7: analyze motion and camera behavior

Cut detection does not classify motion. For each contiguous shot or meaningful range, inspect:

- Camera position: top-down, frontal, profile, over-the-shoulder, aerial, macro
- Framing: extreme close-up, close-up, medium, wide, insert, product shot
- Camera motion: pan, tilt, dolly, push-in, pull-back, orbit, crane, handheld drift
- Optical motion: zoom, focus pull, depth-of-field change, lens distortion
- Subject motion: direction, speed, acceleration, gesture, deformation
- Motion-design behavior: object replacement, stop motion, parallax, morph, simulated camera move
- Stabilization: locked-off, mechanically smooth, intentionally shaky
- Speed treatment: real time, slow motion, speed ramp, time lapse, freeze frame, reverse

Do not call every scale change a camera zoom. It may be a digital crop, graphic replacement, object movement, or a cut between matched compositions. State the evidence.

Optional quantitative checks:

```bash
# Detect long frozen sections.
ffmpeg -hide_banner -i "$SOURCE" \
  -vf "freezedetect=n=-60dB:d=0.5" \
  -map 0:v:0 -f null - 2> "$WORK/freeze.log"

# Detect black frames and fades through black.
ffmpeg -hide_banner -i "$SOURCE" \
  -vf "blackdetect=d=0.05:pix_th=0.10" \
  -an -f null - 2> "$WORK/blackdetect.log"
```

Use optical flow only when the user needs measured motion paths or stabilization analysis. It is usually unnecessary for a descriptive creative breakdown.

## Phase 8: recover and verify on-screen text

Use full-resolution frames around the moment text is sharpest. Text may blur during motion or appear for only a few frames.

Basic OCR:

```bash
tesseract "$WORK/frames-cuts/text-frame.png" \
  "$WORK/ocr/text-frame" \
  --psm 6
```

Preprocess difficult text:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$WORK/frames-cuts/text-frame.png" \
  -vf 'scale=iw*2:ih*2,format=gray,eq=contrast=1.4:brightness=0.03' \
  "$WORK/ocr/text-frame-prepped.png"
```

OCR is a draft, not the final transcription. Verify spelling, punctuation, and line breaks against adjacent source frames. If text remains unreadable, say so rather than reconstructing it from context.

In the report, distinguish:

- Exact readable text
- Partially readable text with uncertain characters
- Decorative or pseudo-text glyphs
- Inferred meaning

## Phase 9: analyze audio

Extract a lossless analysis track:

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -vn -acodec pcm_s16le \
  "$WORK/audio/track.wav"
```

### Silence

```bash
ffmpeg -hide_banner -i "$SOURCE" \
  -af 'silencedetect=noise=-35dB:d=0.15' \
  -f null - \
  2> "$WORK/audio/silence.log"
```

Tune the threshold to the source. Quiet ambience is not silence.

### Loudness and dynamics

```bash
ffmpeg -hide_banner -i "$SOURCE" \
  -filter_complex 'ebur128=peak=true' \
  -f null - \
  2> "$WORK/audio/loudness.log"
```

Record integrated LUFS, loudness range, and true peak when relevant.

### Spectrogram and waveform

```bash
ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -lavfi 'showspectrumpic=s=1600x900:legend=1:color=viridis' \
  "$WORK/audio/spectrum.png"

ffmpeg -hide_banner -loglevel error \
  -i "$SOURCE" \
  -filter_complex 'showwavespic=s=1600x500:colors=white' \
  -frames:v 1 \
  "$WORK/audio/waveform.png"
```

Inspect whether transients align with cuts, logo hits, gestures, or motion peaks. Describe synchronization only when the waveform/spectrum or direct playback supports it.

### Speech and captions

- Check for subtitle streams with `ffprobe`.
- Run speech recognition when dialogue or voice-over matters.
- Verify names, numbers, and technical terms manually.
- Do not infer “no dialogue” merely because the file has no caption track.

If no transcription tool is available, limit claims to measurable structure: silence, loudness, transients, sustained bands, and timing.

## Phase 10: inspect visual design

Across the complete timeline, record:

### Content

- People, objects, locations, products, interfaces, text, and symbols
- Entry and exit of each important element
- Repeated props, shapes, gestures, or visual callbacks

### Composition

- Subject placement and negative space
- Symmetry, grids, leading lines, depth layers, and visual hierarchy
- How composition changes before and after cuts

### Color and lighting

- Dominant palette and accent colors
- Warm/cool balance, saturation, contrast, black level, and highlight behavior
- Hard versus soft light, practical sources, silhouettes, backlighting, and exposure shifts
- Palette changes tied to narrative beats

### Texture and image character

- Film grain, digital noise, paper, metal, glass, fabric, particles, bloom, haze
- Sharpness, motion blur, shallow focus, chromatic aberration, lens flare
- Photographic, illustrated, rendered, composited, or mixed-media qualities

### Editing language

- Hard cuts, jump cuts, match cuts, dissolves, wipes, flashes, fades, split screens
- Average shot length and deliberate pacing changes
- Repetition, escalation, visual rhymes, setup/payoff, and brand reveal structure

### Tone and narrative function

- What the viewer is meant to feel at each phase
- How image, movement, typography, and sound create that effect
- What each shot contributes: establish, explain, contrast, intensify, reveal, or resolve

## Phase 11: build the chronology

The main timeline should use contiguous ranges with no gaps. Each row should include enough detail to reconstruct the viewing experience.

Recommended schema:

| Time                | Frames | Shot and imagery                                                    | Motion and camera           | Edit/transition               | Text/audio                                  | Function and tone                    |
| ------------------- | -----: | ------------------------------------------------------------------- | --------------------------- | ----------------------------- | ------------------------------------------- | ------------------------------------ |
| 00:00.000–00:01.750 |   0–41 | What is visible, including placement, color, light, and fine detail | Camera and subject behavior | How the range begins and ends | Exact text and supported audio observations | Why the shot exists and how it feels |

For a shorter report, combine the columns into one detailed description. Keep the timestamp and frame range explicit.

### Coverage rule

Every frame must belong to one and only one chronological range:

```text
First range starts at 00:00.000 / frame 0.
Each next range starts where the previous range ends.
Final range ends at the decoded duration / final frame.
No gaps. No unexplained overlap.
```

Use exact boundary semantics consistently. For example, treat rows as half-open ranges `[start, end)` except the final row.

### Literal per-frame output

If the user requires one line per frame, generate a machine-readable companion file:

```csv
frame,timestamp,range_id,description,change_from_previous,confidence
0,0.000,R001,"...","first frame",high
1,0.041667,R001,"...","exposure slightly increases",high
```

Do not write 800 identical prose rows when a range plus a per-frame change column communicates the same information. The Markdown report should remain range-based and readable.

## Phase 12: write the final report

Use this structure unless the user requests another format:

```markdown
# [Video title] — full extraction and frame-range analysis

Source:
Analyzed source:
Method:
Known limitations:

## Executive visual reading

## Technical metadata

## Editing, camera, and pacing

## Full chronological breakdown

## On-screen text transcript

## Motion and transition vocabulary

## Color, lighting, and texture

## Audio structure and synchronization

## Motifs and narrative interpretation

## Uncertainties and confidence notes

## Artifact index
```

Start with what the film is doing, not with a list of tools. The methodology belongs near the metadata or at the end.

Descriptions should be concrete. Name what appears, where it sits, how it moves, how the shot changes, and what happens next. Avoid generic phrases such as “cinematic visuals,” “dynamic transitions,” or “engaging atmosphere” unless the report explains the specific mechanism.

## Quality gates

Do not call the analysis complete until all of these pass.

### Source

- [ ] The highest-quality available source was selected or the limitation was documented.
- [ ] Downloaded duration matches the browser/player duration within expected container tolerance.
- [ ] Resolution, codec, frame rate, frame count, and audio streams were verified.
- [ ] The source file was preserved unchanged.

### Timeline

- [ ] The entire duration was sampled.
- [ ] Per-frame scene scores or equivalent boundary analysis covered the whole decoded stream.
- [ ] All candidate cuts were manually checked with neighboring frames.
- [ ] The chronology begins at frame 0 and ends at the final decoded frame.
- [ ] There are no gaps in described ranges.
- [ ] Pans, zooms, focus pulls, morphs, and speed changes are not mislabeled as cuts.

### Text and audio

- [ ] Important on-screen text was checked at full resolution.
- [ ] OCR wording was visually verified or marked uncertain.
- [ ] Audio claims are supported by playback, transcription, waveform, spectrogram, or measured statistics.
- [ ] Silence thresholds and loudness values include units and settings.

### Interpretation

- [ ] Observed facts are separated from inferred meaning.
- [ ] Motifs cite repeated visual evidence.
- [ ] Tone is explained through color, light, motion, composition, editing, and sound.
- [ ] Uncertainty is stated instead of filled with guesses.

### Deliverables

- [ ] The Markdown report path is clear.
- [ ] The artifact directory contains metadata, cut evidence, contact sheets, and audio diagnostics.
- [ ] Temporary or failed artifacts are either removed or labeled.
- [ ] The user receives a concise completion message with coverage statistics.

## Common failures and recovery

### `yt-dlp` cannot extract the URL

1. Update `yt-dlp`.
2. Confirm the URL works in a browser.
3. Inspect available formats and authentication requirements.
4. Use permitted browser cookies or capture the HLS/DASH manifest through CDP.
5. If the media is DRM-protected, stop and document the limitation.

### The browser exposes only a `blob:` URL

The blob belongs to the page’s MediaSource pipeline and cannot be downloaded as a normal URL. Use the page URL with `yt-dlp` or inspect network requests for manifests and media segments.

### The browser requirement is “use my existing Chrome”

Do not launch Chrome. Connect through the installed extension relay or an already-running remote-debugging endpoint. Before closing anything, identify processes by command-line profile path and PID; never terminate the user’s normal Chrome process to clean up an accidental automation instance.

### FFmpeg lacks `drawtext`

Generate unlabeled contact sheets and calculate timestamps from sample order, or label them with ImageMagick. Do not block the analysis on timestamps burned into images.

### PySceneDetect crashes or disagrees with FFmpeg

Use FFmpeg’s full per-frame scene-score output as the primary record. Treat both detectors as candidate generators and settle boundaries through source-frame inspection.

### Scene detection produces too many candidates

Raise the threshold, collapse candidates within 0.2–0.5 seconds into local maxima, and inspect clusters as possible flashes, particle motion, or morphs.

### Scene detection misses transitions

Review contact sheets for dissolves, matched compositions, slow exposure fades, wipes, and graphic morphs. Lower the threshold only around suspicious sections.

### The last frame fails to decode

Compare container duration, decoded frame count, and the final valid presentation timestamp. Document the missing or malformed tail. Do not silently invent the end state.

### HDR frames look washed out

Inspect color-transfer metadata and create a tone-mapped analysis proxy while preserving the HDR original. State that visual color judgments use the proxy.

### The video is too long

Analyze hierarchically: chapters, scenes, shots, then dense samples only around transitions and requested sections. Keep a coverage map so skipped dense regions are explicit rather than accidental.

## Minimal reproducible command set

For a short public video, this is the smallest useful pipeline:

```bash
URL='https://example.com/video-page'
WORK='tmp/video-analysis'
mkdir -p "$WORK"/{frames-dense,frames-cuts,contact-sheets,audio}

python3 -m yt_dlp \
  -f 'bestvideo+bestaudio/best' \
  --merge-output-format mp4 \
  -o "$WORK/source.%(ext)s" \
  "$URL"

SOURCE="$WORK/source.mp4"

ffprobe -v error -show_format -show_streams -count_frames \
  -of json "$SOURCE" > "$WORK/ffprobe.json"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -vf 'fps=4,scale=1280:-2' -q:v 2 \
  "$WORK/frames-dense/frame-%05d.jpg"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -vf 'fps=4,scale=640:-2,tile=4x4:padding=4:margin=4' \
  "$WORK/contact-sheets/sheet-%03d.jpg"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -vf "select='gt(scene,0)',metadata=print:file=$WORK/scene-scores.txt" \
  -an -f null -

ffmpeg -hide_banner -i "$SOURCE" \
  -af 'silencedetect=noise=-35dB:d=0.15' \
  -f null - 2> "$WORK/audio/silence.log"

ffmpeg -hide_banner -i "$SOURCE" \
  -filter_complex 'ebur128=peak=true' \
  -f null - 2> "$WORK/audio/loudness.log"

ffmpeg -hide_banner -loglevel error -i "$SOURCE" \
  -lavfi 'showspectrumpic=s=1600x900:legend=1:color=viridis' \
  "$WORK/audio/spectrum.png"
```

This command set creates evidence. The analysis is complete only after the full timeline, cut boundaries, text, motion, sound, and interpretation have been inspected and written into contiguous ranges.
