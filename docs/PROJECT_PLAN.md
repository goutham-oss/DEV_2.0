# DEV 2.0 — Architecture Analysis, Technical Specifications & Development Plan

**Project Repository:** `DEV-2.0` ([https://github.com/goutham-oss/DEV-2.0](https://github.com/goutham-oss/DEV-2.0))  
**Target Platform:** Windows (with extensible abstraction for cross-platform)  
**Primary Tech Stack:** Python 3.11+, Google Gemini API, ElevenLabs API  

---

## 1. Executive Summary & Concept Analysis

DEV 2.0 is conceived as an AI-powered personal desktop companion utilizing a **hierarchical, modular multi-agent architecture**:
```
User (Voice / Text)
       │
       ▼
 ┌────────────┐       Internal Message Protocol
 │Master Agent│ ──────────────────────────────────────┐
 └─────┬──────┘                                       │
       │ Intent & Routing                             │
       ▼                                              ▼
┌──────────────┐                             ┌────────────────┐
│ System Agent │                             │App / File / ...│
└──────┬───────┘                             └────────────────┘
       │ Skill Dispatch
       ▼
 ┌───────────┐
 │VolumeSkill│ ──> Windows Audio Endpoints (pycaw)
 └───────────┘
```

### Key Strengths of the Proposed Concept
1. **Decoupled Responsibilities:** Separation between reasoning (Master Agent) and execution (Specialized Agents & Skills) prevents prompt bloat and prevents hallucinated direct execution.
2. **Capability Extensibility:** The strict hierarchy `Capability -> Skill -> Agent -> Master Agent` enables plugin-style expansion without modifying core orchestration logic.
3. **Dedicated Reasoning Layer:** Utilizing Google Gemini solely for conversational context, natural-language comprehension, and intent delegation—while maintaining an internal deterministic agent bus—keeps the system robust and predictable.

---

## 2. Identified Missing Architectural Requirements

Before code implementation begins, several architectural aspects missing from the initial high-level brief must be formally specified:

### 2.1. Internal Agent Communication Protocol (IACP)
The prompt mandates an internal communication system independent of Gemini. We must define:
- **Strict Structured Schemas (Pydantic v2):**
  - `TaskRequest`: `task_id`, `trace_id`, `agent_name`, `skill_name`, `parameters`, `requires_confirmation`, `timeout_seconds`.
  - `TaskResponse`: `task_id`, `status` (`SUCCESS`, `FAILED`, `REQUIRES_CLARIFICATION`, `CANCELLED`), `data`, `error_message`, `execution_time_ms`.
- **Communication Pattern:** Async Request-Response with event telemetry over an in-process asynchronous bus (`asyncio` queue / event dispatcher) with timeouts and cancellation tokens.

### 2.2. Safety, Permissions & Human-in-the-Loop (Guardrails)
Desktop assistants have dangerous access levels (file deletion, closing unsaved apps, system state changes).
- **Action Risk Classification:**
  - *Tier 0 (Safe/Read-only):* Get volume, check time, search files, read battery. (Auto-execute).
  - *Tier 1 (Benign Mutation):* Adjust volume, set brightness, open browser URL. (Execute with voice/text feedback).
  - *Tier 2 (Destructive/Irreversible):* Close unsaved application, delete files, system shutdown/restart, batch operations. (**Requires explicit confirmation prompt before execution**).
- **Execution Guard:** Master Agent halts pipeline and prompts: *"Are you sure you want to delete 'report.docx'?"* before passing execution to the specialized agent.

### 2.3. Agent & Skill Discovery / Registry
- A self-describing **`AgentRegistry`** where specialized agents register their metadata and skills dynamically.
- Each skill exports a standardized JSON schema (compatible with Gemini Function Calling / Tool Declarations) so the Master Agent automatically inherits skill capabilities without hardcoded prompt adjustments.

### 2.4. Conversational Context & Memory Architecture
- **Short-term Memory (Working Context):** Sliding window of conversational turns (user utterances + agent summaries).
- **Task Memory (Multi-turn workflows):** Tracking pending clarifications (e.g., User: *"Close the browser"* -> Agent: *"You have 3 windows open. Close Chrome or Edge?"*).
- **Persistent Preferences (Long-term Store):** User defaults (preferred browser, default volume step, app aliases like *"VS Code"* for `Code.exe`). Local SQLite or JSON store.

### 2.5. Voice Pipeline Latency & Audio Flow
A voice assistant feels sluggish if round-trip latency exceeds 1.5–2 seconds.
- **Audio Pipeline Phases:**
  1. *Audio In:* Microphone capture (vad/push-to-talk/wake-word).
  2. *STT (Speech-to-Text):* Options include local Whisper (low latency, zero API cost, offline) vs Cloud STT.
  3. *Reasoning & Dispatch:* Gemini parses intent and calls internal tool.
  4. *Agent Execution:* Local OS command execution (<50ms).
  5. *Audio Out (TTS):* ElevenLabs streaming audio output (`/v1/text-to-speech/{voice_id}/stream`) played through speaker buffer.
- **Interruption / Barge-in:** If the user speaks while DEV is talking, the audio output stream must immediately cancel.

---

## 3. Proposed Project Structure

A clean, modular Python project structure designed for extensibility and strict separation of concerns:

```text
DEV-2.0/
├── docs/
│   ├── ARCHITECTURE.md          # Detailed architecture & protocol specifications
│   ├── PROJECT_PLAN.md          # Phased roadmap and status tracker
│   └── adr/                     # Architectural Decision Records
│       ├── ADR-001-agent-protocol.md
│       ├── ADR-002-windows-os-layer.md
│       └── ADR-003-voice-pipeline.md
├── config/
│   ├── dev_config.yaml          # Application settings (timeouts, thresholds, voices)
│   └── logging_config.yaml      # Structured logging settings
├── dev_core/                    # Core framework engine
│   ├── __init__.py
│   ├── config.py                # Pydantic Settings & environment manager
│   ├── bus/                     # Internal Communication Protocol
│   │   ├── __init__.py
│   │   ├── messages.py          # TaskRequest, TaskResponse, Event models
│   │   └── dispatcher.py        # Async in-memory message dispatcher
│   ├── registry/                # Dynamic discovery
│   │   ├── __init__.py
│   │   └── agent_registry.py    # Agent & Skill catalog
│   ├── base/                    # Abstract Base Classes
│   │   ├── __init__.py
│   │   ├── base_agent.py        # BaseAgent interface
│   │   └── base_skill.py        # BaseSkill interface
│   ├── memory/                  # Session & persistent memory
│   │   ├── __init__.py
│   │   └── session_memory.py    # Short-term context & multi-turn state
│   ├── security/                # Safety & confirmations
│   │   ├── __init__.py
│   │   └── guardrails.py        # Action risk tiering & verification hooks
│   └── master_agent/            # Central orchestrator
│       ├── __init__.py
│       ├── orchestrator.py      # Master Agent loop
│       ├── gemini_client.py     # Gemini API integration & Tool binding
│       └── prompt_templates.py  # System prompts & persona guidelines
├── agents/                      # Specialized Agent Implementations
│   ├── __init__.py
│   ├── system_agent/            # Hardware & OS settings
│   │   ├── __init__.py
│   │   ├── agent.py
│   │   └── skills/
│   │       ├── volume_skill.py
│   │       ├── brightness_skill.py
│   │       └── power_skill.py
│   ├── application_agent/       # Process & App management
│   │   ├── __init__.py
│   │   ├── agent.py
│   │   └── skills/
│   │       ├── launch_app_skill.py
│   │       ├── close_app_skill.py
│   │       └── list_apps_skill.py
│   ├── file_agent/              # File system operations (Post-MVP)
│   ├── browser_agent/           # Web interaction (Post-MVP)
│   └── utility_agent/           # Math, time, utilities (Post-MVP)
├── audio/                       # Audio & Voice Pipeline
│   ├── __init__.py
│   ├── stt/                     # Speech-to-Text adapters
│   │   ├── base_stt.py
│   │   ├── whisper_stt.py       # Local fast Whisper
│   │   └── cloud_stt.py
│   ├── tts/                     # Text-to-Speech adapters
│   │   ├── base_tts.py
│   │   └── elevenlabs_tts.py    # ElevenLabs streaming TTS
│   └── voice_controller.py      # Audio I/O loop, VAD & barge-in handler
├── interfaces/                  # User interaction frontends
│   ├── cli/                     # Interactive CLI / REPL for testing
│   │   └── main.py
│   └── gui/                     # Future tray app or desktop UI
├── tests/                       # Automated test suite
│   ├── unit/                    # Test individual skills & protocol serialization
│   ├── integration/             # Test MasterAgent -> Dispatcher -> SystemAgent
│   └── mocks/                   # Mock Gemini & ElevenLabs fixtures
├── .env.example                 # Template for GEMINI_API_KEY, ELEVENLABS_API_KEY
├── pyproject.toml               # Poetry/Hatch/Pip dependencies
└── README.md                    # Project documentation & setup instructions
```

---

## 4. Key Technical Risks & Proposed Mitigations

| Risk | Impact | Root Cause | Proposed Mitigation |
| :--- | :--- | :--- | :--- |
| **Windows OS API Fragility** | High | Controlling brightness across mixed monitor setups (DDC/CI desktop monitors vs laptop WMI) or audio endpoints (`pycaw`) can fail or throw COM errors. | Use hardened wrappers (`screen_brightness_control` with fallback handling, `pycaw` with COM threading initialization `coinitialize()`). Isolate OS calls inside skill try-except blocks with descriptive error feedback. |
| **Voice Round-Trip Latency** | High | Cloud STT + Gemini reasoning + Cloud TTS (ElevenLabs) can accumulate 2–4s delay, making conversational flow feel sluggish. | Use streaming TTS where audio playback starts on the first generated sentence tokens. Consider lightweight local STT (e.g., `faster-whisper`) for instant transcription, keeping external roundtrip bounded to Gemini + ElevenLabs. |
| **Accidental Destructive Execution** | Critical | LLM misinterprets an ambiguous phrase (*"close everything"*, *"delete the test folder"*) and terminates work or deletes user files. | Strict Action Risk Classification (Tier 0 to Tier 2). Destructive tasks require explicit confirmation state before skill execution. |
| **UWP / Windows Store App Incompatibility** | Medium | Standard executable launches (`subprocess.Popen`) fail on Windows Store packaged applications (e.g., Windows Calculator, WhatsApp, Spotify). | Implement hybrid application discovery: Windows Registry search + `shell:AppsFolder` UWP protocol activation (`explorer.exe shell:AppsFolder\...`). |
| **API Rate Limits & Cost Accumulation** | Medium | Continuous voice interaction generating numerous Gemini tokens and ElevenLabs character conversions. | Cache common static responses (e.g., *"Volume set to 50%"* can use pre-cached or lightweight local audio). Use Gemini Flash models (`gemini-2.5-flash` or `gemini-1.5-flash`) for low latency and high rate limits. |

---

## 5. Architectural Decision Proposals (ADRs for Alignment)

Before implementing Phase 1, the following foundational decisions are proposed:

### ADR-001: Language & Async Model
- **Proposal:** Python 3.11+ using native `asyncio`.
- **Reasoning:** Async I/O is critical for concurrent message dispatch, non-blocking audio streaming, and parallel Gemini API calls. Python has first-class libraries for Windows OS APIs (`pywin32`, `pycaw`, `psutil`) and official Google GenAI / ElevenLabs SDKs.

### ADR-002: Gemini Integration Strategy
- **Proposal:** Use Gemini Function Calling / Tool Declarations dynamically compiled from registered skills.
- **Reasoning:** Instead of asking Gemini to output loose JSON text that requires brittle regex parsing, Gemini's structured Tool Calling gives schema-validated arguments for direct mapping into `TaskRequest`.

### ADR-003: Internal Communication Mechanism
- **Proposal:** In-Memory Async Message Bus with strict Pydantic schemas.
- **Reasoning:** Avoids heavy dependencies (like RabbitMQ or Redis) while strictly separating the Master Agent from specialized agent code. If DEV later moves to multi-process or network microservices, the Pydantic protocol remains identical.

### ADR-004: Windows Management Libraries
- **System Agent:**
  - Volume: `pycaw` (Windows Core Audio API) + `comtypes`.
  - Brightness: `screen_brightness_control` (supports both WMI for laptops and DDC/CI for external monitors).
- **Application Agent:**
  - Process monitoring & termination: `psutil`.
  - Launching: `subprocess` + `os.startfile` + Windows `App Paths` registry fallback.

---

## 6. Phased Implementation Roadmap

### Phase 0: Project Setup & Core Foundation (Immediate Next Step)
- [ ] Initialize Git repository configuration & `.gitignore`.
- [ ] Set up Python environment dependencies (`pyproject.toml` or `requirements.txt`).
- [ ] Implement `dev_core.base` (`BaseAgent`, `BaseSkill`).
- [ ] Implement `dev_core.bus` (`TaskRequest`, `TaskResponse`, `MessageDispatcher`).
- [ ] Implement `dev_core.registry` (`AgentRegistry`).
- [ ] Build unit tests validating agent registration and message dispatching without any external APIs.

### Phase 1: MVP Core (Master Agent, System Agent, Application Agent, CLI Interface)
- [ ] Implement `SystemAgent`:
  - `VolumeSkill` (set, get, mute/unmute).
  - `BrightnessSkill` (set, get).
- [ ] Implement `ApplicationAgent`:
  - `LaunchAppSkill` (launch by name/alias).
  - `CloseAppSkill` (terminate by process/window name).
  - `ListAppsSkill` (inspect active windows).
- [ ] Implement `MasterAgent`:
  - Gemini SDK integration with dynamic tool registration.
  - Intent resolution, task generation, and result synthesis.
- [ ] Build interactive Text CLI interface (`interfaces/cli/main.py`) to test and verify end-to-end task execution.

### Phase 2: Audio & Voice Pipeline
- [ ] Speech-to-Text integration (Modular STT adapter with local Whisper / SpeechRecognition).
- [ ] Text-to-Speech integration (ElevenLabs streaming client with speaker queue).
- [ ] Voice Controller orchestration (Listen -> Transcribe -> Master Agent -> Speak).
- [ ] Audio interruption / barge-in handling.

### Phase 3: File, Browser & Utility Agents + Guardrails
- [ ] `FileAgent`: Safe file search, read, move, delete (with Tier 2 confirmation).
- [ ] `BrowserAgent`: Launch URLs, search web queries.
- [ ] `UtilityAgent`: Time, date, calculations, system specs.
- [ ] Session memory & preferences store (user app aliases, default configurations).

### Phase 4: Desktop Interface, Packaging & Advanced Capabilities
- [ ] System Tray / Desktop Overlay UI.
- [ ] Global hotkey / Push-to-talk activation.
- [ ] Extensibility plugin loader for user-defined third-party skills.
