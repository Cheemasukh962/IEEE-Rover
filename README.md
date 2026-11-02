# IEEE-Rover

A desk rover that finds trash, picks it up and drops it in a bin, and never drives off the edge.

- **Python is the brain.** Camera, YOLO detection and decisions run on a laptop (later possibly a Raspberry Pi 5).
- **C++ is the reflexes.** A small Arduino Uno R4 sketch drives the motors and gripper, reads the edge sensors, and stops the rover on its own if anything goes wrong.
- The two sides talk in short text lines (`D 35 20`, `G 90`, `S`, `E FRONT`), so each side can be built and tested without the other.

## Start here

| Read | For |
|---|---|
| [PHASES.md](PHASES.md) | **What to work on.** The 15 deliverables, grouped into phases, with what can start now, what waits, and the checkpoint for each one |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How we work together: claiming a deliverable, branches, PRs, setup |
| [docs/rover-software-plan.md](docs/rover-software-plan.md) | The full technical plan: hardware, architecture, code samples, dataset, training, firmware, risks |

## Status

- Repo created. No code yet.
- **Next up:** deliverable #1 (skeleton + shared interfaces). After that, all of Phase B can start in parallel.

## Decisions so far

- **Python brain + C++ reflexes.** The Uno has 32 kB of RAM and can't run a vision model, so vision and ML stay in Python.
- **Build everything on a laptop first.** A webcam stands in for the rover camera, and a mock link stands in for the rover.
- **Camera: option A (Pi Zero 2 W as the rover's eyes) is the recommendation.** The Pi Camera Module 3 only plugs into a Raspberry Pi, and the parts list has none. Needs a Pi Zero 2 W, a Standard–Mini camera cable and a microSD card. *The team still needs to confirm this and order the parts.*
- **Safety lives only on the Uno.** It stops when there's no command for 500 ms, on any edge sensor, and it clamps speeds. None of this depends on Wi-Fi or Python.
- **Unknown objects are left alone.** PICK needs high confidence across several frames.
- **Timeline: about 3–4 weeks**, not the 8 weeks in the original plan. Writing the code is fast with AI help; the slow parts are photos/labeling, parts shipping, wiring and calibration.

## Open questions

- Confirm camera option A, B or C, and order parts.
- Where trash goes: a low bin on the desk, or a drop zone at the edge.
- Gripper style: claw, scoop or pusher. This decides which trash classes we can pick.
- Exact model of the IR edge sensors (HIGH or LOW over empty space).
- Final trash class list and "never pick" items (deliverable #9).
- Whether anyone's laptop has an NVIDIA GPU (otherwise train on Kaggle or Colab).
- Who owns which lane: vision, brain, hardware.
