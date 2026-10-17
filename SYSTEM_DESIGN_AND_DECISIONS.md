# Project Parashurama: Technical Architecture & System Decisions

> **Project Concept**: An intelligent AI Tutor that takes user input (via Text or Speech) and parallelly explains the concept on a dynamic Whiteboard with synchronized Voice (TTS) and live visual representations (drawings, formulas, flowcharts, and text).

---

## 1. High-Level System Architecture

```mermaid
flowchart TD
    User([User]) -->|Text or Speech Audio| FE[Frontend Client (Vite + React)]
    
    subgraph Frontend [Client Layer]
        FE --> Mic[Audio Capture / Text Input]
        FE --> AudioPlayer[Web Audio Engine (Synced Player)]
        FE --> Canvas[Interactive Whiteboard (Canvas/SVG)]
    end

    subgraph Backend [Backend Orchestrator (FastAPI)]
        Mic -->|Audio Stream / Prompt| STT[STT Engine: Deepgram / Groq Whisper]
        STT --> Transcript[User Intent / Topic]
        Transcript --> LLM[LLM Reasoning: Groq Llama-3.3]
        
        LLM -->|Structured Script + Visual Markers| Parser[Sync Planner & Splitter]
        Parser -->|Speech Text + Cue Points| TTS[TTS Engine: Deepgram Aura / Edge-TTS]
        Parser -->|Visual Action Graph / Timeline| ActionQueue[Board Action Frames]
    end

    TTS -->|Audio Packets + Timestamps| AudioPlayer
    ActionQueue -->|Timed Drawing Events (WebSocket/SSE)| Canvas
    AudioPlayer -.->|Playback Time Synchronization| Canvas
```

---

## 2. Technical Decision Matrix & Questionnaire

Review the questions below. Fill in `[x]` for your chosen preference, or provide custom instructions under **Additional Notes**.

---

### Phase 1: Whiteboard Engine & Visual Representations

The whiteboard needs to render concepts clearly while matching the pacing of an instructor writing/drawing on a board.

#### Question 1.1: Which rendering technology should power the whiteboard?
- [ ] **Option A (Recommended - Hybrid Canvas & SVG with Rough.js)**:
  - Natural hand-drawn look (like Excalidraw).
  - Supports geometric shapes, arrows, handwriting text, math formulas (KaTeX), and clean auto-layouts.
  - Very smooth 60fps animations.
- [ ] **Option B (Excalidraw Core Embedded)**:
  - Uses the full open-source Excalidraw engine.
  - Pros: Rich built-in shapes, pan/zoom, eraser, laser pointer.
  - Cons: Heavier bundle, programmatic remote-control requires interacting with Excalidraw's internal state.
- [ ] **Option C (Fabric.js / Konva.js Canvas)**:
  - Standard object-oriented 2D canvas with shapes, text, and timeline animations.
  - More corporate / diagrammatic look rather than hand-drawn blackboard look.
- [ ] **Option D (Mermaid.js + Markdown + KaTeX Stream)**:
  - Non-canvas approach: dynamically generates standard diagrams, flowcharts, and markdown notes in real-time.

*Your Choice:* **Option A**  
*Additional Notes:* 

---

#### Question 1.2: What visual primitives should the AI Tutor be capable of generating?
*(Select all that apply)*
- [x] **Animated handwriting/callout text** (simulating chalk/marker strokes)
- [x] **Diagrams & Flowcharts** (nodes, labeled arrows, decision boxes)
- [x] **Mathematical & Scientific formulas** (rendered via LaTeX/KaTeX)
- [x] **Coordinate grids & 2D Graphs** (plots, functions, axes)
- [x] **Bullet points & Summary cards**
- [ ] **Code snippets with syntax highlighting**
- [x] **Complex Anatomical & Biological Schematics (Human Organs, Biology, Physics)**

---

#### Question 1.3: How should complex scientific diagrams and human organs be drawn accurately? *(New Addition)*
*Context: Basic shapes (circles, lines) cannot draw realistic organs like the heart, lungs, or brain.*

- [x] **Option A (Recommended - Hybrid Vector Asset Library + Progressive Chalk Draw-In + Dynamic LLM Callouts)**:
  - **Pre-indexed SVG Line-Art Library**: Curate high-quality, transparent vector schematics for core subjects (Anatomy: Heart, Brain, Nephron, Digestive system, DNA; Physics: Optics, Circuits; Chemistry: Molecular structures).
  - **Live "Draw-in" Effect**: The frontend animates the SVG lines using `stroke-dashoffset` or `roughjs.path()` so it renders stroke-by-stroke on the board as if sketched live in chalk.
  - **Dynamic LLM Callouts & Highlights**: The LLM directs arrows, highlights, and labels to specific organ components synchronized with the voice (e.g. lighting up the *Left Ventricle* when mentioned in audio).
- [ ] **Option B (Pure LLM-Generated SVG Bezier Paths)**:
  - The LLM writes raw SVG `<path d="...">` coordinates for any topic.
  - *Trade-off*: Flexible for any concept, but LLMs often produce irregular or distorted anatomical proportions.
- [ ] **Option C (AI Image Generation to Line-Art Vector Conversion)**:
  - Generate a diagram via fast diffusion/line-art model, vectorize to SVG, and trace onto the board.
  - *Trade-off*: Adds 2-4s latency before board starts drawing.

*Your Choice:* **Option A**  
*Additional Notes:* Organ library with progressive chalk-stroke animation and synchronized pointer callouts.

---

### Phase 2: Audio-Visual Synchronization (The Core Challenge)

To achieve the effect of an instructor speaking while simultaneously drawing on the board, the audio playback and visual elements must be strictly synchronized.

#### Question 2.1: How should synchronization between speech and board drawing be handled?
- [ ] **Option A (Recommended - Timestamped Cue Events via Web Audio API)**:
  - The LLM outputs a dual-track response:
    1. Speech script broken into sentences/phrases.
    2. Visual actions tied to cue tags: `[cue:1] Draw coordinate axis`, `[cue:2] Plot parabola`.
  - The TTS generates audio with word/sentence boundary timestamps (or estimated duration chunks).
  - The frontend audio player uses `audioContext.currentTime` to trigger board strokes exactly at each cue point.
- [ ] **Option B (Streaming Chunk Protocol)**:
  - The backend sends paired micro-chunks over WebSocket: `{ audio_chunk_base64: "...", board_actions: [...] }`.
  - The frontend plays chunks sequentially in a queue.
- [ ] **Option C (Turn-based Action Sequence)**:
  - The board renders step-by-step; speech explains step 1, waits until step 1 finishes drawing, then moves to step 2.

*Your Choice:* **Option A**  
*Additional Notes:*

---

### Phase 3: Speech Input (STT) & Speech Output (TTS)

You have existing API keys for **Deepgram** (`DEEPGRAM_API`) and **Groq** (`GROQ_API`) in your `.env`.

#### Question 3.1: Which STT (Speech-to-Text) provider should capture user questions?
- [ ] **Option A (Recommended - Deepgram Nova-2 / Live WebSocket)**:
  - Near-instantaneous streaming transcription with high accuracy and word-level timings.
- [ ] **Option B (Groq Whisper-large-v3)**:
  - Extremely fast batch transcription via Groq API (transcribes 10s of audio in <300ms).
- [ ] **Option C (Client-side Web Speech API)**:
  - Zero server cost, runs directly in Google Chrome / Edge; fallback to text if unsupported.

*Your Choice:* **Option A**  
*Additional Notes:*

---

#### Question 3.2: Which TTS (Text-to-Speech) provider should voice the tutor?
- [ ] **Option A (Recommended - Deepgram Aura)**:
  - Ultra-low latency (<200ms TTFB), conversational AI voices designed for real-time agents.
- [ ] **Option B (Edge-TTS)**:
  - Free, high-quality Microsoft Azure neural voices without API cost.
- [ ] **Option C (ElevenLabs)**:
  - Hyper-realistic emotional and educational voices, but higher cost and slightly higher latency.
- [ ] **Option D (Browser Web Speech Synthesis)**:
  - Zero latency and zero cost, but robotic and varies across devices.

*Your Choice:* **Option A**  
*Additional Notes:*

---

### Phase 4: LLM Orchestration, Models & Groq Test Matrix

#### Question 4.1: Which model(s) should generate the explanation and whiteboard storyboard?
- [x] **Option A (Multi-Model Support & Groq Test Matrix)**:
  - **Primary Realtime Driver**: **Groq `llama-3.3-70b-versatile`** (super fast ~300+ tokens/sec, ideal for streaming whiteboard cues without latency).
  - **Reasoning Driver**: **Groq `deepseek-r1-distill-llama-70b`** (excels at structuring difficult mathematical proofs and biological step-by-step breakdowns).
  - **Preview & Large OSS Models (e.g. 120B / Open Source Weights)**: Pluggable model switcher to benchmark and test larger open-source weights as Groq introduces/tests them.
- [ ] **Option B (Dual LLM: Fast Planner + Deep Reasoner)**:
  - A fast 8B model generates the first cue within 200ms to start the whiteboard immediately, while a larger model refines subsequent stages.
- [ ] **Option C (Cloud Fallback: Claude 3.5 Sonnet / GPT-4o)**:
  - High fidelity for intricate spatial layouts, with higher latency.

*Your Choice:* **Option A (with Groq Benchmarking Harness for 70B, 120B/OSS preview)**  
*Additional Notes:* Build a pluggable backend model switcher with an automated benchmark script (`backend/benchmarks/test_models.py`) to test throughput, time-to-first-token, and JSON cue generation reliability across Groq models.

---

#### Question 4.2: How should the LLM represent anatomical structures on the board? *(New Addition)*
- [x] **Option A (Semantic Asset Identifier + Coordinated Micro-Cue Animations)**:
  - When explaining the human heart, the LLM specifies:
    ```json
    { "type": "schematic", "asset": "anatomy/heart", "draw_duration_ms": 3000 }
    ```
  - Followed by synchronized spotlight callouts:
    ```json
    { "type": "callout", "target": "left_ventricle", "label": "Left Ventricle", "speech_sync": "blood is pumped to the aorta" }
    ```
- [ ] **Option B (Inline Generative SVG Path)**:
  - The model outputs `<path d="...">` directly in the JSON response.

*Your Choice:* **Option A**  
*Additional Notes:* Ensures anatomically accurate drawings that render like chalkboard art with zero distortion.

---

### Phase 5: Communication Protocol & Frontend Stack

#### Question 5.1: How should the Frontend communicate with the Backend?
- [ ] **Option A (Recommended - Full-duplex WebSockets)**:
  - Allows bi-directional streaming: user can speak/interrupt, backend streams back synchronized audio buffers and drawing events simultaneously.
- [ ] **Option B (Server-Sent Events [SSE] for output + HTTP POST for input)**:
  - Simpler to debug, supports streaming text/events, but audio streaming requires chunking.
- [ ] **Option C (REST API with Polling / Complete Response)**:
  - Generates entire lecture before rendering. (Higher initial delay, simplest implementation).

*Your Choice:* **Option A**  
*Additional Notes:*

---

#### Question 5.2: What should the UI theme and layout look like?
- [ ] **Option A (Recommended - Sleek Dark Slate / Chalkboard Mode)**:
  - High-contrast blackboard/greenboard or modern charcoal slate.
  - Vibrant chalk strokes (cyan, neon amber, emerald, white).
  - Floating tutor voice visualizer + interactive timeline slider.
- [ ] **Option B (Minimalist Clean Whiteboard)**:
  - Light mode, dry-erase marker aesthetic (black, blue, red pens).
- [ ] **Option C (Split Screen: Chat/Audio Deck on Left, Giant Canvas on Right)**.

*Your Choice:* **Option A**  
*Additional Notes:*

---

### Phase 6: User Controls & Interactivity

#### Question 6.1: What controls should the user have over the lecture?
*(Select all that apply)*
- [x] **Pause / Resume** playback (both audio and drawing freeze together)
- [x] **Rewind / Fast-forward timeline slider**
- [x] **Clear Board / Download Notes as PDF or PNG**
- [x] **Interrupt & Ask Follow-up** ("Wait, explain step 2 again!")
- [x] **User Freehand Pen** (user can draw on top of AI's whiteboard)
- [x] **Playback Speed Selector** (0.75x, 1x, 1.25x, 1.5x)

*Additional Notes:*

---

## 3. Data Flow Specification: Board Action DSL / JSON Schema

Here is the proposed lightweight schema the LLM will output to direct the whiteboard:

```json
{
  "speech_script": "Let's understand Newton's Second Law of Motion. Force equals mass times acceleration.",
  "cue_points": [
    {
      "cue_id": "c1",
      "timestamp_offset_ms": 0,
      "speech_snippet": "Newton's Second Law of Motion",
      "actions": [
        {
          "type": "text",
          "content": "Newton's Second Law: F = m · a",
          "position": { "x": 100, "y": 80 },
          "style": { "color": "#f59e0b", "fontSize": 28, "handwritten": true }
        }
      ]
    },
    {
      "cue_id": "c2",
      "timestamp_offset_ms": 2400,
      "speech_snippet": "Force equals mass times acceleration",
      "actions": [
        {
          "type": "box",
          "bounds": { "x": 90, "y": 60, "width": 420, "height": 70 },
          "stroke": "#38bdf8",
          "roughness": 1.5
        },
        {
          "type": "formula",
          "latex": "\\vec{F} = m \\frac{d\\vec{v}}{dt}",
          "position": { "x": 100, "y": 180 }
        }
      ]
    },
    {
      "cue_id": "c3",
      "timestamp_offset_ms": 5200,
      "speech_snippet": "In the human heart, oxygen-depleted blood enters through the superior vena cava into the right atrium.",
      "actions": [
        {
          "type": "schematic_sketch",
          "asset": "anatomy/heart_cross_section",
          "position": { "x": 580, "y": 120 },
          "scale": 0.85,
          "draw_style": "chalk_stroke_animation",
          "draw_duration_ms": 3200
        },
        {
          "type": "callout_pointer",
          "target_part": "right_atrium",
          "label": "Right Atrium (Deoxygenated)",
          "arrow_from": { "x": 500, "y": 180 },
          "arrow_to": { "x": 620, "y": 200 },
          "color": "#38bdf8",
          "highlight_pulse": true
        }
      ]
    }
  ]
}
```

---

## 4. Immediate Next Steps & Milestones

1. **Step 1**: Confirm the questions in this document.
2. **Step 2**: Initialize Frontend with Vite + React + Canvas (Rough.js) setup.
3. **Step 3**: Configure FastAPI backend with WebSocket endpoint, integrating Groq (LLM) and Deepgram (STT/TTS).
4. **Step 4**: Implement the Audio-Visual Synchronization Engine in frontend Web Audio API.
5. **Step 5**: Test end-to-end prompt: text/speech input $\rightarrow$ speech generation $\rightarrow$ synchronized whiteboard drawing.
