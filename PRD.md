# Product Requirements Document
## Circuit Diagnostic AI — "The AI Lab Mentor"

**Version:** 1.1 (revised based on AI Solutions Architect review)
**Author/Owner:** Priya, 3rd-year ECE student
**Builder profile:** No-code/low-code builder using Flowise; beginner programmer; ₹0 budget
**Target outcome:** One exceptional, demo-able, completable GitHub project in 2–3 months

---

## 1. Engineering Philosophy (Read This First)

Circuit Diagnostic AI does not attempt to be the smartest AI. It attempts to be the most systematic electronics troubleshooter.

The AI behaves like an experienced laboratory engineer:

> **Observe → Ask → Measure → Eliminate → Conclude → Explain → Document**

That sequence is the entire product. Everything else — the RAG pipeline, the memory system, the report generator — exists to support those seven verbs.

**The LLM is 20% of this project. The other 80% is the engineering workflow.**

Most students build "AI assistants" — a chat box that calls an LLM. What recruiters and professors notice is an "AI workflow" — a system with structure, memory, rules, and a process that would exist even if you swapped the underlying model.

The architecture of Circuit Diagnostic AI follows from this directly:

- The AI does not answer freeform questions. It runs a diagnostic process.
- The AI does not chat. It asks one targeted question at a time and waits.
- The AI does not respond from training data alone. Every technical claim must be retrieved from real documentation.
- The AI does not forget. It maintains structured project state across the session.
- The AI does not just fix problems. It explains the engineering principle behind every recommendation.
- The AI does not end with a suggestion. It ends with a structured professional report.

---

## 2. Engineering Design Principles

These four principles govern every prompt, every flow node, and every output in the system. When in doubt, return to these.

- **Principle 1 — Never guess hardware specifications. Always retrieve from documentation.** If no relevant chunk is retrieved, say so explicitly. A wrong datasheet number damages a circuit. A disclosed uncertainty does not.
- **Principle 2 — Always explain WHY.** Never provide an answer without the engineering reasoning behind it. Not "replace the resistor" — but "replace the 220Ω resistor with 330Ω because at 3.3V, Ohm's Law gives I = 15mA, which exceeds the ESP32 GPIO source limit of 12mA."
- **Principle 3 — Every diagnosis must end with four things.** Root Cause / Evidence that supports it / Confidence level / Recommended confirming test before applying the fix. A diagnosis without these four fields is incomplete.
- **Principle 4 — If uncertain, ask. Never invent.** When the evidence does not yet support a conclusion, ask another targeted diagnostic question. Expressing calibrated uncertainty is correct engineering behaviour.

---

## 3. Diagnostic Reasoning Rules

These rules govern the AI's behaviour during the diagnostic loop. They are implemented as system prompt constraints, not optional guidelines.

- **Rule 1 — Minimum evidence gate.** Never conclude a root cause before asking at least 3 diagnostic questions. A symptom description alone is never sufficient for diagnosis.
- **Rule 2 — Power before hardware.** Never recommend replacing a component before verifying the power supply is stable and within spec. Power supply faults cause the majority of apparent hardware failures.
- **Rule 3 — Software before hardware.** Always eliminate software causes (wrong library, wrong I2C address, uninitialized peripheral, missing delay) before assuming physical hardware damage.
- **Rule 4 — Measurement before speculation.** When two hypotheses are equally plausible, recommend a specific measurement (voltage, continuity, I2C scan) that would distinguish between them. Do not speculate when evidence is obtainable.

---

## 4. Problem Statement

When a circuit misbehaves, engineers and students fail in two ways:

1. **Unstructured trial-and-error** — swapping components, re-flashing firmware, and guessing without a systematic hypothesis-elimination process.
2. **Generic AI assistance** — existing LLMs answer one-shot questions but have no memory of the board, no awareness of the real datasheets, and no diagnostic workflow. Every question resets to zero.

No lightweight tool combines guided clinical-style questioning, documentation grounding, and persistent session context into a single diagnostic workflow — especially for ECE students and makers.

---

## 5. Target Users

- ECE/EEE undergraduates debugging coursework and personal projects (primary)
- Embedded systems hobbyists and makers working with ESP32
- Lab teaching assistants who want a structured "first-line diagnostic tool" before student escalation
- (Future) Early-career embedded engineers doing board bring-up

---

## 6. MVP Scope

Version 1 contains exactly **6 core features + 3 supporting features + 6 workflow enhancements**.

Nothing outside this list is built for v1, regardless of how good the idea is. The out-of-scope list is in Section 16.

---

## 7. Diagnostic State Machine (Core Architecture)

Every feature either feeds into or is produced by this state machine. Implement this first. This is the feature to brag about — it is the thing that separates this project from "AI chatbot."

Every diagnostic session moves through exactly these states in order. The system knows which state it is in at all times, and each state has defined entry conditions, actions, and exit conditions.

There is one AI. Everything else is engineering workflow structure built around it in Flowise. The state machine is implemented via a combination of system prompt state tracking and Flowise memory nodes that record the current state name alongside the session history.

| State | Entry | Action | Exit |
|---|---|---|---|
| **1. Initialization** | User opens session | Collect board, components, power supply, description of symptom | Project Memory populated → State 2 |
| **2. Problem Collection** | Memory initialized | Ask clarifying questions about the symptom until symptom is unambiguous | Symptom clearly defined → State 3 |
| **3. Evidence Gathering** | Symptom defined | Ask ONE targeted question per turn, retrieve relevant RAG chunks, update hypothesis scores after each answer. **Rule: minimum 3 questions before proceeding.** | ≥3 questions answered AND (one hypothesis >85% OR turn limit reached) → State 4 |
| **4. Hypothesis Elimination** | Evidence gathered | Explicitly list active hypotheses with scores; identify what evidence has ruled out each dropped hypothesis | Top hypothesis identified → State 5 |
| **5. Measurement Request** | Leading hypothesis identified | Request specific confirming measurement (voltage, I2C scan, continuity check) that would confirm or rule out the leading hypothesis | User provides measurement result → State 6, OR user cannot measure → skip to State 6 |
| **6. Root Cause Identification** | Measurement result received (or skipped) | State root cause, evidence trail, confidence score, and engineering rationale. Must include all four fields from Design Principle 3. | Root cause stated → State 7 |
| **7. Fix Recommendation** | Root cause identified | Provide step-by-step fix with engineering reasoning for each step. Include prevention tip for future. | Fix provided → State 8 |
| **8. Learning Summary** | Fix provided | Summarise the engineering principle demonstrated. Link to relevant datasheet sections and documentation. Optionally ask: "Why do you think this happened?" | Summary complete → State 9 |
| **9. Diagnostic Report Generation** | Learning summary complete | Assemble all session data into the structured Diagnostic Report (Feature 6). Export as Markdown / PDF. | Report delivered → Session complete |

---

## 8. Core Features (Must-Have for MVP)

### Feature 1 — Guided Diagnostic Mode ⭐⭐⭐⭐⭐
This is the USP. It gets 40% of total implementation effort. Without it, this is just another RAG chatbot.

**What it does:** Runs an adaptive, state-machine-style diagnostic questioning loop. The AI asks exactly one question at a time, chosen based on all previous answers, to narrow down the root cause.

**Requirements:**
- Must ask exactly one question per turn. Never dumps a list of questions. Never says "check these 5 things."
- Must maintain an internal hypothesis set that updates after every answer. Hypotheses the user's answers rule out must be dropped.
- Must explain on request why it is asking a particular question: "I'm asking about your power source because brownout resets are the most common cause of random ESP32 reboots, per the technical reference manual."
- Must know when to stop: if confidence in one root cause is high, stop and move to reporting. If the turn limit (8–10 questions) is reached without resolution, honestly report the narrowed-but-unconfirmed hypothesis set.
- Must never fabricate a confident diagnosis when evidence is insufficient. Stating uncertainty is a design value, not a bug.
- Every substantive claim must reference the datasheet/documentation (Feature 2). The diagnostic loop and RAG are inseparable.

### Feature 2 — Datasheet-Aware RAG ⭐⭐⭐⭐⭐
Ground every claim in real documentation, not model hallucination.

**What it does:** Every technical statement the AI makes during a diagnostic session is retrieved from a curated knowledge base of real datasheets and reference documents, not generated from training memory.

**Knowledge base for v1 (ESP32 focus only):**
- ESP32 Datasheet
- ESP32 Technical Reference Manual
- Arduino-ESP32 documentation
- DHT22 datasheet
- OLED (SSD1306) datasheet
- Relay module reference
- Common ESP32 error log signatures and their documented causes

Do not add STM32, Arduino Uno, or Raspberry Pi Pico documentation for v1. Architecture should allow it later (separate collection/namespace per board) but adding the content is explicitly deferred.

**Requirements:**
- Relevant document chunks must be retrieved before generating any substantive technical claim.
- Responses drawing on retrieved content should indicate the source (e.g., "Per ESP32 TRM Section 4.1.2...").
- If no relevant chunk is retrieved for a claim, the AI must flag this: "This is based on general knowledge, not retrieved documentation." It must not silently present ungrounded claims.

### Feature 3 — Project Memory ⭐⭐⭐⭐⭐
The AI never asks the same question twice. It remembers everything about the session.

**What it does:** Stores and reads a structured profile of the user's project throughout a session (and ideally across sessions).

**Structured project state schema** (this is the engineering state, not a chat log):

```json
{
  "project": {
    "board": "ESP32 DevKit V1",
    "power_source": "USB 5V",
    "components": [
      { "name": "SSD1306 OLED", "interface": "I2C", "address": "0x3C" },
      { "name": "DHT22", "interface": "GPIO", "pin": 4 },
      { "name": "Relay module", "interface": "GPIO", "pin": 5 }
    ],
    "libraries": ["Adafruit SSD1306", "DHT sensor library"],
    "current_symptom": "OLED blank on power-up"
  },
  "diagnostic_state": {
    "current_state": "EVIDENCE_GATHERING",
    "questions_asked": 3,
    "hypotheses": [
      { "label": "Missing I2C pull-up resistors", "confidence": 0.82 },
      { "label": "Wrong I2C address", "confidence": 0.61 },
      { "label": "Incorrect SDA/SCL pins", "confidence": 0.34 }
    ],
    "eliminated_hypotheses": [
      { "label": "Power supply fault", "eliminated_by": "VIN measured at 4.98V, within spec" }
    ]
  },
  "measurements": [
    { "point": "VIN", "value": "4.98V", "timestamp": "10:17" },
    { "point": "3V3", "value": "3.29V", "timestamp": "10:17" },
    { "point": "GPIO21 (SDA)", "value": "3.3V", "timestamp": "10:19" }
  ],
  "tests_completed": [
    { "test": "Voltage check at VIN and 3V3", "result": "PASS" },
    { "test": "I2C scanner sketch", "result": "PASS — 0x3C detected" },
    { "test": "Pull-up resistor check", "result": "FAIL — not installed" }
  ],
  "session_timeline": [
    { "time": "10:15", "event": "Session started — OLED blank" },
    { "time": "10:17", "event": "Power supply confirmed stable" },
    { "time": "10:19", "event": "I2C address confirmed 0x3C" },
    { "time": "10:22", "event": "Pull-up resistors identified as missing" }
  ]
}
```

**Requirements:**
- The initialization state (State 1) populates the project fields before diagnostics begin.
- `hypotheses` list is updated (scores adjusted, entries eliminated with evidence) after every user answer.
- `measurements` and `tests_completed` are append-only; they are never overwritten.
- Minimum: persists for the duration of one session. Target: persists across sessions so a user returning days later has their project context intact.
- Flowise memory nodes are the primary implementation mechanism; the schema above is the structure stored in memory.

### Feature 4 — Component Compatibility Checker ⭐⭐⭐⭐
Catch wiring mistakes before they damage hardware.

**What it does:** Given a proposed or reported component connection, checks compatibility against datasheet specs and flags mismatches with engineering reasoning.

**Examples of what it catches:**
- "I connected relay to GPIO34" → GPIO34 is input-only on ESP32. It cannot source current to drive a relay. Use GPIO5 or another output-capable pin.
- "I'm using AMS1117 3.3V with a Li-ion battery" → AMS1117 needs >1V dropout. Li-ion drops to ~3.0V at low charge, which can cause instability. A buck converter is more appropriate.
- "I have a 5V sensor connected to ESP32 GPIO" → ESP32 GPIO is not 5V tolerant. This will damage the GPIO pin. Use a voltage divider or level shifter.

**Requirements:**
- Must pull relevant specs (voltage ratings, current limits, logic levels, pin capabilities) from the RAG knowledge base for both components being compared.
- Must produce: verdict (compatible / incompatible / compatible with caveats) + specific numeric reasoning + recommended fix if incompatible.
- Can be invoked standalone (a direct compatibility question) or triggered automatically by the Guided Diagnostic Mode when it suspects a compatibility-related fault.

### Feature 5 — Error Log Analyzer ⭐⭐⭐⭐
Turn a wall of serial monitor text into a structured diagnosis.

**What it does:** Accepts pasted error logs or serial monitor output and identifies likely causes with documentation-grounded explanations.

**Recognized signature classes:**
- Guru Meditation Error (core panic — distinguish firmware vs hardware causes)
- `rst:0x...` (BROWNOUT_RST) — power supply undervoltage
- `rst:0x...` (TG1WDT_SYS_RESET) — watchdog timer reset
- `Backtrace:` — stack trace (identify null pointer / stack overflow patterns)
- `E (WIFI)` prefixed messages — Wi-Fi initialization and connection failures
- Stack canary watchpoint triggered — stack overflow
- `rst:0x...` codes generally — map to their documented meanings

**Requirements:**
- Log input is freeform pasted text. The AI parses it without requiring structured input.
- Recognized signatures are explained in plain English with the probable cause and relevant datasheet/TRM reference.
- Log analysis output feeds directly into the Guided Diagnostic Mode as evidence — it narrows the hypothesis set rather than existing as a separate isolated feature.

### Feature 6 — Professional Diagnostic Report Generator ⭐⭐⭐⭐⭐
The artifact a recruiter, professor, or lab partner will actually read.

**What it does:** At the end of every diagnostic session, produces a clean, structured, exportable report covering the complete diagnostic trail.

**Report sections:**
1. Problem Statement — symptom as reported by user
2. Project Context — board, components, power supply
3. Diagnostic Questions & Answers — full Q&A record
4. Hypotheses Considered — all hypotheses, including ruled-out ones and the evidence that eliminated them
5. Root Cause Identified — primary finding with confidence level
6. Confirming Test(s) — specific test(s) to verify the diagnosis before applying the fix
7. Recommended Fix — step-by-step resolution
8. Engineering Rationale — why the fault produces the observed symptom (engineering principle)
9. Prevention Tips — how to avoid this class of fault in the future
10. Documentation References — datasheets and sections cited during the session

**Requirements:**
- Report reads as a coherent document, not a raw chat transcript dump.
- Exportable as Markdown (minimum) and PDF (target).
- Clean formatting: this is what gets shared, screenshotted, and put in a GitHub README.

---

## 9. Supporting Features (Included in MVP)

**Supporting Feature A — Engineering Reasoning (WHY, always)**
Every recommendation must explain the underlying principle. This is a system-level prompt engineering constraint that applies everywhere, not a separate feature.
- Never: "Replace the resistor."
- Always: "Replace the 220Ω resistor with 330Ω. At 3.3V supply, Ohm's Law gives I = V/R = 3.3/220 ≈ 15mA, which exceeds the ESP32 GPIO max source current of 12mA. 330Ω limits current to 10mA, within spec."

**Supporting Feature B — Confidence Score**
Display a ranked hypothesis list with simple percentage confidence scores during the diagnostic session. Example:

```
Current hypotheses:
● GPIO output current exceeded — 91%
● Power supply undervoltage — 72%
● Loose ground connection — 53%
```

Scores update after each user answer. This makes the diagnostic reasoning visible and teaches users to think probabilistically. Implementation: a structured field in the AI's response that the Flowise output parser extracts and displays separately.

**Supporting Feature C — Learning Resources**
At session end (appended to the Diagnostic Report), automatically include:
- Relevant datasheet sections to read
- Relevant ESP-IDF or Arduino-ESP32 documentation pages
- Example projects or code snippets addressing the identified root cause

This transforms each debugging session into a mini learning event, which aligns with the educator/student use case.

---

## 10. Workflow Enhancement Features (Also MVP)

These six features are Flowise-friendly, low-code to build, and dramatically improve the perceived polish and utility of the product.

**WF1 — Troubleshooting Tree (Visual Diagnostic Map)**
Instead of a pure chat thread, maintain a visible tree/flowchart of the diagnostic path taken, e.g.:

```
Symptom: OLED Blank
├─ Power? → ✓ 3.3V confirmed
├─ I2C Address? → ✓ 0x3C confirmed
├─ Pull-ups? → ✗ Missing — LIKELY CAUSE
└─ Library Init? → Not yet checked
```

Shows the user exactly where in the diagnostic process they are. Makes the structured reasoning visible rather than implicit. Can be implemented as a structured output field the frontend (Flowise widget) renders.

**WF2 — Diagnostic Checklist**
At appropriate points in the diagnostic loop, the AI generates a checklist of physical verifications the user should do, not just questions to answer, e.g.:

```
Before we continue, please verify these:
☐ Measure VIN at the ESP32 3V3 pin (expect 3.3V ± 0.1V)
☐ Confirm SDA wire is connected to GPIO21
☐ Confirm SCL wire is connected to GPIO22
☐ Confirm 4.7kΩ pull-up resistors are on both SDA and SCL lines
```

User confirms each item. The checklist completion state feeds back into memory. This bridges the gap between "AI asks questions" and "user needs to go back to the bench."

**WF3 — Board Profile (ESP32-Specific Knowledge)**
Before any diagnostic session, load a structured ESP32 board profile that the AI uses as ground truth for GPIO questions, ADC constraints, and peripheral restrictions.

ESP32 board profile includes:
- Output-capable GPIOs (vs input-only: GPIO34, 35, 36, 39)
- ADC-capable pins and which ADC channels they map to
- Strapping pins that affect boot if pulled high/low (GPIO0, 2, 12, 15)
- I2C defaults (GPIO21 = SDA, GPIO22 = SCL in Arduino)
- SPI defaults (GPIO23 = MOSI, GPIO19 = MISO, GPIO18 = CLK)
- UART0 defaults (GPIO1 = TX, GPIO3 = RX)
- Pins with ADC interference during Wi-Fi use
- GPIO current limits (12mA source, 20mA sink per pin; 1.2A total chip GPIO)
- Known strapping pin gotchas (GPIO12 high at boot → flash voltage failure on some modules)

This profile is loaded as context on every session. The diagnostic AI consults it when answering GPIO-related questions. It is a JSON/structured config, not a freeform document — easy to update and easy to add new boards to later.

**WF4 — Common Failure Library (RAG-Backed)**
A curated collection of 100+ documented ESP32 and peripheral failure patterns stored in the vector knowledge base. Examples:
- ESP32 brownout on USB power (cause: USB cable resistance + simultaneous Wi-Fi + motor draw)
- OLED blank despite correct wiring (cause: missing I2C pull-up resistors or wrong I2C address)
- Relay chattering (cause: back-EMF without flyback diode; or inadequate current from GPIO)
- Servo jitter (cause: PWM interference, or power not separated from logic)
- Wi-Fi fails to connect (cause: RSSI too low, wrong auth mode, or RTC memory corruption from prior crash)
- DHT22 reading -999 or NaN (cause: too-short delay after initialization, or missing pull-up)
- GPIO36/39 reading noise (cause: ADC2 unusable during Wi-Fi — known ESP32 limitation)

Each entry includes: symptom description, common causes, diagnostic questions to ask, confirming test, fix, datasheet reference. These entries are RAG-retrievable and feed directly into the Guided Diagnostic Mode.

**WF5 — Lab Mode (Step-by-Step Measurement Protocol)**
A special session mode where instead of asking questions in chat, the AI walks the user through a physical measurement protocol one step at a time — and the user enters measurement results that become diagnostic evidence. Example:

```
Lab Mode: Power Supply Investigation
Step 1 of 5
Set your multimeter to DC voltage mode.
Measure the voltage at the ESP32 VIN pin relative to GND.
What reading do you get?
[User enters: 4.8V]

Step 2 of 5
Good. 4.8V is within acceptable range for the onboard 3.3V regulator.
Now measure at the 3V3 pin (the regulated output).
What do you get?
[User enters: 2.9V]

⚠️ 2.9V is below the 3.3V nominal. This confirms the 3.3V regulator is under
stress — likely from excessive load current. Let's investigate what's drawing
the current...
```

Lab Mode is invoked automatically when the diagnostic state identifies a power or signal integrity issue that requires physical measurement to resolve.

**WF6 — Session Timeline**
Displays a running timestamped log of the diagnostic session's key events, e.g.:

```
10:15 Session started — OLED blank, I2C, SSD1306
10:17 Power confirmed (3.3V at 3V3 pin)
10:19 I2C address confirmed (0x3C via scanner sketch)
10:22 Pull-up resistors: not installed — identified as likely cause
10:24 Fix applied: 4.7kΩ resistors added to SDA and SCL
10:25 OLED initialized successfully — RESOLVED
```

Easy to implement as a structured append-only memory field. Makes the session feel professional. The timeline is included in the Diagnostic Report automatically.

---

## 11. Technical Stack (Strictly Constrained)

Use only this stack for v1:

| Component | Tool | Rationale |
|---|---|---|
| Orchestration & flow | Flowise | No-code visual builder; handles agents, chains, memory, retrieval |
| LLM | Gemini free tier (or equivalent free API) | ₹0 budget |
| Vector store | Flowise built-in vector store or free-tier Chroma/Qdrant | Free, Flowise-native integration |
| Memory | Flowise conversation memory nodes | No additional setup |
| Knowledge base | Curated PDFs loaded via Flowise document loaders | No custom code |
| Frontend | Flowise embedded chat widget | No frontend build required |
| Report export | Markdown (Flowise output); manual PDF export at MVP | Simple, no dependencies |

**Explicitly removed from v1 stack:**
- React / Next.js — unnecessary; Flowise widget is sufficient
- FastAPI / Node.js — unnecessary; Flowise handles orchestration
- PostgreSQL — unnecessary for MVP; Flowise memory handles session context
- ChromaDB (self-hosted) — use Flowise-integrated vector store instead
- OCR / Vision models — deferred entirely to v3
- Any cloud service with a paid tier requirement

If an implementation decision requires choosing between a simpler/more-manual approach and a more complex/automated approach, always choose simpler for v1.

> **Note (actual build):** The live implementation swapped Gemini for **Groq (`llama-3.3-70b-versatile`)** after hitting a Gemini free-tier quota bug, and added **Cohere Embeddings + Pinecone** as the working RAG pairing after Gemini Embeddings and HuggingFace Inference embeddings both failed. See the [Phase 1](./build-logs/PHASE1_LOG.md) and [Phase 3](./build-logs/PHASE3_LOG.md) build logs for the full troubleshooting record. The PRD's own §20 instructions anticipate exactly this kind of substitution ("never recommend a paid service, always propose a free alternative").

---

## 12. Knowledge Base Build Plan

Build the knowledge base in this order. Do not proceed to a later tier until the prior tier is loaded and verified:

**Tier 1 (build first — core diagnostic capability):**
- ESP32 Datasheet (official Espressif PDF)
- ESP32 Technical Reference Manual (official Espressif PDF — focus on GPIO, Power, Boot, ADC chapters)
- ESP32 Arduino core documentation

**Tier 2 (add next — peripheral support):**
- SSD1306 OLED datasheet
- DHT22 / AM2302 datasheet
- Generic relay module reference (HiLink or equivalent)

**Tier 3 (add last before v1 completion):**
- Common ESP32 failure pattern library (manually authored, structured — see WF4)
- Common ESP32 error log signatures with documented causes

Do not add STM32, Arduino Uno, Raspberry Pi Pico, or other board documentation in v1. The architecture (separate collection per board) allows adding them later without modifying the diagnostic engine.

---

## 13. Success Criteria

**Qualitative (demo-readiness)**

The MVP is complete and demo-ready when a user can:

1. Describe an ESP32 symptom in plain language and be walked through an adaptive diagnostic sequence — one question at a time, not a static FAQ.
2. Receive at least one visible, cited datasheet-grounded explanation during the session.
3. Have the system remember their board and component context without re-asking already-answered questions.
4. Paste a serial log and receive a correct identification of at least the five most common ESP32 boot/reset error signatures.
5. Ask a GPIO compatibility question and receive a spec-based verdict with numeric reasoning.
6. Complete a session and receive an exported, well-formatted Diagnostic Report.
7. See confidence scores update as the session progresses.
8. Demo the full flow end-to-end in under 10 minutes for a recruiter or professor.

**Quantitative (engineering benchmark)**

The system must successfully diagnose the correct root cause in at least **80% of the 50 curated ESP32 test scenarios** (Section 14) using guided questioning, without producing any unsupported hardware claims not backed by retrieved documentation.

This is the benchmark. It is measurable, repeatable, and gives the project an engineering evaluation standard that most student AI projects never define.

---

## 14. Testing Plan

Build this test set before or alongside the system, not after. It is the only way to know if the diagnostic engine is actually improving. Aim for **50 scenarios** before launch.

Each scenario is a card with: symptom description the user types in / correct root cause / the expected diagnostic path the AI should take / pass/fail criteria.

### Tier 1 — Power & Boot (10 scenarios, highest priority)

| # | Symptom Input | Expected Root Cause | Pass Criteria |
|---|---|---|---|
| T1 | "ESP32 keeps resetting every few seconds when I connect the motor" | Brownout — motor inrush draws excessive current | Identifies power as likely cause within 3 questions; cites TRM brownout section |
| T2 | "ESP32 won't boot, blue LED flickers once and dies" | GPIO0 pulled LOW (boot mode conflict) | Identifies strapping pin issue; cites GPIO0 boot mode behaviour |
| T3 | "ESP32 resets exactly when Wi-Fi connects" | USB cable resistance + Wi-Fi TX current spike causing brownout | Identifies power supply insufficiency and Wi-Fi current draw |
| T4 | "My serial monitor shows rst:0x1 (POWERON_RST) repeatedly" | Unstable power supply on VIN | Correctly maps reset reason code; requests VIN measurement |
| T5 | "ESP32 won't enter deep sleep — keeps waking immediately" | Wake pin floating or GPIO configuration error | Identifies GPIO config before assuming hardware fault |

### Tier 2 — GPIO & Peripheral Wiring (15 scenarios)

| # | Symptom Input | Expected Root Cause | Pass Criteria |
|---|---|---|---|
| T6 | "I'm trying to write to GPIO34 but nothing happens" | GPIO34 is input-only | Identifies within 1–2 questions; cites board profile |
| T7 | "OLED screen is blank even though I2C scanner finds 0x3C" | Missing I2C pull-up resistors | Reaches pull-up hypothesis; requests pull-up check |
| T8 | "Relay makes a clicking sound but doesn't actually switch" | Back-EMF without flyback diode OR insufficient GPIO drive current | Either hypothesis accepted with supporting evidence |
| T9 | "DHT22 always reads -999 or NaN" | Missing pull-up resistor on data pin, or delay too short after init | Identifies pull-up or timing as first two hypotheses |
| T10 | "Servo jitters randomly" | Servo and logic share the same power supply | Identifies shared power / insufficient decoupling |
| T11 | "My ADC reads 0 even when I apply 3.3V" | ADC2 channel used while Wi-Fi is active — known ESP32 limitation | Correctly identifies ADC2/Wi-Fi conflict; cites TRM |
| T12 | "5V sensor connected to ESP32 GPIO and now pin reads wrong" | GPIO not 5V-tolerant; possible damage | Identifies 5V intolerance; recommends level shifter |

### Tier 3 — Software / Firmware Causes (10 scenarios)

| # | Symptom Input | Expected Root Cause | Pass Criteria |
|---|---|---|---|
| T13 | "Guru Meditation Error: Core 0 panic'd (LoadProhibited)" | Null pointer dereference or invalid memory access | Identifies software root cause before suggesting hardware check |
| T14 | "Stack canary watchpoint triggered" | Stack overflow — recursive function or large local array | Identifies stack overflow cause; cites Xtensa exception docs |
| T15 | "Wi-Fi connects but disconnects every 30 seconds" | RSSI too low OR DHCP timeout | Asks about signal strength and router distance first |
| T16 | "I2C scanner finds no devices" | Wrong SDA/SCL pin assignment in code, or missing Wire.begin() | Eliminates hardware before concluding software misconfiguration |

### Tier 4 — Component Compatibility (10 scenarios)

| # | Symptom Input | Expected Root Cause | Pass Criteria |
|---|---|---|---|
| T17 | "Can I power a 12V relay coil directly from GPIO?" | No — GPIO cannot source 12V; needs transistor driver | Correct incompatibility verdict with numeric reasoning |
| T18 | "I'm using AMS1117 3.3V LDO with a Li-ion battery directly" | Dropout voltage risk below 4.3V battery; buck converter preferred | Identifies dropout issue with specific voltage numbers |
| T19 | "I connected a buzzer between GPIO and GND" | If active buzzer: may exceed GPIO source current limit | Requests buzzer current spec before concluding |

Fill remaining scenarios to reach 50 total before v1 launch. Each new session with a real user that produces a previously-untested fault is a candidate for a new test scenario.

---

## 15. Evaluation Protocol

Run the test suite:
- Before any prompt change — baseline score.
- After any prompt change — regression check.
- After adding new knowledge base documents — coverage improvement check.
- Before any demo — verify no regressions.

**How to run a test scenario:**
1. Start a fresh session (clear memory).
2. Enter exactly the symptom text from the test card.
3. Answer the AI's questions using the "truthful patient" approach — answer as if you are the user described by the scenario. Do not volunteer information the AI has not asked for.
4. Record: (a) whether the correct root cause was reached, (b) how many questions it took, (c) whether any unsupported claims were made, (d) whether citations appeared.
5. Mark the scenario PASS or FAIL.

**Track over time:**

| Version | Date | Pass Rate | Avg Questions | Hallucination Count | Notes |
|---|---|---|---|---|---|
| v0.1 | — | — | — | — | Initial prompt |
| v0.2 | — | — | — | — | After RAG tuning |
| v1.0 | — | — | — | — | MVP launch |

This table goes in the GitHub README. A project with an evaluation table that shows measurable improvement over time is genuinely uncommon at student level.

---

## 16. Out of Scope for v1 (Hard Line)

These are good ideas. They are not for v1.

| Feature | Reason Deferred | Target Version |
|---|---|---|
| Breadboard / schematic image analysis | Requires computer vision, image models, OCR | v3 |
| Oscilloscope waveform analysis | Requires signal understanding, vision | v3 |
| Logic analyzer interpretation | Requires vision + signal domain knowledge | v3 |
| KiCad / netlist parsing | Requires significant programming | v3 |
| PCB design review | Requires significant programming | v3 |
| Live ESP32 integration (serial bridge) | Requires backend programming | v2 |
| Automatic sensor calibration | Requires live hardware connection | v2 |
| Multi-board support (STM32, Pico, etc.) | Architecture supports it; content deferred | v2 |
| Viva/quiz mode | Nice-to-have; doesn't advance core diagnostic value | v2 |
| Custom React/Next.js frontend | Flowise widget is sufficient for demo | v2 |
| Firmware code review | Requires code parsing | v2 |
| Power budget calculator | Standalone tool, not diagnostic flow | v2 |

---

## 17. Versioned Roadmap

- **v1 (now — 2 to 3 months):** All 6 core features + 3 supporting features + 6 workflow features, ESP32 only, Flowise stack, free tier everything.
- **v2 (after v1 is solid):** Live ESP32 serial bridge, additional board support (STM32 or Pico via new KB collection), viva mode, custom frontend.
- **v3 (later):** Computer vision — breadboard image analysis, schematic understanding, oscilloscope waveform interpretation.

---

## 18. GitHub Repository Structure

```
circuit-diagnostic-ai/
├── README.md                    ← Project overview + demo GIF/screenshots
├── flowise/
│   └── flows/
│       ├── diagnostic-agent.json       ← Main diagnostic flow export
│       ├── compatibility-checker.json
│       └── report-generator.json
├── knowledge-base/
│   ├── esp32/
│   │   ├── ESP32_Datasheet.pdf
│   │   ├── ESP32_TRM.pdf
│   │   └── ... (other datasheets)
│   └── failure-library/
│       └── common-failures.md          ← Manually authored failure pattern library
├── prompts/
│   ├── diagnostic-system-prompt.md     ← The core diagnostic state machine prompt
│   ├── compatibility-prompt.md
│   └── report-template.md
├── board-profiles/
│   └── esp32.json                      ← ESP32 pin capabilities, restrictions, defaults
├── sample-sessions/
│   └── oled-blank-diagnostic.md        ← Example session showing full diagnostic flow
├── test-suite/
│   ├── scenarios.md                    ← All 50 test scenarios (symptom, expected cause, pass criteria)
│   └── results-log.md                  ← Version-by-version evaluation table (pass rate, hallucination count)
├── docs/
│   ├── architecture.md
│   ├── setup-guide.md
│   └── engineering-philosophy.md       ← Design principles and reasoning rules in full
└── screenshots/
    └── (demo screenshots for README)
```

> **Note (actual build):** The live repo structure diverged slightly for practical reasons — `build-logs/` replaces `docs/` as the primary paper trail (per-phase logs rather than a single architecture doc), and `prompts/` holds the merged multi-mode system prompts described in [PHASE4_LOG.md](./build-logs/PHASE4_LOG.md). The underlying separation of concerns (board-agnostic engine vs. board-specific knowledge) was preserved throughout.

---

## 19. Resume Entry

> Designed and built Circuit Diagnostic AI, an AI-driven engineering workflow platform that guides embedded-systems engineers through adaptive, documentation-grounded hardware fault diagnosis using Flowise and Retrieval-Augmented Generation. The system combines a structured diagnostic state machine, persistent session memory, datasheet-cited reasoning, and automated professional report generation — functioning as an intelligent lab mentor rather than a generic chatbot.

---

## 20. Instructions for Implementing AI Tool

When this PRD is given to an AI assistant (ChatGPT, Claude, Gemini, etc.) for implementation help:

- Treat Sections 6, 7, and 8 as the complete and only scope for v1. Do not add features.
- Default to Flowise-native solutions at every step. Explain any code in beginner-friendly terms with complete copy-pasteable setup instructions.
- Never recommend a paid service. Always propose a free alternative.
- Preserve the separation between board-specific knowledge/content and the board-agnostic diagnostic engine logic (described in Section 9 and the board-profiles directory).
- When proposing a Flowise flow, describe the node connections step by step — Priya is learning Flowise visually, not reading code.
- Flag any step that is likely to take more than a few hours for a beginner, and suggest how to simplify or defer it.
- Ask clarifying questions when a requirement is ambiguous rather than silently assuming scope.
- The project must be completable in 2–3 months by a student with beginner programming experience. Keep that constraint in view for every decision.
