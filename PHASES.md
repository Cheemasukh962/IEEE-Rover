# Software Deliverables by Phase

Each deliverable (#1–#15) is a task one person can claim. Phases show what can start now and what has to wait.

## How the code connects

Every frame goes through one pipeline. Each deliverable is one stage of it:

```
 camera (#2) → detector (#3) → brain: rules (#9) + state machine (#10) → steering (#5) → link → Uno firmware (#8) → motors
                                                                                           ↑
                                                         #6 mock (laptop only) / #13 USB / #14 Wi-Fi
                                                         edge events come back up the same link

 Side track (makes the detector smarter):
 data tools (#7) → dataset (#11) → training (#12) → new best.pt loaded by detector (#3)
```

Each later phase replaces one stage and leaves the rest alone:

- **Our own model:** the detector loads `best.pt` instead of `yolo26n.pt`.
- **Real rover:** the mock link is replaced by the serial or Wi-Fi link.
- **Rover camera:** the webcam input is replaced by the rover's stream.

The brain code never changes for any of these swaps. That's why #1 (the shared interfaces) comes first.

### Shared interfaces (fixed in #1)

- `Detection(name, conf, box=(x1, y1, x2, y2), track_id)`
- `camera.latest() -> frame`
- `link.send(line)` / `link.receive() -> line or None`
- Commands to the Uno: `D <left> <right>`, `G <angle>`, `S`, `P`
- Replies from the Uno: `E FRONT` / `E REAR`

Changing an interface after #1 is merged needs a PR the whole team approves.

---

## All deliverables

| # | Deliverable | Files | Done when | Waits on |
|---|---|---|---|---|
| 1 | Skeleton + shared interfaces | folder layout, `main.py`, `requirements.txt` | Everyone clones it and `python main.py --camera webcam --link mock` runs | Nothing (do first, about an hour) |
| 2 | Camera input | `vision/camera.py` | `latest()` returns the newest frame from a webcam or a video file | Nothing |
| 3 | Detector wrapper | `vision/detector.py` | Turns YOLO / YOLOE output into a list of `Detection`, with tracking IDs | Nothing |
| 4 | Text-prompt test | `scripts/yoloe_test.py` | Per-class hit rate on 30+ desk photos, plus the best prompt wording | Nothing |
| 5 | Steering math | `brain/controller.py` | `drive_toward()` passes unit tests: left target turns left, close target returns `GRAB` | Nothing |
| 6 | Mock link + debug overlay | `link/mock.py`, overlay drawing | Prints commands, can fake edge events, draws boxes/state/command on the video | Nothing |
| 7 | Data tools | `scripts/snap.py`, `record.py`, prelabel, split | Takes photos at rover height, pre-labels them with YOLOE, splits them 70/20/10 | Nothing |
| 8 | Arduino firmware | `firmware/rover_uno/rover_uno.ino` | Compiles and follows the `D`/`G`/`S`/`P`/`E` protocol | Nothing to write it; the Uno to test it |
| 9 | Trash rules | `brain/actions.py` | Class list, action table and "never pick" list are merged | A 15-minute team decision |
| 10 | State machine | `brain/state_machine.py`, `targets.py` | Replaying a video through the mock link (#6) gives the right states and commands | #1; uses #5 and #9 but can stub them |
| 11 | Dataset | `data/` | About 100 labeled rover-height photos per class | #7 tools, then photo-taking time |
| 12 | Train + evaluate | `scripts/train.py`, `evaluate.py`, Kaggle notebook | `best.pt` scores mAP50 ≥ 0.80 and no keep item is marked PICK | #11 (script can be built now on a public dataset) |
| 13 | Serial link + teleop | `link/serial_link.py`, `scripts/teleop.py` | Drive the rover from the keyboard over USB | Uno + #8 |
| 14 | Pi relay + Wi-Fi link | `rover_pi/relay.py`, `link/wifi_link.py` | The laptop gets frames from the rover and sends commands back | Pi Zero 2 W arriving |
| 15 | Integration + tuning | `main.py`, configs | Paper ball → grab → bin, plus stopping at the edge, 10 out of 10 tries | Everything |

---

## Phase A: Foundation (start now, about an hour)

| # | Deliverable | Context |
|---|---|---|
| 1 | Skeleton + shared interfaces | Sets up the folders and the `Detection` / `camera.latest()` / `link.send()` shapes that every other file uses. Without it, everyone invents their own format and nothing fits together. |

**Checkpoint:** `python main.py --camera webcam --link mock` runs on everyone's laptop.

## Phase B: Build the parts in parallel (start now, no hardware needed)

These can all run at the same time because each one only depends on the interfaces from #1.

| # | Deliverable | Context: what it plugs into |
|---|---|---|
| 2 | Camera input | The start of the pipeline. Feeds frames to #3. It always gives the newest frame, so the rover reacts to now and not to old frames. Later the rover stream plugs in here. |
| 3 | Detector wrapper | Takes a frame from #2 and returns `Detection` objects for the brain (#10). It hides YOLO's details, so swapping in our trained model later is a one-line change. |
| 4 | Text-prompt test | A side experiment using #3. It shows which objects YOLOE finds without training, and that decides how much the dataset (#11) has to cover. |
| 5 | Steering math | Turns a target box into the `D left right` command that goes to the link. Pure math, so it can be tested without a camera or a robot. |
| 6 | Mock link + debug overlay | A fake robot that prints commands. It lets #10 be tested end to end on a laptop. The overlay is how the team will debug the rest of the project. |
| 7 | Data tools | Tools for collecting and pre-labeling photos. The side track (#11 → #12) can't start until these exist. |
| 8 | Arduino firmware | The far end of the link. It obeys commands and enforces the safety stops. The protocol is already fixed, so it can be written now and flashed once the Uno is wired. |
| 9 | Trash rules | The team decision on which classes get PICK, LEAVE or DROP_TARGET. Used by #10 to choose targets. |

**Checkpoint:** each part passes its own test with no hardware.

## Phase C: The brain works on a fake robot (after #1; better once #5, #6 and #9 exist)

| # | Deliverable | Context |
|---|---|---|
| 10 | State machine | The center of the pipeline. It takes detections from #3, uses #9 to decide, uses #5 to steer and sends through #6. This is where SEARCH → APPROACH → GRAB → FIND_BIN → DROP happens. |

**Checkpoint:** the full loop works on a laptop with the pretrained model, using a bottle as the stand-in trash. Replay a desk video and the right commands come out. This is the first point where the whole system works in software.

## Phase D: Our own model (after #7; runs alongside Phase C)

| # | Deliverable | Context |
|---|---|---|
| 11 | Dataset | Photos taken from the rover's height. The pretrained model can't see paper balls or wrappers, so this is what fixes that. |
| 12 | Train + evaluate | Produces `best.pt`, which #3 loads. Nothing else in the pipeline changes. |

**Checkpoint:** swap `best.pt` into the Phase C loop and it now finds our real trash classes.

## Phase E: Connect the real rover (after the parts arrive and #8 is done)

| # | Deliverable | Context |
|---|---|---|
| 13 | Serial link + teleop | Replaces the mock link with a USB cable to the Uno. Drive the rover by hand first to test the firmware and motors. |
| 14 | Pi relay + Wi-Fi link | Replaces the USB cable with Wi-Fi through the Pi Zero, and the webcam with the rover's camera. The brain doesn't change. |

**Checkpoint:** the Phase C brain drives the real rover toward a bottle.

## Phase F: Integration and tuning (after everything)

| # | Deliverable | Context |
|---|---|---|
| 15 | Integration + tuning | Calibrate speeds, the grab line, turn gain and edge sensors on the real desk, then run the demo again and again. |

**Checkpoint:** paper ball → grab → bin, plus stopping at the desk edge, 10 out of 10 tries.

---

## Suggested split for 3 people

- **Vision:** #3 → #4 → #7 → #11 → #12
- **Brain:** #1 → #5 → #6 → #10
- **Hardware:** #8 → #13 → #14
- **Everyone:** #9, #15

## Timeline

About 3–4 weeks. AI speeds up writing the code; the slow parts are taking and labeling photos, parts shipping, wiring, and calibrating on the real desk.
