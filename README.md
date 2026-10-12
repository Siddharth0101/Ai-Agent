# Voice Agent

A planned personal assistant centered on voice and text conversations, with tools, private familiar voices, saved memories, document help, learning modes, reminders, reviewed shopping/ticket booking, and explanatory videos.

## Project status

Planning stage. This repository currently contains a Bun + TypeScript starter. `index.ts` prints a greeting; the React interface, LangChain agent, tools, and speech services are planned and have not been implemented or installed yet.

Follow the [delivery roadmap](./docs/roadmap.md) for milestone order, demos, completion gates, and progress. **Next: M1-A, scaffold the working frontend/backend/shared packages.**

## What we are building

The first version will let a user:

- Start a voice session, speak a question, and have it submitted automatically after about **3 seconds of silence**.
- Keep listening for the next question while the voice session is enabled. Speaking again interrupts an active reply and starts a new utterance.
- Detect the spoken language and reply in that language, including Urdu. Target broad multilingual coverage, with clear recovery when a selected speech model does not support the input language.
- See the transcription and the assistant's response in a conversation view.
- Receive a spoken answer through audio playback.
- Adapt speech delivery to the request and conversation context: expressive narration for stories, clear pacing for explanations, and neutral delivery for factual answers. Let the user override the style while retaining their selected voice.
- Ask follow-up questions with context from the current session.
- Get answers from tools, starting with arithmetic and current time.
- See listening, recording, transcription, thinking, tool use, synthesis, playback, and error states, with a visible microphone indicator.
- See the final text answer immediately while the app prepares its audio.
- Finish a recording manually, cancel an unfinished turn, stop playback, or end the voice session. Keep manual recording and text input available as fallbacks.

Example: ask “What is 24 multiplied by 7?” The app transcribes the question, the LangChain agent calls the calculator tool, and the app displays and speaks the answer.

## Technology and architecture

Open [the interactive architecture diagram](./docs/architecture.html) in a browser. It includes the full feature system, hands-free browser audio threads, the manual baseline, scaling, and React structure views. The written design and Mermaid diagrams are in [docs/architecture.md](./docs/architecture.md), with [runtime/recovery/scaling](./docs/scaling.md) and [browser audio](./docs/browser-audio.md) details.

| Area | Decision |
| --- | --- |
| Frontend | **React + TypeScript**, using **atomic design** for components |
| Frontend tooling | Vite is the proposed build tool; not configured yet |
| Backend | TypeScript service; Bun is the initial runtime and package manager |
| AI agent | **LangChain for TypeScript**, running on the backend |
| Tools | Modular backend tools with validated inputs and a central registry |
| Speech | **ElevenLabs** for speech-to-text and text-to-speech, behind replaceable backend adapters |
| Browser audio | Native MediaRecorder baseline; AudioWorklet + Dedicated Worker for the streaming target |
| Hands-free conversation | Speech detection, automatic submission after 3 seconds of silence, and interruption when the user speaks again |
| Languages | Automatic language detection and same-language text/spoken replies; Urdu is required and model-specific coverage must be verified |
| Speech delivery | Automatic context-based delivery presets, a manual style override, and a user-selected speaker voice |
| Responsiveness | Keep sample processing outside React; bound buffers and publish compact UI updates |
| Conversation context | Session-scoped, in-memory history for the MVP; saving personal items remains opt-in |
| Saved feature data | Proposed PostgreSQL for owned saved records, account budgets, reminders, operations and job metadata; private object storage for saved/expiring assets |
| Background work | Separate bounded indexing, rendering, notification and reconciliation processes when their features arrive; ordinary voice turns stay direct |
| Horizontal scaling | Shared temporary Redis before voice replicas, with atomic ownership/replay and no persistence/backups; provider quotas shared across all pools |
| Approved extensions | Voice profiles/memory, recordings, documents, tutor/routines, reminders, action previews, voice shopping/booking, and videos |
| Agent model provider | To be selected before the first working agent integration |
| Speech models and voice | Select ElevenLabs models/voice during integration; verify Urdu and the supported language set |

The proposed voice pipeline is:

```text
Microphone → speech-to-text → LangChain agent ↔ tools
                                  ↓
                         Final answer as text
                                  ↓
                         text-to-speech → playback
```

React handles recording, playback, and the interface. The backend handles speech provider calls, the LangChain agent, tool execution, and credentials. Typed questions skip transcription and enter the same agent flow. In hands-free mode, 3 seconds of detected silence ends the utterance; the final transcript is then submitted once. Speech before that timeout continues the same question. Speech after submission begins a new question and cancels any unfinished previous reply. Already finalized answers remain in the conversation when speech is interrupted.

Start with one modular backend. As extensions arrive, add owned durable records and separate worker processes so rendering, parsing, and scheduled delivery cannot block voice or consume all interactive provider capacity. Purchases/bookings have durable operation IDs and receipt reconciliation independent of voice interruption. Add voice replicas only after shared ownership/replay and quota tests pass. PostgreSQL, Redis, object storage and worker dispatch are proposed architecture defaults, not installed infrastructure; deployment vendors and concrete limits remain open and will be measured.

ElevenLabs is the selected initial speech provider. Use its standalone speech APIs through the planned backend SDK, `@elevenlabs/elevenlabs-js`, with LangChain handling answers and tools. The speech integration is not installed or implemented yet; the selected models must support the required languages, including Urdu. The app owns silence timing, interruption, and conversation history.

Temporary history is an app storage policy. Speech/model providers have separate retention and account settings, which must be documented before live integration. Provider zero retention is not assumed. Select each speech model with its supported API endpoint and transport; see [the integration decisions](./docs/architecture.md#decisions-to-make-during-implementation).

The expanded scope includes contextual speech delivery. Choose a small validated preset from the request/context, with the user's explicit preference taking priority. The backend maps that preset to supported ElevenLabs controls after the answer is finalized; visible answers and conversation history keep the original text without delivery tags. Evaluate the Eleven v4 family for expressive speech, verify language and streaming compatibility, and fall back to neutral delivery when a style is unavailable. See [the delivery design](./docs/architecture.md#contextual-speech-delivery). This is planned functionality.

The streaming target transcribes audio while the user speaks and plays TTS audio as it arrives after the answer is finalized. React owns controls and audio lifecycle; an AudioWorklet handles sample capture/playout and a Dedicated Worker handles necessary conversion, speech detection, silence timing, framing, and streaming transport. The microphone stays active during the enabled voice session except for secure-input pauses such as payment/login verification; the initial native recording flow remains a manual fallback. Worker use and provider latency will be profiled separately. See [the hands-free design](./docs/browser-audio.md#hands-free-conversation) for turn boundaries and interruption behavior.

## Planned extension: familiar-voice companion

Add an opt-in mode where users converse with an AI using an authorized loved one's voice. Keep voice identity, conversational style, and optional saved memories separate, with a clear AI-generated label. Users choose which stories or facts to supply; the assistant cites that context instead of inventing personal memories.

Voice enrollment and saved memories require explicit storage choices, private access, and removal controls. ElevenLabs cloning involves provider storage; conversation history remains temporary by default. See [the companion design](./docs/architecture.md#familiar-voice-companion-planned-extension) and [the approved feature roadmap](./docs/feature-ideas.md). This extension is planned, not implemented.

## Planned extension: explanatory videos

After the core voice assistant, add an on-demand video explanation for how-to questions. For example, an IRCTC cancellation/refund question can produce a verified explanation with narration, captions, screenshots, and highlighted steps. Check current official instructions and the relevant ticket case before writing the tutorial; use genuine interface assets rather than invented screens.

ElevenLabs provides the narration. A renderer such as Remotion is the proposed first approach for assembling screenshots and annotations into a video. This keeps ElevenLabs usage in the speech adapters. Generated video clips through ElevenLabs are an optional alternative; its [Image & Video API](https://elevenlabs.io/docs/eleven-api/guides/cookbooks/image-and-video) currently requires Pro or above, separately from website video access.

Give the text/spoken answer promptly and render the requested video as a separate background job with progress, cancellation, and a private expiring result. Video completion does not automatically start playback. This extension is not implemented; see [the video design](./docs/architecture.md#explanatory-videos-planned-extension).

## Planned extension: voice shopping and ticket booking

Describe products or a trip by voice, compare supported options, and review a draft cart or itinerary. Initial targets are Flipkart, Blinkit, Zepto, and IRCTC/authorized ticketing partners. Show exact available details, total charges, location/date/passenger information, and relevant terms before explicit confirmation for a purchase or booking. Changes to the approved proposal require fresh confirmation.

Live execution depends on verified supported provider access; consumer checkout integrations are not implemented or guaranteed by seller/catalog APIs. Start with one supported provider or a clearly labelled checkout handoff. Required payment/login verification stays in the provider's secure flow. The app tracks verified receipts and uncertain outcomes without automatically repeating purchases. See [the commerce design](./docs/voice-commerce.md).

## Approved expanded scope

All previously suggested features are now part of the planned project scope. They will be delivered in phases after the core voice flow:

| Phase | Planned features |
| --- | --- |
| 1: Core usability | Short/detailed answers, pacing/replay, transcript correction, pronunciation preferences, estimated usage and budgets |
| 2: Companion and memory | Authorized voice profiles, separate persona settings, an original-message library, and explicit saved memories |
| 3: Learning and retrieval | Tutor/language/interview practice, one-tap routines, and source-backed PDF/note answers |
| 4: Scheduling and actions | Timers/reminders and an agenda; reviewable action cards; voice shopping/ticket booking through verified integrations or checkout handoff |
| 5: Videos | Verified narrated walkthroughs with captions/highlights, background rendering, and playback controls |

The [approved feature scope](./docs/feature-ideas.md) contains acceptance expectations for every feature; the [delivery roadmap](./docs/roadmap.md) turns those phases into sequential working milestones. Product scope approval does not enable automatic personal-data retention or authorize future messages/account changes: each save, enrollment, scheduled reminder, or external action has its own user choice. All features remain unimplemented.

Three additional ideas are recorded separately for consideration: searchable current/saved conversations, extra accessibility controls, and a pre-session language/voice preview with a microphone check. Basic semantic controls and keyboard access are already core requirements. These three additions have not yet been approved.

## React component structure

All UI components follow the five levels of [atomic design](https://atomicdesign.bradfrost.com/chapter-2/):

| Level | Responsibility | Examples |
| --- | --- | --- |
| Atoms | Small reusable UI primitives | `Button`, `Icon`, `TextInput`, `Badge` |
| Molecules | Small combinations with one UI responsibility | `MicrophoneControl`, `MessageBubble`, `ToolStatus` |
| Organisms | Complete sections of the interface | `VoicePanel`, `ConversationPanel`, `ToolActivityPanel` |
| Templates | Page layouts with slots for content | `AssistantLayout` |
| Pages | Templates connected to actual feature state | `AssistantPage` |

Hooks and services handle recording, requests, and conversation logic outside this UI hierarchy. See [AGENTS.md](./AGENTS.md) for development rules and [docs/architecture.md](./docs/architecture.md) for the planned folder structure and runtime behavior.

The architecture document includes [deployment diagrams](./docs/architecture.md#deployment-and-scaling), a [turn sequence](./docs/architecture.md#turn-sequence), and [performance and reliability rules](./docs/architecture.md#performance-and-reliability). Start with one modular backend; introduce shared session state before adding replicas. The baseline uses HTTP uploads, streamed public events, and authorized audio delivery; the streaming target adds a WebSocket transport to the same coordinator.

## Run the current starter

With Bun installed, run these commands from the repository root:

```bash
bun install
bun run index.ts
```

This runs the existing greeting only. No frontend/backend development, build, or test scripts are configured yet. App commands and a placeholder-only `.env.example` will be documented when the frontend and backend are scaffolded. Provider secrets will stay on the backend.

## Implementation roadmap

The detailed checklist and release gates live in [docs/roadmap.md](./docs/roadmap.md). Planning and initial speech-provider selection are complete; all implementation milestones are not started.

| Order | Milestone | Scope |
| --- | --- | --- |
| M1 | Foundation and text agent | Bun workspaces, atomic React UI, LangChain, calculator/time, reliable sessions |
| M2 | Manual voice baseline | ElevenLabs STT/TTS, recording, transcript, spoken reply, fallbacks |
| M3 | Complete core voice MVP | 3-second silence, continuous listening/interruption, streaming target, Urdu/language handling, contextual delivery |
| M4 | Everyday controls and budgets | Approved Phase 1; durable owned ledger before account-wide budgets |
| M5 | Familiar voices and selected memory | Approved Phase 2; authenticated ownership and durable private storage first |
| M6 | Documents, practice, and routines | Approved Phase 3 |
| M7 | Reminders and action approval | Approved Phase 4 foundations |
| M8 | Voice shopping and tickets | Approved Phase 4 commerce; supported handoff/one verified adapter before more providers |
| M9 | Explanatory videos | Approved Phase 5 |

Finish one working milestone and its acceptance checks before advancing. M2 is an integration baseline; the full core MVP is M3. Delivery dates will follow measured implementation progress. Session context remains temporary by default; saved profiles, memories, recordings, documents, routines, and reminders are deliberate storage choices. New suggestions remain separately unapproved.

## Technical references

- [LangChain TypeScript overview](https://docs.langchain.com/oss/javascript/langchain/overview)
- [LangChain tool definitions](https://docs.langchain.com/oss/javascript/langchain/tools)
- [LangChain voice agent guide](https://docs.langchain.com/oss/javascript/langchain/voice-agent)
- [ElevenLabs JavaScript SDK](https://github.com/elevenlabs/elevenlabs-js)
- [ElevenLabs Scribe realtime transcription](https://elevenlabs.io/docs/eleven-api/resources/libraries/scribe-stt/javascript-scribe)
- [ElevenLabs speech models and supported languages](https://elevenlabs.io/docs/overview/models)
- [ElevenLabs expressive audio controls](https://elevenlabs.io/docs/help-center/product/core-capabilities/text-to-speech/how-do-audio-tags-work-with-eleven-v3-and-v4)
- [ElevenLabs Image & Video API](https://elevenlabs.io/docs/eleven-api/guides/cookbooks/image-and-video)
- [Remotion video renderer](https://www.remotion.dev/docs/renderer)
- [Thinking in React](https://react.dev/learn/thinking-in-react)
- [Vite getting started](https://vite.dev/guide/)
