# DEV 2.0 — AI-Powered Personal Computer Assistant

DEV 2.0 is an AI-powered desktop assistant built on a modular, multi-agent architecture. It allows users to interact using natural language and voice, delegating tasks through a specialized agent hierarchy.

## Architecture Highlights
- **Master Agent:** Natural language comprehension, intent extraction, task planning (powered by Google Gemini), and response synthesis.
- **Specialized Agents:** Dedicated agents for System control, Applications, Files, Browser, and Utilities.
- **Skill-Based Modularity:** Fine-grained skills within agents (`VolumeSkill`, `BrightnessSkill`, `LaunchAppSkill`, etc.).
- **Internal Agent Communication Protocol (IACP):** Decoupled, schema-validated task routing independent of external AI APIs.
- **Voice Pipeline:** Low-latency speech interaction combining speech-to-text with ElevenLabs streaming text-to-speech.

## Project Documentation
Detailed architectural blueprints, protocol definitions, and development roadmap can be found in:
- [docs/PROJECT_PLAN.md](docs/PROJECT_PLAN.md)
