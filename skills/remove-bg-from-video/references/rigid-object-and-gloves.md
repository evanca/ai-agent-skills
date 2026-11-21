# Rigid object with colored gloves

Use this approach when a phone or similar rigid object is held in colored gloves against a light background. It is a case-specific alternative to full video segmentation, not a universal phone detector.

## Why a single color key fails

A white key can remove the game screen, pale phone rim, and bright glove highlights. Dark-bezel contours can become open when fingers overlap the frame, causing large holes in the screen. Dilation recovers missing rim pixels but also retains a strip of background. GrabCut can classify a pale case as background and remove discontinuous segments of its frame.

A convex hull can close an object interior, but it can also bridge real gaps and retain unwanted background. Use geometry only for the rigid object; preserve glove gaps independently.

## Separate stable geometry from changing content

1. Choose a sharp reference frame with a readable outer silhouette. Mark the actual case outline, including protrusions and asymmetric corners. A rounded rectangle is sufficient only when the footage supports it; do not cut off buttons to simplify the model.
2. Build tracking patches around several visible corners or bezel sections. Mask out the changing screen content and as much background as possible. Four distributed patches can constrain a nearly front-facing phone well.
3. Match patches within a plausible motion neighborhood. Fit a robust transform from the correspondences. Similarity/partial affine is appropriate for limited in-plane motion; perspective changes require a homography or richer tracking. Reject degenerate geometry and inspect occluded or weak patches. Template scores can be high at the wrong location.
4. Warp the protected phone silhouette to each frame. Combine it with a glove mask calibrated from the inspected footage. Color thresholds, search regions, radii, and coordinates must be derived per source and scale with working resolution.
5. Remove isolated specks without deleting thin fingers or filling real gaps. Preserve low-saturation highlights inside confidently tracked glove regions. Apply a narrow antialiased boundary; wide blur hides errors and introduces halos.

For major motion, update search windows from accepted positions or introduce additional reference poses. Temporal smoothing should suppress detection noise without lagging behind genuine motion. Inspect fingers crossing the object and fast movements specifically.

Keep the foreground color and alpha separate. A clear case may contain transmitted background color in the original pixels: a geometrically correct mask alone cannot remove that contamination. Refine edge color or use a more capable matting method when the required result exposes this issue.

## HLG proof conversion example

For one HLG phone clip, this FFmpeg filter preserved blue highlights that a brighter conversion clipped. It is an inspected starting point, not a universal HDR preset:

```text
zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709,tonemap=tonemap=hable:desat=0:peak=10,zscale=t=bt709:m=bt709:r=full,format=rgb24
```

Confirm the source color metadata and compare the proof visually. Select peak/exposure and output color tags for the actual source; do not reuse these parameters for SDR, LOG, or unknown transfers.

## Reusable implementation boundaries

Keep tracking/calibration, subject-mask combination, rendering, and proof-sheet generation modular. Store source-specific geometry and thresholds as explicit configuration, not defaults for unrelated clips. Probe counts or decode to EOF; preserve timestamps for variable-rate footage instead of assuming a fixed 30 fps. Verify subprocess return codes and finish encoders before declaring an export complete.

Useful evidence includes reference calibration, representative cutout PNGs, a dark-background motion proof, per-frame tracking/area diagnostics, and exported-alpha checks. Retain only artifacts that help reproduce or assess the final treatment.
