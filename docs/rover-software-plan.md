# Trash-Sorting Rover: Software Plan

Snapshot of the shared plan doc as of Nov 1, 2026. Live, editable version: https://claude.ai/artifact/9V1X4LfxakMmqbfNr6tCRg

## Summary

We can have a detector naming objects on a photo within an hour, and the full see-decide-act loop working in about 8 weeks. The plan builds every piece of software in Python on a laptop first, with a webcam standing in for the rover. The rover joins later as a body that obeys simple commands.

- **Python is the brain, for good.** Vision, ML and decisions stay in Python on a laptop (or later a Raspberry Pi). None of it gets ported to C++, because the Uno can't run a vision model.
- **C++ is only the reflexes.** A short Arduino sketch on the Uno drives the motors and gripper servo, reads the edge sensors and IMU, and stops the rover on its own if anything goes wrong.
- **ML in three steps, cheapest first:** a pretrained model (no training), then text-prompt detection (still no training), then our own small model trained on photos from the rover's point of view.
- **One hardware fix before buying:** the Pi Camera Module 3 only plugs into a Raspberry Pi, and the parts list has none.

## Fix first: the camera needs a Raspberry Pi

The Pi Camera Module 3 Wide only works with Raspberry Pi boards that have a camera (CSI) connector. The ESP32 on the list is a plain AiTrip ESP-WROOM-32 dev board with no camera input, and the Uno has none either.

| Option | What changes | Cost vs. current list | Good | Trade-off |
| --- | --- | --- | --- | --- |
| **A. Pi Zero 2 W as the rover's eyes (recommended)** | Add a Pi Zero 2 W ($15) and a Standard–Mini camera cable (about $3), because the Zero's camera port is the small 22-pin type. It streams the camera to the laptop over Wi-Fi and passes commands to the Uno. | +$18, or +$9 if the ESP32 goes back | Keeps the 120° autofocus camera. Same camera moves to a Pi 5 later. | Laptop must be nearby. Wi-Fi adds delay. Needs a microSD card and a 5 V supply. |
| B. ESP32 camera board | Replace the Pi camera and the ESP32 with a Seeed XIAO ESP32S3 Sense (about $14), which streams its own small camera | about −$40 | Cheapest, one tiny board | Lower image quality (OV2640 or OV3660 sensor). Streaming firmware is in C++. |
| C. Raspberry Pi 5 on the rover | Add a Pi 5 ($110 for 4GB at PiShop) and the same cable. The model runs on the rover. | about +$113, plus a 5 V supply and cooler | No laptop or Wi-Fi needed. YOLO26n runs at about 67 ms per frame (Ultralytics). | Priciest: 2026 memory shortages raised Pi 5 prices. Draws the most power. |

Go with **A now**: the laptop runs all the ML during development anyway. If the rover later has to work without the laptop, the same Python code moves to a Pi 5 (option C).

- **The Yahboom kit's driver board does the encoder work.** Its 4-channel driver has its own STM32 chip and takes commands over I2C or serial. The Uno sends it speeds instead of counting encoder pulses itself.
- **Skip the 360° camera for version 1.** A 120° camera that turns in place sees all the way around in three stops.

## How the software works

Python on the laptop sees and decides; C++ on the Uno moves and protects. They talk in short text lines, so each side can be built and tested on its own.

Diagram (system layout, option A):
- Laptop, Python ("the brain"): YOLO detector (finds objects: class, confidence, box) → Brain (action table: pick or leave; state machine picks a target) → Steering (target position to wheel speeds: `D left right`).
- Wi-Fi: camera frames go up to the laptop; commands come down.
- Pi Zero 2 W, Python ("eyes and relay"): Camera Module 3 (120°, CSI ribbon) → relay.py (streams frames to the laptop, passes each command line to the Uno over USB).
- USB serial: commands go down to the Uno; edge events come up.
- Arduino Uno R4, C++ ("reflexes and safety"): rover_uno.ino obeys D (drive), G (gripper) and S (stop) lines with speeds clamped, and stops on its own when an edge sensor sees empty space or 500 ms pass with no command. Wired to: driver board + 4 motors (I2C or serial), IR edge sensors (digital pins), MPU-6500 IMU (I2C), gripper servo (PWM pin).

Each frame, about 10 times a second:

1. **Grab** the newest camera frame and drop any older ones.
2. **Detect:** YOLO returns a class, a confidence and a box for each object.
3. **Decide:** the action table says pick, leave or bin, and the state machine chooses a target.
4. **Act:** the target's position becomes two wheel speeds, sent as one line such as `D 35 20`.
5. **Obey:** the Uno applies it, checking the edge sensors and the command clock on every pass of its own loop.

**Why not port it all to C++?** The Uno R4 has 32 kB of RAM, while the YOLO26n model file alone is 5.3 MB. Vision stays in Python wherever it runs, and the C++ stays small.

## Tools and setup

| Tool | Used for | Notes |
| --- | --- | --- |
| Python 3.12 | All brain code | Ultralytics supports Python 3.8 and up |
| VS Code + Python extension | Editing and debugging | |
| Git + one shared GitHub repo | Working on the same code | One branch per feature, merged by pull request |
| Ultralytics (`pip install ultralytics`) | Detection (YOLO26), text-prompt detection (YOLOE), training, tracking, export | Installs PyTorch with it |
| OpenCV (`opencv-python`) | Camera frames, drawing boxes, printed ArUco/AprilTag markers | |
| pyserial | Talking to the Uno over USB | |
| Roboflow (free tier), CVAT or Label Studio | Drawing boxes on photos (labeling) | Roboflow also hosts public datasets and exports YOLO format |
| Kaggle Notebooks or Google Colab | Free GPU for training | Kaggle doesn't publish its weekly GPU quota; users commonly report about 30 hours |
| Arduino IDE 2 | Uno firmware in C++ | Add the Uno R4 board package |
| Raspberry Pi Imager + Pi OS Lite | Setting up the Pi Zero 2 W in Phase 6 | picamera2 is the Pi's Python camera library |

```text
rover/
  vision/     camera.py (webcam | video file | rover stream), detector.py (YOLO wrapper)
  brain/      actions.py (class -> action table), targets.py, state_machine.py, controller.py
  link/       mock.py, serial_link.py, wifi_link.py  (all share send() / receive())
  firmware/   rover_uno/rover_uno.ino  (C++, the only non-Python code)
  rover_pi/   relay.py  (runs on the Pi Zero 2 W)
  data/       desk.yaml, images/, labels/
  scripts/    snap.py, record.py, train.py, evaluate.py, teleop.py
  main.py     python main.py --camera webcam --link mock
```

## Phase 0 (Day 1): a detector on your own photo

1. Make a project folder, then run `python -m venv .venv`, activate it, and `pip install ultralytics`.
2. Put a pencil, an apple, a crumpled paper ball and a bottle on the desk. Take a phone photo from low down, about where the rover's camera will sit. Save it as `desk.jpg`.
3. Run:

```python
from ultralytics import YOLO

model = YOLO("yolo26n.pt")      # nano model, pretrained on COCO; downloads on first run
results = model("desk.jpg")
results[0].show()                # opens the photo with boxes drawn

for box in results[0].boxes:
    print(model.names[int(box.cls)], round(float(box.conf), 2), box.xyxy[0].tolist())
```

4. Live video: `yolo predict model=yolo26n.pt source=0 show=True`.

Expect the apple and the bottle right away. It won't find the pencil, the paper ball or a trash can, because COCO's 80 classes don't include them.

**Done when:** both partners have run it on a photo and on the webcam.

## Phase 1 (Week 1): our own classes, still no training

```python
from ultralytics import YOLOE

model = YOLOE("yoloe-26s-seg.pt")
model.set_classes(["pencil", "apple", "crumpled paper ball", "candy wrapper",
                   "soda can", "apple core", "small trash can"])
results = model.predict("desk.jpg", conf=0.25)
results[0].show()
```

Needs Ultralytics 8.4.0 or newer; the first `set_classes` call downloads a text encoder of about 254 MB.

1. Take 30 to 50 test photos at rover height: different spots, lighting and clutter.
2. Run YOLOE on all of them and save the annotated images.
3. Tally hits and misses per class in a shared sheet.
4. Try other wordings and keep the best ones.

**Done when:** a per-class hit rate on 30+ photos and the best prompt list.

## Phase 2 (Week 1): decide what counts as trash

| Class | Action | Why |
| --- | --- | --- |
| paper_ball | PICK | The most common desk trash |
| wrapper | PICK | Light and easy to grab |
| can | PICK | Only if the gripper can hold one |
| apple_core | PICK | Food waste |
| pencil | LEAVE | A keep item |
| apple | LEAVE | A whole apple is food, not trash |
| bin | DROP_TARGET | Where trash goes |

```python
ACTIONS = {
    "paper_ball": "PICK", "wrapper": "PICK", "can": "PICK", "apple_core": "PICK",
    "pencil": "LEAVE", "apple": "LEAVE",
    "bin": "DROP_TARGET",
}

def action_for(name, conf, frames_seen):
    action = ACTIONS.get(name, "LEAVE")             # anything unknown is left alone
    if action == "PICK" and (conf < 0.6 or frames_seen < 3):
        return "LEAVE"                               # be sure before throwing anything away
    return action
```

- Unknown means leave. PICK needs high confidence across several frames.
- Only list trash the 3D-printed gripper can actually hold.
- Consider a printed ArUco/AprilTag marker on the bin (OpenCV detects it with no training).

**Done when:** agreed class list, action table, and a "never pick" set of 3 to 5 keep items.

## Phase 3 (Weeks 2–3): build the dataset

| Source | What it gives | Catch |
| --- | --- | --- |
| COCO (inside `yolo26n.pt`) | Bottle, cup, apple, banana, book, phone and more | No pencil, paper ball, wrapper or bin |
| TACO | About 1,500 litter photos with 4,784 labeled objects; Roboflow copy is CC BY 4.0 | Outdoor scenes |
| Roboflow Universe | Community trash and desk-object sets in YOLO format | Label quality varies |
| Kaggle | Many garbage datasets, some with YOLO boxes | Many are classification-only, like TrashNet (2,527 single items, no boxes) |
| **Our own rover-height photos** | Our desk, lighting, camera | We label them; the most valuable source |

How many: Ultralytics suggests roughly 50–200 images per class for a narrow fine-tune and 500–2,000+ for robust results. Aim for about 100 of our own per class in version 1, plus a few hundred public ones.

Collect at the camera's real height and tilt; vary position, distance, lighting, clutter; add photos with none of our classes and look-alikes to keep.

YOLO format: one `.txt` per image, one line per object `class_id x_center y_center width height` (0 to 1).

```yaml
# data/desk.yaml
path: data
train: images/train
val: images/val
test: images/test
names: [paper_ball, wrapper, can, apple_core, pencil, apple, bin]
```

Pre-label with YOLOE (classes in the same order as `names`), then correct by hand:

```python
from pathlib import Path

out = Path("data/raw_labels")
out.mkdir(parents=True, exist_ok=True)
for r in model.predict("data/raw", stream=True, conf=0.3):
    rows = [f"{int(c)} {x:.4f} {y:.4f} {w:.4f} {h:.4f}"
            for c, (x, y, w, h) in zip(r.boxes.cls.tolist(), r.boxes.xywhn.tolist())]
    (out / (Path(r.path).stem + ".txt")).write_text("\n".join(rows))
```

Write labeling rules down; split by session (test set from a different day and lighting).

**Done when:** about 100 labeled rover-height images per class, split roughly 70/20/10.

## Phase 4 (Weeks 3–4): train and test our model

```python
from ultralytics import YOLO

model = YOLO("yolo26n.pt")      # start from COCO weights, never from scratch
model.train(data="desk.yaml", epochs=100, imgsz=640, batch=16)
metrics = model.val(data="desk.yaml", split="test")
print(metrics.box.map50)          # overall score, 0 to 1
```

Train on a free Kaggle or Colab GPU; download `runs/detect/train/weights/best.pt`. Read per-class precision/recall and the confusion matrix (a keep item read as trash is the expensive mistake).

Suggested version-1 targets: mAP50 ≥ 0.80 on the test set; no "never pick" item marked PICK; ≥10 FPS on the laptop (YOLO26n ≈ 39 ms per image on CPU in Ultralytics' benchmark). Improve loop: shoot 20–50 more photos of each failure case, label, retrain.

## Phase 5 (Weeks 2–5, in parallel): from detections to decisions

Build with the pretrained model first ("bottle" = stand-in trash, "apple" = keep item) while the other partner builds the dataset.

```python
model = YOLO("best.pt")
while True:
    frame = camera.latest()                    # newest frame only; old ones are dropped
    result = model.track(frame, persist=True, conf=0.4, verbose=False)[0]
    command = brain.step(parse(result))        # the state machine decides
    link.send(command)                         # mock prints it; the real link sends it to the Uno
```

```python
def drive_toward(box, frame_w, frame_h, v_max=40, k_turn=50, grab_line=0.85):
    x1, y1, x2, y2 = box
    error = ((x1 + x2) / 2 - frame_w / 2) / (frame_w / 2)   # -1 far left, +1 far right
    closeness = y2 / frame_h                                 # box bottom low in frame = close
    if closeness > grab_line and abs(error) < 0.1:
        return "GRAB"
    forward = max(15, v_max * (1 - closeness))               # slow down near the target
    turn = k_turn * error
    return f"D {int(forward + turn)} {int(forward - turn)}"  # left, right speed in %
```

State machine (one clockwise lap per piece of trash): SEARCH (turn ~40°, stop, look) → APPROACH (trash seen in 3 frames) → GRAB (at the grab line) → FIND_BIN (holding it) → GO_TO_BIN (bin seen) → DROP (at the bin) → SEARCH (dropped). APPROACH → SEARCH if the target is lost for 1 s; GRAB → SEARCH if the grab failed twice; SEARCH → DONE after a full turn with nothing to pick. Any state: an edge event from the Uno → BACK_OFF (reverse, turn away) → SEARCH.

Test without hardware: `link/mock.py` prints commands and fakes edge events; replay rover-height videos with `python main.py --camera test_run.mp4 --link mock`; draw an overlay of boxes, target, state and command.

## Phase 6 (Weeks 5–7): connect the real rover

| Direction | Message | Meaning |
| --- | --- | --- |
| Brain to Uno | `D <left> <right>` | Wheel speeds in %, −100 to 100 |
| Brain to Uno | `G <angle>` | Gripper servo angle, 0 to 180 |
| Brain to Uno | `S` | Stop now |
| Brain to Uno | `P` | Heartbeat |
| Uno to Brain | `E FRONT` or `E REAR` | Edge sensor sees empty space; motors already stopped |

Safety rules live only on the Uno: no command for 500 ms → stop; front edge → stop and accept only backing up (rear edge: only forward); speeds clamped.

```cpp
// rover_uno.ino: the rover's reflexes. Python decides; this sketch obeys and keeps it safe.
#include <Servo.h>

const int EDGE_FRONT[2] = {2, 3};        // IR sensors: front-left, front-right
const int EDGE_REAR[2]  = {4, 5};        // rear-left, rear-right
const int GRIPPER_PIN   = 9;
const int NO_DESK       = HIGH;          // many IR modules read HIGH over empty space: check yours
const unsigned long WATCHDOG_MS = 500;   // no command for this long = stop

Servo gripper;
String line;
unsigned long lastCommand = 0;
bool frontStop = false, rearStop = false;
int curL = 0, curR = 0;

void setMotors(int left, int right) {    // speeds in %, -100..100
  curL = left; curR = right;
  // TODO: send to the Yahboom 4-channel driver over I2C or serial (see Yahboom's tutorial)
}

bool overEdge(const int pins[2]) {
  return digitalRead(pins[0]) == NO_DESK || digitalRead(pins[1]) == NO_DESK;
}

void handle(const String& cmd) {
  int l, r;
  if (sscanf(cmd.c_str(), "D %d %d", &l, &r) == 2) {
    l = constrain(l, -100, 100);
    r = constrain(r, -100, 100);
    if (frontStop && (l > 0 || r > 0)) { l = 0; r = 0; }   // at a front edge: only back up
    if (rearStop && (l < 0 || r < 0))  { l = 0; r = 0; }   // at a rear edge: only go forward
    setMotors(l, r);
  } else if (cmd.startsWith("G ")) {
    gripper.write(constrain(cmd.substring(2).toInt(), 0, 180));
  } else if (cmd == "S") {
    setMotors(0, 0);
  }
  lastCommand = millis();                // any line, including "P", feeds the watchdog
}

void setup() {
  Serial.begin(115200);
  for (int i = 0; i < 2; i++) { pinMode(EDGE_FRONT[i], INPUT); pinMode(EDGE_REAR[i], INPUT); }
  gripper.attach(GRIPPER_PIN);
  setMotors(0, 0);
}

void loop() {
  while (Serial.available()) {           // read without blocking
    char c = Serial.read();
    if (c == '\n') { handle(line); line = ""; }
    else if (c != '\r' && line.length() < 32) line += c;
  }
  bool front = overEdge(EDGE_FRONT), rear = overEdge(EDGE_REAR);
  if (front && !frontStop) { setMotors(0, 0); Serial.println("E FRONT"); }
  if (rear && !rearStop)   { setMotors(0, 0); Serial.println("E REAR"); }
  frontStop = front;
  rearStop = rear;
  if (millis() - lastCommand > WATCHDOG_MS && (curL || curR)) setMotors(0, 0);
}
```

Bring-up order: bench test with wheels in the air → teleop from the laptop over USB (pyserial) → add the Pi Zero 2 W relay → collect rover-recorded data and retrain → calibrate (distance curve, grab line, k_turn, minimum speed, IR height/polarity) → close the loop with a partner on the stop key.

**Done when:** the rover drives to a paper ball, grabs it, drops it in the bin, and stops at the desk edge in 10 out of 10 tries.

## Phase 7 (Week 8): tune, then optionally move onboard

Tune thresholds, frames-to-confirm, turn gain and speeds; add photos for weak lighting; log every run for replay. Optional Pi 5: `model.export(format="ncnn")` (fastest on Raspberry Pi; ~67 ms per frame for YOLO26n at 640 px), then run the same `main.py` with `--camera picamera --link serial`. Later: a second camera on the gripper (Pi 5 has two camera connectors) and segmentation for grasp points.

## How each part maps to the software

| Part | Job | Talks to | Software | Language |
| --- | --- | --- | --- | --- |
| Laptop | The brain | Pi Zero 2 W over Wi-Fi | Ultralytics YOLO, OpenCV, `brain/` | Python |
| Pi Zero 2 W (to add) | Eyes and relay | Camera (CSI), Uno (USB), laptop (Wi-Fi) | picamera2, `relay.py` | Python |
| Camera Module 3 Wide | 120° view of the desk | Pi Zero 2 W via Standard–Mini cable | picamera2 | — |
| Arduino Uno R4 | Reflexes and safety | Pi (USB); driver board (I2C or serial) | `rover_uno.ino` | C++ |
| Yahboom 4WD kit driver board | Powers 4 × 520 encoder motors, reads encoders | Uno | Its own STM32 firmware | — |
| IR sensors (10-pack) | Four for desk edges, one in the gripper | Uno digital pins | `rover_uno.ino` | C++ |
| CANADUINO MPU-6500 IMU | Heading | Uno over I2C | MPU-6500 Arduino library | C++ |
| Servo + 3D-printed gripper | Grabs trash | Uno PWM pin | Arduino Servo library | C++ |
| AiTrip ESP32 | Not needed with option A | — | — | — |

## Timeline and who does what

About 8 weeks at a few hours a week each. A = vision and ML partner, B = robot and brain partner.

| Phase | When | Owner |
| --- | --- | --- |
| 0 · Day-1 detector | Day 1 | both |
| 1 · Text-prompt test | Week 1 | A |
| 2 · Trash rules | Week 1 | both |
| 3 · Dataset | Weeks 2–3 | A |
| 4 · Train and test | Weeks 3–4 | A |
| 5 · Brain + fake robot | Weeks 2–5 | B |
| 6 · Real rover | Weeks 5–7 | B, A helps |
| 7 · Tune and demo | Week 8 | both |
| Demo: rover clears the desk | end of week 8 | both |

## Risks and open questions

| Risk | Plan |
| --- | --- |
| Public data doesn't match the rover's view | Rover-height photos from Phase 3; rover-recorded data in Phase 6 |
| Throwing away a keep item | Unknown means leave, high PICK threshold, 3-frame confirm, "never pick" test set |
| Wi-Fi lag or dropouts | Stop-and-look; Uno watchdog and edge stops never depend on Wi-Fi |
| Gripper can't hold some items | Only classes the gripper fits; one retry, then move on |
| Edge sensors fooled by a dark or glossy desk | Test on the real desk before self-driving runs |
| Lighting changes | Training photos in every lighting used |

Open questions: which camera option (A, B or C); where trash goes (low bin on the desk, or drop zone at the edge); gripper style (claw, scoop or pusher); exact model of the 10-pack edge sensors; hours per week each; whether either laptop has an NVIDIA GPU.

## Key sources

Ultralytics YOLO26, YOLOE, Raspberry Pi and training-tips docs (docs.ultralytics.com); TACO (github.com/pedropro/TACO, arXiv 2003.06975); TrashNet (github.com/garythung/trashnet); Raspberry Pi Zero 2 W product page; PiShop Pi 5 4GB listing; Tom's Hardware on 2026 Pi price increases; BerryBase Standard–Mini camera cable; Arduino UNO R4 Minima docs; Yahboom chassis kit and 4-channel driver (RobotShop); CANADUINO 26617 MPU-6500 (DigiKey); desertcart listing for ASIN B07WCG1PLV.
