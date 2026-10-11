---
name: pixinsight-processing
description: This skill should be used when the user asks to "process this master light in PixInsight", "stretch / denoise / color calibrate my astro image", "process my integrated image", "make a starless version", or to export a processed astrophotography image (XISF, FITS, TIFF, JPG) using PixInsight on their own computer. It drives the PixInsight desktop app with computer use and short PJSR scripts; no MCP server is needed.
---

# Processing astro images in PixInsight

Take an integrated master (usually a WBPP `masterLight_...xisf`) from linear data to finished files by driving PixInsight on the user's computer with computer use. Do the image work with short PJSR (PixInsight JavaScript) scripts, not by clicking through process dialogs: scripts are repeatable, their console output says what happened, and process dialogs are slow and fragile to operate.

Two reference files hold the detail. Read both before the first action:

- `references/driving-pixinsight.md`: how to write, save and run scripts reliably, and what does not work.
- `references/pjsr-snippets.md`: the script for each processing stage.

## Ground rules

- Never overwrite the source master or an earlier result. Save new files next to the source. For a redo, add a suffix (`_v2`) and keep the previous version unless told otherwise.
- Run one stage per script. After each, read the console and look at the image, then decide the next stage. Saving is always its own final script, run only after looking at the result.
- Report what the console and the image actually show. State any failure or fallback plainly.
- Stay inside PixInsight. Do not use the console's `!` shell escape.
- Request computer-use access to PixInsight only, and prefer background control so the user can keep working.
- PixInsight reads and writes the image files itself, so no separate folder access is needed.

## Before starting

1. Confirm PixInsight is running and request access to it.
2. Check which tools are installed by listing the `Process` menu categories. Note whether StarNet2 is present, and whether any AI tools (BlurXTerminator, NoiseXTerminator, StarXTerminator, MLDenoise) are present and configured. Use only what is installed; MLDenoise needs a model file set before it can run.
3. List the folder (`ls "<path>"` in the Process Console) and choose the input. Prefer an `_autocrop` master if present, otherwise the plain master. Ignore files that are already stretched or processed. Tell the user which file was chosen.

Ask the user only when it changes the result and cannot be inferred: for example which of several masters to use. Otherwise proceed and state the choices made.

## Workflow

Each stage's script is in `references/pjsr-snippets.md`.

### 1. Open and measure

Open the file, give the view a short id, and print size, whether it has an astrometric solution, and per-channel median and MAD. A background median well above about 0.05 means a very bright sky: it changes the background-reference limits in stage 3 and limits how much faint signal can be recovered. Tell the user.

### 2. Background removal

Choose by how much of the frame the object fills.

- Object with real sky around it (most nebula fields): GradientCorrection with defaults.
- Object filling the frame (M31, large galaxies, large nebula complexes): GradientCorrection treats the faint outer disk or halo as a gradient and removes it. Use a plane fit instead (AutomaticBackgroundExtractor, degree 1, subtraction).

### 3. Color calibration

Try SpectrophotometricColorCalibration first. It needs the astrometric solution. Its defaults assume a Sony color sensor and an average spiral galaxy white reference; say so, since the user's camera may differ.

If it fails, fall back to BackgroundNeutralization followed by ColorCalibration, using a small sky patch in the corner farthest from the object as the background reference. Check the console: a background reference printed as zero means no pixels fell inside the low/high limits, so the step did nothing. Raise the upper limit and rerun.

Apply an auto screen stretch and look: neutral background, plausible star colors, no strong cast.

### 4. Stretch

Apply a permanent HistogramTransformation derived from the auto-stretch values, to a background of about 0.18 to 0.20, then reset the screen stretch.

### 5. Star separation (if StarNet2 is installed)

Keep a copy, run StarNet2 on the working image, and derive a stars-only layer from the copy. If StarNet2 is missing or fails, skip stages 5 and 7 and process with stars in place, using gentler curves.

### 6. Starless work

- Noise reduction with MultiscaleLinearTransform, strongest on the finest scales.
- Contrast and saturation with CurvesTransformation. Use the nebula or galaxy starting points in the snippets and adjust after looking.
- Galaxies: keep the faint outer disk and a natural falloff from centre to edge. Do not apply HDRMultiscaleTransform by default; it darkens the core and flattens the inner galaxy. If the core is clipped in the data, leave it white rather than darkening it. Offer core compression as an option instead of applying it.
- Emission nebulae: be careful with green removal (SCNR), which can damage teal OIII.

Save a starless copy before putting stars back.

### 7. Stars back

Screen the star layer back, slightly reduced: about 0.85 for dense star fields, 0.9 otherwise.

### 8. Look, then save

View the whole frame and some detail. If something is clearly wrong, fix it before saving.

Default outputs, next to the source: 32-bit XISF, 16-bit TIFF, JPG, and starless TIFF and JPG when a starless copy exists. Convert a copy, never the working image, to 16 or 8 bit. Confirm each save line in the console.

## Final report

Lead with what was saved and where, and what the result looks like. Then, briefly:

- which input file was used;
- the stages in order, with any failure or fallback;
- visible defects left in the image (star halos, corner color patches, residual gradient);
- what was left behind: iconized helper images, the detached Script Editor and Process Console windows (double-click a title bar to dock it again), and the script file in the home folder;
- if any typed text may have gone to the image workspace instead of a text field, say so.
