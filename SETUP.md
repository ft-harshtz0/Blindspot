# BlindSpot — Laptop Setup Checklist (do this top to bottom)

**For:** an **Intel Core Ultra laptop with an Arc 140T iGPU** and **32 GB shared
RAM** (no discrete NVIDIA GPU — the iGPU shares system memory, so there's no
separate VRAM pool to worry about).
**Goal:** get from "just cloned the repo" to "it narrates the scene out loud,
accelerated on the Arc iGPU" with zero guesswork.

> Follow the steps **in order**. After each step there's a ✅ **check** (what you
> should see) and a 🛠️ **if it fails**. Don't move to the next step until the
> check passes. Total time: ~20–30 min (most of it is downloads).

---

## Step 0 — What you need before starting

- [ ] Windows laptop with an **Intel Core Ultra** CPU and **Arc 140T** iGPU
- [ ] **32 GB RAM** (shared with the GPU — the more free RAM, the more the iGPU
      can use for model weights + frame buffers)
- [ ] A working **webcam** (built-in is fine)
- [ ] **Internet** (for downloads + the neural voice)
- [ ] ~5 GB free disk (models download/export on first run)
- [ ] The **latest Intel Graphics driver** installed (Arc 140T needs a recent
      driver for the OpenVINO GPU plugin to work — see Step 4)

---

## Step 1 — Install Python 3.11

We need **Python 3.11** (or 3.10). **NOT 3.12 or 3.13** — the audio model
(TensorFlow) has no wheels for those and the install will fail later.

1. Download Python 3.11 from https://www.python.org/downloads/release/python-3119/
   (pick **Windows installer 64-bit**).
2. In the installer, **tick "Add python.exe to PATH"**, then Install.

✅ **Check** — open a **new** PowerShell window and run:
```powershell
py -3.11 --version
```
You should see `Python 3.11.x`.

🛠️ **If it fails** (`py` not found): reinstall and make sure you ticked "Add to
PATH", then open a fresh terminal.

---

## Step 2 — Get the code

```powershell
cd D:\
git clone https://github.com/scorp2006/BlindSpot.git
cd BlindSpot
```

✅ **Check**: `ls` shows `run.py`, `config.py`, `blindspot`, `README.md`.

---

## Step 3 — Create + activate a virtual environment

Keeps all the AI libraries isolated so they can't break your system Python.

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

✅ **Check**: your prompt now starts with `(.venv)`.

🛠️ **If activation is blocked** ("running scripts is disabled"): run this once,
then retry the activate line:
```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

> ⚠️ From now on, **every** command assumes you see `(.venv)` in the prompt. If
> you open a new terminal, re-run `.\.venv\Scripts\Activate.ps1` first.

---

## Step 4 — Update the Intel GPU driver, then install torch + OpenVINO

There's no NVIDIA GPU here, so we skip CUDA entirely. YOLO gets accelerated on
the Arc 140T through **OpenVINO** (Intel's inference runtime, which talks to the
iGPU directly). This needs an up-to-date Intel Graphics driver.

1. Update your Intel Graphics driver from
   https://www.intel.com/content/www/us/en/download-center/home.html
   (search "Arc & Iris Xe Graphics - WHQL") or via **Intel Driver & Support
   Assistant**. Reboot after installing.
2. Install a plain (CPU) torch build. It's only used to load/export model
   weights — the actual YOLO inference runs through OpenVINO on the iGPU, not
   through torch:
   ```powershell
   pip install torch torchvision
   ```

✅ **Check**:
```powershell
python -c "import torch; print('torch', torch.__version__)"
```
You just want this to print a version with no errors — `cuda.is_available()`
being `False` here is **expected and fine**; OpenVINO handles the GPU, not torch.

🛠️ **If the driver update didn't take**: reboot, then confirm in
*Settings → System → Display → Advanced display* (or Device Manager) that the
GPU shows as "Intel(R) Arc(TM) 140T Graphics" with no yellow warning icon.

---

## Step 5 — Install the core libraries (Tiers 1 & 2) + verify the GPU plugin

```powershell
pip install -r requirements-core.txt
```
This pulls YOLO (ultralytics), OpenCV, **OpenVINO**, the neural voice, and the
VLM client. Takes a few minutes.

✅ **Check 1** — no red `ERROR` lines when it finishes.

✅ **Check 2** — OpenVINO can see the Arc iGPU:
```powershell
python -c "from openvino import Core; print(Core().available_devices)"
```
You want `GPU` in the printed list (alongside `CPU`).

🛠️ **If `GPU` is missing**: the driver from Step 4 didn't install correctly, or
you need a reboot. Update it from Intel's site, reboot, and re-run the check.
*(Everything still works on `CPU` — just slower — so you can continue and fix
this later if you're in a hurry.)*

🛠️ **If a build error mentions a compiler / wheels**: make sure you're on Python
3.11 (Step 1) inside the venv (Step 3). 99% of install pain is wrong Python.

---

## Step 6 — Export YOLO to OpenVINO so it runs on the Arc iGPU

The `.pt` weights ultralytics downloads are a **torch** format; to run on the
iGPU we export them once to **OpenVINO IR** format (FP16, which is faster and
lighter on shared RAM than FP32):

```powershell
yolo export model=yolov8m.pt format=openvino half=True
```
This downloads `yolov8m.pt` (~50 MB) if needed, then writes a
`yolov8m_openvino_model/` folder next to it (`.xml` + `.bin`).

Now point BlindSpot at the exported model instead of the raw `.pt`:
```powershell
$env:BLINDSPOT_YOLO = "yolov8m_openvino_model/"
```
(Add this `$env:` line to your terminal routine, or edit the default in
`config.py` → `YOLO_MODEL`, so you don't have to retype it every session.)

✅ **Check**: the folder `yolov8m_openvino_model` exists and contains an `.xml`
file.

> `config.YOLO_DEVICE` is `"auto"` by default — `blindspot/vision.py` detects
> the `_openvino_model` folder and automatically asks OpenVINO for
> `intel:gpu` (falling back to `intel:cpu` if the plugin isn't available). You
> don't need to set the device yourself.

---

## Step 7 — Smoke-test the pieces ONE AT A TIME 🧩

This is the important part — **test each module alone** so you know exactly what
works before running everything together.

### 7a. Camera
```powershell
python -m blindspot.camera
```
✅ A window opens showing your webcam. Press **q** to close.
🛠️ Black window / error? Another app may be using the camera. Or edit
`config.py` → `CAMERA_SOURCE = 1` and retry.

### 7b. Voice (downloads nothing; needs internet for the neural voice)
```powershell
python -m blindspot.speech
```
✅ You **hear** three spoken test lines.
🛠️ No sound? Check volume/output device. If the neural voice fails, it auto-falls
back to an offline robotic voice — you'll still hear *something*.

### 7c. Vision (uses the OpenVINO export from Step 6)
```powershell
python -m blindspot.vision
```
✅ The terminal prints `Model yolov8m_openvino_model/ loaded on device:
intel:gpu`, a webcam window opens with **green boxes** around objects, and the
scene is printed in the terminal (e.g. `A person close ahead.`). Press **q** to
quit.
🛠️ If it says `loaded on device: intel:cpu` instead of `intel:gpu`: OpenVINO
can't see the GPU plugin — recheck Step 5's Check 2 (driver/reboot). It still
works, just slower.

### 7d. 🎉 TIER 1 — the live narrator (this is a real demo)
```powershell
python run.py narrator
```
✅ It **watches your webcam and describes the scene out loud** as things change.
Press **q** (or Ctrl+C) to stop. **If this works, you have a working demo.**

### 7e. VLM client — free "stub" mode (no GPU, no cost)
```powershell
python -m blindspot.vlm
```
✅ Prints a `(stub)` answer. This proves the brain-client wiring works **without
spending any GPU money**. (Turning on the real brain is a later step — see
`vlm_space/README.md`. Leave it in stub mode for now.)

---

## Step 8 — (LATER) Audio + voice questions (Tier 3)

Only do this once Tier 1 works and you're ready for the multimodal part.

```powershell
pip install -r requirements-audio.txt
winget install Gyan.FFmpeg
```
Then **close and reopen** PowerShell (so ffmpeg is on PATH), re-activate the venv,
and test:
```powershell
python -m blindspot.audio     # prints sounds it hears (talk, play a horn on YouTube)
python -m blindspot.listen    # hold SPACE, speak a question, release -> see the text
```
🛠️ If TensorFlow won't install: you're almost certainly not on Python 3.11.
This is exactly why we pinned it. Everything in Tiers 1–2 still works without it.

---

## Step 9 — (OPTIONAL) Phone as the camera

Walk-around demo. The **phone is just the camera**; the laptop still runs all AI.

1. Install the **"IP Webcam"** app on the phone (free, Android).
2. Open it → *Start server*. It shows a URL like `http://192.168.1.42:8080`.
3. Phone and laptop must be on the **same Wi-Fi**.
4. On the laptop, edit `config.py`:
   ```python
   CAMERA_SOURCE = "http://192.168.1.42:8080/video"   # use YOUR phone's URL
   ```
5. Run `python run.py narrator` — it now uses the phone's camera.

---

## Quick reference — everyday commands

```powershell
.\.venv\Scripts\Activate.ps1          # activate the env (do this every new terminal)
$env:BLINDSPOT_YOLO = "yolov8m_openvino_model/"   # use the iGPU-accelerated model
python run.py narrator                # Tier 1: live narrator (safe, always works)
python run.py full --no-audio         # everything except sound
python run.py full                    # everything (needs Step 8 done)
```

## The 3 things that cause 95% of problems
1. **Wrong Python** — must be 3.11, inside the venv (prompt shows `(.venv)`).
2. **Forgot to activate the venv** (or forgot to re-set `$env:BLINDSPOT_YOLO`)
   in a new terminal.
3. **OpenVINO can't see the GPU** — usually an outdated Intel Graphics driver.
   Update it, reboot, and re-run `python -c "from openvino import Core;
   print(Core().available_devices)"`. It still runs on `intel:cpu` meanwhile,
   just slower.

If stuck on any step, copy the exact error text back to the team chat — the step
number + error is enough to fix it fast.
