# CLAUDE.md

Context for Claude Code sessions on this repo. Keep it short and current. Update **Current status** and **Decisions log** at the end of every work session.

## Working agreement (read first)
Raz wants to understand every part and process of this project. Raz makes the decisions, and Claude advises.
- **Explain before changing.** Before any change (code, config, docs, installs, git operations), explain *what* will change, *why*, and *how*. Then wait for Raz's go-ahead.
- **Recommend, don't decide.** When there's a choice, lay out the options with their trade-offs and give a recommendation. Raz picks the direction.
- **Teach new things before using them.** Explain any new concept, tool or library (e.g. `hypothesis`) before it shows up in the code.
- **Small steps.** Make one decision or one change at a time. Don't bundle unrelated changes or run ahead to the next step.

## Project
Adaptive traffic-light controller: given where the cars are at a junction, choose the best signal phase.
- **Purpose:** portfolio project for an ML / AI engineer job search. Optimize for a finished, polished, reproducible result over breadth.
- **Centerpiece:** a reinforcement-learning (RL) controller, rigorously benchmarked against strong classical baselines in simulation (SUMO).
- **Secondary:** a computer-vision pipeline that turns junction camera footage (including 360/fisheye) into lane-level state. It uses pretrained models, applied well.
- **Setting:** a typical Israeli urban 4-way junction: right-hand traffic, Israeli signal sequence, 2 lanes per approach. See **Junction specification**.

## Terminology
Each level is written in its own style, so two levels never look alike. Use these terms precisely, in code and in conversation.

| Level | What it is | Count | How we write it | Example |
|---|---|---|---|---|
| **Road** | One of the two streets that cross | 2 | Always in words, never abbreviated | "north–south road" |
| **Approach** | One side of a road, where cars enter | 4 | One capital letter | `N` |
| **Lane** | Each approach has a **left lane** (left turns + U-turns) and a **thru lane** (straight + right turns) | 8 | Approach + lane, in words | "N thru lane" |
| **Movement** | The path a car takes: approach + turn | 12 (+4 U-turns) | Words: left turn / straight / right turn / U-turn | "N right turn" |
| **Light** (signal group) | The light controlling one lane | 8 | `approach-lane` | `N-left`, `N-thru` |
| **Phase** | A set of lights that are green together | 8 | `P1`–`P8` + a short name | P3 "North only" |

- Movements say "straight", and lights say "thru". `N-thru` is the light for two movements: N straight and N right turn.
- Old abbreviations like "N-S" and "N-SR" are retired.
- **Conflict**: two movements whose paths cross, or merge into the same exit road. Movements from the same approach only split apart, so they never conflict.
- **Protected turns only**: a green is a green. No phase contains conflicting movements.
- **Phase set**: the phases a controller may choose from, defined in YAML.
- **Transition**: what each light does when the phase changes. A light that stops goes green → yellow → red. After the all-red clearance, a light that starts goes red+yellow → green. There's no flashing green. A light that is green in both the old and new phase stays green.
- **Cycle**: one pass through a fixed phase order. Only fixed-time has one.
- **Decision step**: how often the controller is asked "which phase next?".
- **SUMO state**: SUMO's own name for this is "phase": one row of its light program, including yellow rows. Don't confuse it with our phase.

## Junction specification
A 4-way junction with right-hand traffic. Each approach has a left lane and a thru lane, so there are 8 lights: `N-left`, `N-thru`, `E-left`, `E-thru`, `S-left`, `S-thru`, `W-left`, `W-thru`.

### Conflict check (exhaustive, 2026-11-04)
**Method.** Walk clockwise around the junction's edge: N in, N out, E in, E out, S in, S out, W in, W out. Each movement connects an entry point to an exit point. Two movements from different approaches **cross** if their endpoints alternate around the edge, and **merge** if they end on the same exit road. A script checked every combination.

| Level | Checked | Conflict | Compatible |
|---|---|---|---|
| Movements (pairs from different approaches) | 54 | 28 (16 crossing + 12 merging) | 26 |
| Lights (all pairs) | 28 | 20 | 8 |
| On/off combinations of all 8 lights | 256 | 239 unsafe | 17 safe: 1 all-red, 8 with one light, 8 with two lights |

- The 16 crossing points match the textbook number for a 4-way junction.
- **No safe combination has 3 or more lights.** Every light on one road conflicts with every light on the other road. On each road, the 4 lights form a square whose diagonals conflict (a left turn vs. oncoming straight/right), so any 3 of the 4 include a conflict:
```
 N-left ──── N-thru        ──── = can be green together
   │    ╲  ╱    │          ╲╱   = conflict
   │     ╳      │
   │    ╱  ╲    │
 S-left ──── S-thru        (the east–west road has the same square)
```
- **U-turns** from the left lane (16 movements: 96 pairs, 16 crossing + 24 merging) add conflicts at the movement level. The light-level conflicts and the 8 phases stay exactly the same.

### The 8 phases (main phase set)
| Phase | Name | Green lights | Road |
|---|---|---|---|
| P1 | N & S thru | `N-thru` + `S-thru` | north–south |
| P2 | N & S left | `N-left` + `S-left` | north–south |
| P3 | North only | `N-left` + `N-thru` | north–south |
| P4 | South only | `S-left` + `S-thru` | north–south |
| P5 | E & W thru | `E-thru` + `W-thru` | east–west |
| P6 | E & W left | `E-left` + `W-left` | east–west |
| P7 | East only | `E-left` + `E-thru` | east–west |
| P8 | West only | `W-left` + `W-thru` | east–west |

- **4-phase comparison set:** P1, P2, P5, P6. This is the standard paired cycle, used by fixed-time and actuated.
- **Single-light combinations** are safe but never worth choosing, since a phase that adds a second light lets more cars through. They aren't in the action set. They do appear **during transitions**, though, so the safety guard must accept them.

### Switching between phases
Two phases are never green at the same time, because each phase is already the largest safe combination. What matters is which phases **share a light**:
```
              P1  N & S thru
          N-thru /       \ S-thru
     P3 North only       P4 South only
          N-left \       / S-left
              P2  N & S left          (P5–P8 form the same ring)
```
- **Neighbours in the ring share one light**, which stays green through the switch. For example, P1 → P3: `N-thru` keeps flowing, `S-thru` goes yellow → all-red, then `N-left` goes red+yellow → green.
- **Across the ring** (P1↔P2, P3↔P4) and **between roads**, no light is shared, so it's a full switch.

### Assumptions
1. Merging into the same exit road counts as a conflict. This is the standard, safe rule.
2. No pedestrians. They're an optional extension, out of scope for now.
3. U-turns are allowed from the left lane and are part of the simulated traffic. They don't change the lights or the phases.

## Architecture
```
camera video ──► perception ──► JunctionState ──► controller ──► safety guard ──► phase
                 (detect, track,   (per-lane queue,   (fixed-time /     (min green,
                  map to lanes)     approaching,       actuated /        yellow, all-red,
                                    waiting time)      max-pressure /    conflict matrix)
                                                       RL)
SUMO simulator ─────────────────► JunctionState  (used for training and evaluation)
```
- `JunctionState` (in `interface/`) is the **only** contract between perception and control. Controllers never see pixels, and perception never knows which controller is running.
- Every controller implements the same `Controller` interface (`controllers/base.py`).

## Non-negotiables
1. **Safety is enforced outside the learned policy.** `safety/phase_guard.py` wraps every controller, and the RL agent can only pick from safe actions.
2. **No result without baselines.** Every controller is compared against fixed-time, actuated and max-pressure.
3. **Honest evaluation.** Use held-out traffic scenarios, 10+ seeds, and report confidence intervals. Never tune on the evaluation scenarios.
4. **Reproducible.** Every number in the README comes from a single documented command. Seed everything. Configs live in YAML, not hard-coded.

## Metrics
Average wait per vehicle, 95th-percentile wait (fairness), throughput, average queue length.

## Tech stack
Python 3.11 via `uv` (`pyproject.toml` + `uv.lock`) · SUMO + TraCI/libsumo · Gymnasium (`sumo-rl` TBD) · Stable-Baselines3 (DQN, PPO) · PyTorch · TensorBoard/W&B · Ultralytics YOLO · ByteTrack via `supervision` · OpenCV · pytest + `hypothesis` (property-based tests for the safety guard) · Docker · GitHub Actions · Streamlit

## Repo layout (planned)
```
sim/          SUMO network + seeded route generators (low/med/high/rush-hour/asymmetric)
controllers/  base.py, fixed_time.py, actuated.py, max_pressure.py, rl_agent.py
safety/       phase_guard.py
rl/           env wrapper, state/reward definitions, train.py, configs/
perception/   detector.py, tracker.py, lane_zones.py, undistort.py
interface/    JunctionState dataclass
eval/         benchmark.py, plots, stats
demo/         side-by-side video renderer, Streamlit app
tests/
```

## Commands
_To be filled in as they exist._
- Tests: `TBD`
- Benchmark: `TBD` (target: `python eval/benchmark.py --scenarios all --seeds 10`)
- Train: `TBD`

## Roadmap
- **M0**: CLAUDE.md, README, repo setup ✅
- **M1**: SUMO junction, baseline controllers, safety guard and its tests, benchmark script ← _current_
- **M2**: RL controller (DQN → PPO), reward ablations, held-out evaluation (centerpiece)
- **M3**: perception (YOLO + ByteTrack → lane counts), counting-error evaluation, fisheye handling
- **M4**: integration, side-by-side demo video, Docker, polished README
- Optional: multi-junction grid, pedestrians/emergency vehicles, ONNX + latency benchmark

## Future direction (out of scope)
- Eventually, make the controller work for other junction types (T-junctions, more lanes, other lane layouts). This is **not** a goal for this project.
- Rule for now: when it costs nothing extra, junction-specific facts (lights, conflict matrix, phase set) stay in config rather than code. Don't build generic multi-junction support.

## Current status
- 2026-11-04: M0 done. CLAUDE.md, README, MIT LICENSE and .gitignore are committed and pushed to https://github.com/raztayar/smart-traffic-lights (public, with topics set). No code yet.
- 2026-11-04: M1 planning done. We settled the junction layout, the terminology, the phase set (confirmed by an exhaustive conflict check), the evaluation split and the tooling. No code yet.
- **Next step:** decide own TraCI loop vs. `sumo-rl`, then install SUMO and set up the environment with `uv`.

## Open questions (M1)
- Add the conflict check as a test? It would recompute conflicts from the lane layout and check them against the YAML conflict matrix. Recommended once that YAML exists. The check script used on 2026-11-04 was throwaway and isn't in the repo.
- Own TraCI loop vs. `sumo-rl`. Recommended: our own loop, so safety stays in our guard. Decide before writing the sim loop.
- `JunctionState` contents: **deep dive pending (Raz asked not to forget).** It must stay camera-observable, and it probably includes the current phase and time in phase.
- Exact Israeli transition durations (yellow, all-red, red+yellow). Verify against Ministry of Transport guidelines.

## Decisions log
- 2026-11-04: Target ML/AI-engineer roles, so the RL controller and benchmark are the centerpiece and vision stays pragmatic.
- 2026-11-04: All control development happens in simulation (SUMO). Real footage is used only for perception and the demo.
- 2026-11-04: `JunctionState` is the single interface between perception and control, so the two halves can be built and tested separately.
- 2026-11-04: The safety guard sits outside the policy, so the RL agent can never produce unsafe phases.
- 2026-11-04: Work style: Claude explains and recommends before every change, and Raz decides (see Working agreement).
- 2026-11-04: Simulate a typical Israeli junction, so it is realistic and matches the footage we'll use later.
- 2026-11-04: 2 lanes per approach. Lane assignment: the left lane carries left turns (and U-turns), and the thru lane carries straight + right turns. This is the only assignment that supports all 8 phases.
- 2026-11-04: No flashing green (it's uncommon in cities). The transition is yellow → all-red → red+yellow.
- 2026-11-04: Protected turns only, with no permissive turns. Every phase is fully conflict-free.
- 2026-11-04: The main phase set has all 8 conflict-free phases (P1–P8). The phase set is YAML config, and a 4-phase comparison set (P1, P2, P5, P6) exists. Raz's reasoning: don't limit the controller, since the extra options are part of what lets an adaptive controller win.
- 2026-11-04: An exhaustive conflict check confirmed exactly 8 phases. No safe combination has 3+ lights (see Junction specification).
- 2026-11-04: Terminology agreed: road / approach / lane / movement / light / phase, each written in its own style (see Terminology).
- 2026-11-04: Merging into the same exit road counts as a conflict. U-turns from the left lane are included in the traffic. Pedestrians are out of scope (optional extension).
- 2026-11-04: Generalizing to other junction types is a future direction, out of scope. Junction-specific facts stay in config where that's free.
- 2026-11-04: The safety guard works per light (its own timers for each light), because switching between neighbouring phases leaves one light green.
- 2026-11-04: Benchmark fairness: max-pressure gets the same 8 phases as RL (the like-for-like test). Fixed-time and actuated use the 4-phase comparison set, as a real junction would. A 4-vs-8 RL ablation is optional.
- 2026-11-04: Train/held-out split: tune and train on low/med/high with seeds 0–999. Evaluate on all 5 scenarios with seeds 1000–1009. Rush-hour and asymmetric are never used for tuning. Report seen and unseen patterns separately.
- 2026-11-04: Use `hypothesis` property-based tests for the safety guard.
- 2026-11-04: Use `uv` (changed from venv + pip): it's faster, and its lockfile pins every package's exact version for reproducibility.
