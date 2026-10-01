# BVR Practice Game (human vs self-improving RL F-16)

> DRAFT skeleton (Step 0). Sections marked _TODO_ are filled in as each step lands.

## 1. Overview
A beyond-visual-range (BVR) air-combat practice game. A human pilot flies one F-16 in real time
against an RL-trained F-16. After every fight the AI runs a short background update round and is
only accepted if it does at least as well on a fixed benchmark (it never gets worse).
Goal (professor's requirement): "a self-fighting game/simulation so a fighter pilot can practise on it".
Context: DRDO research internship + college project.

FlightGear is only a 3D viewer. Physics = JSBSim. Combat rules (missiles, radar, kills) = Python.

### Status
| Step | What | Status |
|---|---|---|
| 0 | Repo audit vs upstream, run all tests | in progress |
| 1 | Fix turn/altitude coupling (B1) + control test matrix | planned |
| 2 | Fix reward (B2), missile damage (B3), launch speed (B4), end_reason | planned |
| 3 | Scenarios, recording, metrics | planned |
| 4 | Radar, lock, RWR | planned |
| 5 | Game mode v1 (real time, panel, launcher, debrief) | planned |
| 6 | BT opponent styles + fixed benchmark | planned |
| 7 | RL env wrapper, checks, random baseline, PPO smoke test | planned |
| 8 | Base PPO training + evaluation | planned |
| 9 | PPO opponent in game mode | planned |
| 10 | Human-clone model | planned |
| 11 | Per-flight update loop + safety check | planned |
| 12 | End-to-end: 10 human fights | planned |

## 2. Architecture
```
human (keys) / AI policy
      | heading, altitude, throttle, fire
      v
PID autopilot (jsb_gym/utils/autopilot.py) -> aileron / elevator / rudder
      v
JSBSim F-16 (Level 1 simObjects) <-> missiles (AIM-120 model) <-> radar/RWR (planned)
      v
FlightGear 3D (UDP 5550/5551) + 2D pygame panel (planned)
      v
recording (obs/actions/events/Tacview) -> per-flight update loop (planned)
```
BVRGym levels: **L1** `jsb_gym/simObjects` (aircraft, missile) -> **L2** `jsb_gym/agents`
(aircraft + missiles, BT/RL agents) -> **L3** `jsb_gym/envs` (Gymnasium envs). Each level has a `config/` folder.

## 3. Folder structure
| Path | What | Ours? |
|---|---|---|
| `aircraft/`, `engine/`, `systems/`, `scripts/` | JSBSim model + init scripts (do not modify) | upstream |
| `data_classes/` | dataclasses for agent/aircraft/missile configs | upstream |
| `data_output/flightgear*.xml` | JSBSim -> FlightGear UDP output (5550 Blue, 5551 Red) | upstream |
| `data_output/tacview/` | Tacview CSV logs | upstream |
| `fg/ogel/` | missile model for FlightGear (copy to `~/.fgfs/Aircraft/`) | upstream |
| `jsb_gym/simObjects/` | L1: `F16BVR`, `AAMBVR`, `FDMObject` | upstream |
| `jsb_gym/agents/` | L2: `BaseBVRAgent`, `BTBVRAgent`, `RLBVRAgent` | upstream |
| `jsb_gym/agents/player_agent.py` | human wrapper | ours (to import) |
| `jsb_gym/envs/` | L3: `BVRBase`, `BVREnv_BTvsBT`, `WvrEnv` | upstream |
| `jsb_gym/bts/` | behavior trees (BVR + WVR) | upstream |
| `jsb_gym/utils/` | autopilot, PID, geometry, Tacview logger | upstream (autopilot modified by us) |
| `tests/test_*.py` (upstream 6) | aircraft, missile, agent, tacview, BTvsBT, wvr | upstream |
| `tests/player_controller.py`, `test_player_vs_ai.py`, `test_*_response.py`, `test_turn_altitude_hold.py` | our control + player tests | ours (to import) |
| `main.py` | PPO training entry point | upstream |

## 4. Installation (Ubuntu 22.04)
_TODO (Step 0 completion): system packages, Miniconda, `conda env create -f environment.yml`,
FlightGear PPA `ppa:saiarcot895/flightgear`, F-16 aircraft download, `cp -r fg/ogel ~/.fgfs/Aircraft/`,
Tacview, verify script, common errors._

## 5. How to run
Upstream tests (from repo root):
```bash
python -m tests.test_agent && python -m tests.test_aircraft && python -m tests.test_missile
python -m tests.test_tacview && python -m tests.test_BTvsBT && python -m tests.test_wvr
```
FlightGear viewers (our setup; upstream uses `--aircraft=f16-block-52 --fdm=null`):
```bash
fgfs --fdm=external --native-fdm=socket,in,60,,5550,udp --aircraft=f16-block-32 --airport=ESSA --multiplay=out,10,127.0.0.1,5000 --multiplay=in,10,127.0.0.1,5001
fgfs --fdm=external --native-fdm=socket,in,60,,5551,udp --aircraft=f16-block-32 --airport=ESSA --multiplay=out,10,127.0.0.1,5002 --multiplay=in,10,127.0.0.1,5003
```
_TODO: game mode, controls table, training, TensorBoard, evaluation, update loop, Tacview debrief._

## 6. How it works
### Observation (verified, upstream `BaseEnv.from_obs2nn`)
Shape (40, 15) = last 40 steps x 15 features (all from Blue's view, **perfect information** today):
`sin/cos bearing to enemy, sin/cos own heading, own mach, own altitude, distance (0..120 km),
sin/cos enemy's bearing to us, sin/cos enemy heading, enemy mach, enemy altitude,
own_missile_active (always 0), enemy_missile_active (always 0)`.
### Action (verified)
3 values in [-1, 1] -> heading 0..360 deg, altitude 3000..12000 m, throttle 0.49 (<=0) or 0.69 (>0, afterburner).
Firing is **not** an action today: `RLBVRAgent` auto-fires when < 60 km and within ±40°.
### Reward (verified)
Sparse terminal: +1 if Red dies, otherwise -1 (bug B2: Blue death is never checked separately).
_TODO: radar, opponents, human-clone, update loop, benchmark._

## 7. Configuration reference
_TODO: every config file + parameter._

## 8. Experiment log
### Step 0 (cloud audit of upstream 05e2189; jsbsim 1.3.1, gymnasium 1.3.0, py_trees 2.6.0, pandas 3.0.6, Python 3.11)
| Test | Result | Time |
|---|---|---|
| tests.test_agent | PASS (4 missiles loaded) | <1 s |
| tests.test_aircraft | PASS (runs; no FG listening) | 54 s |
| tests.test_missile | PASS (runs) | 14 s |
| tests.test_tacview | PASS (ended by 600 s timeout) | 8 s |
| tests.test_BTvsBT | PASS (ended by 1200 s timeout) | 15 s |
| tests.test_wvr | PASS | 3 s |

Upstream autopilot baseline (hold 7000 m, command turn at t=5 s, back to 0 at t=35 s, 300 m/s):
45° turn: alt 6988–7233 m, heading settles at 38.8° (≈6° error); 90°: 6790–7237 m; 180°: drops to 4885 m.

## 9. Known limitations / future work
Simplified radar and missiles, 1v1 only, no EW. Future: 1v2, 1vMany, WVR opponents
(Control Zone, AggressiveShooter, ConservativeShooter already exist in `jsb_gym/envs/WvrEnv.py`).

## Credits & license
Based on **BVRGym by xcwoid**: https://github.com/xcwoid/BVRGym — GPL-3.0 (this project stays GPL-3.0).
Paper: *BVR Gym: A Reinforcement Learning Environment for Beyond-Visual-Range Air Combat*, arXiv:2403.17533
(describes an older version; the code is the reference).
