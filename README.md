---
title: OpenBeni
emoji: 🏁
colorFrom: blue
colorTo: purple
sdk: docker
app_port: 7860
license: mit
---

# OpenBeni

## SDK 文档

- [MONDO SDK使用简介](<open-beni-sdk-docs/MONDO SDK使用简介.md>)
- [Beni SDK 开放协议手册（HTML）](open-beni-sdk-docs/protocol-reference.html)
- [MONDO SDK User Guide (English)](<open-beni-sdk-docs/MONDO SDK User Guide.en.md>)
- [SDK Protocol Reference (English)](open-beni-sdk-docs/protocol-reference.en.html)
- [HMS Fault Code Reference (English)](open-beni-sdk-docs/hms-fault-codes.en.html)

Canonical source: [Mondo-Robotics/open-beni](https://github.com/Mondo-Robotics/open-beni). Development and pull requests belong in this repository; the Hugging Face Space is a deployment of this source.

Play the browser game at [Hugging Face Spaces](https://huggingface.co/spaces/mondo-robotics/open-beni).
This repository contains the complete web game, server, training source, robot assets and policies under the MIT license.
See [the web README](web/README.md) for local development, loading checks and deployment.

Build and run the game from source with Docker:

```bash
docker build -t open-beni .
docker run --rm -p 7860:7860 open-beni
```

Open `http://localhost:7860`. Hugging Face proxies port 7860 to the Space's HTTPS URL; players do not enter a port.
Set `MAX_ROOMS=0` to disable multiplayer; the default is 10 rooms.
For a build without a runtime CDN, pass `--build-arg VITE_ASSET_BASE_URL=`.

# BENI mjlab

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Flat-terrain velocity tracking, recovery and jumping for the BENI robot, built on
[mjlab](https://github.com/mujocolab/mjlab) and MuJoCo Warp.

## Recommended Environment

- Linux x86_64.
- Python **3.12**.
- An NVIDIA GPU (at least 8 GB memory) with a driver compatible with CUDA 12.8 for training.
  CPU execution is available for small runs.
- [uv](https://docs.astral.sh/uv/) for Python and dependency management.

## Getting Started

Install dependencies from the repository root:

```bash
uv sync --locked
```

## Training

Train the BENI robot for 15,000 PPO iterations:

```bash
uv run python -m beni_rl.train Beni-Jump-Flat --env.scene.num-envs 4096 --agent.max-iterations 15000 --agent.seed 42
```

Run a small training check that automatically uses CPU when CUDA is unavailable:

```bash
uv run python -m beni_rl.train Beni-Jump-Flat \
  --env.scene.num-envs 4 --agent.max-iterations 10
```

CPU runs use the same MuJoCo Warp backend and are slower than GPU training.

Checkpoints and TensorBoard logs are saved under `logs/rsl_rl/beni_jump_flat/`.

Inspect all training options with `uv run python -m beni_rl.train Beni-Jump-Flat --help`.

### Multi-GPU training

`python -m beni_rl.train` automatically uses all visible CUDA devices, up to eight.
`--gpu-ids '[0,1]'` selects indices within `CUDA_VISIBLE_DEVICES`; `--gpu-ids None`
selects CPU.

### Resume train

Resume a checkpoint with additional PPO updates:

```bash
uv run --locked python -m beni_rl.train Beni-Jump-Flat \
  --agent.resume True --agent.load-run '<run>' --agent.load-checkpoint model_4000.pt \
  --agent.max-iterations 3000
```

Resume restores the stage, sampling progress, difficult-state membership and
learning-rate ceiling.

## Evaluation

Play the policy in the browser using Viser, from the repository root:

```bash
uv run play Beni-Jump-Flat --checkpoint-file "assets/models/policy.pt" --num-envs 1 --viewer viser
```

Once Viser is ready, play automatically opens the default browser in a new window
(or a tab, depending on browser settings). The default address is
`http://localhost:8080`; if that port is occupied, the browser opens the port Viser
actually selects. If no browser is available, use the URL printed in the terminal.

### Controlling the robot in Viser

Open **Controls → Commands → Twist** and check **Enable** to control the selected
robot with the sliders. When unchecked, commands are sampled automatically.

| Control | Range | Requested behavior |
| --- | --- | --- |
| `lin_vel_x` | −2 to 2 m/s | Forward for positive values, backward for negative values. |
| `lin_vel_y` | Fixed at 0 | Lateral motion is disabled. |
| `ang_vel_z` | −5 to 5 rad/s | Left turn for positive values, right turn for negative values, viewed from above. |
| `base_pitch (rad)` | −0.1 to 0.1 rad | Requested forward/backward body tilt (pitch). |
| `base_roll (rad)` | −0.1 to 0.1 rad | Requested left/right body tilt (roll). |
| `base_height (m)` | 0.10 to 0.12 m | Requested body height; larger values request a taller stance, with 0.11 m at the midpoint. |

For example, set `lin_vel_x = 0.5` and `ang_vel_z = 0` to drive forward, or set
`lin_vel_x = 0` and `ang_vel_z = 1` to request a left turn in place. Set both to
zero to request standing still; keep **Enable** checked. Combining forward speed
and yaw rate requests a curved path. These are policy commands, so the actual
motion and pose depend on the trained policy's tracking ability.

Open **Controls → Commands → Jump** (the folder shows the fixed reference height)
and click **Request one jump** to trigger a single jump for the selected robot.
Jumps are triggered manually during play. To try a standing jump, keep Twist
**Enable** checked, set both velocity commands and body tilt to zero, and set
`base_height (m) = 0.11`. Wait for **Measured state** to show `ready=True` before
clicking. You can also request a jump while driving forward once the robot is
balanced. Training, play and the BENI web profile use a fixed 0.18 m jump reference,
measured as base rise above the nominal standing height, with a matching height
command. Checkpoint settings and uploaded model fingerprints do not change this
reference; actual wheel clearance depends on the trained policy. The original
trajectory file is retained as an archive; earlier policies require their matching
source revision to reproduce their behavior.

The robot must meet the balance and wheel-support conditions continuously for
0.2 seconds before a request can be accepted. **Last request** shows `accepted`
or a rejection reason: `not ready`, `cooldown`, `busy` or `disabled`. Before
requesting another jump, let the robot recover its balance and wait for both
the 1-second cooldown and the previous jump's landing monitoring to finish.
`busy` can remain visible for several seconds after landing; wait and click
again to submit a new request.

Use **Simulation → Pause** to freeze the simulation, or **Reset Environment** to
reset the robot after a fall. Jump requests made while paused are processed
after resuming the simulation.

### Other play options

Play a trained policy, replacing `<run>` with a saved checkpoint:

```bash
uv run play Beni-Jump-Flat \
  --checkpoint-file "logs/rsl_rl/beni_jump_flat/<run>/model_best.pt" \
  --num-envs 1 \
  --viewer viser
```

To inspect the robot without a checkpoint:

```bash
uv run play Beni-Jump-Flat \
  --agent zero \
  --num-envs 1 \
  --viewer viser
```

## Export and deployment (ONNX + MNN)

Export a checkpoint on CPU:

```bash
uv run --locked --group export python -m beni_rl.headless_export \
  --run_dir "logs/rsl_rl/beni_jump_flat/<run>" \
  --checkpoint model_best.pt
```

Omit `--checkpoint` to select the highest-numbered `model_*.pt` in the run.

The command performs `.pt → ONNX → MNN` and writes both deployment artifacts
under `<checkpoint-dir>/exported/<checkpoint-stem>/`:

| File | Deployment use |
| --- | --- |
| `policy.onnx` | Policy for ONNX Runtime and the browser viewer. |
| `policy.mnn` | Policy for MNN deployed on the robot. |

## Browser game

The browser MuJoCo + ONNX game lives in [`web/`](web/README.md). Build from a
full repository checkout with Node.js 22.13+ and Python 3.9+ available as
`python3`; no training dependencies are needed.

```bash
cd web
npm ci
npm run build
npm start
```

Open `http://localhost:3000/`. The Node server listens on `0.0.0.0:3000` and
enables Multiplayer by default; use `npm run dev` for development. Serving
`web/dist/` alone supports Single Player only. Public Multiplayer deployment
needs the Node server on the same origin.

## Repository structure

Key files and directories:

```text
.
├── assets/
│   ├── beni_base/
│   │   ├── meshes/                                  # Robot visual meshes (OBJ)
│   │   ├── mjcf/beni.xml                            # MuJoCo robot model
│   │   ├── textures/                                # Robot textures
│   │   └── urdf/beni.urdf                           # URDF robot model
│   ├── models/
│   │   ├── policy.pt                                # Checkpoint for play
│   │   ├── policy.onnx                              # ONNX policy
│   │   ├── policy.mnn                               # MNN policy
│   │   ├── policy_best.pt                           # Local best checkpoint
│   │   └── policy_best.onnx                         # Local best ONNX policy
│   └── motions/
│       ├── base_jump_h0.18.npz                      # Current Jump reference trajectory
│       ├── base_jump_h0.25.npz                      # Retained original Jump reference
│       ├── recovery_initial_states_train.npz        # Original recovery library
│       ├── recovery_contact_states_train.npz        # Contact-stratified training states
│       └── recovery_contact_states_heldout.npz      # Independent recovery assessment states
├── python/beni_rl/
│   ├── __init__.py                                  # Task registration
│   ├── _viser_logging.py                            # Viser WebSocket log filtering
│   ├── actuator.py                                  # Motor and passive spring models
│   ├── beni_jump_env_cfg.py                         # Jump training and play environment
│   ├── beni_robot.py                                # Robot and actuator configuration
│   ├── beni_velocity_flat_env_cfg.py                # Velocity tracking environment
│   ├── distributed.py                               # Shared multi-GPU normalization
│   ├── headless_export.py                           # ONNX/MNN exporter
│   ├── jump.py                                      # Jump commands, measurements and rewards
│   ├── jump_evaluation.py                           # Jump trials for best-checkpoint selection
│   ├── mdp.py                                       # Shared observations, rewards and events
│   ├── play.py                                      # Play entry point and browser launch
│   ├── recovery.py                                  # Recovery rewards and state restoration
│   ├── recovery_bank.py                             # Bank validation and physical strata
│   ├── recovery_curriculum.py                       # Recovery sampling and curriculum
│   ├── recovery_evaluation.py                       # Independent locomotion checks
│   ├── recovery_gate.py                             # Recovery checks before Jump training
│   ├── recovery_sampling.py                         # Adaptive sampling of difficult recovery states
│   ├── rl_cfg.py                                    # PPO configuration and training runner
│   ├── train.py                                     # Training entry point
│   ├── velocity_command.py                          # Velocity, pose and gimbal commands
│   └── wheel_safety.py                              # Wheel overspeed termination and diagnostics
├── web/
│   ├── app/                                         # Browser UI
│   ├── client/                                      # Game session and input handling
│   ├── components/                                  # UI components
│   ├── game/                                        # Browser simulation and rendering
│   ├── hooks/                                       # React hooks
│   ├── lib/                                         # Shared UI utilities
│   ├── public/                                      # Static config and generated runtime assets
│   ├── scripts/                                     # Asset and runtime preparation
│   ├── server/                                      # Multiplayer server
│   ├── shared/                                      # Client/server race rules and protocol
│   ├── tools/                                       # Demo recorder and verification tools
│   └── README.md                                    # Web development instructions
├── pyproject.toml                                   # Python package and dependencies
├── uv.lock                                          # Locked Python dependencies
├── LICENSE
└── README.md
```

## License

OpenBENI is licensed under the [MIT License](LICENSE).
Dependencies, including [mjlab](https://github.com/mujocolab/mjlab/blob/main/LICENSE),
retain their own licenses.

## Acknowledgments

Built with [mjlab](https://github.com/mujocolab/mjlab) and
[MuJoCo Warp](https://github.com/google-deepmind/mujoco_warp).
