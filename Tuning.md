# Fine-Tuning Strategy for AAOS AI Voice Assistant (Gemma 4)

## Executive Summary
When deploying Large Language Models (LLMs) like **Google Gemma 4** on **Android Automotive OS (AAOS)**, automakers must avoid the pitfall of "training use cases." Hardcoding specific user intents into model weights creates brittle, unscalable software that behaves like a glorified `if/else` tree.

Instead, fine-tuning should focus on **capabilities, API schemas, and behavioral constraints**, relying on **Retrieval-Augmented Generation (RAG)** and **Dynamic Function Calling** for execution at runtime.

---

## 1. What to Train vs. What NOT to Train

```
                  ┌──────────────────────────────────────────────┐
                  │          AI Assistant Architecture           │
                  └──────────────────────┬───────────────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
   ┌───────────────────────────┐                   ┌───────────────────────────┐
   │    DO TRAIN (Fine-Tune)   │                   │    DO NOT TRAIN (SFT)     │
   ├───────────────────────────┤                   ├───────────────────────────┤
   │ • Schema & Tool Calling   │                   │ • Specific Use Cases      │
   │ • In-Cabin Voice Features │                   │ • Manuals & Vehicle Specs │
   │ • Conciseness & Persona   │                   │ • Static Business Logic   │
   │ • Safety & Refusal Rules  │                   │ • Dynamic External Data   │
   └───────────────────────────┘                   └───────────────────────────┘
```

### A. WHAT TO TRAIN (Generalizable Capabilities)

Focus fine-tuning on structural understanding, domain grammar, and behavioral boundaries:

1. **API & Tool-Calling Proficiency (Schema Adherence):**
   * *Target:* Teach the local model (Gemma 4 E2B/E4B) how to output strict, valid JSON or C++ structures that directly interface with AAOS `CarPropertyManager` and `CarInputManager`.
   * *Example:* Mapping arbitrary natural phrasing (*"My back is cold"*, *"Warm up my seat"*, *"It's freezing in row 1"*) into the correct property signature (`HVAC_STEERING_WHEEL_HEAT` / `HVAC_SIDE_HEATING`).

2. **Acoustic & In-Cabin Speech Adaptations:**
   * *Target:* Fine-tune native multimodal audio layers on audio recorded in real driving conditions (road noise, HVAC blowing, windshield wipers, multi-passenger cross-talk).

3. **Concise Spoken Response Formatting:**
   * *Target:* Enforce ultra-short, voice-friendly outputs to comply with automotive driver distraction regulations (e.g., outputting *"Setting temperature to 21°"* instead of long multi-sentence explanations).

4. **Safety Guardrails & System Safety Refusals:**
   * *Target:* Align the model (via Direct Preference Optimization / DPO) to strictly refuse dangerous driver actions (e.g., attempting to turn off headlights or driver assistance features while driving at high speeds).

---

### B. WHAT NOT TO TRAIN (Runtime Data & Hardcoded Logic)

Do **not** bake static facts, dynamic data, or explicit business rules into model weights:

1. **Specific Dialogue / Use-Case Scenarios:**
   * *Why:* Hardcoding scenarios like *"If the user asks for coffee, suggest Starbucks"* is brittle and breaks when phrasing changes. Use function calling to trigger an external POI / Navigation API instead.

2. **Vehicle Owners' Manuals & Technical Specs:**
   * *Why:* Vehicle manuals change per trim level, model year, and market region. Baking manual text into the weights leads to hallucinations and massive model retraining costs. Use **Retrieval-Augmented Generation (RAG)** at runtime instead.

3. **Dynamic Data (Traffic, Battery/Fuel Status, Weather):**
   * *Why:* Real-time data must be injected dynamically into the context window via AAOS Vehicle Hardware Abstraction Layer (VHAL) APIs, not fine-tuned into weights.

4. **Complex Multi-Step Business Logic:**
   * *Why:* Sequential workflows (e.g., payment flows, service appointment bookings) should be managed by a deterministic state machine / backend orchestrator, using the LLM only for intent parsing and dialogue generation.

---

## 2. Deep Dive: Strategy Comparison

| Metric | Training Use Cases / Logic (Anti-Pattern) | Training Capabilities & Schemas (Best Practice) |
| :--- | :--- | :--- |
| **Scalability** | **Low:** Every new vehicle feature requires dataset re-annotation and full retraining. | **High:** Adding new features only requires updating the JSON tool schema provided in the prompt. |
| **Robustness** | **Fragile:** Fails if user input deviates slightly from the trained dialogue examples. | **Resilient:** Generalizes to novel human phrasing and slang because it learned intent extraction, not exact syntax. |
| **Model Size Efficiency**| **Bloated:** Requires larger model capacity to memorize thousands of specific dialogue trees. | **Compact:** Small edge models (e.g., Gemma 4 E2B/E4B) excel when only responsible for intent parsing and schema generation. |
| **Maintenance Cost** | **High:** Constant continuous retraining cycles needed across different vehicle variants. | **Low:** Model weights stay static for long periods; domain logic updates via lightweight software/RAG updates. |

---

## 3. Edge vs. Cloud Training Matrix (Gemma 4 Family)

### A. Edge Tier: Gemma 4 E2B / E4B (In-Vehicle local execution)
* **Hardware Target:** Qualcomm Snapdragon Cockpit (SA8255P / SA8775P) or NVIDIA DRIVE.
* **Tuning Method:** Supervised Fine-Tuning (SFT) on QLoRA + 4-bit/8-bit Quantization (AWQ/INT8).
* **Primary Objective:** Ultra-low latency ($\le 200\text{ms}$), offline availability, and zero-hallucination vehicle control schema generation.

### B. Cloud Tier: Gemma 4 26B / 31B (Remote processing layer)
* **Hardware Target:** Enterprise GPU Clusters / Cloud Infrastructure.
* **Tuning Method:** LoRA + RAG Pipeline + Direct Preference Optimization (DPO).
* **Primary Objective:** Complex reasoning, multi-turn travel planning, ambient manual lookup, and contextual safety validation.

---

## 4. Key Takeaways & Recommendations

1. **Treat the Model as a Compiler:** The local LLM should act as a natural language compiler that translates messy human speech into structured, deterministic code (JSON/APIs) for AAOS.
2. **Decouple Data from Model Weights:** Use **RAG** for vehicle documentation and **APIs (VHAL/CAN)** for real-time telemetry.
3. **Keep Logic in Code, Intelligence in Weights:** Let standard Android/Java/C++ code handle state management, conditional flows, and vehicle rules. Let the AI handle understanding and expression.
