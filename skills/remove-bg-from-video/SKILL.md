---
name: remove-bg-from-video
description: Use when removing or replacing a video background, exporting footage with transparency or a chroma key background, or fixing cutout halos and missing subject edges.
---

# Remove Background from Video

Produce a temporally stable cutout that retains the subject's silhouette and interior detail. Keep the source unchanged. Prefer deterministic segmentation, matting, tracking, and compositing over generative reconstruction when the original subject must remain exact.

## Establish the actual constraints

Probe the source with `ffprobe`: dimensions, display rotation, frame rate/time base, frame count, duration, audio, pixel format, color primaries, transfer function, and HDR metadata. Work in displayed orientation; avoid applying rotation twice.

Inspect early, middle, and late frames plus difficult moments: motion blur, fingers crossing an object, pale edges, gaps, and changing screens. Determine what belongs to the subject before choosing a key. White phone screens, reflections, clear cases, and pale glove highlights can overlap the background's color.

Retain source resolution, cadence, duration, and audio unless the requested or approved deliverable specifies otherwise. A smaller proof is useful, but disclose its resolution and color conversion; do not silently treat it as a native-resolution master. Decode to EOF for a full render rather than copying a previous clip's frame count.

## Choose the mask from visible evidence

- Uniform background with distinct subject colors: use a color key with edge cleanup and spill suppression.
- Rigid objects with similar-colored interiors or rims: track a protected object silhouette and combine it with separate masks for hands or other moving parts. Read [rigid-object-and-gloves.md](references/rigid-object-and-gloves.md) for the phone/glove approach and its limitations.
- Complex backgrounds, deformable subjects, hair, or substantial motion blur: use suitable video segmentation/matting or tracked rotoscoping. Independent still-image masks often flicker; add temporal consistency and inspect motion.

Do not erode the whole subject just to eliminate halos. Separate geometry errors from color contamination: recover the silhouette first, then refine alpha and foreground edge colors. Keep genuine internal holes, such as the gap between a finger and an object. Protect opaque subject interiors from accidental keying.

HDR/HLG/PQ footage needs an explicit color path. Inspect tone-mapped SDR proofs before deriving a color mask: clipped highlights can make colored gloves appear white. Preserve HDR when required; otherwise document the SDR conversion and tag the export consistently.

## Prove the edges before scaling up

Start with a small set of representative frames. Compare source and cutout at matching scale, including enlarged edge crops on dark and contrasting backgrounds. Check:

- Entire object frame and protrusions retained; screen/interior fully opaque where appropriate.
- No residual background strip or white fringe around the object or hands.
- Real gaps preserved, no holes in highlights, and no detached mask specks.

Then render a short motion sample—roughly ten seconds when appropriate—including difficult movement. Look for jitter, tracking drift, popping alpha, and changing edge thickness. If the user requested a sample approval checkpoint, deliver the concrete sample and pause there. Otherwise, continue within the authorized scope once verification is satisfactory. Existing approval carries forward; do not add another approval gate.

Once approved, preserve the accepted treatment, render the remaining/full authorized footage, and inspect later poses and scene changes. If the later footage defeats the method, adjust and validate the affected region rather than assuming sample success covers it.

## Export and verify

Prefer an alpha-capable editing master such as ProRes 4444 MOV when transparency is useful. Ordinary H.264 MP4 does not preserve alpha. Supply a lightweight MP4 composited on a dark background for review when helpful. If chroma key delivery is preferred, choose a saturated color absent from the subject and avoid compression that creates poor key edges.

Match straight/premultiplied alpha conventions to the compositor and identify them when relevant. Retain original RGB inside opaque areas; treat edge decontamination separately from global grading.

Verify the encoded file, not only intermediate arrays:

- Probe codec, alpha-capable pixel format, dimensions, duration, and frame count/cadence against the intended source range.
- Decode exported alpha; sample known background, opaque interior, and boundary regions. An alpha-capable codec alone does not prove useful transparency.
- Inspect exported frames across the whole interval and the final frame; play the proof when judging temporal smoothness. Tracking scores and area changes help locate failures but do not prove visual quality.

Keep final outputs clearly named and separate from failed experiments. Deliver absolute file links, duration/resolution, and any material limits. Distinguish sampled visual checks from frame-by-frame inspection; do not promise perfect edges from command success alone.
