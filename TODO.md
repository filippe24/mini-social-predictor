# TODO

## Done
- 2026-11-18: Read the brief, looked at the header of the sample take S001/i001 (summary only), wrote the plan.
- 2026-11-19: Answers received. Decisions:
  - Units: work internally in metres. Input unit set per source in config, checked against hip width; the aligned_exports source (inches) is supported too.
  - Source: `.aligner/exports/*_.trimmed.bvh` by default, `aligned_exports/actors/*_aligned.bvh` optional. Both actors of a take must come from the same source.
  - Output: `F:/jovane_f/data_f/dualflow_labels`.
  - Takes: `S\d{3}/i\d{3}` only. K0XX, S0XX and calibration folders are skipped. Takes that are not aligned are listed in takes.csv but not labelled by default.
  - Project `mini-social-predictor`, package `social_state`, CLI `social-state`. Config in YAML.
  - Two stages: `signals` (slow, from BVH, cached) and `label` (fast, from the cache) so thresholds can be tuned quickly.

- 2026-11-19 (2): Actor names come from the aligner. Its table is `dataset.aligner.yaml` (sessions → actors, `character: J_`) at `D:/Users/jovane/local-dualflow/`; there is none on the G: copy. Each take's `.aligner/sources.yaml` `belongs:` takes priority. Our config has a rename map on top. Every `S\d{3}` folder is used, including the placeholder `S0XX` (no BVH → status `missing_bvh`). Unreviewed takes are marked. The package stays `social_state`.

## Next
1. Project setup, config, `bvh.py` + tests.
2. `dataset.py`, `signals.py`, `labels.py` + tests.
3. Outputs, CLI, single-take run on S001/i001 (ask first).
4. Quick-look plot, Blender script.

## Waiting on user
- Confirm the path to `dataset.aligner.yaml` (D: local copy) and how to map its session keys to the G: scene folders.
- What "finer timing" means for the past-change labels (shorter look-back window, a higher label rate, or both as settings).
- Full-dataset run, Blender check, thresholds, `scenario` column.
