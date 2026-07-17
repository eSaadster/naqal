# Image-to-video process

1. **Generate a still image first**
   - Start with an image model.
   - Make one strong cover/frame image in your target aspect ratio.
   - Leave safe margins so the subject doesn’t touch edges.
   - If composition is bad, regenerate until it works.

2. **Use that still as the start frame for clip 1**
   - Switch to video mode.
   - Use a frames/start-frame workflow if the tool supports it.
   - Set the still image as the **Start frame**.
   - Prompt only for motion and continuity, not a new scene.

3. **Render clip 1**
   - Keep it short.
   - Ask for smooth continuous motion with no cuts.
   - Preserve any areas you want stable across clips.

4. **Extract the exact last frame of clip 1**
   - Save the true final frame as a still image.
   - This matters: not “near the end,” but the exact last frame.

5. **Use that extracted frame as the start frame for clip 2**
   - Create the next video clip.
   - Set the extracted last frame from clip 1 as the **Start frame** for clip 2.
   - Prompt the next motion/change from that exact moment onward.

6. **Repeat for every new clip**
   - For each clip:
     - render clip
     - extract exact last frame
     - feed it in as next clip’s start frame
   - This creates seamless visual continuity.

7. **Verify the seam**
   - Extract:
     - the last frame of clip 1
     - the first frame of clip 2
   - Compare them side by side.
   - If the workflow worked, they should be identical or effectively seamless.

8. **Join clips with no transitions**
   - Concatenate clips in order.
   - Do **not** add crossfades or dissolves if the boundary frames already match.

9. **Upscale if needed**
   - If the generated clips are low-res, upscale the final combined master after stitching.

10. **Optionally export frames**
   - If needed for editing, analysis, or further generation, extract the final video into a numbered image sequence.

## Core principle

**End frame of clip N = start frame of clip N+1**

That’s what lets you turn separate generated clips into one continuous-looking video.
