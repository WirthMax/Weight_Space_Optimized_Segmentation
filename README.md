# Weight-Space Optimized Segmentation

This repo holds the evaluation code and paper figures for a study of **weight-space programming**:
Claude improves a trained segmentation model by editing its weight tensors directly. It uses no
training, no gradients and no ground-truth labels.

Each editing session runs in its own sandbox and produces an edited checkpoint, `<weights>_CLAUDE`.
This repo collects the predictions of every original model and its edited counterpart, scores both
against ground truth, and generates the figures for the paper.

## How the edits are made

Every model is edited in a separate **editing sandbox** with the `weight-space-segmentation`
Claude Code skill. A sandbox holds the model repo, the weights, the paper and unlabeled images. For
Cellpose-SAM the sandbox is `/mnt/Vol_b/cellpose`, and the skill is in
`.claude/skills/weight-space-segmentation/` there.

The skill works in phases:

| Phase | What happens |
|---|---|
| 0 Setup | Hashes the original checkpoint (SHA-256) and creates the editable copy `<weights>_CLAUDE`. Scans `data/` for anything that looks like a label file. Splits the images into `dev` (studied while designing edits) and `check` (held out). |
| 1 Understand | Reads the paper and the repo's inference path: architecture, output head, pre- and post-processing, and which checkpoint keys are actually loaded. |
| 2 Harness | Builds `predict.py`, which calls the repo's own model, preprocessing and post-processing code. |
| 3 Diagnose | Predicts on `dev` and catalogs the failure modes by looking at mask panels. |
| 4 Edit loop | One hypothesis per edit, each written as `edits/NNN_name.py` that declares the tensors it changes. `apply_edits.py` always rebuilds `_CLAUDE` from the original plus all kept edits. It checks that the original's hash is unchanged, that only the declared tensors changed, and that keys, shapes and dtypes match. Each edit is judged visually, then kept or moved to `edits/rejected/`. |
| 5 Confirm | Predicts the held-out `check` images once, after the edit set is final, and verifies the integrity record. |
| 6 Report | Writes `REPORT.md`, `EDIT_LOG.md`, `integrity.json` and the full conversation transcript (`CONVERSATION.md` / `.jsonl`). |

Ground rules during editing:

- No ground truth from any source, and no stand-in for it (such as another model's predictions).
- No metric or evaluation scripts.
- No gradients, optimizers or automated search.
- No re-estimation of BatchNorm statistics unless the user explicitly allows it.
- The original checkpoint is read-only.
- Behavior changes only through the weights: the repo code is unmodified, and the edited checkpoint
  has the same keys, shapes and dtypes.
- No internet access.

Statistics computed from model outputs alone may inform an edit, but the decision to keep or revert
it is made by looking at the masks.

## Why scoring happens here and not in the sandbox

An edit only counts as label-free if ground truth never reaches the sandbox. So all scoring
happens here, and data flows in one direction:

```
editing sandbox (one per model)                 this repo
──────────────────────────────                  ─────────
REPORT, EDIT_LOG, edits/, integrity, ── copy ──▶ runs/<model>/<run>/
transcript
predictions (original + _CLAUDE)     ── copy ──▶ runs/<model>/<run>/predictions/
no labels anywhere                               ground_truth/ ──▶ results/ ──▶ figures/
```

Nothing is ever written back into a sandbox: no ground truth, no metrics, no figures.

## Model runs

| Model | Sandbox | Checkpoints | Kept edits | Status |
|---|---|---|---|---|
| Cellpose-SAM | `/mnt/Vol_b/cellpose` (commit `a54cb48`, `cellpose/` unmodified) | `weights/cpsam_v2` (sha `0f1cc3f7…`) → `weights/cpsam_v2_CLAUDE` (sha `7dc8f15c…`) | **001** flow-field gain ×1.2: the dY/dX rows of the 1×1 readout (`out.weight[:128]`, `out.bias[:128]`); the cellprob rows are unchanged | Editing done. Test-set masks were exported for both checkpoints (68 images, 6,725 → 6,777 masks). Not yet imported here; ground-truth scoring pending. |

`weights/cyto3` is also in the Cellpose sandbox, but the Cellpose 4 repo cannot load Cellpose 3
models, so it was not edited.

Add one row per run as more models are edited.

### Cellpose-SAM: what to know before scoring

- **Images.** 864 unlabeled images: `data/test` (68), `data/train` (540) and `data/train_cyto2`
  (256), named `NNN_img.png`. The counts and names match the Cellpose cyto/cyto2 dataset, which
  stores ground truth as `NNN_masks.png`. This still needs to be confirmed when the ground truth is
  added.
- **Training overlap.** According to the edit report, `train` and `train_cyto2` come from the
  model's own training data. The headline numbers should come from `test`.
- **Dev/check split.** During editing, 18 of the 68 test images were in `dev` and were looked at
  while designing edits. The other 50 were held out in `check`. Results are reported for each subset
  and for all 68 images. The split files are `weight_space_work/splits/dev_all.txt` and
  `check_final.txt` in the sandbox.
- **Exported predictions.**
  - `data/outputs/{Original,CLAUDE}/<stem>_cp_masks.png`: uint16 instance labels for the 68 test
    images, made with the Cellpose-SAM paper's evaluation settings (tile 256, no diameter
    estimation or resizing, no test-time augmentation, tile overlap 0.1, cellprob threshold 0.0,
    flow threshold 0.4). Each folder has a `manifest.json`, and `run_comparison.json` confirms the
    two runs used identical settings.
  - `weight_space_work/preds/{baseline,e001}` (40 dev images) and `{baseline_check,final_check}`
    (824 check images): instance masks in `inst/<split>__<stem>.png`.
- **Expected effect.** According to the edit report, the edit only changes which masks pass the
  flow-error QC. On held-out images it added about 1.4% masks and removed 0.08%, and every
  pre-existing mask boundary stayed the same. Gains should therefore show up mainly as recall, with
  some false-positive slivers and a few very small cells lost.

## Evaluation protocol (planned)

- **Identical pipelines.** The original and `_CLAUDE` checkpoints go through the same code,
  parameters, device and image list; only the weights file differs. Every prediction set has a
  manifest that records the weights' SHA-256, the parameters, the device, the library versions and
  the repo commit.
- **No tuning on ground truth.** Inference parameters are fixed in advance from the model's paper and
  are the same for both checkpoints.
- **Metrics.** Instance-segmentation metrics come from the model repo's own code where it has any.
  For Cellpose this is `cellpose.metrics.average_precision`: AP = TP / (TP + FP + FN) at several IoU
  thresholds. Scores are given per image and aggregated, as paired comparisons between the original
  and edited checkpoints, split into dev/check and test/train subsets.

## Repository layout (planned)

```
runs/<model>/<run_id>/       imported, read-only record of one editing session
    REPORT.md  EDIT_LOG.md  integrity.json  applied_edits.json  edits/
    CONVERSATION.md  CONVERSATION.jsonl.gz
    predictions/{original,claude}/…  + manifest.json
ground_truth/<dataset>/      labels for scoring (gitignored, never copied to a sandbox)
src/                         shared code: loading, metrics, plot style
scripts/                     import, evaluation and one script per figure
results/                     metric tables (CSV)
figures/                     paper figures (PDF + PNG)
```

## Environment

- **Cellpose code:** conda env `CLAUDE_CP` (Python 3.8, torch 2.4.1+cu121). Scripts put the sandbox
  repo `/mnt/Vol_b/cellpose` first on `sys.path`, so they use its code rather than the installed
  package. The base env has no `cellpose`.
- **Hardware:** 2 × Tesla V100-SXM2-32GB. **GPU 1 silently corrupts arithmetic.** The Cellpose-SAM
  session found that `x*2+1` was off by up to 7 and fp32 matmuls by up to 85, with no ECC errors.
  Run all inference on `cuda:0`. `device_selftest()` in the sandbox's `weight_space_work/predict.py`
  refuses any device that fails the check.
