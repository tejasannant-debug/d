# CLAUDE.md — project rules for BVRGym game + RL work

## What this is
Human-vs-AI BVR air-combat practice game on top of BVRGym (xcwoid/BVRGym, GPL-3.0).
Human flies one F-16 in real time vs an RL F-16; after every fight the AI runs a short
update round and is only accepted if it passes a fixed benchmark. 1v1 only for now; keep modules
separable (agent, radar, opponent, env, game, training, update loop) for later 1v2 / 1vMany.

## Hard rules
- Read the code before changing it. Trust code over the paper (arXiv 2403.17533) and over prompts.
- Never modify JSBSim model files: `aircraft/`, `engine/`, `systems/`, `scripts/*.xml`.
- Extend/fix in place; no rewrites of the autopilot or BVRGym core.
- New parameters go in config files following the 3-level pattern:
  `jsb_gym/simObjects/config` (L1) -> `jsb_gym/agents/config` (L2) -> `jsb_gym/envs/config` (L3).
- No magic numbers, fixed seeds, one change per experiment.
- A step is done only when its tests PASS; then update README.md, commit `stepN: ...`, and STOP for approval.
- Out of scope: multi-agent, jamming/chaff, cockpit/VR/joystick, attention networks.

## Commands (run from repo root, conda env `bvr`)
```bash
conda activate bvr
# upstream tests (must keep passing)
python -m tests.test_agent
python -m tests.test_aircraft      # streams to FlightGear UDP 5550 if running
python -m tests.test_missile
python -m tests.test_tacview
python -m tests.test_BTvsBT
python -m tests.test_wvr
# training
python main.py
```

## Facts verified in code (upstream 05e2189)
- 1 env step = 0.5 s sim (F16 Sim_time_step=6 x Control_time_step=10 JSBSim runs at 120 Hz).
- Obs (40, 15): rolling history of 15 features, see README "Observation".
- Action (3,) in [-1,1] -> heading 0..360, altitude 3000..12000 m, throttle 0.49 / 0.69.
- BT agents always use throttle 0.49.
