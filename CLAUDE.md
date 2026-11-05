# CLAUDE.md

Context for Claude Code sessions on this repo. Keep it short and current. Log every decision in **Decisions log** as soon as it's made, and keep **Current status** current.

## Working agreement (read first)
Raz wants to understand every part and process of this project. Raz makes the decisions, and Claude advises.
- **Explain before changing.** Before any change (code, config, docs, installs, git operations), explain *what* will change, *why*, and *how*. Then wait for Raz's go-ahead.
- **Recommend, don't decide.** When there's a choice, lay out the options with their trade-offs and give a recommendation. Raz picks the direction.
- **Teach new things before using them.** Explain any new concept, tool or library (e.g. `hypothesis`) before it shows up in the code.
- **Small steps.** Make one decision or one change at a time. Don't bundle unrelated changes or run ahead to the next step.
- **Log decisions immediately.** As soon as Raz decides something, record it in the Decisions log and update every affected section here and in README.md. Don't wait for the end of the session.
- **Pins.** When Raz says "pin", add the item to `docs/pinned-questions.md` (things to decide later).

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
- **Transition**: what each light does when the phase changes. A light that stops goes green → yellow → red. A light that starts goes red+yellow → green, but only after the intergreen time from every conflicting light has passed. A light that is green in both the old and new phase stays green. No flashing green (see **Signal timing**).
- **Conflict matrix**: yes/no for each pair of lights: may they ever be green together? Symmetric. It comes from geometry (the conflict check below).
- **Intergreen time** (זמן בין-ירוקים): the minimum time from the end of one light's green to the start of a *conflicting* light's green. It's directional: A → B can differ from B → A.
- **Intergreen matrix**: the intergreen time for every ordered pair of conflicting lights. The pairs it contains are exactly the conflict matrix.
- The Israeli guidelines call a light מופע (signal group) and a phase שלב (stage).
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
- **Neighbours in the ring share one light**, which stays green through the switch. For example, P1 → P3: `N-thru` keeps flowing and `S-thru` goes yellow → red. Once the intergreen time `S-thru` → `N-left` has passed, `N-left` goes red+yellow → green.
- **Across the ring** (P1↔P2, P3↔P4) and **between roads**, no light is shared, so it's a full switch.

### Assumptions
1. Merging into the same exit road counts as a conflict. This is the standard, safe rule.
2. No pedestrians. They're an optional extension, out of scope for now.
3. U-turns are allowed from the left lane and are part of the simulated traffic. They don't change the lights or the phases.

### Signal timing (official Israeli guidelines)
Source: Ministry of Transport, *הנחיות לתכנון רמזורים* (June 2025), [gov.il PDF](https://www.gov.il/BlobFolder/guide/planning_traffic_lights/he/%D7%94%D7%A0%D7%97%D7%99%D7%95%D7%AA%20%D7%9C%D7%AA%D7%9B%D7%A0%D7%95%D7%9F%20%D7%A8%D7%9E%D7%96%D7%95%D7%A8%D7%99%D7%9D.pdf). Local copy: `docs/references/` (PDF page numbers below).

| Item | Official rule | Where |
|---|---|---|
| Vehicle light sequence | red → red+yellow → green → [flashing green] → yellow → red | §6.2, p. 178 |
| Red+yellow | 2 s | §6.2 |
| Yellow | 3 s | §6.2 |
| Minimum red | 3 s | §6.2 |
| Flashing green | 3 s, only for lights on roads with a speed limit of 60 km/h or more. It counts toward minimum green. | §4.3, p. 101 |
| Minimum green | 6 s on urban streets. 10 s for the main road at urban-arterial or non-urban junctions. | §4.3, table 4.1 |
| Intergreen time | Calculated for every pair of conflicting movements: T = max(3, ⌈T2 − T3⌉). T2 = the clearing car's time to clear the conflict point, T3 = the entering car's time to reach it. Between two lights, take the largest value over their movements. Round up to whole seconds. | §5, pp. 148–174 |
| Intergreen matrix | Covers every conflicting pair. Each light appears once as clearing (מפנה) and once as entering (נכנס). | §5.7, p. 174 |

## Architecture
```
camera video ──► perception ──► JunctionState ──► controller ──► safety guard ──► phase
                 (detect, track,   (list of cars:     (fixed-time /     (min green,
                  map to lanes)     lane, position,    actuated /        yellow, red+yellow,
                                    speed, wait;       max-pressure /    intergreen matrix)
                                    + signal state)    RL)
SUMO simulator ─────────────────► JunctionState  (used for training and evaluation)
```
- `JunctionState` (in `interface/`) is the **only** contract between perception and control. Controllers never see pixels, and perception never knows which controller is running.
- **`JunctionState` contents** (decided 2026-11-05): a list of cars + the signal state + the clock time `t`.
  - Each car: `id` (track ID), `lane` (the lane it's in, or came from if it's inside the junction), `location` (approach zone / inside the junction), `distance_m` (to the stop line, approach zone only), `speed_mps`, `wait_s` (total time stopped while in view), `vehicle_type`.
  - Per-lane counts such as `queue` and `approaching` (moving towards the stop line) are helper functions computed from the list.
  - Signal state comes from the safety guard, never from perception: the current phase, each light's state and time in that state, time in phase, and whether a transition is in progress.
  - SUMO and perception both produce the same raw car list, and one shared builder turns it into `JunctionState`, so every definition is identical in sim and on video. It must stay camera-observable: only cars inside the observation zone, no routes or turn intentions, and no SUMO-only data.
  - Metrics come from SUMO's ground truth, never from `JunctionState`.
- Every controller implements the same `Controller` interface (`controllers/base.py`).

## Non-negotiables
1. **Safety is enforced outside the learned policy.** `safety/phase_guard.py` wraps every controller, and the RL agent can only pick from safe actions.
2. **No result without baselines.** Every controller is compared against fixed-time, actuated and max-pressure.
3. **Honest evaluation.** Use held-out traffic scenarios, 10+ seeds, and report confidence intervals. Never tune on the evaluation scenarios.
4. **Reproducible.** Every number in the README comes from a single documented command. Seed everything. Configs live in YAML, not hard-coded.

## Metrics
Average wait per vehicle, 95th-percentile wait (fairness), throughput, average queue length.

## Tech stack
Python 3.11 via `uv` (`pyproject.toml` + `uv.lock`) · SUMO + TraCI/libsumo (our own sim loop, no `sumo-rl`) · PyYAML (`safe_load` only) · Gymnasium · Stable-Baselines3 (DQN, PPO) · PyTorch · TensorBoard/W&B · Ultralytics YOLO · ByteTrack via `supervision` · OpenCV · pytest + `hypothesis` (property-based tests for the safety guard) · Docker · GitHub Actions · Streamlit

## Repo layout (planned)
An installable package in the src layout. Imports look like `from smart_traffic_lights.safety import phase_guard`.
```
src/smart_traffic_lights/
  sim/          SUMO network + seeded route generators (low/med/high/rush-hour/asymmetric)
  controllers/  base.py, fixed_time.py, actuated.py, max_pressure.py, rl_agent.py
  safety/       phase_guard.py
  rl/           env wrapper, state/reward definitions, train.py, configs/
  perception/   detector.py, tracker.py, lane_zones.py, undistort.py
  interface/    JunctionState dataclass
  eval/         benchmark.py, plots, stats
  demo/         side-by-side video renderer, Streamlit app
tests/
docs/           pinned-questions.md, references/ (official PDFs, local only, not in git)
pyproject.toml  project settings + dependencies   (uv.lock pins exact versions; .python-version = 3.11)
```

## Commands
_To be filled in as they exist._
- Setup: `uv sync` (creates `.venv/` from `uv.lock`)
- Run anything in the project environment: `uv run <command>`
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
- 2026-11-05: Reviewed the open questions. Decided: our own sim loop, an intergreen matrix, and the `JunctionState` contents. Read the official Israeli signal guidelines (see **Signal timing**). Installed `uv` 0.13.7 via Homebrew. Set up the uv project (`uv init --package`, Python 3.11.18). No code yet.
- **Next step:** install SUMO (how is not decided yet).

## Open questions (M1)
- **Intergreen values.** What uniform value to start with? Later, per-pair values could be calculated with the official formula from our SUMO geometry. Also unclear: does red+yellow fall inside the intergreen time or after it?
- **Speed limit.** Assume 50 km/h (a typical urban street)? Under the guidelines that means no flashing green, which matches our earlier decision for a better reason.
- **`JunctionState` details:** the speed below which a car counts as stopped, the observation-zone length, and the virtual detector distance.

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
- 2026-11-05: Our own sim loop over TraCI, not `sumo-rl`. sumo-rl's source confirms it has no all-red or red+yellow, uses one timer for the whole junction, takes over SUMO's light program, and would run the baselines under different transition rules than RL. We code against the API that traci and libsumo share: libsumo for fast training, traci + `sumo-gui` for debugging.
- 2026-11-05: The guard uses an **intergreen matrix** in YAML (directional, one entry per ordered pair of conflicting lights), following the Israeli guidelines. It starts with one uniform value, and per-pair values can be filled in later.
- 2026-11-05: `JunctionState` = a list of cars + the signal state + time (see Architecture). Raz's reasoning: keeping each car's wait allows more flexible rewards, such as fairness terms. Cars inside the junction are included. Scenario name, seed and time of day stay out of the RL observation.
- 2026-11-05: The conflict matrix isn't a separate table. It's the set of pairs in the YAML intergreen matrix. A test recomputes conflicts from the lane layout (the edge-walk method from 2026-11-04) and checks they match that set exactly. It also checks that every phase in every phase set is conflict-free.
- 2026-11-05: The official reference PDFs live in `docs/references/` but stay out of git (size, and redistribution of Ministry documents). CLAUDE.md and the README link to the sources.
- 2026-11-05: The code is an installable package in the src layout (`src/smart_traffic_lights/`), so tests and scripts import it the same way and our generic module names (`sim`, `eval`, `interface`) can't clash with other libraries.
- 2026-11-05: uv manages Python 3.11 (pinned in `.python-version`) instead of the Mac's system Python, so CI and Docker get the same version.
- 2026-11-05: YAML is read with PyYAML `safe_load`, never `load`, which can build arbitrary Python objects.
- 2026-11-05: Work style: log decisions immediately, not at the end of the session. "Pin" → `docs/pinned-questions.md`.
