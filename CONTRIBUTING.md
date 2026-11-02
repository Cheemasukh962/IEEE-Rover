# Working on the rover together

## Claiming a deliverable

1. Pick a deliverable from [PHASES.md](PHASES.md) whose "Waits on" column is satisfied.
2. Tell the team (group chat or a GitHub issue) which number you're taking, so two people don't build the same thing.
3. Make a branch for it, build it, and open a PR.
4. It merges when someone **other than you** has run its checkpoint ("Done when") and it passes.

## Branches and PRs

- One branch per deliverable, named after it: `d1-skeleton`, `d3-detector`, `d8-firmware`.
- Never commit straight to `main`. Everything goes through a pull request.
- Keep PRs to one deliverable. Small PRs get reviewed faster.
- Pull `main` before you start and again before you open the PR.

## Shared interfaces

These are fixed in deliverable #1. Every file builds against them, which is what lets people work in parallel:

- `Detection(name, conf, box=(x1, y1, x2, y2), track_id)`
- `camera.latest() -> frame` (always the newest frame; old ones are dropped)
- `link.send(line)` / `link.receive() -> line or None` (the mock, serial and Wi-Fi links all share this)
- Commands to the Uno: `D <left> <right>` (wheel speeds, −100 to 100), `G <angle>` (0 to 180), `S` (stop), `P` (heartbeat)
- Replies from the Uno: `E FRONT` / `E REAR` (edge seen; motors already stopped)

**Changing any of these needs a PR that the whole team approves**, because it breaks everyone else's code.

## Setup

Python 3.12:

```bash
python -m venv .venv
# Windows:  .venv\Scripts\activate
# Mac/Linux: source .venv/bin/activate
pip install ultralytics opencv-python pyserial
```

Quick check that the detector works (downloads the model on first run):

```bash
yolo predict model=yolo26n.pt source=0 show=True
```

Firmware: Arduino IDE 2 with the Uno R4 board package.

## What not to commit

- `.venv/`
- Model weights (`*.pt`, `*.onnx`, exported folders). Share trained weights through a GitHub release or a shared drive.
- Raw photos and the dataset (`data/images/`, `data/raw/`). These get big fast. Keep them on Roboflow or a shared drive; commit only `data/desk.yaml` and the labeling rules.
- `runs/` (training output)

`.gitignore` already covers these.
