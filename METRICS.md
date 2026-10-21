# Runtime Metrics Snapshot

Measured on the demo device on **2026-10-21 14:25–14:34 JST**, release build of `demo_30thaug`
(commit `e550d82d`, APK SHA-256 `e805e94f…38c8e`, installed 14:24:52). All numbers come from
`adb` (root) against the live app process; nothing here is estimated from code unless marked.

Reproduction commands are at the end. Re-run this snapshot after any change to the model file,
`LLMManager`, the system prompt, or the tool registry.

---

## 1. Device

| Item | Value |
|---|---|
| Hardware | Pixel Tablet (`tangorpro`), AOSP Automotive build, Android 17 (SDK 37) |
| SoC | Google Tensor G2 (`gs201`) |
| CPU | 8 cores: 4× Cortex-A55 @ 1.80 GHz, 2× Cortex-A78 @ 2.35 GHz, 2× Cortex-X1 @ 2.85 GHz |
| GPU | Arm Mali-G710 (`mali0`), DVFS 202–848 MHz |
| RAM | 7.6 GB total (`MemTotal` 7 600 328 kB); ~1.4 GB available with the app resident |
| Power at capture | USB powered, battery 9 %, charging at ~0.88 A |
| Thermal at capture | BIG cluster 81 °C, MID 69 °C, LITTLE 71 °C, GPU 49 °C, skin 39 °C (skin status 1 = light throttling) |

> Battery was at 9 % during capture. Keep the tablet above ~30 % when benchmarking; the kernel
> throttles aggressively at low charge and TTFT will drift upward.

---

## 2. Model

### 2.1 LLM (on-device)

| Item | Value |
|---|---|
| File | `gemma-4-E2B-it.litertlm` |
| Path on device | `/storage/emulated/10/Android/data/com.poc.assistant/files/` (user 10 → `/data/media/10/…`) |
| Size | 2 588 147 712 bytes (2.41 GiB) |
| Format | LiteRT-LM bundle (`LITERTLM` header), created 2026-04-28 by the ODML authors |
| Family / size class | Gemma 4, **E2B** = "effective 2 B" parameter class (Per-Layer-Embedding architecture; the bundle does not record an exact parameter count or weight dtype) |
| Sub-models in bundle | `tf_lite_prefill_decode`, `tf_lite_mtp_drafter` (speculative-decoding drafter), `tf_lite_embedder`, `tf_lite_per_layer_embedder`, `tf_lite_vision_encoder` + `vision_adapter`, `tf_lite_audio_encoder_hw` + `audio_adapter` |
| Modalities used by the app | Text only (vision/audio sub-models present but unused) |
| Chat template | Gemma tool-calling template (`<|tool_call>`, `<|tool_response>`), overridden by the app's own `<TOOL>name(args)</TOOL>` convention in the system prompt |

### 2.2 Engine configuration (`LLMManager.kt`)

| Setting | Value |
|---|---|
| Runtime | `com.google.ai.edge.litertlm:litertlm-android:0.16.0` (`liblitertlm_jni.so`, 21.5 MB) |
| Backend | `Backend.GPU()` (ML Drift / OpenCL on Mali); CPU fallback only if GPU init throws |
| `maxNumTokens` (KV cache / context window) | **4096** (hard-floored in code; `app_prefs.max_tokens`) |
| Speculative decoding | `ExperimentalFlags.enableSpeculativeDecoding = true` for Gemma models; the `mtp_drafter` caches exist on disk, so it is active |
| Shader / weight cache | Persistent, `no_backup/litert_gpu_cache/` |
| Engine instance | One `Engine` + one `Conversation` for the process, serialised by `inferenceMutex`; session reset = `resetConversation()`, never re-init |
| Prewarm | After init, the system prompt + `"Input: Hello / Output: Hello! …"` is pushed through the model so the KV cache already holds the system prompt |

### 2.3 Cold-start timeline (from logcat, this boot)

| Phase | Start → End | Duration |
|---|---|---|
| `WakeWordService` "Booting LLM in background" → `LLM Initialized successfully` | 14:24:54.648 → 14:24:59.897 | **5.2 s** |
| KV-cache prewarm (system prompt) | 14:24:59.9 → 14:25:04.477 | **4.6 s** |
| Total to "ready for first query" | | **≈ 9.8 s** |

GPU program/weight caches were regenerated at 14:12 (fresh install). First-ever init after a cache
wipe takes longer than the 5.2 s above.

### 2.4 GPU-side cache files (`/data/user/10/com.poc.assistant/no_backup/litert_gpu_cache/`)

| File | Size | Status |
|---|---|---|
| `gemma-4-E2B-it.litertlm_…_mldrift_weight_cache.bin` | 783 MB | **active** (GPU-resident transformer weights) |
| `gemma-4-E2B-it.litertlm_….mtp_drafter_mldrift_weight_cache.bin` | 42 MB | active (drafter weights) |
| `…_mldrift_program_cache.bin` + `…mtp_drafter…program_cache.bin` | 12.5 MB + 0.5 MB | active (compiled kernels) |
| `gemma-4-E2B-it.litertlm_1790229436_…xnnpack_cache` (2026-09-28) | 788 MB | stale (older model hash, CPU path) |
| `model.litertlm_…xnnpack_cache` + `model.litertlm_…mldrift_weight_cache.bin` (2026-10-13) | 1 154 MB + 1 146 MB | stale (previous model file) |

**≈ 3.1 GB of stale cache is reclaimable** (`app data` dir is 3.7 GB; the live caches need ~0.85 GB).

### 2.5 Other ML models bundled in the APK

| Model | Use | Size |
|---|---|---|
| `assets/model/` (Vosk: `Gr.fst`, `HCLr.fst`, `final.mdl`, `final.ie`) | Wake word "hey iris", always-on | 68 MB + `libvosk.so` 8.9 MB |
| `universal_sentence_encoder.tflite` | `SemanticSearchManager` intent/domain classifier | 5.9 MB |
| `models/bge-small-en-v1.5-mediapipe.tflite`, `models/bge_small_en_v1_5_int8.tflite` | Embeddings (two variants shipped) | 34 MB + 34 MB |
| `gesture_recognizer.task`, `face_landmarker.task` | DMS (MediaPipe tasks-vision 0.10.14) | 8.0 MB + 3.6 MB |
| `yamnet.tflite` | Audio event detection | 4.0 MB |
| `libmlkit_google_ocr_pipeline.so` | OCR | 10 MB |

APK: `app-release.apk` 291 MB; installed code path 279 MB.

---

## 3. Prompt and context

| Item | Tokens |
|---|---|
| **System prompt** (`Sys` in `UiPerfMetrics`) | **1 306** (constant across all 21 logged turns) |
| Per-query dynamic block (telemetry + date/time + location + language directive + filtered `[Available Tools]` + query) | ~200–390 (logged as "Prompt Size: 807–1550 chars ≈ 201–387 tokens") |
| Total context at generation (`Ctx`) | 1 403–1 716 (mean 1 510 over 20 local turns) |
| Context window | 4 096 → **~2 400–2 700 tokens of headroom** for multi-turn history before `KvCacheCompactionManager` must trim |
| Typical output | ~17 tokens ("Estimated Output Tokens") |
| Tools in registry (`vehicle_skills_registry.json`) | 130 (`CUSTOM_KOTLIN` + `GENERIC_VHAL_WRITE`); only 3–5 are injected per query by domain routing |

> The code comment in `LLMManager.kt` ("system prompt (2701 tokens)") is stale; the measured
> value is 1 306.

---

## 4. Latency and throughput

### 4.1 Voice path (`AssistantSession`), 20 local turns from this boot's logcat

| Metric | Mean | Median | Min | Max |
|---|---|---|---|---|
| TTFT (ms) | **1 110** | 1 070 | 928 | 1 365 |
| Decode speed (tokens/s) | **24.0** | 23.4 | 10.4 | 38.1 |
| Context tokens | 1 510 | — | 1 451 | 1 595 |

One cloud-routed turn (hybrid cascade, `Route=Cloud`): TTFT 1 267 ms, 82.9 tok/s, total 1 472 ms.

### 4.2 Injected test queries (`LocalLLMActivity` → `TEST_QUERY`), measured with the sampler running

| # | Query | Prompt Δ (tok) | Ctx | TTFT (ms) | tok/s | Gen total (ms) | Tool time |
|---|---|---|---|---|---|---|---|
| 1 | "what is the cabin temperature right now" | 345 | 1 668 | 1 766 | 31.7 | 2 303 | `getTemperature` 6 ms |
| 2 | "turn on the AC and set the temperature to 70" | 279 | 1 612 | 1 228 | 21.5 | 2 485 | VHAL write 67 ms, `setTemperature` 176 ms |
| 2b | agentic observation turn after the tools | 201 | 1 524 | 1 308 | 23.2 | 2 040 | — |
| 3 | "tell me a fun fact about electric cars" | 387 | 1 716 | 2 389 | 19.8 | 3 550 | — |

Observations:

- **TTFT is ~1.1 s on the voice path and 1.2–2.4 s on the dashboard path.** `CLAUDE.md`/`AGENTS.md`
  budget is **< 600 ms**. The gap is prefill of the ~200–390 dynamic tokens on the GPU; the
  1 306-token system prompt is already in the KV cache from prewarm.
- Longer dynamic blocks cost more: query 3 (387 tokens) took 2.4 s TTFT vs 1.2 s for query 2 (279 tokens).
- An action query that calls tools runs **two** generations (tool call + observation), so wall time
  to the spoken result is ~4.5 s even though each generation is ~2.3 s.
- Decode speed of 20–30 tok/s is for ~17-token outputs; short outputs under-report the steady-state
  rate because the first tokens include speculative-decoding warm-up.
- Decode TPS across turns varies 10–38 tok/s. The low end (10.4) coincides with thermal status 1.

---

## 5. Process resources

### 5.1 Memory (`dumpsys meminfo com.poc.assistant`, app idle with model loaded)

| Category | PSS | Notes |
|---|---|---|
| **Total PSS** | **2.74 GB** (2 741 880 kB) | ≈ 36–38 % of device RAM |
| Graphics (`GL mtrack`) | 1.88 GB | GPU buffers: transformer weights + KV cache + activations |
| Native heap | 541 MB (762 MB reserved) | LiteRT-LM host-side allocations, Vosk, TFLite |
| Other mmap | 244 MB | model file / cache mmaps |
| Code (.so/.dex/.oat/.apk) | 43 MB PSS (153 MB RSS) | |
| Java heap | 14.7 MB (19 MB reserved) | |
| Stack | 1.3 MB | 48 threads |
| Swap | 21 MB | |
| `VmHWM` (peak RSS this process lifetime) | 3.09 GB | |

GPU memory owned by the process (`dumpsys gpu`): **1.914 GB** of 2.643 GB global GPU allocation.

Under inference RSS rose only from 2 678 MB → 2 768 MB (+90 MB); the working set is essentially
static once the model is loaded.

### 5.2 CPU

Measured as scheduler ticks/s for the PID (100 Hz, so 100 = one full core), 1 s samples:

| State | App CPU | Which threads |
|---|---|---|
| Idle, model loaded, wake-word listening | **25–35 %** of one core (`top`: 21–28 %) | `DefaultDispatcher` worker (Vosk decode loop) 16.5 %, main thread 4.3 %, RenderThread 0.5 % |
| During prefill + decode | **50–100 %** of one core (peaks 83, 80, 102 ticks/s) | LiteRT GPU driver thread + tokenizer/callback thread |
| System-wide idle | 800 % cpu capacity, ~660–690 % idle | |

The LLM itself runs on the GPU; the CPU cost during inference is the OpenCL driver, sampling, and
the per-token callback chain. The always-on ~0.3 core is wake-word listening, not the LLM.

### 5.3 GPU

| State | Mali utilisation (`/sys/class/misc/mali0/device/utilization`) | GPU clock |
|---|---|---|
| Idle (launcher + overlay compositing) | noisy 0–80 % (system-wide counter; SurfaceFlinger/launcher dominate) | 302–471 MHz |
| During prefill + decode | **99–100 %** sustained for the whole generation | **848 MHz (max)** pinned |
| Between generations | drops back within 1 s | returns to 351–471 MHz |

The utilisation counter is for the whole GPU, so the idle figure is not attributable to the app;
the inference figure is unambiguous because the clock pins at max for exactly the generation window.

### 5.4 CPU frequency behaviour during inference

Big cores (`cpu6`) sit at 2.05–2.19 GHz throughout and do not boost to 2.85 GHz for inference;
little cores stay at 1.2–1.8 GHz. The LLM is GPU-bound; CPU DVFS is driven by UI/wake word.

---

## 6. Storage footprint on device

| Location | Size |
|---|---|
| Installed APK (`/data/app/…/com.poc.assistant-…`) | 279 MB |
| App data (`/data/user/10/com.poc.assistant`, mostly `litert_gpu_cache`) | 3.7 GB (≈ 0.85 GB live, ≈ 3.1 GB stale caches) |
| Model file (external files dir) | 2.59 GB |
| **Total** | **≈ 6.6 GB** (≈ 3.5 GB after clearing stale caches) |

---

## 7. Quick-reference summary

| Metric | Value |
|---|---|
| Model | Gemma 4 E2B instruction-tuned, LiteRT-LM bundle, 2.41 GiB, GPU backend, speculative decoding on |
| Context window | 4 096 tokens |
| System prompt | 1 306 tokens |
| Typical total context | ~1 500 tokens |
| Cold start to ready | ≈ 9.8 s (5.2 s engine + 4.6 s prewarm) |
| TTFT (voice path, median) | 1.07 s (budget 0.6 s) |
| Decode speed (median) | 23 tok/s local; 83 tok/s cloud fallback |
| Full generation, typical | 2.0–2.5 s per generation; ~4.5 s for tool-call + observation |
| RAM (PSS) | 2.74 GB, of which 1.88 GB GPU graphics |
| GPU memory | 1.91 GB |
| CPU idle / inference | ~0.3 core / 0.5–1.0 core |
| GPU during inference | 100 % at 848 MHz |
| Threads | 48 |
| On-disk | 6.6 GB (3.5 GB after cache cleanup) |

---

## 8. How this was measured (reproduce)

```bash
adb root
P=$(adb shell pidof com.poc.assistant)

# Memory
adb shell dumpsys meminfo com.poc.assistant
adb shell dumpsys gpu | grep "Proc $P"
adb shell "cat /proc/$P/status | grep -E 'VmRSS|VmHWM|VmSwap|Threads'"

# CPU (process + per thread)
adb shell "top -b -n 3 -d 2 | grep -E 'poc.assistant|%cpu'"
adb shell "ps -T -p $P -o TID,PCPU,TIME,CMD" | sort -k2 -rn | head

# GPU
adb shell cat /sys/class/misc/mali0/device/utilization /sys/class/misc/mali0/device/cur_freq

# Thermal / power
adb shell "dumpsys thermalservice | grep Temperature"
adb shell "dumpsys battery | grep -E 'level|powered'"

# Model + caches
adb shell ls -la /data/media/10/Android/data/com.poc.assistant/files/
adb shell ls -la /data/user/10/com.poc.assistant/no_backup/litert_gpu_cache/
adb shell "head -c 65536 /data/media/10/Android/data/com.poc.assistant/files/gemma-4-E2B-it.litertlm | strings -n 6 | head -60"

# Latency / tokens (already logged by the app)
adb logcat -d | grep -E "UiPerfMetrics: LLM metrics updated|LLMLatency|LLMMetrics|Prompt Size"

# Drive a query without the mic (needs LocalLLMActivity alive; quote the query inside the shell string)
adb shell am start -n com.poc.assistant/.LocalLLMActivity --user 10
adb shell "am broadcast -a com.poc.assistant.TEST_QUERY --es query 'what is the cabin temperature' --ez reset false --user 10"
```

A 1 Hz sampler that records CPU ticks, RSS, GPU utilisation/clock and CPU clocks for the PID is
what produced §5; it is a 20-line `/system/bin/sh` loop over `/proc/$P/stat`, `/proc/$P/status`
and the sysfs nodes above, pushed to `/data/local/tmp/` and run with `nohup`.



8. DMS vision models (gesture + face)
Static facts from the bundles in app/src/main/assets/ and the code in vision/dms/. Runtime CPU/GPU/latency for this pipeline was not measured; the app does not log per-frame inference time (only init). Input shapes below were read from the TFLite flatbuffers.

8.1 Where it runs
Item	Value
Runtime	MediaPipe Tasks Vision 0.10.14 (libmediapipe_tasks_vision_jni.so, 14 MB)
Delegate	Delegate.GPU first (Mali OpenCL via TFLite GPU delegate); falls back to Delegate.CPU (XNNPACK) if GPU init throws. Same process as the LLM, so both share the Mali-G710.
Running mode	RunningMode.IMAGE (synchronous per frame, not the VIDEO/LIVE_STREAM tracker)
Owner	DriverMoodCaptureService (foreground service) → DmsCameraManager (CameraX, front camera, STRATEGY_KEEP_ONLY_LATEST) → DriverMonitoringManager.processFrame()
Frame rate	Throttled to one AI pass per 100 ms (≤ 10 fps) in both FULL and LITE profiles (DmsCaptureProfile)
Analysis resolution	FULL: camera default; LITE: 480×360
Per-frame work	DriverFaceProcessor (if mood enabled) → HandGestureProcessor (if gestures enabled) → HealthRppgProcessor (CPU, if rPPG enabled), on a single coroutine
Enabled by	SET_DMS_PIPELINE broadcast / app_prefs (gestures, mood), camera sensor privacy gate (DmsCameraSensorGate)
Not running by default	Camera and both models are only loaded while the capture service is up; released when the sensor is turned off
8.2 Gesture model: gesture_recognizer.task (8.0 MB on disk)
A MediaPipe bundle of four TFLite models chained inside the task runtime:

Stage	File	Size	Input	Output
1. Palm detection	hand_detector.tflite	2.34 MB	[1,192,192,3] f32	palm boxes (SSD anchors)
2. Hand landmarks	hand_landmarks_detector.tflite	5.48 MB	[1,224,224,3] f32 (crop)	21 landmarks ×3 + handedness + presence
3. Gesture embedder	gesture_embedder.tflite	0.55 MB	hand[1,21,3], world_hand[1,21,3], handedness[1,1]	128-d embedding
4. Canned classifier	canned_gesture_classifier.tflite	8 KB	[1,128] f32	8 classes
Classes (from the bundled labels.txt): None, Closed_Fist, Open_Palm, Pointing_Up, Thumb_Down, Thumb_Up, Victory, ILoveYou. Stages 1–2 are image CNNs and are what the GPU delegate accelerates; 3–4 are tiny MLPs over landmarks. Stage 1 only re-runs when no hand is tracked from the previous frame.

App-side post-processing (HandGestureProcessor, DmsVictorySignPolicy, DmsMediaGestureArbiter):

Setting (dms_gesture_config.json)	Value
minConfidenceScore	0.45
debounceMs	800
Mappings	Pointing_Up→wake, Thumb_Up/Thumb_Down→volume, Open_Palm→pause, Victory→play, ILoveYou→favorite, Closed_Fist→none
Synthetic gestures from landmark trajectory (not from the model)	Swipe_Left/Swipe_Right → prev/next track, Pinch → mute, Two_Fingers_* → prev/next
8.3 Face model: face_landmarker.task (3.6 MB on disk)
Stage	File	Size	Input	Output
1. Face detection (BlazeFace short-range)	face_detector.tflite	0.23 MB	[1,128,128,3] f32	face boxes
2. Face mesh	face_landmarks_detector.tflite	2.55 MB	[1,256,256,3] f32 (crop)	478 landmarks
3. Blendshapes	face_blendshapes.tflite	0.96 MB	[1,146,2] f32 (landmark subset)	52 blendshape scores
setNumFaces(2) (driver + passenger), so stages 2–3 run up to twice per frame. The app reads blendshapes (eyeBlink*, browDown*, mouthFrown*, …) for drowsiness, distraction/gaze and emotion (thresholds in dms_gesture_config.json), and the cheek landmark feeds rPPG.

8.4 Memory
Component	Size	Basis
Gesture model weights (4 files)	8.0 MB	on disk, loaded via AssetManager mmap
Face model weights (3 files)	3.6 MB	on disk
Input tensors per frame (f32)	0.44 MB (192²) + 0.60 MB (224²) for hands; 0.20 MB (128²) + 0.79 MB (256²) per face	shape × 4 bytes
Analysis frame (LITE 480×360 RGBA bitmap + MPImage copy)	~0.7 MB × 2	resolution × 4 bytes
Runtime working set (GPU delegate programs/buffers + activations + MediaPipe graph)	not measured; typically tens of MB for this pair of task bundles	—
For comparison, the LLM holds ~1.9 GB of GPU memory, so the DMS models are < 1 % of the app's footprint. The visible cost of DMS is CPU/GPU time at up to 10 fps and the camera HAL buffers, not memory. Measure it with the same sampler as §5 while toggling SET_DMS_PIPELINE once a device is attached.
