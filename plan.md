# Inclusive Classroom — 10-Day Hackathon MVP Architecture

Assume: mic + webcam + smart TV/browser + internet already in classroom. No edge hardware. Web app only.

---

# 1. MVP System Architecture

```
Browser (Teacher) ──mic/cam──▶ Next.js Frontend ──WebSocket──▶ FastAPI Backend ──▶ AI APIs
                                       │                              │
                                       │◀─────caption/avatar/notes────┤
                                       ▼                              ▼
                              Browser (Student TV/device)      PostgreSQL (store)
```

One Next.js app (teacher view + student view, different routes). One FastAPI backend. One Postgres DB. One WebSocket connection per classroom session. That's it.

| Component | Classification |
|---|---|
| Next.js frontend (teacher + student views) | Must Build |
| FastAPI backend, monolith | Must Build |
| WebSocket session per classroom | Must Build |
| PostgreSQL (single DB) | Must Build |
| Redis | Nice to Have (only if WebSocket fan-out across >1 backend instance needed — for a demo, skip it) |
| LangGraph / LangChain / MCP / RAG / Vector DB | Future Scope — none needed for MVP, see reasoning below |

**"Can this MVP work without it?" check:**
- LangGraph → No agents looping/branching at demo scale, it's a linear pipeline. **Remove.**
- LangChain → Just calling 2-3 APIs directly with `requests`/`httpx`. **Remove.**
- MCP → No ERP/LMS to integrate in 10 days. **Remove.**
- RAG → Notes/quiz can be generated straight from the transcript with a good prompt; no need to retrieve NCERT chapters for a demo. **Remove.**
- Vector DB → Only needed if doing RAG. **Remove.**

Result: **Next.js + FastAPI + Postgres + 3-4 AI API calls.** That's the whole stack.

---

# 2. Feature A — Deaf/HoH Student (Speech → Caption → Sign Avatar)

**Flow:** Teacher Speech → STT → Captions → Sign Avatar → Student View

### Architecture
```
Teacher Mic (browser) → WebSocket audio chunks → FastAPI
   → STT API (Whisper API or Deepgram streaming)
   → text → FastAPI broadcasts via WebSocket
   → Student browser: renders caption text
   → Student browser: maps caption → pre-recorded ISL video clips (word/phrase lookup)
```

### Components
| Component | Classification |
|---|---|
| Browser mic capture (`MediaRecorder` / `getUserMedia`) | Must Build |
| WebSocket audio streaming to backend | Must Build |
| STT integration (Whisper API, simplest path) | Must Build |
| Live caption rendering (student + teacher screen) | Must Build |
| ISL avatar via **pre-recorded clip lookup** (common words/phrases matched from caption text) | Must Build (this is the realistic 10-day version) |
| Generative 3D ISL avatar (real-time animation synthesis) | Future Scope — not feasible in 10 days, no usable open ISL generation model exists off-the-shelf |
| Multi-language translation before captioning | Nice to Have (only if time permits, else English-only captions for MVP) |

### User flow
1. Teacher opens `/teacher`, clicks "Start Lecture," browser asks mic permission.
2. Audio streamed in ~2-3s chunks over WebSocket to FastAPI.
3. FastAPI forwards chunk to Whisper API, gets text back.
4. FastAPI broadcasts `{type: "caption", text}` to all connected clients in that session.
5. Student `/student/[sessionId]` page renders caption text live + looks up matching word(s) in a small JSON map of phrase→video URL, plays the clip if matched.
6. If no match found, just show captions (graceful, visible fallback — not silent).

### Required models/APIs
- **STT:** OpenAI Whisper API (`whisper-1`) — simplest, no self-hosting, good enough latency for a demo. Alternative: Deepgram (has native streaming, slightly better real-time UX, still simple).
- **Sign avatar:** No model — a **curated dictionary** of ~50-100 pre-recorded ISL video clips for common classroom words ("hello," "today," "homework," subject names). This is the only realistic ISL approach in 10 days; say this explicitly to judges as a deliberate scoping decision, not a gap.

### APIs (internal)
```
POST /session/start          → create classroom session, returns session_id
WS   /ws/{session_id}        → bidirectional: audio in, captions/avatar-cue out
GET  /session/{id}/clip-map  → returns word→video-url dictionary for student client
```

---

# 3. Feature B — Speech-Impaired Teacher (Sign/Gesture → Text → Voice)

**Flow:** Teacher Sign/Gesture → Gesture Recognition → Text → Voice → Classroom Output

### Architecture
```
Teacher Webcam (browser) → frame capture (every ~200ms) → WebSocket → FastAPI
   → MediaPipe Hands/Pose (runs in-browser via mediapipe.js, OR server-side via Python mediapipe)
   → keypoints → simple gesture classifier (small pre-trained set of gestures, e.g. 15-20 signs)
   → matched gesture → text label
   → FastAPI: text → TTS API → audio
   → broadcast text (caption) + audio (TTS) to classroom
```

### Components
| Component | Classification |
|---|---|
| Webcam frame capture in browser | Must Build |
| MediaPipe Hands keypoint extraction (use **MediaPipe in-browser JS** — zero backend ML infra, runs client-side) | Must Build |
| Gesture-to-text matching for a **small fixed gesture set** (~15-20 classroom-relevant signs/gestures: yes, no, stop, repeat, question, numbers) | Must Build |
| Full ISL gloss → sentence reconstruction (LLM-based) | Future Scope — too unreliable to demo well in 10 days without a real ISL gesture dataset |
| TTS for recognized text | Must Build |
| Full continuous sign-language sentence recognition | Future Scope |

### User flow
1. Teacher opens `/teacher`, toggles "Sign Mode."
2. Browser runs MediaPipe Hands directly on webcam feed (client-side, no server round-trip needed for keypoints — huge latency win and zero backend ML work).
3. Browser does simple rule-based/nearest-neighbor matching of keypoint pattern against the ~15-20 pre-defined gesture templates (this can literally be cosine similarity on normalized keypoint vectors — no training pipeline needed).
4. Matched gesture label sent to FastAPI via WebSocket.
5. FastAPI sends label text to TTS API, gets audio back, broadcasts both text + audio URL to classroom session.
6. Classroom TV/student browsers display caption and play synthesized audio.

### Required models/APIs
- **Keypoints:** `@mediapipe/tasks-vision` (Google's JS package) — runs entirely client-side, no model hosting needed.
- **Gesture matching:** No trained model needed for MVP — nearest-neighbor / cosine similarity against a small hand-recorded template set (record 15-20 gestures yourself during prep days, store as reference vectors).
- **TTS:** Browser-native `SpeechSynthesis` API (zero cost, zero latency, works offline-in-browser) for MVP demo. Upgrade path: ElevenLabs or Google Cloud TTS API if higher-quality voice is wanted and time allows.

### APIs (internal)
```
WS  /ws/{session_id}        → same socket as Feature A, message type differentiates: {type:"gesture", label, audio_url?}
POST /tts                   → {text} → {audio_url} (only needed if not using browser-native SpeechSynthesis)
```

**Honest scoping note for judges:** real continuous ISL recognition is an unsolved research problem even outside hackathon constraints. The credible 10-day demo is a **fixed gesture vocabulary** (like a remote control, not full sign language) — frame it as "Phase 1 gesture commands," not "we built ISL recognition."

---

# 4. Feature C — Lecture Intelligence (Speech → Notes → Quiz)

**Flow:** Teacher Speech → Transcript → AI Summary → Notes → Quiz

### Architecture
```
Full transcript (accumulated from Feature A's STT output, stored as lecture progresses)
   → at "End Lecture" click: full transcript → single LLM API call
   → prompt: "summarize into structured notes" → Notes (Markdown)
   → second LLM call: notes → "generate 5 quiz questions" → Quiz (JSON)
   → store both in Postgres, display to teacher for one-tap approve → visible to students
```

No RAG. No NCERT grounding for MVP — just transcript-in, notes-out. (Curriculum grounding is future scope.)

### Components
| Component | Classification |
|---|---|
| Transcript accumulation during lecture (just append STT chunks to a string/array) | Must Build |
| "End Lecture" trigger → notes generation | Must Build |
| LLM call for notes (single prompt, no RAG) | Must Build |
| LLM call for quiz generation (single prompt, JSON mode) | Must Build |
| Teacher approval step before publishing | Must Build (cheap to build, makes demo feel trustworthy) |
| RAG grounding against NCERT content | Future Scope |
| Curriculum-outcome traceability | Future Scope |
| Quiz auto-grading / LMS publishing | Future Scope |

### User flow
1. Teacher clicks "End Lecture."
2. FastAPI takes the full accumulated transcript, sends to GPT-4o-mini (or similar) with a "generate structured study notes" prompt.
3. Notes returned, stored in Postgres, shown to teacher in a review panel.
4. Teacher clicks "Generate Quiz" → second LLM call using the notes as input, JSON-mode output (5 MCQs).
5. Teacher clicks "Publish" → notes + quiz become visible on `/student/[sessionId]/notes`.

### Required models/APIs
- **Notes + Quiz generation:** OpenAI GPT-4o-mini (cheap, fast, good enough quality, JSON mode support for the quiz) — or Claude API equivalent. One model handles both tasks, just different prompts.

### APIs (internal)
```
POST /lecture/{id}/end          → finalizes transcript, triggers notes generation
POST /lecture/{id}/notes        → returns generated notes (Markdown)
POST /lecture/{id}/quiz         → returns generated quiz (JSON)
POST /lecture/{id}/publish      → marks notes+quiz visible to students
GET  /lecture/{id}/notes        → student-facing fetch
```

---

# 5. Database Schema

Single Postgres DB, 5 tables. No sharding, no multi-DB.

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    teacher_name TEXT,
    started_at TIMESTAMP DEFAULT now(),
    ended_at TIMESTAMP,
    status TEXT DEFAULT 'active'  -- active | ended | published
);

CREATE TABLE transcript_segments (
    id SERIAL PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    text TEXT,
    source TEXT DEFAULT 'speech',  -- speech | gesture
    created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE notes (
    id SERIAL PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    content_markdown TEXT,
    approved BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE quizzes (
    id SERIAL PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    questions_json JSONB,
    approved BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE sign_clips (
    id SERIAL PRIMARY KEY,
    word TEXT UNIQUE,
    video_url TEXT
);
```

That's it. No `schools`, `students`, `accessibility_profiles`, `attendance` tables — out of scope for a single-classroom demo. Add those back only if the demo needs multi-classroom switching.

---

# 6. API Design

```
POST   /session/start                  → create session, returns {session_id}
POST   /session/{id}/end                → mark ended, trigger notes pipeline

WS     /ws/{session_id}                 → main real-time channel (audio in, captions/gesture/avatar-cue out)

POST   /lecture/{id}/notes/generate     → calls LLM, stores notes
POST   /lecture/{id}/quiz/generate      → calls LLM, stores quiz
POST   /lecture/{id}/publish            → sets approved=true on notes+quiz

GET    /lecture/{id}/notes              → student fetch
GET    /lecture/{id}/quiz               → student fetch
GET    /sign-clips                      → word→video_url dictionary (loaded once on student client init)
```

13 endpoints total (incl. WS). No versioning, no auth complexity needed for a demo (add a single shared session-code join flow if time allows, skip real auth).

---

# 7. WebSocket Flow

One WebSocket connection per browser tab, all multiplexed through one `/ws/{session_id}` route. Message-type discriminated, not separate sockets per feature.

```
Teacher client                    FastAPI                         Student client(s)
     │                               │                                  │
     │──audio_chunk (binary)───────▶│                                  │
     │                               │──Whisper API call               │
     │                               │◀─text──                         │
     │                               │──broadcast {type:"caption",text}─▶│ renders caption
     │                               │                                  │ looks up sign_clips[word]
     │                               │                                  │ plays clip if match
     │                               │                                  │
     │──gesture_keypoints (json)────▶│ (only if Sign Mode toggled)      │
     │                               │──match against templates        │
     │                               │──TTS call                       │
     │                               │──broadcast {type:"gesture",      │
     │                               │   label, audio_url}─────────────▶│ shows caption + plays audio
     │                               │                                  │
     │──"end_lecture"───────────────▶│──triggers notes/quiz pipeline   │
     │◀──{type:"notes_ready"}────────│                                  │
```

**Message schema (single channel, typed):**
```json
{ "type": "caption" | "gesture" | "notes_ready" | "quiz_ready", "payload": { ... } }
```

One socket, one schema, broadcast to all clients in that session's room (FastAPI's WebSocket manager keeps a `dict[session_id, list[WebSocket]]` in memory — no Redis pub/sub needed at single-instance demo scale).

---

# 8. Folder Structure

```
inclusive-classroom/
├── frontend/                       (Next.js)
│   ├── app/
│   │   ├── teacher/
│   │   │   └── page.tsx            (mic capture, sign-mode toggle, lecture controls)
│   │   ├── student/
│   │   │   └── [sessionId]/
│   │   │       ├── page.tsx        (live captions + avatar clip player)
│   │   │       └── notes/page.tsx  (notes + quiz view)
│   │   └── layout.tsx
│   ├── components/
│   │   ├── CaptionDisplay.tsx
│   │   ├── SignClipPlayer.tsx
│   │   ├── GestureCapture.tsx      (MediaPipe wrapper)
│   │   └── NotesPanel.tsx
│   ├── lib/
│   │   ├── websocket.ts
│   │   └── mediapipe.ts
│   └── package.json
│
├── backend/                        (FastAPI, monolith)
│   ├── main.py                     (app init, route registration)
│   ├── routes/
│   │   ├── session.py
│   │   ├── lecture.py
│   │   └── websocket.py
│   ├── services/
│   │   ├── stt.py                  (Whisper API wrapper)
│   │   ├── tts.py                  (TTS wrapper / browser-native fallback note)
│   │   ├── llm.py                  (notes + quiz generation calls)
│   │   └── gesture_match.py        (cosine similarity matcher)
│   ├── db/
│   │   ├── models.py
│   │   └── database.py
│   ├── data/
│   │   └── sign_clips.json         (word→video_url map, also gesture templates)
│   └── requirements.txt
│
└── README.md
```

Two folders. No `services/` microsplit, no `infra/` Kubernetes manifests, no `agents/` directory.

---

# 9. Development Timeline (10 Days)

| Day | Focus | Deliverable |
|---|---|---|
| **1** | Setup: Next.js + FastAPI skeleton, Postgres connected, WebSocket echo test working | Boilerplate running end-to-end (empty but connected) |
| **2** | Feature A: mic capture in browser → audio chunks → Whisper API → captions rendered on teacher screen | Teacher sees their own live captions |
| **3** | Feature A: broadcast captions to student view via WebSocket room, basic two-page demo working | Teacher speaks, student browser shows live captions |
| **4** | Feature A: build/record 50-100 ISL clip dictionary, implement word-lookup + clip player on student side | Captions trigger matching sign clips |
| **5** | Feature B: MediaPipe Hands integrated client-side, record gesture templates, build cosine-similarity matcher | Webcam detects a hand sign, logs matched label to console |
| **6** | Feature B: wire gesture match → TTS → broadcast → classroom output; test end-to-end | Teacher signs "yes/no/stop" → classroom hears synthesized voice |
| **7** | Feature C: transcript accumulation, "End Lecture" button, notes generation LLM call, render notes panel | Click button, get real generated notes from a real lecture |
| **8** | Feature C: quiz generation LLM call, approve/publish flow, student notes/quiz page | Full notes→quiz→publish→student-view loop working |
| **9** | Polish: UI pass (Tailwind cleanup), error handling (mic permission denied, API timeouts), demo script rehearsal | App looks presentable, doesn't crash on edge inputs |
| **10** | Buffer + demo rehearsal: fix whatever broke on Day 9, record backup demo video, prep judge Q&A answers (honest framing of gesture vocab limits, ISL clip-library scope) | Stable demo + backup video + talking points |

**Risk buffer built in:** Day 10 is intentionally not a build day — hackathon demos die from last-minute live-demo failures, not missing features. A recorded backup video is non-negotiable.

---

# 10. Future Scope Architecture (explicitly NOT for the 10-day build)

Keep this as a separate slide — "what we'd build next," not part of the MVP demo.

| Future capability | Why deferred |
|---|---|
| Edge devices (Raspberry Pi/Jetson) for offline classroom operation | Requires hardware procurement + embedded deployment testing — incompatible with a software-only 10-day sprint |
| Generative real-time ISL avatar (full sentence-level sign generation) | No production-ready open ISL generation model exists; needs a real research/data effort (ISLRTC corpus partnership) |
| LangGraph multi-agent orchestration | Only justified once there are actually multiple branching/looping workflows to coordinate — MVP's pipeline is linear |
| RAG over NCERT/state board content for curriculum-grounded notes | Valuable for production trust/accuracy, unnecessary complexity for a demo where "AI generated decent notes" is already impressive |
| MCP integration with School ERP / LMS / Attendance / Curriculum systems | No real school IT system to integrate against in a hackathon; would be building against a mock anyway |
| Multi-classroom / multi-school tenancy, district dashboards | Demo is single-classroom; multi-tenant data modeling adds schema complexity with no demo payoff |
| Kubernetes, multi-region deployment, Kafka/event-bus | Solves a scale problem the MVP doesn't have yet |
| Braille display integration | Needs physical hardware testing, separate output pipeline — phase 3+ |
| Full continuous ISL recognition (not fixed gesture vocabulary) | Genuinely unsolved problem at research level, not a 10-day scope item |
| Parent SMS/WhatsApp digest, emotion/engagement analytics | Nice product extensions, zero relevance to "does the core accessibility loop work" demo question |

This table doubles as your judge Q&A defense: when asked "why didn't you build X," the answer is already written — point at this slide.