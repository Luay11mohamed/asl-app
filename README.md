# ASL Live — FastAPI backend + browser frontend + TTS

A web port of the original desktop camera-inference script. Same models,
same feature math, same gloss → NLP pipeline — but the webcam and
MediaPipe landmark extraction now run **in the browser**, streaming
compact feature vectors to a FastAPI backend over a WebSocket, which
runs the PyTorch models and gloss/NLP logic and streams predictions back.
Sentences are spoken aloud with the browser's built-in text-to-speech.

```
Browser (camera + MediaPipe Tasks JS)          FastAPI backend
──────────────────────────────────────         ─────────────────────────
getUserMedia → HandLandmarker/PoseLandmarker
  → extractFrameFeatures() (155-dim)   ──WS──▶  frame buffer (60×155)
  → extractStaticFeatures() (63-dim)            → motion features (308-dim)
                                                 → ASLv3Classifier / ASLStaticModel
control buttons (mode/add/word/undo/…) ──WS──▶  gloss buffer, OOV, gloss→text
prediction + state              ◀──WS──         top-5 probs, HUD state
speechSynthesis.speak(sentence)  (client-only, no audio sent to server)
```

## Why the split?

Doing MediaPipe hand/pose detection in the browser (via
`@mediapipe/tasks-vision`) means the server never needs webcam access or
a Python MediaPipe/XNNPACK install — it only receives small numeric
vectors, which is far cheaper than streaming video frames over a socket
and works the same whether the backend is on localhost or a remote host.

## 1. Get the model files

Backend (PyTorch checkpoints — same files the original script used):

```
backend/checkpoints/final_dynamic_model.pt      # ASLv3Classifier checkpoint (feat_dim=308)
backend/checkpoints/final_static_model.pth      # ASLStaticModel state_dict
backend/checkpoints/label_encoder.joblib        # static-model class order
```

Frontend (MediaPipe Tasks `.task` files — same ones used by the desktop
script's `HAND_MODEL_PATH` / `POSE_MODEL_PATH`):

```
frontend/models/hand_landmarker.task
frontend/models/pose_landmarker_lite.task
```

Get them from https://ai.google.dev/edge/mediapipe/solutions/vision if
you don't already have them.

## 2. (Optional) plug in your real NLP pipeline

If you have working `oov_handler.py` / `gloss_to_text_ollama.py` files
(the ones the original script imported), drop them into
`backend/app/` — `nlp_bridge.py` will import them automatically. If
they're absent, a built-in rule-based fallback (word-join + capitalize +
period) is used instead, so the app still runs end-to-end without Ollama.

## 3. Run the backend

```bash
cd backend
pip install -r requirements.txt
# point at your checkpoint files if they're not in ./checkpoints/
export DYNAMIC_CHECKPOINT_PATH=./checkpoints/final_dynamic_model.pt
export STATIC_CHECKPOINT_PATH=./checkpoints/final_static_model.pth
export STATIC_LABEL_ENCODER_PATH=./checkpoints/label_encoder.joblib
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

The frontend is mounted by the same server, so open:

```
http://localhost:8000
```

(No separate frontend server needed — `frontend/` is served as static
files. If you'd rather run it standalone, e.g. `python -m http.server`
inside `frontend/`, just make sure `WS_URL` in `app.js` still resolves
to your backend.)

## 4. Controls

Same as the original desktop app, as on-screen buttons and matching key
shortcuts:

| Key | Action |
|---|---|
| C | toggle STATIC (letter/digit) ↔ DYNAMIC (word) model |
| A | add current prediction to gloss/spelling buffer |
| W | finish spelling the current word |
| B | undo last letter/word |
| R | reset the rolling frame buffer |
| L | toggle LLM usage for sentence generation |
| G | run OOV handling + gloss→text and display the sentence |
| X | clear gloss/spelling buffer and sentence |

Plus two web-only additions:
- **🔊 Speak sentence** — replays the last generated sentence via TTS.
- **Auto-speak** — automatically speaks each newly generated sentence.

## Notes / assumptions

- TTS is done client-side with the Web Speech API (`speechSynthesis`) —
  no audio ever touches the server, and it works fully offline in most
  browsers. If you'd prefer server-side TTS (e.g. for a fixed voice or
  to save an audio file), swap in a library like `pyttsx3` or a cloud
  TTS API inside a new `/api/tts` endpoint and call it from `app.js`
  instead of `speechSynthesis`.
- Landmark handedness/mirroring conventions in `app.js` mirror the
  python `extract_frame_features` exactly (camera-left hand = signer's
  right hand, etc.) — this assumes the browser feed isn't additionally
  flipped before landmark detection (only the CSS/video preview is
  mirrored for a natural "mirror" look; detection runs on the
  unmirrored MediaPipe stream).
- One `Session` per WebSocket connection holds the whole state machine
  (frame buffer, gloss buffer, mode, smoothing history) — equivalent to
  the local variables in the original `run_inference()` loop, just
  scoped per browser tab instead of per process.
