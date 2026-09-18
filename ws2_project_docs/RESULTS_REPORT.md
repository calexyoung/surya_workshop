# Active-Region Segmentation: Results Report

**Status as of 2026-09-18.** The `real`-budget run is no longer running. Seed 0 completed and
early-stopped; seed 1 ended at epoch 6 without early-stopping and without exporting a
checkpoint; seeds 2-4 were never started. §7 has been rewritten from the per-epoch
`metrics.csv` files rather than from the mid-run snapshot it previously carried.

Step 6 of `3_errors_ar.ipynb` **has now been run on the `real` validation set**
(2026-09-18, `runs/nbconvert/3_errors_real_analysis.ipynb`, 28/28 cells, no errors). It
reused the two existing checkpoints rather than retraining, so §7 now carries this run's own
label floor, calibration, threshold sweep and bootstrap intervals rather than the medium
run's.

This report gathers what was measured across the four notebooks in
`downstream_apps/ar_segmentation/` — the baseline ladder, the Surya fine-tune, and the
error analysis — for readers who want the numbers and what they mean without re-running
anything. The mechanics are documented in `NOTEBOOKS_EXPLAINED.md`,
`ERRORS_NOTEBOOK_EXPLAINED.md` and `GROUND_TRUTH_EXPLAINED.md`.

---

## 1. Executive summary

| finding | value |
|---|---|
| Surya + adapters, 400 training images, seed 0 **kept checkpoint** (epoch 9) | **IoU 0.627**, precision 0.736, recall 0.811 |
| The same curve's best epoch (epoch 10) | IoU 0.637 — *not* what the run keeps: `ModelCheckpoint` monitors `val_loss`, and epoch 10's was worse |
| 0.627 against the **0.6275** label floor | level: the gap is far below the ±0.008 bootstrap width on the model's own score |
| Seeds completed at the real budget | **1 of 5**; seed 1 stopped at epoch 6 with no exported checkpoint |
| Surya + adapters, 60 training images (final medium run) | IoU 0.437, 95 % CI [0.424, 0.449]; 0.465 at the best calibrated threshold |
| Seed-to-seed spread vs sampling uncertainty (medium) | 0.025 vs 0.021 — comparable; a single run is not quotable to three decimals |
| Label floor: agreement between ±40 G and ±60 G versions of the answer key | **0.6275** (real validation set); 0.632 on the medium set |
| Run-to-run spread at the real budget | **0.0039**, against a bootstrap width of 0.0167 — a flip from the medium budget, where the two matched |
| Calibration | raw scores over-confident (scale 0.46, shift −1.98); stated-vs-observed gap 0.0098 → 0.0015 after correction |
| Strongest predictor of per-image score | distance of the labelled regions from disk centre, r = −0.94 (solar rotation + line-of-sight foreshortening) |
| First run of the error notebook | collapsed to predicting nothing; three causes found and fixed |

The short version: with enough data the foundation model reaches the point where its
disagreement with the answer key is about the same size as the answer key's disagreement
with itself. Below that point model error dominated; at it, further gains on pixel overlap
cannot be distinguished from a different choice of threshold in the labelling rule.

Two caveats keep that from being a finished result. The two numbers being compared come
from **different validation sets**, and the ensemble that would say how much of the 0.627 is
seed luck **was never completed**. Both are §9's first items.

---

## 2. What was measured, and how

Every model is scored with the same code (`ARMetrics`) on the same pairs of SDO image and
hourly active-region outline. Scores are **overlap** (IoU, Dice), never accuracy — active
regions cover ~1 % of the frame, so "background everywhere" is 99 % accurate and useless.
The error notebook adds, for each score:

- a **bootstrap interval** over validation images (sampling uncertainty);
- a **calibration** of the model's probabilities, fitted on a random half of the validation
  set and reported on the other half;
- **confidence contours** at 0.25 / 0.5 / 0.75 on the calibrated map;
- an **ensemble** of independently seeded adapter sets (training uncertainty);
- a **label floor**: the answer key re-derived at ±40 G and ±60 G, and the agreement between
  those two equally defensible keys.

---

## 3. The collapse, and its causes

The first run of `3_errors_ar.ipynb` (8 images, 2 epochs) produced a textbook failure:

| epoch | loss | IoU | precision | recall |
|---|---|---|---|---|
| 0 | 0.021 | 0.0054 | 0.71 | 0.0054 |
| 1 | 0.013 ↓ | **0.000035** | 1.00 | 0.000035 |

The loss improved while the model learned to predict nothing. Three causes, all now handled
in the notebook:

| cause | evidence | fix |
|---|---|---|
| The images were the quietest in the archive | a naive date cap selected mid-January 2011: 0.101 % AR coverage, 987 background pixels per AR pixel | date-window selection (solar maximum, 2013–2015): 0.9–1.6 % coverage, up to 16× the signal per image |
| The rare pixels were not up-weighted | `ar_pos_weight: null`; the penalty for "predict nothing" was 0.24 | weight measured from the selected outlines (√ of the imbalance); with a Dice term the same prediction costs 8.4 |
| Learning rate 10× too high for Surya | the config's own comment said 1e-4; the notebook passed 1e-3 | rate chosen per model |

The diagnostic that exposed it — and the one to watch on any run — is **precision and
recall together**, not the loss.

---

## 4. Medium budget: 60 training images, 60 validation, 2 seeds

Two runs. The first used the last epoch's weights, a disc-shaped dilation in the label
floor, and a chronological calibration split; the second corrected all three (§6, §7).

| quantity | provisional | **final** |
|---|---|---|
| seed 0, IoU at 0.5, pooled over 60 images | 0.427 [0.411, 0.442] | **0.437 [0.424, 0.449]** |
| calibration scale *a*, shift *b* | 0.443, −0.915 | 0.459, −1.975 |
| stated-vs-observed probability gap, before → after | 0.0077 → 0.0031 | 0.0098 → **0.0015** |
| evaluation half, uncalibrated IoU | 0.369 (confounded) | **0.444** (matches the full set) |
| best calibrated threshold → IoU | 0.12 → 0.381 | 0.28 → **0.465** |
| seeds 0 / 1 at that threshold | 0.402 / 0.383 | **0.459 / 0.417** |
| seed spread vs one seed's bootstrap width | 0.010 vs 0.032 | **0.025 vs 0.021** |
| answer-key reconstruction agreement (60 images) | 60/60 ≥ 0.6, median 0.826 | 60/60, median 0.813 |
| **label floor** | 0.639 | **0.632** |
| verdict | headroom | best 0.459 < 0.632: model error dominates |

### Training curves at 60 images

Both seeds reached their lowest validation loss at **epoch 1 of 10** and then overfit: seed 0
val loss 0.533 → 0.652 by epoch 9 while IoU stayed flat at ~0.43. Recall drifted 0.82 → 0.60
and precision 0.44 → 0.58 — toward under-prediction, never collapse. The lesson taken into
the `real` run: more *images*, not more epochs, plus early stopping and keeping the best
epoch's weights.

### What the ensemble showed

With the best-epoch weights the two seeds differ by 0.025 in IoU, about the width of one
seed's bootstrap interval. Training randomness and sampling uncertainty are the same size at
this budget; the honest report is the pair (0.459, 0.417) or a mean with a spread.

---

## 5. The limb effect: what actually drives per-image score

Per-image IoU on the medium validation set fell from 0.53 on the first images to 0.36 on the
last. The validation images are chronological (three days, 2013-01-16 → 01-18), and:

| per-image IoU correlates with | r |
|---|---|
| mean distance of the labelled regions from disk centre | **−0.94** |
| labelled area (coverage) | +0.90 |
| number of regions | +0.19 |
| agreement between the labelling rule and the released key | −0.19 |

| day | IoU | coverage | radial position (0 = centre, 1 = limb) |
|---|---|---|---|
| 16 Jan | 0.491 | 1.5 % | 0.45 |
| 17 Jan | 0.411 | 1.2 % | 0.53 |
| 18 Jan | 0.360 | 0.9 % | 0.60 |

The same regions rotated toward the limb; their line-of-sight-thresholded labelled area
shrank by a third; the model's overlap fell with it. Agreement between rule and key stayed
flat, so this is solar rotation and foreshortening, not the model or the labels misbehaving.
It is the limitation `GROUND_TRUTH_EXPLAINED.md` predicts, now measured.

**Consequence for method.** The notebook had split the validation set for calibration by
index — day 1 to fit, days 2–3 to evaluate — which put the easy images in the fit (0.470) and
the hard ones in the evaluation (0.368). A seeded random split gives 0.435 vs 0.403, i.e.
noise. The split is now random.

---

## 6. The answer key

The outlines are the ARPIL task of SuryaBench (Roy *et al.*, *Scientific Data* 13, 712,
2026): from the HMI line-of-sight magnetogram, threshold at ±50 G, drop regions under 100 px,
dilate with "a rectangular filter of size 10 pixels", take the intersection of the dilated
polarity maps as the polarity-inversion-line map, and keep active-region blobs that contain
one. Findings from reproducing it:

| finding | evidence |
|---|---|
| Size filter must precede dilation | reversed, agreement with the released key collapses to 0.01–0.09 |
| "Rectangular, size 10" = square brush grown 10 px, not a disc, not a 10 × 10 box | 12 timestamps: PIL agreement better on 12/12 (median 0.74 → 0.80); footprint 9/12 (≈ 0.85 → 0.86); a 10 × 10 box scores 0.70 |
| Good days reproduce closely; May 2010 does not | 2011-01-16: 0.66–0.92; 2010-05-13: ≈ 0.0–0.18. The 5,291 masks from 2010 were absent from the preprint (119,454 masks) and present in the published version (121,963 — matches the release exactly) |
| The two stored maps use different encodings | `union_with_intersect` 0/255, `intersection` 0/1 |
| AR coverage varies 78× over the solar cycle | 0.023 % (2019) to 1.80 % (2014); "≈ 1 %" is a solar-maximum figure |

The residual gap on good days (footprints ≈ 0.85 rather than 1.0) most likely comes from how
an "AR containing a PIL" is delimited, which the paper does not specify.

---

## 7. Real budget: 400 training / 400 validation images — what the run produced

**Setup.** Images spread evenly across 2013–2015 (`window-stratified`, ≈ 131–137 per year in
each split; the first-400 alternative would have been 17 consecutive days of March 2013, the
same regions rotating). Up to 20 epochs, early stopping with patience 3, and the checkpoint
selected on **`val_loss`**. Download: 774 files, 457 GB, completed in under 1 h 50 m.
Launched 2026-09-17 00:09. Five seeds were planned; **one completed.**

### Seed 0 — completed, early-stopped after 13 epochs

| epoch | val_loss | IoU | precision | recall |
|---|---|---|---|---|
| 0 | 0.475 | 0.492 | 0.622 | 0.698 |
| 1 | 0.452 | 0.518 | 0.678 | 0.686 |
| 2 | 0.412 | 0.531 | 0.616 | 0.795 |
| 3 | 0.400 | 0.562 | 0.701 | 0.742 |
| 4 | 0.360 | 0.589 | 0.707 | 0.780 |
| 5 | 0.356 | 0.579 | 0.650 | 0.842 |
| 6 | 0.347 | 0.598 | 0.702 | 0.802 |
| 7 | 0.336 | 0.599 | 0.679 | 0.836 |
| 8 | 0.350 | 0.611 | 0.771 | 0.747 |
| **9 — kept** | **0.320** | **0.627** | **0.736** | **0.811** |
| 10 | 0.323 | 0.637 | 0.778 | 0.781 |
| 11 | 0.325 | 0.620 | 0.716 | 0.822 |
| 12 | 0.327 | 0.604 | 0.663 | 0.873 |

Epoch 9 set the lowest `val_loss`; epochs 10–12 failed to beat it and patience 3 fired. The
run took 00:09 → 10:11 (≈ 10 h, ≈ 46 min/epoch); `epoch=9-step=4000.ckpt` was written at
08:28 and exported as `surya_seed0_real_..._es3_best.ckpt` at 10:11.

**The kept checkpoint is epoch 9, IoU 0.627 — not epoch 10's 0.637.** Epoch 10 has the better
overlap and the worse `val_loss`, and `val_loss` is what `ModelCheckpoint` monitors. 0.637 is
the maximum of a curve, not a result: quoting it would be reporting a checkpoint the run did
not keep and cannot reproduce from its own selection rule. Whether `val_loss` is the right
monitor for a task *scored* on overlap is a real question, and is now §9's third item.

### Seed 1 — ended at epoch 6, not usable

| epoch | val_loss | IoU | precision | recall |
|---|---|---|---|---|
| 0 | 0.475 | 0.465 | 0.517 | 0.820 |
| 1 | 0.440 | 0.505 | 0.589 | 0.777 |
| 2 | 0.392 | 0.564 | 0.687 | 0.760 |
| 3 | 0.373 | 0.594 | 0.739 | 0.756 |
| 4 | 0.379 | 0.546 | 0.585 | 0.888 |
| 5 | 0.342 | 0.597 | 0.686 | 0.822 |
| 6 | 0.331 | 0.603 | 0.677 | 0.847 |

`val_loss` was still setting new lows at epoch 6, so early stopping had not fired — the run
simply stopped. Last checkpoint 14:13, last metrics row 14:28, no final checkpoint exported
to `runs/errors/`, and the pid in `real.pid` is gone. **Cause not established.** Seed 1 is
therefore not an ensemble member, and the seed-to-seed spread at the `real` budget is
unmeasured. (The 0.025 spread in §4 is the *medium* budget's and does not transfer.)

### What the run does and does not establish

1. **400 images are not memorised after ten passes.** At 60 images the best epoch was the
   second; here `val_loss` was still setting new lows at epoch 9.
2. **Epoch 0 alone beat the entire medium run** — 0.492 against a best of 0.465, on a broader
   and harder validation set spanning three Januaries.
3. **The model is level with the label floor, not past it.** Per-image IoU 0.6275 against a
   floor of 0.6275, measured on this run's own images. Those are two different quantities
   that coincide, and the gap between them is far below the ±0.008 bootstrap width on the
   model's score. The earlier claim that IoU had *crossed* the floor rested on epoch 10's
   0.637, which the run does not keep.
4. **The floor is no longer borrowed, but it is still a January floor.** Step 6 computes it
   from the first 60 validation images, and the validation set is time-ordered, so all 60
   fall in 2013-01-16 → 01-19. It is this run's own data; it is not yet a 2013–2015 floor.
5. **Which IoU you quote changes the verdict.** Pooled and calibrated at the best threshold
   the model scores 0.6493, above the floor; per-image at 0.5 it scores 0.6275, level with
   it. The per-image figure is the one the training curve reports, so it is the one §1 and
   the deck quote. The 0.6493 is also optimistic: its 0.36 threshold was chosen on the same
   half it is reported on.

---

## 8. Machine findings

| finding | value |
|---|---|
| The pod's real limits (cgroup), vs what `free`/`nproc` report | **60 GiB, 7 cores**, `oom.group=1` — vs 124 GB, 16 cores on the node |
| What an OOM does | kills and restarts the whole container; `/tmp` and `nohup`'d jobs are lost (happened once, with 10 loader workers) |
| Page cache from the `.nc` reads | charged to the pod, fills to the limit within an epoch, reclaimed on demand: **14.2 million reclaims, 0 OOMs** |
| Unreclaimable memory (anon + shmem), 4 loader workers | ~25 GB steady (medium); 34 GB steady / **40 GB peak** (real, at 05:37); the guard trips at 54 |
| GPU during training | 100 % on an L40S with 4 workers (0 % with 2 — loader-starved) |
| Validation and Step 6 | loader- and EFS-bound, not GPU-bound: the medium analysis took 90 min cold and 19 min with files in page cache |
| Download from S3 | 44–70 MB/s with 16 threads |

---

## 9. Open items

- **Finish the ensemble.** Two members give a seed spread of **0.0039** against a bootstrap
  width of 0.0167 — training randomness is now small next to sampling uncertainty, a flip
  from the medium budget. Treat it as provisional: seed 1 is the under-trained epoch-6
  member, which if anything inflates the spread. A third completed seed would settle it.
- **Spread the label floor across the cycle.** Step 6 takes the first 60 validation images
  and the set is time-ordered, so the 0.6275 floor comes entirely from 2013-01-16 → 01-19.
  Sampling those 60 at random across 2013–2015 is cheap and would make the floor
  representative of the set the model is actually scored on.
- **Decide what `ModelCheckpoint` should monitor.** Epoch 10 scored 0.637 IoU against epoch
  9's 0.627 and was discarded for a worse `val_loss`. If overlap is the reported metric,
  monitoring `val_loss` is a default rather than a decision — and on this run it cost 0.010
  IoU. Monitoring IoU directly, or reporting both, deserves an explicit choice.
- **Per-region scoring** — detection rate, false alarms, per-region position and area error —
  is what a forecaster would use, and what the limb effect (§5) argues for over pixel overlap.
- **Test-time augmentation** as a cheap extra uncertainty source (safe at inference; the
  training pipeline disables flips because the outline would not be flipped with the image).
- **Vary the other two rule parameters** (blob size, brush size) in the label floor, not just
  the threshold.

---

## Provenance

| artefact | location |
|---|---|
| executed medium runs | `runs/nbconvert/3_errors_medium_run.ipynb` (provisional), `3_errors_medium_final.ipynb` (final) |
| real run, seed 0 (complete) | per-epoch metrics `runs/errors_surya_seed0_real_window-stratified_pw9_bce-dice_lr0.0001_es3_best/version_0/metrics.csv`; kept weights `runs/errors/surya_seed0_real_..._es3_best.ckpt` |
| real run, seed 1 (partial) | same path with `seed1`; no exported checkpoint. `3_errors_real.ipynb` was never produced |
| general-audience deck | `ws2_project_docs/ar_segmentation_magnetogram.html` and `.pdf` — 14 slides, matches this report |
| memory / GPU log | `runs/nbconvert/mem_watch2.log` |
| answer-key dataset | Roy *et al.* 2026, DOI 10.1038/s41597-026-06552-5; `nasa-ibm-ai4science/surya-bench-ar-segmentation` |
