# Repository Guidelines

## Project Structure & Module Organization
- `software/`: core robotics code and runnable demos.
- `software/src/robots/`: robot implementations (`xlerobot`, `xlerobot_2wheels`) and configs.
- `software/examples/`: teleop, vision, and control examples (numbered scripts like `4_xlerobot_teleop_keyboard.py`).
- `simulation/`: simulators and assets (`mujoco/`, `Maniskill/`, `Isaac_sim/`).
- `web_control/`: remote control stack with `server/` (FastAPI + Socket.IO) and `client/` (React + Vite + TS).
- `XLeVR/`: lightweight VR monitor tooling.
- `docs/`: Sphinx docs (`en/source`, `zh/source`), plus doc-generation scripts.
- `hardware/`, `others/`: CAD assets and supporting materials.

## Build, Test, and Development Commands
- Web server:
  - `cd web_control/server && python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
  - `python main.py` (runs API + Socket.IO)
- Web client:
  - `cd web_control/client && npm install`
  - `npm run dev` (local dev), `npm run build` (production bundle), `npm run lint` (ESLint)
- Docs:
  - `pip install -r docs/requirements.txt`
  - `make -C docs html` (build), `sphinx-autobuild docs/source docs/build/html` (live preview)
- MuJoCo sim smoke run:
  - `python simulation/mujoco/xlerobot_mujoco.py`

## Coding Style & Naming Conventions
- Python: 4-space indentation, `snake_case` for functions/modules, `PascalCase` for classes, prefer type hints in new code.
- TypeScript/React: components in `PascalCase` (e.g., `RobotVideoCanvas.tsx`), hooks prefixed with `use` (e.g., `useSocket.ts`).
- Follow existing patterns before introducing new abstractions.
- Run `npm run lint` for `web_control/client` changes before PR.

## Testing Guidelines
- No repository-wide enforced test framework or coverage gate currently.
- Required minimum: run relevant example/smoke scripts for your change area.
- Prefer naming Python tests `test_*.py` (existing pattern: `software/test_yolo.py`, `simulation/Maniskill/examples/test_xbox.py`).
- For web changes, lint must pass and the UI should be manually validated in browser.

## Commit & Pull Request Guidelines
- Use short, imperative commit subjects (history pattern: `Fix ...`, `Update ...`, `Refactor ...`).
- Keep PRs focused and small; split unrelated work.
- Before large work, discuss approach on the related issue.
- In PR description include:
  - what changed and why,
  - linked issue (`Fixes #123` when applicable),
  - validation steps run,
  - screenshots/video for UI or behavior changes.
