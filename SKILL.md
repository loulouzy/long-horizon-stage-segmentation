---
name: long-horizon-stage-segmentation
description: >-
  Design per-frame stage prompts, build causal signal-based automatic trajectory
  segmentation scripts, and integrate usable-only auto-segmentation into data
  collection, trajectory editing, dataset export, and online inference trackers
  for any long-horizon robotic manipulation task (single-arm or bimanual).
  Use when a user wants to decompose a multi-step manipulation task into stage
  prompts, write or refine an automatic episode segmenter from joint/gripper
  trajectories, add stage-level quality flagging, or wire automatic segmentation
  after data collection into an editor or training export pipeline.
---

# Long-Horizon Task Stage Segmentation & Prompting

End-to-end procedure for turning any multi-step robotic manipulation task into
per-frame stage prompts, a deterministic signal-based auto-segmentation script,
and a usable-only collection/editor/export pipeline.

## Workflow Overview

Execute these five phases in order. Do not skip Phase 1—task descriptions almost
always omit hidden transitions (handovers, parallel arm prep, re-grasps, or
gripper twitches) that only appear in real recordings.

### Phase 1: Empirical Inspection of Real Demonstrations

Before proposing any stage list or threshold, inspect 3–5 real demonstrations:

1. **Extract multi-camera filmstrips** (overview + wrist views every 1–2 s)
   annotated with timestamp, gripper commands/states, and key joint angles.
2. **Plot all action & state channels plus joint speeds** across the episodes
   and tabulate every gripper open/close pulse (`start`, `end`, `duration`,
   `peak`, `post_state_median`) and major joint extremum.
3. **Check for hidden structure**:
   - Object/side swaps across episodes (determines whether prompts must refer to
     spatial roles rather than object identities).
   - Unmentioned intermediate maneuvers (e.g. bimanual handovers due to reach
     limits, fixture repositioning, or posture resets).
   - Concurrency between arms and whether parallel actions finish in variable
     order across episodes.
   - Data artifacts: brief pre-reach gripper twitches, partial/narrow releases on
     thin parts, joint-limit saturation, or merged gripper open pulses.

### Phase 2: Stage Decomposition & Prompt Design

Design stages and prompts using [Prompt & Boundary Design Reference](./references/prompt-and-boundary-design.md):

1. **Resolve non-Markovian ambiguity**: Split wherever visually similar states
   require different future actions (e.g. holding an object before vs. after an
   operation, empty gripper between object 1 and object 2 vs. end of task, or
   cycle `k of N` in repetitive motions).
2. **Use strictly causal physical boundaries**: Every cut must trigger on a
   clear joint/gripper event detectable without lookahead so offline labels and
   an online `StageTracker` stay aligned.
3. **Handle bimanual concurrency & handovers cleanly**:
   - Do not cut mid-motion on Arm A just because Arm B finished a parallel grasp;
     describe both roles in one stage and check Arm B as a quality invariant.
   - When two parallel sub-tasks can complete in either order, trigger the next
     stage at `max(event_A, event_B)` (a causal two-latch barrier).
   - End a bimanual handover at **giver release** (after receiver close +
     kinematic cross-over check), never prematurely at receiver close.
4. **Keep prompts concise, role-based, and isolated**:
   - Lowercase imperative ASCII English, short enough to fit comfortably inside
     the VLA prompt + discrete-state token budget.
   - Use spatial/role nouns (`left object`, `right hand`) instead of brand or
     appearance names when object identities vary.
   - Use distinct, non-overlapping stage keys per task so stage labels and
     indices can never cross-contaminate across tasks.

### Phase 3: Signal-Based Auto-Segmentation Script

Build a standalone, dependency-light segmentation module following
[Pipeline Integration Reference](./references/pipeline-integration.md):

1. **Keep core segmentation lightweight**: `segment_episode(action, fps)` should
   require only `numpy` (and `scipy.signal` only if cyclic peak/width detection
   is needed) so it runs inside minimal collection/editor environments where
   `matplotlib` or `pandas` may not be installed.
2. **Anchor on high-SNR landmarks first**:
   - Locate major kinematic maneuvers first, then match preceding grasp pulses
     using the **last** sustained open→close pulse before motion commitment
     (immune to initial twitches and re-grasp retries).
   - Support merged release+grasp pulses (first open = release 1; last close
     before next landmark = grasp 2) and distinct release thresholds for wide vs.
     narrow objects (always enforcing `closed_thresh < open_thresh`).
3. **Emit structured errors, two-tier flags, and verdicts**:
   - Raise `SegmentError(reason, partial_events)` when a required landmark is
     missing.
   - Separate **hard defects** from **soft hints** (cohort median/MAD outliers
     with a minimum cohort size `>= 5` and a non-zero scale floor so saturated
     joints do not trigger false alarms).
   - Classify each episode into `ok`, `hint`, `count` (boundaries sound despite
     non-critical tail/count warning), `check` (requires human review), or
     `exclude` (wrong arm mode / invalid demo).
4. **Support both raw recordings and exported datasets**:
   - Auto-detect raw episode directories vs. exported training datasets (e.g.
     LeRobot v2.1), strictly filtering by task instruction and never clobbering
     existing outputs when zero episodes match.
   - Sanitize `NaN`/`Inf` to `None` before any strict-JSON (`allow_nan=False`)
     write.

### Phase 4: End-to-End Pipeline Integration

Wire automatic segmentation across the data lifecycle (details in
[Pipeline Integration Reference](./references/pipeline-integration.md)):

1. **Usable-only gate everywhere**: Only auto-segment recordings where
   `status == 'complete'`, `usable is True`, and task/arm layout match. Never
   auto-segment or index unusable/rejected episodes, and never mutate raw
   `episode.json` hashes after recording closes.
2. **Post-collection hook**: Immediately after saving a usable episode, run
   single-episode auto-segmentation and atomically upsert (`flock` + temp-file
   rename) into a task-scoped index co-located in the dataset directory (avoids
   polluting `$HOME` during tests). Wrap in non-fatal error handling so a
   segmentation bug never breaks recording persistence.
3. **Trajectory editor integration**:
   - On import of a usable episode, embed the auto-stage record in the replay
     bundle manifest; pre-fill initial segment annotations **only** when verdict
     is sound (`ok`/`hint`/`count`), leaving `check` episodes unlabelled with a
     warning badge for review.
   - Dynamically scope stage colors, dropdown choices, sub-stage controls, and
     suspect filtering to the active task/library (blocking UI signals while
     repopulating dropdowns so dirty flags do not trip).
   - Store all stage boundaries in **source-frame coordinates** so timeline
     trims/deletes keep boundaries exact; handle `number is None` for pre-export
     raw demos.
4. **Export & online tracker parity**:
   - In dataset export, enforce task-namespace matching for both manual labels
     and auto-segment indices; fall back to the whole-episode prompt on
     unlabelled or `check`/`exclude` episodes.
   - When deploying online, implement a causal `StageTracker` on measured
     `observation.state` using the same transition rules and evaluate frame
     agreement + transition delay offline on demonstrations first.

### Phase 5: Verification Checklist

Do not finish until every item passes:

- [ ] **Visual boundary audit**: Render a contact sheet of actual camera frames
  at *every* detected stage boundary across the real demonstrations and verify
  physical alignment by eye.
- [ ] **Signal & report audit**: Inspect generated `report.md`, `summary.csv`,
  and annotated signal plots for zero unexpected hard flags and clean stage
  durations.
- [ ] **Unit tests on edge cases**: Cover normal trajectory, pre-reach gripper
  twitch, merged open pulses, flipped parallel finish order, narrow-object
  release, failed handover kinematics, missing tail release, missing required
  landmarks (`SegmentError`), and task-filtered dataset loaders.
- [ ] **Lifecycle integration tests**: Verify usable vs. unusable gating in both
  post-collection save and editor import, `check`-verdict non-auto-commit,
  editor task switching & suspect filtering, and export namespace isolation.
- [ ] **Environment parity**: Confirm core segmentation + editor flows pass in
  the lightweight runtime Python environment (without `matplotlib`/`pandas`) as
  well as the plotting/export environments.
