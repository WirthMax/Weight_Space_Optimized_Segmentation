# CLAUDE.md

This repo evaluates weight-space edits and makes the paper figures. The editing happens elsewhere,
in per-model **sandboxes**, using the `weight-space-segmentation` skill. This repo imports the
sandbox outputs, scores the original and `_CLAUDE` checkpoints against ground truth, and generates
the figures. README.md has the background, the run table and the planned layout.

- Skill source (read it to understand a run): `/mnt/Vol_b/cellpose/.claude/skills/weight-space-segmentation/`
  (`SKILL.md`, `references/edit_playbook.md`, `scripts/`)
- Cellpose-SAM sandbox: `/mnt/Vol_b/cellpose`. Run record in `weight_space_work/` (`REPORT.md`,
  `EDIT_LOG.md`, `integrity.json`, `edits/`, `splits/`, `preds/`), test-set masks in `data/outputs/`,
  transcript in `CONVERSATION.md` / `CONVERSATION.jsonl` (54 MB).

## Rules

These rules protect the paper's central claim, that the edits were made without labels.

1. **Sandboxes are read-only from here.** Never write, move or delete anything under a sandbox
   (currently `/mnt/Vol_b/cellpose`). That includes caches, `__pycache__` and outputs of scripts you
   run from it. Copy artefacts into this repo; nothing goes the other way. Above all, ground truth,
   metrics, scores and figures must never end up in a sandbox, because a later editing session
   there could read them and the edits would no longer be label-free.
2. **Ground truth stays in `ground_truth/`.** It is gitignored. Never copy or symlink it into a
   sandbox or into a path you pass to a sandbox script as `--data`.
3. **No weight editing from this repo.** This repo has access to ground truth. If asked to improve a
   model, that work belongs in a sandbox session that follows the skill. Say so rather than doing it
   here.
4. **Imported run records are immutable.** Files under `runs/<model>/<run_id>/` are the record of a
   session. Don't edit them; derived data goes to `results/`.
5. **Fair comparison.** Run the original and `_CLAUDE` checkpoints with identical code, parameters,
   device and image list. Write a manifest for every prediction set (weights SHA-256, parameters,
   device, torch/library versions, repo commit). Before scoring, check both hashes against the
   run's `integrity.json`.
6. **No tuning on ground truth.** Fix inference parameters in advance (from the model's paper or
   repo defaults) and use the same values for both checkpoints. Don't choose thresholds,
   subsets or metrics after seeing which one favors the edit.
7. **Report every subset.** Report `dev` and `check` images separately (dev images were inspected
   during editing). Report test and training-set images separately too. Show losses next to gains.
8. **GPU 0 only.** GPU 1 on this machine silently corrupts arithmetic. Run a device self-test before
   inference; reuse `device_selftest` from the sandbox's `weight_space_work/predict.py` (import it,
   don't modify it).

## Environment

- Use `conda run -n CLAUDE_CP python …` (Python 3.8, torch 2.4.1+cu121) for anything that imports
  `cellpose`. Put `/mnt/Vol_b/cellpose` first on `sys.path` so the sandbox repo's code is used.
  It must stay at commit `a54cb48` with `cellpose/` unmodified. The base env (Python 3.13) has
  no `cellpose`.
- Code that runs outside the sandbox must not write `__pycache__` into it: set
  `PYTHONDONTWRITEBYTECODE=1` when importing from `/mnt/Vol_b/cellpose`.

## Cellpose-SAM facts needed for evaluation

- Checkpoints: `weights/cpsam_v2`, sha256 `0f1cc3f7ecdd8a037a57c6c48d9d8921391be4cbce3fa9f13c3e3a2e1253c667`;
  `weights/cpsam_v2_CLAUDE`, sha256 `7dc8f15cf55644f83e17f288c8d10684afcac2f8e03700d3ea503db8c9016795`.
  The only kept edit is 001, flow gain ×1.2 on `out.weight[:128]` and `out.bias[:128]`.
- Evaluation parameters (Cellpose-SAM paper methods): `bsize=256, diameter=None, augment=False,
  tile_overlap=0.1, cellprob_threshold=0.0, flow_threshold=0.4`, with repo defaults otherwise.
  `data/outputs/` in the sandbox was produced this way by `weight_space_work/generate_test_masks.py`.
- Images: `data/{test,train,train_cyto2}/NNN_img.png` (68 / 540 / 256). The ground-truth naming is
  presumably `NNN_masks.png` (Cellpose dataset convention); confirm when adding it.
- Split membership: `weight_space_work/splits/dev_all.txt` (40, of which 18 test) and
  `check_final.txt` (824, of which 50 test). Paths are relative to `data/`, e.g. `test/051_img.png`.
  In `preds/*/inst/` the file stems are the path with `/` replaced by `__`: `test__051_img.png`.
- Metrics: `cellpose.metrics.average_precision(masks_true, masks_pred, threshold=[...])` returns
  AP, TP, FP and FN per image.

## Conventions

- Each figure is made by one script in `scripts/`. It reads `results/` tables (and predictions or
  images for qualitative panels) and writes PDF and PNG to `figures/`. Scripts must be deterministic
  and rerunnable.
- Large artefacts (prediction PNGs, transcripts, ground truth) are kept out of git, or compressed
  where they are part of the record (`CONVERSATION.jsonl.gz`). Small tables and manifests are
  committed.
