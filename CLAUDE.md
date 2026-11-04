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
- **Setting:** a typical Israeli urban 4-way junction: right-hand traffic, Israeli signal sequence, 2 lanes per approach.

## Glossary
Use these terms precisely, in code and in conversation.
- **Approach**: one road entering the junction (N, S, E, W).
- **Lane assignment**: each approach has a left lane (left turns only) and a right lane (straight + right).
- **Movement**: approach + direction of travel (e.g. N-left). There are 12 in total.
- **Signal group**: one light controlling one lane's movements. There are 8: N-L, N-SR, S-L, S-SR, E-L, E-SR, W-L, W-SR.
- **Conflict**: two movements whose paths cross or merge. In our layout, the only conflicts within an axis are left vs. oncoming straight & right (N-L vs. S-SR, S-L vs. N-SR). Every N–S group is treated as conflicting with every E–W group, which is a conservative simplification.
- **Phase**: a set of non-conflicting signal groups that are green together. All turns are protected (a green is a green).
- **Phase set**: the phases a controller may choose from, defined in YAML. The main set has 8 phases: per axis, opposite straights, opposite lefts, all of one approach, all of the other.
- **Transition**: the per-signal-group sequence between phases: green → yellow → red, then the all-red clearance, then red+yellow → green. There's no flashing green. Groups that are green in both phases stay green.
- **Cycle**: one pass through a fixed phase order. Only fixed-time has one.
- **Decision step**: how often the controller is asked "which phase next?".
- **SUMO state**: SUMO's own name for this is "phase": one row of its light program, including yellow rows. Don't confuse it with our phase.

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

## Current status
- 2026-11-04: M0 done. CLAUDE.md, README, MIT LICENSE and .gitignore are committed and pushed to https://github.com/raztayar/smart-traffic-lights (public, with topics set). No code yet.
- 2026-11-04: M1 planning done. We settled the junction layout, the phase set, the glossary, the evaluation split and the tooling. No code yet.
- **Next step:** decide own TraCI loop vs. `sumo-rl`, then install SUMO and set up the environment with `uv`.

## Open questions (M1)
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
- 2026-11-04: 2 lanes per approach. Lane assignment: the left lane is left turns only, and the right lane is straight + right. This is the only assignment that supports every phase below.
- 2026-11-04: No flashing green (it's uncommon in cities). The transition is yellow → all-red → red+yellow.
- 2026-11-04: Protected turns only, with no permissive turns. Every phase is fully conflict-free.
- 2026-11-04: The main phase set has all 8 conflict-free phases: the paired phases plus all-of-one-approach. The phase set is YAML config, and a 4-phase set (paired only) exists for comparison. Raz's reasoning: don't limit the controller, since the extra options are part of what lets an adaptive controller win.
- 2026-11-04: The safety guard works per signal group (its own timers for each group), because switching between some phases leaves certain groups green.
- 2026-11-04: Benchmark fairness: max-pressure gets the same 8 phases as RL (the like-for-like test). Fixed-time and actuated use the standard paired 4-phase cycle, as a real junction would. A 4-vs-8 RL ablation is optional.
- 2026-11-04: Train/held-out split: tune and train on low/med/high with seeds 0–999. Evaluate on all 5 scenarios with seeds 1000–1009. Rush-hour and asymmetric are never used for tuning. Report seen and unseen patterns separately.
- 2026-11-04: Use `hypothesis` property-based tests for the safety guard.
- 2026-11-04: Use `uv` (changed from venv + pip): it's faster, and its lockfile pins every package's exact version for reproducibility.
