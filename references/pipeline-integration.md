# Pipeline Integration & Implementation Reference

Blueprint for implementing a robust auto-segmentation module and wiring it into
data collection, trajectory editing, dataset export, and online inference.

## 1. Module Architecture & Dependency Hygiene

Split responsibilities cleanly across small, testable modules:

| Component | Dependencies | Responsibility |
|---|---|---|
| **Core segmenter** (`segment_episode`, `flag_episodes`, `segment_action_entry`) | `numpy` only (plus `scipy.signal` only if cyclic peak finding is needed) | Pure signal-to-stage segmentation, single-episode & cohort flagging, JSON-safe entry construction |
| **CLI & reporting** (`load_episodes`, `write_outputs`, `build_report`) | Optional `pandas`/`pyarrow` (for parquet datasets) and optional `matplotlib` (gracefully skipped if absent) | Batch run on raw dirs or exported datasets; writes `segments.json`, `summary.csv`, `report.md`, `contact_sheet.png`, `flagged/*.png`, and per-episode plots when cohort `<= 10` |
| **Stage prompt registry** (`stage_prompts.py`) | `numpy` | Task-scoped prompt tables, `stage_prompt`, `parse_label`, `build_index`, `load_index`, `project_stage_plan`, `plan_from_segments`, `autofill_segments`, `frame_prompt`, `prompt_runs` |
| **Collection & editor hooks** | Standard library (`fcntl`, `json`, `pathlib`) | Usable-only automatic segmentation after recording close and during editor import; atomic locked index upsert |

Keeping core segmentation free of `matplotlib`/`pandas` ensures it runs
out-of-the-box in lightweight GUI and robot-collection Python environments.

## 2. Standard Stage Entry & Index Schema

Store all stage intervals in **source-frame coordinates** (`[start, end)` half-open
frame indices relative to the raw recording's frame 0). Because coordinates are
anchored to source frames, trimming or deleting segments in a trajectory editor
never invalidates stage boundaries.

```json
{
  "schema": "auto_segment_index_v1",
  "task": "<task_namespace>",
  "dataset": null,
  "note": "Generated in source-frame coordinates.",
  "episodes": {
    "episode_20260101_120000_abcd1234": {
      "number": null,
      "arm": "both",
      "task": "<task_namespace>",
      "verdict": "ok",
      "flags": [],
      "frames": 1800,
      "counts": {},
      "metrics": {},
      "events": {
        "grasp_1_close": 240,
        "op_1_end": 780
      },
      "stages": [
        {"name": "stage_1", "start": 0, "end": 240},
        {"name": "stage_2", "start": 240, "end": 780}
      ]
    }
  }
}
```

### Strict JSON & Concurrency Rules
- Always sanitize non-finite floats (`NaN`, `±Inf` → `None`) and numpy scalars
  before writing JSON with `allow_nan=False`.
- Guard index updates with an exclusive file lock (`fcntl.flock` on a sibling
  `.<name>.lock` file) + write-to-tmp + `os.fsync` + atomic `Path.replace()`.
- Never mutate raw `episode.json` after recording closes (preserving checksums
  and validation provenance).

## 3. Two-Tier Flagging & Verdict Triage

Separate actionable defects from statistical hints so reviewers focus only on
episodes that truly need attention:

1. **Hard defects** (plain message, no hint prefix):
   - Segmentation failure (`SegmentError`, carrying `partial` events so debug
     plots still show what was detected before failure).
   - Missing required arm grasp, invalid handover kinematics, wrong arm moving
     in single-arm mode, or missing final release.
2. **Soft hints** (explicit prefix such as `提示：`):
   - Stage duration outliers on `log(max(seconds, 0.2))` via robust z-score
     (`|x - median| / max(MAD * 1.4826, min_scale) > 3.5`).
   - Peak joint amplitude outliers via robust z-score with a non-zero
     `min_scale` floor (e.g. `0.12 rad`), and **only when cohort size `>= 5`**
     (prevents false alarms on small batches or joints hitting hardware stops).
   - Borderline handover overlap durations.
3. **Verdict mapping**:
   - `exclude`: wrong arm mode (e.g. idle arm moved significantly in single-arm
     demo).
   - `check`: `SegmentError` or structural hard defect; excluded from automatic
     stage export unless explicitly overridden, and **never** silently written
     into initial editor annotations.
   - `count`: only non-boundary hard flags (e.g. cyclic stroke count off by 1,
     or final tail release trimmed early); stage boundaries remain sound and can
     still produce per-frame stage prompts.
   - `hint`: only soft hints.
   - `ok`: zero flags.

## 4. Usable-Only Automatic Segmentation Lifecycle

Enforce a single shared predicate `is_usable_target_episode(meta)` across
collection, import, and editor UI:

```python
def is_usable_target_episode(meta):
    return (
        isinstance(meta, dict)
        and meta.get('status') == 'complete'
        and meta.get('usable') is True
        and meta.get('sides') == expected_sides
        and normalized_task(meta.get('task')) == normalized_task(expected_task)
    )
```

### Hook 1: Immediately After Data Collection (`stop_episode`)
1. Finalize and validate the raw episode first.
2. If `is_usable_target_episode(result)` is `False`, do nothing.
3. If `True`, run single-episode segmentation and upsert into
   `<data_root>/auto_segment_<task>_index.json` (co-located with the dataset
   root so unit tests in `tmp_path` never pollute `$HOME`).
4. Surface a concise summary in the collector status message (e.g.
   `5 stages ok` vs. `needs review (<reason>)`). Wrap in `try/except` so an
   unexpected segmentation error can never compromise saving the recording.

### Hook 2: During Trajectory Editor Import (`import_episode`)
1. Run `is_usable_target_episode(meta)` before segmenting. Unusable episodes
   receive no embedded auto-stage metadata and keep a single unlabelled segment.
2. For usable episodes:
   - Compute the stage entry from `arrays['action']` and embed it in
     `bundle.json` (e.g. `manifest['<task>_auto_stages']`).
   - Best-effort upsert into `<source_root_parent>/auto_segment_<task>_index.json`.
   - When `verdict in ('ok', 'hint', 'count')` and `stages` is non-empty, split
     the initial segment via `autofill_segments(...)`, run `validate_segments`,
     and save the labelled stage segments directly into initial
     `annotations.json` (preserving episode quality and notes).
   - When `verdict == 'check'`, keep the single unlabelled segment so the
     operator explicitly reviews and corrects it.

### Hook 3: Trajectory Editor UI (`TrajectoryEditor`)
- **Task-scoped controls**: Dynamically switch timeline `stage_task`, stage
  colors, `stage_combo` items, and sub-stage checkboxes (`fine_check`) whenever
  switching libraries, opening a bundle, or editing the task prompt. Always wrap
  `stage_combo` repopulation in `QSignalBlocker` and restore `currentText` to
  avoid spuriously marking the form dirty.
- **Fallback chain for usable bundles**:
  1. `manifest['<task>_auto_stages']`
  2. Merged index (`workspace` + `data_root`, with `data_root` taking priority)
  3. On-the-fly `segment_action_entry(bundle.data['action'])` for legacy usable
     replays imported before auto-segmentation existed
  - Return `None` immediately whenever the bundle's source is not usable!
- **Library list & suspect filter**:
  - Format optional episode numbers safely (`f'#{n:03d} · '` only when
    `n is not None`).
  - Show auto-segment badges and stage-second tooltips for usable imported and
    usable raw episodes; hide badges for unusable episodes.
  - Enable the suspect-only filter (`check`, `exclude`, `count`) per library.

### Hook 4: Dataset Export & Online Inference
- **Export (`export_edited`)**:
  - Resolve `task_kind` from episode metadata / task prompt.
  - Ignore any index entry whose `task != task_kind`, and parse manual segment
    labels strictly with `parse_label(label, task=task_kind)`.
  - Prioritize reviewed manual/editor stage segments over the auto index; fall
    back to the whole-episode prompt when unlabelled or `check`/`exclude`.
  - Verify per-frame prompt roundtrip against the exported dataset rows.
- **Online `StageTracker` (inference)**:
  - Implement transitions causally on measured `observation.state` using the
    same event semantics (grip close, wrist roll return, two-latch parallel
    barrier, giver release).
  - Evaluate offline on demonstration states (`stage_tracker_eval`) to measure
    per-frame agreement and per-boundary transition latency before real-robot
    trials.

## 5. Visual & Automated Verification Template

1. **Boundary camera contact sheet**: For every real episode (rows) and every
   internal stage boundary (columns), paste the overhead RGB frame at
   `stage['end']` with label `ep{k} {prev}->{next} @ {t:.1f}s`. Visually confirm
   consistent physical state down every column.
2. **Synthetic unit tests**: Test normal sequence, pre-reach twitch, merged
   open pulses, both parallel completion orderings, narrow release threshold,
   bad handover posture, missing tail release, `SegmentError` partial events,
   and strict dataset-loader task filtering.
3. **End-to-end integration tests**: Test usable vs. unusable collection close,
   usable vs. unusable import, `check`-verdict protection, legacy-replay auto
   display, library switching, suspect filtering, and export namespace
   isolation.
