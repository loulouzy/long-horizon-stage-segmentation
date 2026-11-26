# Stage Prompt & Boundary Design Reference

This reference captures practical rules for decomposing long-horizon robotic
manipulation tasks into per-frame VLA stage prompts that train cleanly and
transfer reliably to closed-loop real-robot inference.

## 1. Why Stage Prompts Exist (Design Criterion)

Single-frame or short-history VLA policies are effectively Markovian: when two
moments in a long task look visually similar (same object in hand, similar arm
pose), the policy cannot tell whether to repeat, advance, or finish.

Split a task into stages **only where splitting removes genuine behavioral
ambiguity** or isolates a distinct control regime (e.g. precision grasp vs.
continuous trajectory vs. retreat):

- **Good splits**:
  - Before vs. after completing a manipulation on an object (e.g. holding a part
    prior to insertion vs. after completing insertion and preparing to place it).
  - Empty gripper preparing for a second object vs. empty gripper returning to
    rest at episode end.
  - Entering or exiting a cyclic/repetitive action (`k of N` sub-stages so an
    external counter, not a memoryless policy, decides when `N` repetitions are
    finished).
  - Pre-handover positioning vs. post-handover transport.
- **Bad splits**:
  - Cutting in the middle of a single smooth carry/rotation trajectory (for
    example, splitting "lift part" and "rotate part into fixture" at a tiny
    joint angle when operators naturally begin rotating mid-lift). Mid-motion
    prompt switches introduce unnecessary action-chunk discontinuities.
  - Creating sub-second stages shorter than a single action chunk horizon
    (e.g. `< 1.5–2 s` at 30 Hz with `H = 50`). Keep typical stages ~5–20 s.

## 2. Prompt Syntax & Naming Rules

| Aspect | Guideline | Example |
|---|---|---|
| **Language & charset** | Concise lowercase ASCII English, imperative mood | `grasp the left part with the left hand` |
| **Arm/mode prefix** | Prepend a consistent mode tag when multiple arm configurations share the same workspace and similar camera views | `[both arms] ...`, `[one arm] ...` |
| **Object naming** | Prefer **spatial/role nouns** over brand, color, or instance labels whenever objects can swap positions or appearances across episodes | `the right workpiece`, `the left tool` |
| **Explicit hand roles** | Name the active hand(s) explicitly in bimanual tasks so left/right responsibilities are unambiguous | `insert the right part and grasp the left part with the left hand` |
| **Repetition counter** | Append `, k of N` only for cyclic stages where the count is well-defined and verified | `[both arms] wipe the surface, 2 of 3` |
| **Token budget** | Keep every prompt compact (typically 6–20 words) so `prompt + discrete state tokens` stays well below the VLM context limit | Verify worst-case tokenized length before training |
| **Stage key isolation** | Use descriptive, task-specific snake_case stage keys that never collide with other tasks in the same workspace | `grasp_right_part`, `handover_left_part` (not generic `grasp` or `place` if shared with another task) |

## 3. Causal Boundary Patterns

Every offline stage boundary should correspond to a causal event reproducible
from measured joint/gripper state online without lookahead.

### Pattern A: Grasp Completion (`approach_and_grasp -> manipulate`)
- **Signal**: Gripper opens above grasp threshold, then closes below closed
  threshold.
- **Pitfall**: Teleoperators occasionally twitch a gripper open/shut while still
  at home, or open and re-grasp after a missed attempt.
- **Solution**: Anchor on the subsequent manipulation event first, then select
  the **last** sustained open→close pulse (duration `>= 0.4 s`) occurring before
  manipulation commitment.

### Pattern B: Single-Arm Continuous Manipulation (`manipulate -> retreat/next`)
- **Signal**: Primary manipulation joint (e.g. wrist roll, lift height, or
  linear sweep) exceeds a clear peak magnitude (`>= MIN_PEAK`) and returns
  inside `END_FRACTION * peak` (commonly `0.3`).
- **Pitfall**: Splitting at a tiny onset threshold while the arm is still
  approaching the target creates jittery boundaries across demos. Only create a
  separate `move_to_target` stage when the approach is distinctly separated from
  a prior stage (e.g. immediately following a cyclic stage).

### Pattern C: Bimanual Parallel Work (Primary + Secondary Arm)
Two cases occur frequently in real bimanual demos:
1. **Secondary grasp during primary manipulation**: Arm A performs a continuous
   operation while Arm B reaches and grasps the next object mid-stage.
   - **Rule**: Keep this as **one stage** whose prompt states both actions. Do
     not cut Arm A's continuous motion at Arm B's grasp timestamp. Record Arm B's
     grasp completion as a quality metric/flag (e.g. warn if Arm B finished only
     after Arm A ended).
2. **Two parallel completions whose order varies across episodes**: Arm A
   finishes an operation (`event_A`) while Arm B places an earlier object and
   releases (`event_B`). Across demos, sometimes `event_A < event_B` and
   sometimes `event_B < event_A`.
   - **Rule**: Transition to the next stage at **`max(event_A, event_B)`**.
     Online, this is a simple two-latch barrier (`done_A and done_B`) that
     guarantees neither arm is prompted into the next phase before both
     prerequisites are physically satisfied.

### Pattern D: Bimanual Handover (`handover -> transport/place`)
- **Physical sequence**: Giver holds object steady → Receiver moves across
  workspace → Receiver opens and closes gripper on object → brief two-hand
  overlap (~0.5–1.5 s) → Giver opens gripper to release.
- **Rule**: Place the stage boundary at **giver release** (first sustained
  opening of giver gripper after giver's preceding stage ended), **not** at
  receiver close.
  - Why: Switching immediately on receiver close risks telling the giver to drop
    the object if the receiver missed or gripped weakly. Keeping
    `close -> confirm -> release` inside the handover stage lets the policy
    verify the transfer visually and proprioceptively before releasing.
- **Hard invariants to check**:
  1. `receiver_close < giver_release` (receiver actually closed before giver let
     go).
  2. Receiver's lateral/base joint crossed into the handover zone at
     `receiver_close` (prevents mistaking an earlier empty gripper close on the
     far side of the table for a handover grasp).
  3. Two-hand overlap duration (`giver_release - receiver_close`) within a
     reasonable band (soft hint if `< 0.2 s` or `> 4.0 s`).

### Pattern E: Cyclic / Repetitive Motions (`fine` k/N Sub-Stages)
- **Signal**: Repeated narrow strokes or oscillations on a joint.
- **Rule**: Measure dip/peak depth relative to a **local rolling envelope**
  (e.g. rolling maximum over a window wider than one stroke) rather than a
  global baseline, so slow posture drifts and wide plateaus produce ~0 depth.
- **Guards**:
  - Require narrow half-width (`< MAX_WIDTH_S`) so sustained holds are excluded.
  - Require spring-back recovery within `RECOVERY_S` after the peak so a final
    monotonic descent at stage end is never miscounted as an extra stroke.
  - Group strokes into clusters separated by `<= MAX_GAP_S` and select the
    largest cluster so isolated approach/retreat wobbles are ignored.
  - Emit `k of N` fine boundaries (midpoints between adjacent strokes) only when
    the detected count equals expected `N`.

## 4. Gripper Threshold Pitfalls to Check in Every Task

1. **Wide body vs. narrow part release**: Opening from a wide grasp (measured
   hold ~0.40) typically commands `>= 0.55–0.90`, whereas releasing a narrow
   handle or shaft (measured hold ~0.08–0.12) may command only `0.45–0.55`. Use
   separate open thresholds when grasp widths differ, and always pair a lower
   open threshold with a strictly lower closed threshold (`closed < open`, e.g.
   `open=0.35, closed=0.25`).
2. **Merged open pulses**: An arm that releases Object 1 and swings straight to
   Object 2 may keep its gripper open continuously instead of closing empty in
   mid-air. Design mid-episode pulse detection so `release_1 = first_open` and
   `grasp_2 = last_close_before_next_landmark` work identically whether there
   are two separate pulses or one merged pulse.
3. **Action vs. measured state**: Offline segmentation typically runs on
   `action` (works on datasets with action-only columns and cleanly captures
   operator intent without contact stall), whereas online inference tracks
   measured `observation.state`. Inspect both side-by-side during Phase 1 to
   ensure thresholds have generous margin on both streams.
