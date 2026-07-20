# Illumination-Selective Contrast Enhancement for Monocular Depth Estimation

**Kanishk Arora — NIT Hamirpur**

## What this is

A training-free preprocessing study testing whether decomposing an image into
illumination and reflectance layers (bilateral filtering), then applying CLAHE
selectively to the illumination layer only, improves frozen monocular depth
estimation (DPT_Large) — and, critically, *when* it helps versus when it doesn't.

Full results: [`master_results.csv`](./master_results.csv)

## Setup

- **Backbone:** DPT_Large, frozen, no training/fine-tuning
- **Dataset:** TUM RGB-D freiburg3 structure/texture sequences (4 sequences,
  varying geometric structure and visual texture density)
- **Methods compared (7):** raw baseline, SSR Retinex, LIME, standard CLAHE,
  bilateral decomposition alone (no CLAHE), CLAHE-on-reflectance (ablation),
  and the proposed hybrid (CLAHE on illumination only)
- **Metrics:** MAE, AbsRel, RMSE, δ<1.25, least-squares scale alignment

## Key finding

The hybrid method does **not** universally improve depth accuracy. It shows a
clear, mechanistically-explained bounded operating regime:

| Condition | MAE result | δ<1.25 result |
|---|---|---|
| Structure present, texture absent | Hybrid wins (0.151 vs 0.205, +26%) | 78.2% vs 66.4% |
| Texture present, structure absent | Hybrid wins (0.281 vs 0.393, +28%) | 46.8% vs 25.0% |
| Neither present | Roughly flat (0.352 vs 0.346) | 60.6% vs 52.0% |
| Both present (already easy scene) | Hybrid loses (0.276 vs 0.204, -36%) | 58.5% vs 71.5% |

Visual and ablation analysis identified the failure mechanism precisely: in
already well-exposed, feature-rich scenes, the fixed gain multiplier saturates
highlights, and CLAHE compounds this by introducing local-contrast artifacts
on surfaces that had no meaningful illumination signal left to enhance.

## What's been ruled out / attempted honestly

- **Adaptive gain (illumination-std based):** attempted as a fix for the
  failure mode above; did not work, because illumination standard deviation
  did not sufficiently discriminate between failing and succeeding
  conditions (33.6 vs 38.6 average — too similar). Reported as a negative
  result with a specific, stated reason, not hidden.
- **Dataset scope, acknowledged directly:** TUM's structure/texture sequences
  vary geometric structure and texture content, not illumination — they test
  a related but distinct axis from the paper's core illumination hypothesis.

## In progress

- **Multi-Illumination dataset (Murmann et al., 2019) consistency experiment:**
  a direct, ground-truth-free test of the actual illumination-robustness
  hypothesis. Camera and geometry are fixed per scene across 25 controlled
  lighting variants; depth-prediction variance across those 25 variants is
  compared with vs. without hybrid preprocessing. Lower variance under the
  hybrid method would directly support the illumination-stability claim,
  independent of the TUM structure/texture confound.
- Hyperparameter sensitivity check (gain/clipLimit) across both a winning and
  a losing TUM sequence, to test robustness of both results.

## Practical scope of this method

The four TUM sequences represent global extremes — a whole scene that is
uniformly texture/structure-rich or uniformly starved. Most real-world scenes
are not globally uniform; they mix texture-rich and texture-starved regions
within a single frame (e.g. a patterned floor under a blank ceiling, or a
snowy trail with scattered rocks and footprints against open snow). Applying
this method as a blanket, whole-image preprocessing step is therefore not
recommended for general scenes — the results above show it can actively hurt
accuracy on already information-rich content. The method's demonstrated use
case is texture/structure-starved scenes or regions specifically (blank
walls, fog, snow, low-detail industrial surfaces), not general-purpose
preprocessing. A natural extension, motivated directly by this limitation and
by the failed global adaptive-gain attempt above, is per-region rather than
per-image adaptive gain — applying enhancement only to locally texture-starved
patches within a mixed scene, left unexplored here.

## Positioning relative to prior work

Related, but methodologically distinct: RADepthNet and DeLightMono use
*learned*, jointly-optimized illumination/reflectance decomposition in
domain-specific settings (endoscopy). This work uses a *classical,
training-free* decomposition as a pure preprocessing step ahead of a frozen,
general-purpose depth network, evaluated specifically for *when* it helps
versus hurts rather than claiming universal improvement.
