# Project development guidelines

These guidelines apply throughout this repository. Read [README.md](./README.md) for the product scope and [docs/architecture.md](./docs/architecture.md) for the proposed system design. Keep these documents aligned as the implementation develops.

## Required direction

- Build a voice assistant that accepts spoken questions and returns spoken and visible answers.
- Use React + TypeScript for the frontend and atomic design for UI components.
- Use LangChain for agent orchestration and tool calling on the backend.
- Use the existing Bun package manager and preserve strict TypeScript settings.
- Treat the folder structure and MVP choices in the architecture document as initial implementation defaults. ElevenLabs is selected for speech recognition and synthesis; the agent model provider, specific speech models/voice, framework versions, and deployment choices remain open.
- Clearly distinguish planned functionality from implemented functionality in documentation.
- All features in [docs/feature-ideas.md](./docs/feature-ideas.md)'s approved phases are planned project scope. Keep the core voice milestone first, preserve explicit storage/action choices, and do not treat its separately labelled new suggestions as approved features.
- Follow [docs/roadmap.md](./docs/roadmap.md) for delivery order, active work, and milestone gates. Keep its status aligned with verified implementation and the README; the manual voice baseline is an integration milestone, and M3 is the complete core MVP.
- Follow [docs/agent-learning.md](./docs/agent-learning.md) for the approved bounded M10 learning extension after M1–M9. Keep LangChain for TypeScript, the direct agent path, and the core voice milestone first; M10 does not add tickets, services, or infrastructure to M1. The three separately labelled further UX suggestions remain unapproved.
- Follow [docs/scaling.md](./docs/scaling.md) for runtime/recovery boundaries. PostgreSQL for owned durable records, separate temporary Redis before replicas, and private object storage are proposed defaults, not installed infrastructure or selected hosting. Add only the stores/processes required by the active milestone.

## React: atomic design

Place UI components in `apps/web/src/components/` under these directories:

| Directory | Purpose | Examples |
| --- | --- | --- |
| `atoms/` | Reusable primitives with typed props | `Button`, `TextInput`, `Badge` |
| `molecules/` | Small combinations of primitives | `MicrophoneControl`, `MessageBubble` |
| `organisms/` | Complete interface sections | `VoicePanel`, `ConversationPanel` |
| `templates/` | Layouts that receive content through props or slots | `AssistantLayout` |
| `pages/` | Screens that connect layouts to feature state | `AssistantPage` |

Composition rules:

- Higher levels may compose lower levels. Atoms must not depend on molecules, organisms, templates, or pages; lower levels must not import higher levels.
- Components at the same level may compose each other when useful, without circular imports. Do not create a wrapper just to pass through every level.
- Keep atoms and molecules presentation-focused. Pass data and callbacks through props.
- Keep microphone lifecycle, API requests, session state, and audio playback in feature hooks and services. Pages connect that behavior to components.
- Put voice and conversation logic under `features/voice/` and `features/conversation/`. Atomic design applies to UI components, not hooks, services, or backend modules.
- Use PascalCase for component files and names, and `use`-prefixed names for hooks.
- Use shared styles and design tokens instead of duplicating colors, spacing, and typography.
- Provide semantic controls, keyboard access, accessible labels, and visible recording and loading states.

## Browser audio execution

- Follow [docs/browser-audio.md](./docs/browser-audio.md). React/main-thread code owns the DOM, microphone permission, AudioContext setup/lifecycle, controls, and compact status updates.
- Keep the native MediaRecorder path as the baseline/fallback. Do not add a Worker solely for an asynchronous fetch or wrap native recording with redundant custom encoding.
- In the streaming path, use AudioWorklet for bounded, deadline-sensitive sample capture/playout. Keep expensive conversion, encoding, voice activity detection (VAD), silence timing, and transport framing in a Dedicated Worker when required.
- Transfer pooled buffers through typed messages with session ID, turn ID, generation, and sequence number. Do not put PCM frames or every audio meter update into React state.
- Bound worker queues, socket buffers, and playback buffers. Recycle transferred buffers; ignore stale-generation messages. Keep shared memory optional and require cross-origin isolation if introduced.
- Local stop must silence playback immediately and invalidate outstanding audio before waiting for worker/server cancellation. Clean up ports, worklets, workers, tracks, sockets, and audio resources on session disposal.
- Prefer streaming STT during recording and streaming TTS after final answer commit for the performance target. Partial transcripts remain draft input; do not run the agent until the utterance is finalized.
- Make `end_input` a capture-epoch/last-sequence barrier: validate and forward all required frames in order before provider commit. Reject gaps or uncertain continuity rather than treating incomplete input as a full question. Confirmed speech can invalidate/flush the playback worklet directly from the audio worker; React/server work must not delay local audible stop.
- Derive visible status from separate capture, turn-processing, and playback states; transcription can overlap recording and synthesis can overlap playback.

## Backend: LangChain agent

- Keep agent construction, model configuration, prompts, and tool implementations in `apps/server/src/`.
- Start with LangChain's TypeScript `createAgent` API and explicit tool registration, following the [official overview](https://docs.langchain.com/oss/javascript/langchain/overview). Check the documentation for the installed version before writing integrations.
- Keep the agent independent from speech providers and HTTP handlers: it receives text plus session context and produces a final answer plus tool activity.
- Keep speech-to-text and text-to-speech in separate adapters. Changing a speech provider should not require rewriting agent or UI components.
- Use ElevenLabs standalone speech APIs for the initial STT/TTS adapters, with its official JavaScript server SDK where applicable. Keep LangChain as the agent, retain app-owned turn/cancellation rules, and verify model-specific Urdu support before enabling voice.
- Select the speech model together with its supported API/transport, formats and capabilities; model IDs are not interchangeable across endpoints. Document actual speech/model-provider retention before live use; temporary app history does not imply provider zero retention.
- Keep provider credentials and privileged operations on the server. The frontend must not import server code, provider SDKs requiring secret keys, or LangChain agent implementations.
- Give each session its own history. Bound history size, request duration, and tool-call iterations; reset context when the user starts a new conversation.
- A turn works on an isolated history snapshot. Commit finalized context only while its turn ID and session generation are still current. Gate content/progress events and speech dispatch with the same check; late cancelled work must not change backend history. Only the coordinator may emit terminal cancellation events after verifying ownership and the affected turn's cancelled state.
- Map LangChain events to an allowlist of public progress events and final answer text. Do not forward arbitrary model streams: intermediate model text can precede another tool call.
- Keep temporary history in memory for one replica. Before adding replicas, use shared expiring state for history, active-turn ownership, deduplication, cancellation, and event replay. Leases require fencing/version checks.
- Treat shared-state loss/failover as authority uncertainty; invalidate affected old store incarnations/sessions and reject stale work. A Redis lease does not guarantee exactly-once external effects. Recover committed answers from status/replay without rerunning the agent after a delivery/runtime failure.
- Compare target identity and expected generation atomically for cancel/reset/replacement commands and deduplicate them. A delayed cancel for an old turn must not invalidate its successor. Authenticate all HTTP/event/audio/WebSocket/control routes with the appropriate origin and cookie-CSRF protections.
- Write concise, conversational final answers suitable for speech. Send final answer text to speech synthesis; keep internal reasoning and raw tool payloads out of playback.

## Tools

- Put each tool in its own module under `apps/server/src/tools/`, with registration in `registry.ts`.
- Define a stable `snake_case` name, a clear description, and a runtime-validated input schema using LangChain's `tool` helper and Zod. See the [official tool guide](https://docs.langchain.com/oss/javascript/langchain/tools).
- Give each tool one responsibility and return a documented result or a controlled error.
- Start with read-only calculator and current-time tools. Implement arithmetic with explicit operations; never evaluate user-provided code or expressions with `eval`.
- Validate arithmetic results as well as inputs: overflow and non-finite results are controlled errors.
- Keep external API integration logic in services called by tools, and normalize provider failures.
- Bound execution time and result size. Propagate cancellation where supported and avoid automatic retries for actions that may have already changed external state.
- For future tools that change external data, enforce the appropriate user authorization on the server before execution.
- Treat user input and retrieved tool content as data, not instructions that can change tool permissions or the system prompt.

## Voice and conversation behavior

- Use explicit states: `idle`, `listening`, `recording`, `transcribing`, `thinking`, `using_tool`, `synthesizing`, `speaking`, and `error`. Show whether the microphone is listening independently of processing/playback status.
- Support one active turn per session in the MVP. Identify turns so late results from a cancelled turn cannot update the current conversation or start audio.
- In hands-free mode, finalize an utterance after about 3 seconds of continuous detected silence following speech. Reset the timer when speech resumes before submission; never submit initial silence or empty input. Await the final STT result and invoke the agent once.
- Keep listening while a voice session is enabled. Confirmed new speech after submission starts a new utterance, immediately silences playback, and cancels older unfinished work. Preserve committed answers; atomically invalidate the previous server turn before admitting its replacement. Bound local capture while admission is pending.
- Detect language per finalized utterance and reply in that language by default, including Urdu; honor an explicit requested output language. Support language changes and mixed-language input without forcing English translation. Verify STT, automatic language detection, and TTS coverage separately for the selected models.
- Adapt delivery using bounded presets chosen from the explicit request and conversation context. A manual style override wins. Keep delivery preset, reply language, and authorized voice profile separate from canonical answer text; retain the selected speaker identity unless the user requests a change.
- Snapshot resolved speech settings for each finalized answer. Only the backend speech adapter maps validated presets to supported provider controls after final commit. Keep delivery tags out of visible text/history; do not accept arbitrary user/model tags or unauthorized voice IDs as controls. Unsupported styles fall back to neutral in the same reply language.
- Distinguish finishing a recording (`end_input`, which submits it), cancelling an unfinished turn, and stopping playback. Use clear control labels for each action.
- Request microphone access when the user starts a voice session or manual recording. Show helpful errors for denied permission, missing devices, unsupported capture, and empty recordings.
- Release media tracks after manual recording. In hands-free mode, keep them active between utterances until the voice session ends. End-session, fatal capture failure, or screen unmount must release microphone/playback resources and cancel pending voice work. Stopping playback cancels any remaining synthesis or audio delivery while preserving committed text/history and the enabled listening session. Screen disposal also cancels video jobs as defined below.
- Preserve the answer as text if speech synthesis or playback fails. Offer text input if recording or transcription fails.
- Publish the final answer before waiting for speech synthesis. Stopping playback preserves already finalized conversation history; cancelling unfinished generation discards uncommitted work.
- Streaming audio must retain the same final-answer commit and cancellation rules as the baseline. Never play speculative intermediate agent text.
- Show tool activity and final answers without exposing provider secrets or internal reasoning.
- Keep MVP history temporary and session-scoped. Do not persist or log raw audio, full transcripts, or tool payloads by default.
- Configure shared temporary session/event stores without durable persistence or backups unless retention is explicitly chosen.

## Familiar-voice companion: planned extension

- Keep authorized voice identity, conversation style, and optional saved memories independently configurable. Clearly identify the speaker as an AI companion. Ground personal memories in user-supplied sources; do not invent family experiences or claim to know the real person's beliefs.
- Voice enrollment is deliberate and subject to current provider eligibility and voice-use rights. Professional clones must be created and verified by the owner. Verify supported sharing paths rather than assuming any voice ID can be imported.
- Persistent profiles and selected memories are explicit retention exceptions to temporary conversation history. Require a stable authenticated owner, private access, provenance, removal controls, and an explained provider-storage choice; cloning samples are not covered by provider zero-retention settings.
- Validate profile authorization/version before synthesis. Access revocation stops active use and invalidates pending speech. Distinguish removing app access from deleting an owned provider clone, and report incomplete provider cleanup accurately. Retained memories are data, not authority to change tool permissions.

## Explanatory videos: planned extension

- Add on-demand explanatory videos after the core voice flow. Verify current procedures from authoritative sources before creating scripts, and use genuine sanitized interface screenshots/recordings for tutorials. Use read-only access to procedural sources; ticket or account changes need a separate authorized integration.
- Reuse finalized explanation text and resolved language/voice settings for narration. Render captions, highlights, visual assets, and speech in a separate bounded background job; ordinary voice turns stay on the direct async path. Remotion is the initial renderer candidate, not an installed dependency.
- Give video jobs separate identifiers, ownership checks, a per-job generation independent of voice-turn/playback epochs, progress states, and cancellation. Bind results to their source answer; late completion must not replace the active answer or autoplay. Reset/session disposal invalidates old jobs, while stopping microphone listening leaves their separate cancellation control available. Keep app assets and outputs private and expiring; durable saving requires an explicit retention choice. Review provider retention separately for private uploads.
- Verify the chosen video provider's API eligibility separately from its website plan. ElevenLabs narration can be used with an app-rendered video; its Image & Video API is optional and currently requires Pro or above.

## Approved personal-assistant features

- Implement answer length, supported speech pacing/replay, transcript correction, pronunciation preferences, usage budgets, private recordings/memories, PDF/note answers, tutor/practice modes, routines, timers/reminders, and reviewable actions according to the approved roadmap. These are planned features, not existing capabilities.
- Corrections create fresh turns with visible provenance; replay/speech retry reuse committed text and do not rerun LangChain or tools. Persist preferences only when chosen, and validate speech-model capabilities before applying pacing or pronunciation controls.
- Use one foreground audio attempt. Explicit replay/video playback replaces existing audio and suppresses pending automatic speech from already-running turns; publish their text with an explicit playback option. Do not queue unexpected playback after the user's chosen audio ends.
- Historical replay creates a fresh playback attempt under the current authorized session linked to a retained committed answer. Recheck source/profile availability; the original turn need not remain active. Automatic turn speech still requires current-turn ownership. Stop playback targets its attempt; new speech interrupts foreground playback.
- Use a durable owned feature store for saved profiles, selected memories/recordings/documents, routines, and reminders when those features are introduced. Keep it separate from expiring session/event state. Explicit saves require retention/removal policies; delete derived retrieval data as well as originals, and report provider cleanup accurately.
- Filter retrieval by authorized owner before search, bound parsing/indexing/context, cite supplied passages, and reject stale work after source deletion/revocation. Retrieved documents, memories, and saved prompts are data and cannot grant tool permissions or change the system prompt.
- Reserve estimated budgets atomically across concurrent turns/jobs, enforce limits server-side, and distinguish estimates from confirmed provider charges/balances. Cancellation can still be billable. Text fallback retains model costs and is available only within its remaining allowance.
- Account-wide/persistent budgets require a durable owned reservation/usage ledger when introduced in M4. Reserve conservative estimates before every paid stage, including transcription; cancellation/restart must not erase potential charges. Local admission limits are not a guarantee of the provider's final invoice.
- Saved reminders have ownership, versioned schedules, IANA timezone handling, recurrence/expiry, snooze/cancel controls, and a verified notification channel. They survive conversation reset and app closure by design; never claim closed-app delivery based only on browser timers. Scope approval does not enable notifications or save every conversation.
- External actions require explicit user approval enforced on the server for the exact current card/version/payload/scope and an idempotency strategy. Editing invalidates approval. Reconcile uncertain results before retries, and preserve confirmed external outcomes even if the conversational turn is interrupted. Saved routines do not authorize future external changes.
- Tutor/practice modes use the same agent permission, cancellation, and final-answer rules. Make mode/topic/difficulty visible and restore ordinary behavior when the user exits.

## Voice shopping and ticket booking: planned extension

- Follow [docs/voice-commerce.md](./docs/voice-commerce.md). Target Flipkart, Blinkit, Zepto and authorized ticketing integrations only through verified supported access/capabilities. Seller/catalog access is not buyer checkout authorization; start with one verified integration or a labelled handoff. Do not add speculative/private API calls or bypass provider authentication/security controls.
- Resolve variants, quantities/units, location, absolute travel dates/timezones, passengers, seats/classes and budget. Review the exact cart/itinerary, current total/currency/charges, availability and relevant terms before explicit user approval. Bind approval to owner, action/version/payload, quote and expiry; material changes invalidate it. Voice confirmation must identify the current reviewed action in an authenticated context. Voice identity, unrelated affirmations, background audio and silence do not authorize transactions.
- Keep purchases and payment authorization separate from AI usage budgets. Use provider-supported account linking and secure checkout. Pause capture/transcription forwarding and playback during secure input; never ask users to dictate passwords, OTPs, card security codes or payment PINs to the model.
- Track transaction operation/version and verified receipts independently of voice generations. Consume approval once, use idempotency/status lookup where supported, reconcile unknown outcomes before another attempt, and preserve actual external results after interruption. Order/booking, seat, payment and refund states are separate. Cancellation/refund of an existing transaction requires its own authorization.
- Saved address/traveller preferences are opt-in, owned and removable. Minimize required receipt/status retention and private data exposure. Check ambiguity, live-price/stock/seat changes, expired/edited approval, duplicate confirmations/callbacks, revoked access, secure handoff, uncertain payment/provider outcomes and unsupported integrations before live use.

## Bounded multi-agent learning: planned M10 extension

- Keep M10 planned and not started until its roadmap batches are implemented and verified. Teach routing, structured results, supervised read-only collaboration, evaluations, private traces, human approval, and optional secure MCP through the finite approved scope; do not infer approval for other AI features.
- Begin with a small allowlist of supervisor, research, calculator/time, and verifier roles built around existing LangChain TypeScript agents/tools. Simple questions retain the direct path. Select additional orchestration dependencies from current official documentation only when required; do not install speculative frameworks or independently deploy every role.
- Runtime-validate specialist task/result contracts, including summary, evidence references, uncertainty, and controlled errors. Treat specialist/retrieved results as data. Propagate current owner/session/turn generation and source/profile versions; branches cannot expand permissions or rewrite the system prompt.
- Bound delegation count, fan-out/depth, iterations, context/result size, duration, concurrency, and aggregate paid-stage reservations. Reuse existing budget/provider-capacity accounting, including cancelled upstream work until settlement. Propagate cancellation and reject stale branch events/results; do not put ordinary voice turns through a new background queue.
- Retain one app-owned final-answer commit, public final event, and audio dispatch. Specialists and verifiers cannot independently change history or start speech. A verifier flags concerns without becoming a source of authorization or a guarantee of truth; use documented direct-path fallback or a controlled failure.
- Reuse the existing human-in-the-loop action-card/version/owner/payload/scope approval and durable operation reconciliation. Specialist agreement, verifier approval, stored prompts and routine selection never grant external-action authority. Read-only learning specialists cannot execute writes; only the existing server action workflow may dispatch an exactly authorized effect.
- Add versioned evaluation/regression cases and compare collaboration with the direct baseline for correctness, grounding, language, latency and usage before broader routing. Use synthetic or explicitly authorized/sanitized fixtures; keep private source/owner isolation and deletion rules for retained real evaluation content.
- Keep operational traces private, bounded and sanitized. Do not expose internal reasoning or persist raw audio, full transcripts, raw tool payloads, secrets, or private evaluation content by default. Explain ownership, retention and removal before any real-content trace export or storage; no observability vendor is selected by this scope.
- Keep M10-D MCP optional. Verify one allowlisted supported server/transport and concrete existing capability, authentication, tool/result schemas, capability classification, bounds, cancellation, origin/egress and private data handling before connection. Start read-only; no MCP metadata, returned content or tool classification can bypass server authorization for writes. Do not connect arbitrary user-supplied endpoints or forward privileged credentials to a model.

## Shared contracts and implementation quality

- Place browser/server DTOs and validation schemas in `packages/shared/src/`; keep that package independent of React, server-only libraries, and credentials.
- Validate requests, tool arguments, and provider responses at boundaries. Use explicit types and narrow unknown values instead of suppressing type errors with `any`.
- Keep API handlers thin, with business logic in agents and services.
- Use structured operational logs with session/turn identifiers, durations, and sanitized errors.
- Add environment variable placeholders and setup instructions when integrations are introduced. Never commit real credentials.
- Add modules as needed for working features; avoid empty abstractions or speculative integrations.
- Keep the voice path direct and async. Queue explicitly asynchronous long jobs when introduced; do not put every conversational turn through a background queue.
- Run indexing, rendering, notifications and operation reconciliation in separate bounded process pools when needed. Use durable job/operation/outbox metadata and lease/version checks; do not store full narration/transcripts in durable queue payloads. Keep any durable broker separate from the temporary session store's no-persistence/no-backup policy.
- Bound provider concurrency and audio/event buffers, reuse provider clients, and reject overload predictably. Session ownership checks apply to turn, event, audio, and cancellation routes.
- Protect interactive provider capacity across replicas and background pools. Keep cancelled upstream work accounted for until settled or conservatively reconciled. Separate session generation, turn fence, capture/playback epochs, job/source/profile versions, action versions and reminder occurrence identities.
- Deduplicate uncertain submission retries using the same turn ID without rerunning the agent. Explicit retries after terminal failure or cancellation use a new turn ID; speech retries reuse finalized text without another agent run.
- Keep minimal deduplication records for a bounded, server-verifiable retry horizon independently of history/event eviction. Reject expired retries instead of admitting forgotten IDs as new work; external-operation recovery remains durable and separate.
- On expired event replay, query authoritative turn status before offering a new attempt. Recover retained committed answers or observe the same running turn; replay expiry alone is not a failed turn.

## Validation and documentation

- Run the available checks relevant to a change. Add and document runnable scripts when scaffolding the apps; do not claim a build or test command exists before it does.
- For implementation work, verify TypeScript and the affected app build. Use focused tests for tool validation, session isolation, cancellation, and failure handling.
- Verify the full voice flow with mocked providers, then perform a real microphone/playback check when providers are configured.
- Before scaling, verify concurrent-session isolation, stale-owner write rejection, event reconnection without duplicate runs, and bounded resource use. Measure stage latency and provider saturation before setting performance targets.
- For audio worker changes, compare native and worker paths on representative devices; inspect long tasks, frame delivery, underruns, message overhead, memory growth, stop responsiveness, and provider latency independently.
- Cover permission denial, a failed tool, failed transcription, failed synthesis, and a late result after cancellation where relevant to the change.
- Verify the hands-free silence threshold, speech resuming around submission, no empty/duplicate turns, and interruption during processing/playback. Test background noise and speaker echo so the assistant's own voice does not trigger a new question.
- Verify Urdu recognition and spoken output, language changes, mixed-language speech, and uncertain/unsupported-language recovery with representative recordings.
- Verify context-based delivery, manual overrides, model/voice capability checks, and neutral fallback. Test expressive playback echo and cancellation so a late style/synthesis result cannot restart old audio. Speech retries reuse finalized text without rerunning the agent.
- For video changes, verify source accuracy, screenshot/caption readability, narration synchronization, Urdu right-to-left rendering, failed jobs, cancellation/stale completion, and private result expiry.
- For companion/profile changes, verify ownership isolation, grounded memories, profile revocation during synthesis, provider failures, and local/provider removal behavior.
- For approved extension changes, verify private document retrieval/deletion, recurrence and notification deduplication, concurrent budgets, exact action-card approval/idempotency, correction/replay provenance, and failure recovery relevant to that feature.
- For M10 changes, verify the direct-path baseline, routing/result contracts, bounded fan-out/cost, private trace handling, malformed or hostile specialist content, source/owner isolation, stale/cancelled branches, one final answer/audio dispatch, and unchanged server approval/operation recovery. Check optional MCP authentication/capabilities/egress and denial/failure paths only when that adapter is enabled; a verifier vote alone does not satisfy the exit gate.
- For documentation-only changes, check links, consistency, and the diff; no application tests are required.
- For runtime/store changes, verify crash-after-commit recovery, expired worker claims, job duplicates, source deletion during work, quota isolation and durable restoration. Suspend restored external dispatch/notification retries until ownership, deletion and possible prior effects are reconciled; stale backups must not repeat purchases or reminders.
- Update the README when setup commands, implemented features, or architecture decisions change. Report which checks ran and any remaining limitations.
