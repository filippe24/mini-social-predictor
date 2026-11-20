# Task: social-state labeling pipeline (milestone 1)

## Context

We have a multimodal dataset of two-person (dyadic) interactions: Vicon body mocap,
separated voice tracks, Pupil Labs Neon eye tracking on both actors, and Vicon-tracked props.
About 8 hours, 7-9 actors, many different scenarios (roommates, coworkers, friends, emotional scenes).

The research goal is a small, causal model that predicts actor B's high-level social reaction
(proximity, orientation, later gaze target) from actor A's behaviour. The model is a proof that
the dataset carries learnable social structure; it is not the contribution itself.

**This task is milestone 1 only: compute proximity and orientation labels from BVH.**
No gaze, no audio, no props, no model yet. Structure the code so those can be added later.

Existing code that produced the data (read-only for us, do not modify):
- aligner: https://codeberg.org/filippe24/aligner (Python 3.11+, uv, Qt) also in here "C:/Users/jovane/project/aligner",
- vicon_exporter: https://github.com/filippe24/vicon_exporter also in here "C:/Users/jovane/project/vicon_exporter",

## Data facts (checked on a sample take)

| Fact | Value |
| --- | --- |
| Layout | `<root>/sXXX/iXXX/.aligner/exports/` (scene / interaction) |
| Files | One BVH per actor, `<actor>_.trimmed.bvh`, exactly two per take (sample: `R_`, `P_`) |
| Frame rate | 120 fps (Frame Time 0.008333). Read it from each file, do not hard-code |
| Units | Metres |
| Up axis | Z. The floor is the X-Y plane |
| World space | Shared by both actors (sample hip distance 0.84-4.57 m) |
| Sync | Already cropped by aligner. Both sample files have 7,214 frames (~60 s) |
| Skeleton | 60 joints, root `Hips` with 6 channels (`Xposition Yposition Zposition Xrotation Zrotation Yrotation`) |
| Joints needed now | `Hips`, `Head`, `LeftUpLeg`, `RightUpLeg` |
| Rotation order | Differs per joint (five different orders in one file). Parse each joint's `CHANNELS` line |

Props are C3D in the same world space (not needed in milestone 1).

## Rules

- Never write into the dataset folders. All outputs go to a separate output directory given on the command line. "G:/My Drive/temp/dualflow"
- Do not derive facing direction from the hip rotation channels: the rest pose is not axis-aligned.
- If the two BVH files of a take have different frame counts or frame rates, flag the take and skip it. Never guess an offset.
- All thresholds live in one config file (YAML or TOML), not in code.
- Every step must run on both Windows and Linux.

## Project setup

- Python 3.12, managed with **uv** (`uv init`, `uv add`, `uv run`).
- Dependencies for now: numpy, pandas, scipy (rotations), matplotlib (histograms), pyyaml or tomllib, pytest. some are already installed with `uv add`
- Layout: a core package (suggested name `social_state`) with modules `bvh.py` (parser + forward kinematics),
  `signals.py`, `labels.py`, `dataset.py` (take discovery), and a CLI entry point.
- Later (not now): a PySide6 + pyqtgraph viewer that only reads the core package's outputs.
  PyTorch will be added at the modeling stage; on Windows the CUDA build needs its own uv index.

## Processing steps

1. **Find takes.** Walk `sXXX/iXXX/.aligner/exports/`, require exactly two `*.trimmed.bvh`.
   Write `takes.csv`: scene, interaction, actor_a, actor_b, n_frames, fps, duration_s, status,
   plus an empty `scenario` column to be filled in by hand.
2. **Parse and pose.** Parse BVH hierarchy and motion, respecting per-joint channel order.
   Forward kinematics → world positions of the four joints above, per frame.
3. **Validate.** Same frame count and fps for both actors; no NaNs; plausible ranges. Record status in `takes.csv`.
4. **Resample** 120 Hz → 30 Hz by averaging groups of 4 frames (positions only; compute angles after).
5. **Per-actor signals (30 Hz):**
   - pelvis floor position (x, y) from `Hips`
   - pelvis height z (posture: sitting vs standing)
   - forward vector = normalize( proj_floor( (LeftUpLeg − RightUpLeg) × up ) ), up = (0, 0, 1)
6. **Pair signals (30 Hz):**
   - floor distance between the two pelvises
   - for each actor: angle in [0°, 180°] between their forward vector and the floor direction to the partner
7. **Labels** at 5 Hz (every 0.2 s), horizon h = 1 s, for each actor as "B":

   | Label | Classes | Rule (defaults, from config) |
   | --- | --- | --- |
   | proximity_state | close, mid, far | distance at t+h: < 1.2 m close, 1.2-3.6 m mid, > 3.6 m far |
   | proximity_change | approach, stay, retreat | d(t+h) − d(t): < −0.15 m approach, > +0.15 m retreat |
   | orientation_state | facing, oblique, turned_away | B's angle at t+h: < 30° facing, 30-90° oblique, > 90° turned away |
   | orientation_change | turn_toward, hold, turn_away | angle(t+h) − angle(t): < −15° turn toward, > +15° turn away |

   A step is a **transition** for a signal when its change label is not stay/hold.
   Also store the same labels evaluated at time t (B's current/past state), for use as model input later.
8. **Validity mask.** Per 30 Hz frame and per actor. All valid for now, but keep the hook:
   missing data will be marked later, and a label step is invalid if any frame in [t, t+h] is invalid.
9. **Outputs** (in the output directory):
   - one `.npz` per take and actor-as-B: 30 Hz signals, 5 Hz label tracks (as integer class ids + a class-name map), validity mask, metadata (take id, actors, fps)
   - `label_stats.csv`: per take, class counts and transition counts per label
   - histograms (PNG) over all takes: distance, distance change, angle, angle change, with the current thresholds drawn as vertical lines

## Tests (pytest)

- Synthetic BVH pair with a known distance and facing angle → expected signals.
- BVH parser respects different per-joint rotation orders (compare against a hand-built case).
- Distance is identical whichever actor is B.
- Forward vector sign: a synthetic actor walking straight along +X has forward ≈ (1, 0).
- Label counts per take sum to the number of valid steps.

## Deliverables for this milestone

1. Working CLI, e.g. `uv run social-state label --data <root> --out <dir> --config config.yaml`.
2. `takes.csv`, per-take `.npz`, `label_stats.csv`, histograms.
3. A short summary printed at the end: number of takes, usable hours, class balance and transition counts per label.
4. A quick-look script that plots, for one take, distance and both angles over time with the label tracks underneath.
5. A Blender script (`tools/blender_view.py`, run with `blender --python tools/blender_view.py -- --take <take dir> --labels <npz dir>`)
   that imports both BVH files with Blender's built-in BVH importer (Z up: set the importer's up/forward axes accordingly, scale 1),
   sets the scene to 120 fps, and adds the labels so they can be checked while scrubbing:
   - timeline markers at every transition (e.g. `B=R_ approach`), mapped from 5 Hz label time to 120 Hz frames
   - keyframed custom properties on each armature (distance, angle, class id per label) to view in the Graph Editor
   The script runs inside Blender's own Python, so it may only use numpy and the standard library (read the .npz directly, do not import the project package).

Stop after this milestone and report the class balance and transition counts; thresholds will be revised from the histograms before any modeling.

## Roadmap (for code structure only, do not implement now)

2. Baselines: class prior, hand rules (face the speaker, keep distance), B's past state only.
3. Features from A's body (≈15 joints in A's pelvis frame) + pair geometry; tree model (LightGBM) and a small causal GRU; leave-one-actor-out evaluation, macro-F1 on all steps and on transition steps.
4. A's voice (pitch, energy, VAD) and transcript (WhisperX + frozen sentence embeddings).
5. Gaze (Neon) as input and as an output head; props from C3D.
