# mod\_amd — Improvement Roadmap

This document outlines planned improvements to `mod_amd`'s answering machine
detection accuracy and capabilities. The work is organised into three phases,
progressing from quick wins to more advanced techniques.

---

## Phase 1 — Improved Energy-Based VAD (Easy)

The current `classify_frame()` function uses a simple sum of absolute sample
values compared against a fixed `silence_threshold`. This is the most basic
form of voice activity detection and is prone to false positives in noisy
environments and false negatives with quiet speakers.

### 1.1 RMS Energy Calculation

Replace the absolute-sum energy calculation with **Root Mean Square (RMS)**
energy, which is more acoustically meaningful and better represents perceived
loudness.

**Current approach:**
```c
energy += abs(audio[j++]);
score = (uint32_t)(energy / (f->samples / divisor));
```

**Proposed approach:**
```c
energy += (double)audio[j] * audio[j];  // sum of squares
score = (uint32_t)sqrt(energy / sample_count);  // RMS
```

### 1.2 Adaptive Silence Threshold

Instead of a fixed `silence_threshold`, dynamically track the noise floor of
the call and set the threshold relative to it. This handles varying line
conditions and background noise levels.

**Approach:**
- Maintain a running average of the lowest energy frames (noise floor estimate)
- Set the effective threshold as `noise_floor * multiplier` (e.g., 2x–3x)
- Add a configuration parameter `silence_threshold_multiplier` (default: `2`)
- Keep the existing fixed threshold as a fallback / minimum

### 1.3 Zero-Crossing Rate

Combine energy with **zero-crossing rate (ZCR)** to better distinguish speech
from noise. Speech typically has a lower zero-crossing rate than white noise or
static, even at similar energy levels.

**Approach:**
- Count the number of sign changes in the audio samples per frame
- Normalise by frame length to get a rate
- A frame is classified as `VOICED` only if energy exceeds the threshold **and**
  ZCR is below a configurable maximum (e.g., `max_zero_crossing_rate`)

### 1.4 Estimated Effort

- **Complexity:** Low
- **Files changed:** `mod_amd.c` only
- **New config params:** `silence_threshold_multiplier`, `max_zero_crossing_rate`
- **Risk:** Low — the existing heuristic logic remains unchanged; only the frame
  classifier is improved

---

## Phase 2 — Beep Detection (Medium)

Answering machines almost always play a **beep tone** before the recording
period begins. Detecting this beep is the single most impactful improvement for
AMD accuracy, and is what most commercial solutions rely on.

### 2.1 Goertzel Algorithm for Tone Detection

Implement a **Goertzel filter** to efficiently detect sustained single-frequency
tones in the audio stream. Common answering machine beep frequencies are around
**440 Hz**, **480 Hz**, **620 Hz**, and **1000 Hz**.

**Approach:**
- Run Goertzel filters for a small set of target frequencies on each frame
- If a single frequency dominates the frame energy for a sustained duration
  (e.g., 100–500 ms), classify it as a beep
- Add new result/cause: `MACHINE` / `BEEP`

### 2.2 Post-Detection Beep Waiting

After an initial `MACHINE` classification, optionally continue listening for
the beep before firing the event. This allows the caller to start playback at
exactly the right moment (after the beep).

**New channel variables:**
- `amd_beep_detected` — `true` / `false`
- `amd_beep_time_ms` — milliseconds from answer to beep detection

**New config params:**
- `beep_detection` — `true` / `false` (default: `false`)
- `beep_frequencies` — comma-separated list of frequencies to detect (default:
  `440,1000`)
- `beep_min_duration` — minimum tone duration in ms (default: `100`)
- `beep_max_wait` — maximum time to wait for a beep after machine detection
  (default: `10000`)

### 2.3 Estimated Effort

- **Complexity:** Medium
- **Files changed:** `mod_amd.c`, `amd.conf.xml`
- **New dependencies:** None (Goertzel is a simple algorithm, ~30 lines of C)
- **Risk:** Low — beep detection can be added as an optional feature behind a
  config flag

---

## Phase 3 — Machine Learning Classifier (Advanced)

For the highest possible accuracy (95%+), replace or augment the heuristic
approach with a trained **machine learning model** that classifies audio as
human or machine based on acoustic features.

### 3.1 Feature Extraction

Extract meaningful audio features from each analysis window:

- **MFCCs** (Mel-Frequency Cepstral Coefficients) — the standard feature for
  speech/audio classification
- **Spectral centroid** — where the "centre of mass" of the spectrum is
- **Spectral rolloff** — frequency below which a percentage of energy is
  concentrated
- **Pitch / F0** — fundamental frequency tracking
- **Energy contour** — how energy changes over time

### 3.2 Model Architecture

A lightweight model suitable for real-time inference in C:

- **Option A:** Small **1D CNN** operating on MFCC frames — fast, simple
- **Option B:** **GRU/LSTM** for sequential analysis — better temporal modelling
- **Option C:** **Random Forest / XGBoost** on hand-crafted features — no neural
  network dependency, easier to deploy

### 3.3 Inference Runtime

Options for running the model in the FreeSWITCH process:

| Runtime | Pros | Cons |
|---|---|---|
| **TensorFlow Lite** | Well-supported, optimised for edge | Larger dependency |
| **ONNX Runtime** | Framework-agnostic, good C API | Medium dependency |
| **Custom C implementation** | No dependencies | Only works for simple models (e.g., decision trees) |
| **External service (gRPC)** | Flexible, can use any framework | Adds network latency |

### 3.4 Training Data Requirements

- Labelled audio samples of answered calls: **human** vs **machine**
- Ideally 1,000+ samples of each class
- Diverse set: different languages, accents, phone systems, voicemail providers
- Data augmentation: add noise, vary volume, simulate different codecs

### 3.5 Hybrid Approach

Rather than replacing the heuristic entirely, use a **hybrid strategy**:

1. Run the existing heuristic VAD for fast, confident decisions
2. For `NOTSURE` cases (or cases near decision boundaries), invoke the ML model
3. This minimises latency for clear-cut cases while improving accuracy for
   ambiguous ones

### 3.6 Estimated Effort

- **Complexity:** High
- **Files changed:** New files for feature extraction and inference, plus
  `mod_amd.c` integration
- **New dependencies:** ML inference runtime (TFLite, ONNX, or custom)
- **Risk:** Medium — requires training data collection and model validation
- **Timeline:** Significantly longer than Phases 1 and 2

---

## Other Ideas to Explore

### Speech-to-Text Based Detection

Use a speech recognition engine to transcribe the first few seconds and
classify based on text content. Answering machine greetings follow predictable
patterns ("Please leave a message", "is not available", etc.).

- Could integrate with FreeSWITCH's `mod_unimrcp` or a cloud STT API
- Best as a fallback for `NOTSURE` cases
- Adds latency and potentially cost (cloud APIs)

### Cadence / Rhythm Analysis

Analyse the temporal pattern of voice and silence transitions rather than just
counting words:

- Humans: short utterance → long pause (waiting for response)
- Machines: longer utterances → short pauses → more utterances (reading a script)
- Track the ratio and rhythm of voice/silence segments over time
- Could be implemented as an enhancement to the existing state machine

### Use FreeSWITCH Built-in VAD

FreeSWITCH provides a built-in `switch_vad_t` API that is more sophisticated
than the current custom energy check. It handles noise floor tracking and has a
proper voice/silence state machine. Consider replacing `classify_frame()` with
the FreeSWITCH VAD API as a quick improvement.

---

## Priority Summary

| Phase | Improvement | Accuracy Impact | Effort | Dependencies |
|---|---|---|---|---|
| **1** | RMS + adaptive threshold + ZCR | Moderate (+5–10%) | Low | None |
| **2** | Beep detection (Goertzel) | High (+15–20%) | Medium | None |
| **3** | ML classifier | Very High (+20–30%) | High | ML runtime |

**Recommended starting point:** Phase 1, then Phase 2. These two phases alone
should bring detection accuracy from ~75–80% to ~90–95% without adding any
external dependencies.
