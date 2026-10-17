# Project Parashurama: Technical Architecture & System Decisions

> **Project Concept**: An intelligent AI Tutor that takes user input (via Text or Speech) and parallelly explains the concept on a dynamic Whiteboard with synchronized Voice (TTS) and live visual representations (drawings, formulas, flowcharts, and text).

---

## 1. High-Level System Architecture

```mermaid
flowchart TD
    subgraph Vercel [Frontend Layer - Deployed on Vercel]
        User([Student / User]) -->|Voice / Text| UI[React + Vite Blackboard UI]
        AuthModal[Auth & Profile Manager] -->|Session Token| UI
        UI --> CanvasEngine[Rough.js Whiteboard + Canvas + KaTeX]
        UI --> WebAudio[Web Audio Engine - Synced Player]
    end

    subgraph AWS [Backend & Cloud Infrastructure - Deployed on AWS]
        subgraph Compute [AWS ECS Fargate / Docker Container]
            API[FastAPI Gateway]
            WS[Persistent WebSocket Router]
            AuthService[Auth & JWT Verification]
            LLMOrchestrator[LLM Reasoning & Cue Planner - Groq]
            SyncEngine[Audio & Visual Sync Engine]
            KnowledgeTracer[Student Progress & Understanding Engine]
        end

        subgraph Database [Storage & Memory Services]
            RDS[(AWS RDS PostgreSQL + pgvector)]
            Redis[(AWS ElastiCache Redis - Active Sessions)]
            S3[(AWS S3 Bucket - Boards, Audio & Notes)]
        end
    end

    subgraph AI_Providers [AI Cloud Services]
        DeepgramSTT[Deepgram Nova-2 STT]
        DeepgramTTS[Deepgram Aura TTS]
        GroqLLM[Groq Llama 3.3 70B / DeepSeek R1 / 120B]
    end

    UI <==>|Full-Duplex Secure WebSocket (WSS)| WS
    UI -->|REST API (Auth, Profile, History)| API
    API --> AuthService
    AuthService --> RDS

    WS --> DeepgramSTT
    WS --> LLMOrchestrator
    LLMOrchestrator <--> GroqLLM
    LLMOrchestrator <--> RDS
    LLMOrchestrator <--> Redis

    SyncEngine <--> DeepgramTTS
    SyncEngine ==>|Streaming Audio + Timed Cue Frames| WS

    KnowledgeTracer -->|Update Mastery & Topic Progress| RDS
    API -->|Save/Load Board Snapshots| S3
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

### Phase 7: Multi-User Authentication & Profiles

#### Question 7.1: Which authentication provider should be used?
- [x] **Option A (Recommended - Clerk / Supabase Auth)**:
  - Turnkey modern UI modal with Google, GitHub, and email/password login.
  - Generates secure JWTs verified statelessly by FastAPI backend (`Depends(verify_token)`).
  - Instant user profile management and session persistence.
- [ ] **Option B (FastAPI Native OAuth2 + JWT with PostgreSQL)**:
  - Full control over user credentials, passwords hashed with bcrypt in PostgreSQL.
  - No external vendor cost or third-party dependency.
- [ ] **Option C (AWS Cognito)**:
  - Native AWS integration with User Pools and IAM roles.

*Your Choice:* **Option A**  
*Additional Notes:*

---

#### Question 7.2: What student profile attributes should be tracked for personalization?
*(Select all that apply)*
- [x] **Target Educational Level** (e.g. Middle School, High School, Undergraduate, Professional)
- [x] **Preferred Learning Pace** (Concise summary vs. Deep-dive step-by-step)
- [x] **Tutor Persona & Accent** (Encouraging mentor, Academic professor, Socratic questioner)
- [x] **Primary Subject Focus** (STEM, Medicine/Biology, Coding, General Science)

*Additional Notes:*

---

### Phase 8: Memory Storage & Cross-Session Recall

To feel like a real 1-on-1 human tutor, Parashurama must remember the student across sessions (e.g., past doubts, weak topics, and previous lesson notes).

#### Question 8.1: How should the multi-tiered memory system be structured?
- [x] **Option A (Recommended - 3-Tier Memory Architecture)**:
  1. **Working Memory (Redis / ElastiCache)**:
     - Maintains active lecture state, uncommitted whiteboard strokes, and live speech turn buffers during an active WebSocket session.
  2. **Episodic & Relational Memory (AWS RDS PostgreSQL)**:
     - Stores structured lesson history, full lecture transcripts, whiteboard snapshots, and time-stamped cue logs.
  3. **Semantic Memory (Vector Embeddings via `pgvector` on RDS)**:
     - Embeds student queries, past misconceptions, and lecture summaries into vector space.
     - *Recall in action*: When a user struggles with a concept, the tutor recalls: *"Last week when we covered Derivatives, you had a similar question on the chain rule..."*

*Your Choice:* **Option A**  
*Additional Notes:*

---

### Phase 9: User Learning Understanding & Progress Identification

#### Question 9.1: How should student understanding and topic mastery be measured?
- [x] **Option A (Recommended - Bayesian Knowledge Tracing & Concept Graph)**:
  - Topics are organized as a directed Knowledge Graph (e.g., `Calculus` $\rightarrow$ requires `Algebra` $\rightarrow$ requires `Arithmetic`).
  - Student has a **Mastery Probability (0% to 100%)** per concept that updates dynamically after each lecture and comprehension checkpoint.
- [ ] **Option B (Rule-based Progress Log)**:
  - Simple checklist of completed topics and study duration.

*Your Choice:* **Option A**  
*Additional Notes:*

---

#### Question 9.2: How should the AI Tutor assess understanding in real-time?
*(Select all that apply)*
- [x] **Interactive In-Lecture Checkpoints**: The AI Tutor pauses narration, writes a test question on the board, and waits for student verbal or text response.
- [x] **Socratic Diagnostic Feedback**: The tutor analyzes the student's answer, identifies specific misconceptions, and adapts the whiteboard diagram to clarify.
- [x] **Student Learning Analytics Dashboard**: Visual mastery heatmap, retention decay estimates (spaced repetition reminders), and recommended next topics.
- [x] **Post-Lecture Quiz & Summary Card**: Auto-generates 3 quick practice problems at the end of each session.

*Additional Notes:*

---

### Phase 10: Cloud Deployment & DevOps Architecture (Vercel + AWS)

#### Question 10.1: How will the Frontend be deployed on Vercel?
- [x] **Option A (Vercel Edge & Static Hosting)**:
  - Connected directly to GitHub repository (`Parushuram-frontend`).
  - Automatic CI/CD deployments on `git push origin main`.
  - Global CDN edge caching with ultra-low latency assets.
  - Environment variables configured via Vercel dashboard (`VITE_API_URL`, `VITE_WS_URL`).

*Your Choice:* **Option A**  
*Additional Notes:*

---

#### Question 10.2: How will the Backend and Database be deployed on AWS?
- [x] **Option A (Recommended - AWS ECS Fargate + RDS PostgreSQL + S3 + ElastiCache)**:
  - **Compute**: Containerized FastAPI inside Docker running on **AWS ECS Fargate** behind an Application Load Balancer (ALB).
    - *Why Fargate*: Native support for persistent, long-lived WebSockets (`wss://`) without serverless timeout limits.
  - **Database**: **AWS RDS PostgreSQL** with `pgvector` extension for user data, lesson history, and memory embeddings.
  - **Cache**: **AWS ElastiCache Redis** for fast real-time session state and rate-limiting.
  - **Asset Storage**: **AWS S3** bucket for storing whiteboard PNG/SVG exports, lesson PDF notes, and cached audio files.
- [ ] **Option B (AWS App Runner + Supabase + S3)**:
  - Simpler container deployment with automated scaling, using managed Supabase for PostgreSQL.

*Your Choice:* **Option A**  
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
