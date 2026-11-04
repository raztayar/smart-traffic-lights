# CLAUDE.md

Context for Claude Code sessions on this repo. Keep it short and current. Update **Current status** and **Decisions log** at the end of every work session.

## Project
Adaptive traffic-light controller: given where the cars are at a junction, choose the best signal phase.
- **Purpose:** portfolio project for an ML / AI engineer job search. Optimize for a finished, polished, reproducible result over breadth.
- **Centerpiece:** a reinforcement-learning (RL) controller, rigorously benchmarked against strong classical baselines in simulation (SUMO).
- **Secondary:** a computer-vision pipeline that turns junction camera footage (including 360/fisheye) into lane-level state. It uses pretrained models, applied well.

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
Python 3.11 · SUMO + TraCI · `sumo-rl` (Gymnasium) · Stable-Baselines3 (DQN, PPO) · PyTorch · TensorBoard/W&B · Ultralytics YOLO · ByteTrack via `supervision` · OpenCV · pytest · Docker · GitHub Actions · Streamlit

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
- **M0**: CLAUDE.md, README, repo setup ← _current_
- **M1**: SUMO junction, baseline controllers, safety guard and its tests, benchmark script
- **M2**: RL controller (DQN → PPO), reward ablations, held-out evaluation (centerpiece)
- **M3**: perception (YOLO + ByteTrack → lane counts), counting-error evaluation, fisheye handling
- **M4**: integration, side-by-side demo video, Docker, polished README
- Optional: multi-junction grid, pedestrians/emergency vehicles, ONNX + latency benchmark

## Current status
- 2026-11-04: Project scoped. CLAUDE.md and README written. No code yet. GitHub repo not yet connected.
- **Next step:** connect the GitHub repo, then start M1 (install SUMO, build the single 4-way junction network).

## Decisions log
- 2026-11-04: Target ML/AI-engineer roles, so the RL controller and benchmark are the centerpiece and vision stays pragmatic.
- 2026-11-04: All control development happens in simulation (SUMO). Real footage is used only for perception and the demo.
- 2026-11-04: `JunctionState` is the single interface between perception and control, so the two halves can be built and tested separately.
- 2026-11-04: The safety guard sits outside the policy, so the RL agent can never produce unsafe phases.
