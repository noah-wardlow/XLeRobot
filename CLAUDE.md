# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

XLeRobot is a low-cost dual-arm mobile household robot platform built on top of the [LeRobot](https://github.com/huggingface/lerobot) framework. It features dual SO100/SO101 arms (6 DOF each), a 2-motor head, and either a 3-wheel omnidirectional or 2-wheel differential drive base. Hardware uses Feetech STS3215 servo motors connected via two USB serial buses.

This is **not a standalone Python package** (no `pyproject.toml` or `setup.py`). Source files are manually copied into an existing LeRobot installation. There is no CI/CD pipeline, no automated test suite, and no Python linter config.

## Installation & Setup

1. Install LeRobot: `pip install -e .` from the LeRobot repo
2. Copy `software/src/model/` into LeRobot's `/model` folder
3. Copy `software/src/robots/xlerobot/` (or `xlerobot_2wheels/`) into LeRobot's `/robots` folder
4. Copy `software/examples/` into LeRobot's `/example` folder
5. For Raspberry Pi (host/client mode), uncomment `xlerobot_host` and `xlerobot_client` imports in `__init__.py`

## Build & Run Commands

### Robot Software (requires LeRobot environment)

```bash
# Keyboard teleoperation (primary example)
PYTHONPATH=src python software/examples/4_xlerobot_teleop_keyboard.py

# Start the robot host (for remote/Pi operation)
PYTHONPATH=src python -m lerobot.robots.xlerobot.xlerobot_host --robot.id=my_xlerobot
```

### Web Control

```bash
# Backend
cd web_control/server && pip install -r requirements.txt
uvicorn main:app                     # or: python main.py

# Frontend
cd web_control/client && npm install
npm run dev                          # Vite dev server
npm run build                        # tsc -b && vite build (production)
npm run lint                         # ESLint — must pass before PR for web changes
npm run preview                      # preview production build
```

Server is configured via `.env` file (see `.env.example`). Key var: `ROBOT_TYPE` (maniskill|mujoco|xlerobot).

### Simulation

```bash
# MuJoCo
pip install -r simulation/mujoco/requirements.txt
python simulation/mujoco/xlerobot_mujoco.py

# ManiSkill
python simulation/Maniskill/run_xlerobot_sim.py
```

### Documentation (Sphinx)

```bash
pip install -r docs/requirements.txt
make -C docs html                              # build
sphinx-autobuild docs/source docs/build/html   # live preview
```

### Testing / Validation

No test framework or coverage gate. Validate changes by running relevant example/smoke scripts. Two informal test files exist: `software/test_yolo.py` and `simulation/Maniskill/examples/test_xbox.py`. Name new test files `test_*.py`.

## Code Style

- **Python**: 4-space indentation, `snake_case` functions/modules, `PascalCase` classes. Follow existing patterns.
- **TypeScript/React** (`web_control/client/`): `PascalCase` components (e.g. `RobotVideoCanvas.tsx`), `use`-prefixed hooks (e.g. `useSocket.ts`). Run `npm run lint` before PRs.

## Architecture

### Dual Motor Bus Design

- **Bus 1** (`/dev/ttyACM0`): Left arm (motors 1-6, POSITION mode) + head (motors 7-8, POSITION mode)
- **Bus 2** (`/dev/ttyACM1`): Right arm (motors 1-6, POSITION mode) + wheels (motors 7-9 omni or 9-10 diff, VELOCITY mode)

Motor PID: arms/head → P=16, I=0, D=43 (position). Wheels → velocity mode, no PID tuning.

### Robot Variants

| Variant | Class | Config | Base Kinematics |
|---------|-------|--------|-----------------|
| 3-wheel omni | `XLerobot` | `XLerobotConfig` | 3 omniwheels at 240°/0°/120° matrix transform |
| 2-wheel diff | `XLerobot2Wheels` | `XLerobot2WheelsConfig` | `v = (x ± ω*L/2)/r`, wheel_radius=0.05, wheelbase=0.25 |

Both inherit from LeRobot's `Robot` base class.

### Key Source Files

- `software/src/robots/xlerobot/xlerobot.py` — Main 3-wheel robot class (`XLerobot`)
- `software/src/robots/xlerobot/config_xlerobot.py` — Config dataclass (ports, keymaps, cameras); also `XLerobotHostConfig`, `XLerobotClientConfig`
- `software/src/robots/xlerobot/xlerobot_host.py` — ZMQ host (PULL:5555 commands, PUSH:5556 observations, CONFLATE=1)
- `software/src/robots/xlerobot/xlerobot_client.py` — ZMQ client (`XLerobotClient(Robot)`)
- `software/src/robots/xlerobot_2wheels/` — 2-wheel differential variant (parallel structure)
- `software/src/model/SO101Robot.py` — Analytical IK solver (`SO101Kinematics`): 2-link, law of cosines, l1=0.1159m, l2=0.1350m
- `software/src/teleporators/xlerobot_vr/xlerobot_vr.py` — Quest 3 VR teleop (note: directory typo "teleporators")
- `software/src/record.py` — Dataset recording wrapper around LeRobot's recording
- `software/joyconrobotics/` — Nintendo JoyCon Bluetooth controller driver
- `software/examples/4_xlerobot_teleop_keyboard.py` — Primary teleop example with `SimpleTeleopArm`, `SimpleHeadControl`

### Control Flow Pattern

All teleop scripts follow this pattern:
1. Create config → create robot → `robot.connect()`
2. Main loop: read input → compute IK/actions → `robot.send_action(action_dict)` → `robot.get_observation()`
3. `robot.disconnect()` releases motors on exit

**Action dict format**: `{"joint_name.pos": float}` for positions, `{"x.vel": float, "y.vel": float, "theta.vel": float}` for base velocity.

**P-control**: Teleop scripts apply `action = current + kp * (target - current)` with `kp ≈ 0.81` at ~50Hz loop rate.

**IK wrist coupling**: `wrist_flex = -shoulder_lift - elbow_flex + pitch` (keeps end-effector pitch independent of arm pose).

### Motor Normalization Modes

- `DEGREES` — Raw joint angles in degrees
- `RANGE_M100_100` — Normalized to [-100, 100] (default for arms)
- `RANGE_0_100` — [0, 100] range (used for grippers)

### Network Architecture (Host/Client)

For Raspberry Pi deployments over ZMQ:
- Host (on Pi): `XLerobotHost` — ports 5555 (commands) and 5556 (observations), JSON with base64 JPEG-encoded camera images
- Client (on laptop): `XLerobotClient` — connects to host IP
- Watchdog timeout: 500ms (calls `robot.stop_base()` if no commands received)

### Web Control Stack

- **Backend** (`web_control/server/`): FastAPI + async python-socketio, rate-limited at 20 cmd/s (3 violations → 2s penalty). `RemoteCore` connects to robot host via async ZMQ.
- **Frontend** (`web_control/client/`): React 19 + TypeScript 5.8 + Tailwind 4 + Vite 7 + socket.io-client. `useSocket` hook manages connection, keyboard-to-direction mapping, and video streaming.
- REST endpoints: `/health`, `/robot/info`, `/robot/controllers`, `/robot/camera/*`
- Socket.IO events: `move_command`, `start_video_stream`, `stop_video_stream`, `camera_actions`

### VR Teleoperation (XLeVR)

- Meta Quest 3 via WebSocket (HTTPS:8443, WS:8442)
- Config in `XLeVR/config.yaml`
- Delta action control: tracks VR position change per frame, scales deltas (x:170, y:80, z:80), maps to IK with smoothing (alpha=0.1)
- VR event handler: left thumbstick → recording control (exit, rerecord, stop, reset/back position)

### Simulation

- **MuJoCo** (`simulation/mujoco/`): `XLeRobotController` loads `scene.xml` MJCF, renders at 60Hz via `mujoco_viewer`, P-control base (kp=10 linear, kp_rot=100), 0.005 rad/step arm increments
- **ManiSkill** (`simulation/Maniskill/`): `Xlerobot(BaseAgent)` registered as `"xlerobot"`, Gymnasium env with `PushCube-v1` task, full PD controller suite (joint pos/vel/delta + EE pos/pose/delta), 3 cameras (head 256×256, arm cameras 128×128)
- **Isaac Sim** (`simulation/Isaac_sim/`): Placeholder

### Example Scripts (numbered by complexity)

0-3: Single arm control (joint, EE with IK, dual arm, YOLO vision)
4: Full XLeRobot keyboard teleop (dual arms + head + base) — **primary example**
5: Xbox controller teleop
6-7: JoyCon teleop (single arm and full robot)
8: VR teleop (Quest 3)

## Keyboard Controls (Default Teleop)

**Base**: i/k (fwd/back), j/l (left/right), u/o (rotate), n/m (speed up/down), b (quit)
**Left arm**: q/e (shoulder pan), w/s (IK x), a/d (IK y), z/x (pitch), r/f (wrist roll), t/g (gripper), c (reset)
**Right arm**: Numpad 7/9 (shoulder pan), 8/2 (IK x), 4/6 (IK y), 1/3 (pitch), / /* (wrist roll), +/- (gripper), 0 (reset)
**Head**: < /> (motor 1), , /. (motor 2), ? (reset)
