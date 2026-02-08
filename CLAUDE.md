# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

XLeRobot is a low-cost dual-arm mobile household robot platform built on top of the [LeRobot](https://github.com/huggingface/lerobot) framework. It features dual SO100/SO101 arms (6 DOF each), a 2-motor head, and either a 3-wheel omnidirectional or 2-wheel differential drive base. Hardware uses Feetech STS3215 servo motors connected via two USB serial buses.

## Installation & Setup

XLeRobot's software integrates into an existing LeRobot installation:

1. Install LeRobot: `pip install -e .` from the LeRobot repo (see [official guide](https://huggingface.co/docs/lerobot/installation))
2. Copy `software/src/model/` into LeRobot's `/model` folder
3. Copy `software/src/robots/xlerobot/` (or `xlerobot_2wheels/`) into LeRobot's `/robots` folder
4. Copy `software/examples/` into LeRobot's `/example` folder
5. For Raspberry Pi (host/client mode), uncomment `xlerobot_host` and `xlerobot_client` imports in `__init__.py`

## Running the Software

All Python scripts run from within the LeRobot environment with `PYTHONPATH=src`:

```bash
# Start the robot host (for remote/Pi operation)
PYTHONPATH=src python -m lerobot.robots.xlerobot.xlerobot_host --robot.id=my_xlerobot

# Keyboard teleoperation (primary example)
PYTHONPATH=src python -m examples.xlerobot.teleoperate_Keyboard

# Direct local example execution
PYTHONPATH=src python software/examples/4_xlerobot_teleop_keyboard.py
```

### Web Control

```bash
# Backend
cd web_control/server && pip install -r requirements.txt && uvicorn main:app

# Frontend
cd web_control/client && npm install && npm run dev
```

### Simulation (MuJoCo)

```bash
cd simulation/mujoco && pip install -r requirements.txt && python xlerobot_mujoco.py
```

### Documentation

```bash
cd docs && pip install -r requirements.txt && make html
```

## Architecture

### Dual Motor Bus Design

The robot uses two Feetech motor buses over USB serial:
- **Bus 1** (`/dev/ttyACM0`): Left arm (6 motors) + head (2 motors) = 8 motors
- **Bus 2** (`/dev/ttyACM1`): Right arm (6 motors) + wheels (3 omni or 2 differential) = 8-9 motors

### Robot Variants

| Variant | Class | Config | Base Type |
|---------|-------|--------|-----------|
| 3-wheel omni | `XLerobot` | `XLerobotConfig` | 3 omniwheels at 120deg |
| 2-wheel diff | `XLerobot2Wheels` | `XLerobot2WheelsConfig` | Differential drive |

Both inherit from LeRobot's `Robot` base class and share the same arm/head architecture.

### Key Source Paths

- `software/src/robots/xlerobot/xlerobot.py` - Main 3-wheel robot class
- `software/src/robots/xlerobot/config_xlerobot.py` - Config dataclass (ports, keymaps, cameras)
- `software/src/robots/xlerobot_2wheels/` - 2-wheel variant (same structure)
- `software/src/model/SO101Robot.py` - Analytical IK solver for SO101 arms (2-link, law of cosines)
- `software/src/teleporators/xlerobot_vr/` - Quest 3 VR teleoperation (note: directory typo "teleporators")
- `software/src/record.py` - Dataset recording from teleoperation sessions
- `software/joyconrobotics/` - Nintendo JoyCon Bluetooth controller driver

### Example Scripts (numbered by complexity)

0-3: Single arm control (joint, EE with IK, dual arm, YOLO vision)
4: Full XLeRobot keyboard teleop (dual arms + head + base) - **primary example**
5: Xbox controller teleop
6-7: JoyCon teleop (single arm and full robot)
8: VR teleop (Quest 3)

### Control Flow

Scripts follow a common pattern:
1. Create `XLerobotConfig` / `XLerobot` (or Client variant for remote)
2. `robot.connect()` initializes motor buses and cameras
3. Main loop: read input -> compute IK/actions -> `robot.send_action(action_dict)` -> `robot.get_observation()`
4. Actions are dicts mapping `"joint_name.pos"` keys to target positions
5. P-control with configurable `kp` gain smooths commanded positions
6. `robot.disconnect()` releases motors on exit

### Motor Normalization Modes

- `DEGREES` - Raw joint angles in degrees
- `RANGE_M100_100` - Normalized to [-100, 100] (default for arms)
- `RANGE_0_100` - [0, 100] range (used for grippers)

### Network Architecture (Host/Client)

For Raspberry Pi deployments, the robot runs in host/client mode over ZMQ:
- Host (on Pi): `XLerobotHost` at ports 5555 (commands) and 5556 (observations)
- Client (on laptop): `XLerobotClient` connects to host IP
- Watchdog timeout: 500ms (stops robot if no commands received)

### Web Control Stack

- **Backend**: FastAPI + Socket.IO (`web_control/server/`), rate-limited at 20 cmd/sec
- **Frontend**: React 19 + TypeScript + Tailwind + Vite (`web_control/client/`)
- Socket.IO events: `move_command`, `start_video_stream`, `camera_actions`

### VR Teleoperation (XLeVR)

- Uses Meta Quest 3 via WebSocket connection
- Config in `XLeVR/config.yaml` (HTTPS port 8443, WS port 8442)
- Delta action control with IK mapping from hand tracking to arm joints

### Simulation

- **MuJoCo** (`simulation/mujoco/`): Direct physics sim with keyboard control via GLFW
- **ManiSkill** (`simulation/Maniskill/`): RL-compatible Gymnasium environment, registered agent `xlerobot_single`, default task `PushCube-v1`
- **Isaac Sim** (`simulation/Isaac_sim/`): Placeholder for future integration

## Keyboard Controls (Default Teleop)

**Base**: i/k (fwd/back), j/l (left/right), u/o (rotate), n/m (speed up/down), b (quit)
**Left arm**: q/e (shoulder pan), w/s (IK x), a/d (IK y), z/x (pitch), r/f (wrist roll), t/g (gripper), c (reset)
**Right arm**: Numpad 7/9 (shoulder pan), 8/2 (IK x), 4/6 (IK y), 1/3 (pitch), / /* (wrist roll), +/- (gripper), 0 (reset)
**Head**: < /> (motor 1), , /. (motor 2), ? (reset)
