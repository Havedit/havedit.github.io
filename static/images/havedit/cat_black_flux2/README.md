# cat_black_flux2 Web Assets

This directory contains exported visualization assets for the HAVEdit FLUX.2 example:

```text
Sample: 000000000005
Task: Change the color of the cat from orange to black
Model: /home/lili/code/models/FLUX.2-klein-base-9B
Image: /data_ljy/ll/dataset/PIE_Bench/annotation_images/0_random_140/000000000005.jpg
```

The assets are intended for building an offline paper/project webpage. The project code is not needed at webpage runtime; the webpage can load the PNG files and `manifest.json` directly.

## Generation Config

The main generation settings are recorded in `manifest.json`:

```text
num_inference_steps: 28
seed: 42
guidance_scale: 2.5
warmup_steps: 6
threshold: 0.63
enable_bhc: true
enable_trajectory_trust: false
bhc_tau_low: 0.33
bhc_tau_high: 0.63
bhc_lambda_max: 0.15
```

This is the tuned cat-color configuration, not the `threshold=0.91` black_subjectrelease configuration.

## Top-Level Files

```text
original.png
  The input image.

result.png
  The final edited result after 28 denoising steps.

target_mask.png
  PIE-Bench target mask for this sample, if available.

manifest.json
  Machine-readable index for all assets. Use this as the main source for webpage loading.

metrics.json
metrics.csv
  Per-step scalar diagnostics.
```

## Frame Directories

Each frame filename follows:

```text
step_0001.png
step_0002.png
...
step_0028.png
```

The important directories are:

```text
frames/x0/
  Decoded x0 prediction at each denoising step.
  This is the best main slider content.
  It shows what clean image the model currently predicts from the noisy latent.

frames/semantic_prior/
  Per-step semantic prior current.
  More accurate name: semantic_prior_current.
  This is dynamic and changes across steps.

frames/attention/
  Per-step attention map diagnostic.
  This is useful as supplementary visualization.

frames/edit_mask/
  Edit permission map.
  High value means this area is allowed to change.

frames/preserve_weight/
  Preserve/protection weight map.
  High value means this area should stay close to the original.

frames/overlay_edit_mask/
frames/overlay_preserve_weight/
frames/overlay_semantic_prior/
  Heatmap overlays blended onto the corresponding x0 frame.
```

## Recommended Webpage Interaction

A simple webpage can use the following layout:

```text
Original | x0^t slider | Result
```

For the slider panel, use `manifest.json` and load:

```text
frames[i].x0
```

Then provide an overlay toggle:

```text
None
Edit Mask
Preserve Weight
Semantic Prior Current
Attention
```

Suggested default:

```text
Main image: frames/x0/
Default overlay: None
Most useful overlay: Semantic Prior Current
Secondary overlays: Edit Mask, Preserve Weight, Attention
```

## Important Interpretation Notes

### x0^t

`x0^t` is the decoded clean-image prediction from the current denoising step:

```text
x0_pred = x_t - t_norm * v_theta
```

It is useful because it shows how the final edit gradually forms during sampling.

### Semantic Prior Current

`frames/semantic_prior/` stores the current-step semantic prior, also called `latest_semantic_prior_current` in the code.

This map changes over time. It shows where the model's current attention/semantic evidence is concentrated at each step.

### Frozen Semantic Prior

The algorithm also has a frozen semantic prior. With `warmup_steps=6`, the first 6 steps accumulate attention, and the prior is finalized at the end of warmup.

From step 7 onward, this frozen prior is used by the edit/protect routing logic.

Note: this export currently does not save a separate `semantic_prior_frozen/` folder. The existing `frames/semantic_prior/` folder should be interpreted as `semantic_prior_current`, not frozen prior.

### Preserve Weight and Edit Mask

The preserve/edit maps are computed from:

```text
1. frozen semantic prior
2. current v_pred versus v_ref difference
```

The simplified formula is:

```text
deviation = ||v_pred - v_ref||
z_dev = local_zscore(deviation)

preserve_score = sigmoid(
    - alpha * z_dev
    + beta * (1 - frozen_semantic_prior)
)

preserve_weight = clamp((preserve_score - threshold) / soft_band, 0, 1)
edit_mask = 1 - preserve_weight
```

Interpretation:

```text
preserve_weight high:
  protect this region and keep it close to the original image.

edit_mask high:
  allow this region to be edited.
```

In this exported sample, `edit_mask` and `preserve_weight` are stable from step 7 to step 28. This is expected for this configuration: the frozen semantic prior is fixed after warmup, and the thresholded routing decision remains unchanged across later denoising steps.

Therefore:

```text
frames/x0/ and frames/semantic_prior/ are dynamic.
frames/edit_mask/ and frames/preserve_weight/ behave like fixed decision maps after step 7.
```

## Metrics Summary

From `metrics.csv`:

```text
step 1-6:
  edit_ratio and preserve_weight_mean are empty because warmup is still collecting the prior.

step 7-28:
  edit_ratio = 0.7318382263
  preserve_weight_mean = 0.2681617737
  high_preserve_area_ratio = 0.0
```

This means roughly 73.18% of the spatial map is treated as editable under the exported soft mask, while roughly 26.82% is protected.

## Suggested Web Captions

Possible short captions for the webpage:

```text
x0^t Prediction:
  Clean-image prediction reconstructed from the intermediate latent at denoising step t.

Semantic Prior Current:
  Dynamic semantic evidence from the current step's attention.

Edit Mask:
  Region allowed to change by HAVEdit after warmup.

Preserve Weight:
  Region protected from editing after warmup.
```

## Loading Tip

Use `manifest.json` rather than hard-coding filenames. Each frame entry contains paths such as:

```text
frames[i].x0
frames[i].edit_mask
frames[i].preserve_weight
frames[i].semantic_prior
frames[i].attention
frames[i].overlay_edit_mask
frames[i].overlay_preserve_weight
frames[i].overlay_semantic_prior
```

