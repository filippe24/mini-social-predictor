# social_state

Labeling pipeline for a dyadic mocap dataset. Current task: CLAUDE_TASK_labeling.md.

## Folders
- Rely mostly on rules and permissions.

## Commands
- `uv sync`, `uv run pytest`, `uv run social-state ...`
- Code must run on Windows and Linux (pathlib, no shell-specific calls).

## Ask before heavy operations
Stop and ask for confirmation, with the exact command and a time/size estimate, before:
- processing more than one take, or any run expected to take over ~1 minute
- reading any file larger than 1 MB (BVH is ~9 MB: inspect with a short Python snippet, print summaries only)
- installing large packages (e.g. torch) or downloading models/data
- deleting or overwriting files, or writing outside this repo and the output folder
Prefer giving me the command to run myself; I will report the summary back.

## Division of work
I run full-dataset jobs, check labels visually in Blender, choose thresholds, and fill takes.csv `scenario`.
When one of these is needed, stop and tell me what to run or look at.
At the end of each session, update TODO.md: done / next / waiting on user.
