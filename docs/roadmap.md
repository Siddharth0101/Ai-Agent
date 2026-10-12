# Delivery roadmap

Status: planning complete enough to begin implementation; the repository still contains only the Bun + TypeScript starter. No assistant milestone below is implemented or released. **The next work item is M1-A: scaffold the working apps.**

This document owns implementation order, milestone progress, and release gates. [The feature scope](./feature-ideas.md) owns the approved Phase 1–5 features and their acceptance expectations. [The architecture](./architecture.md), [browser audio design](./browser-audio.md), and [commerce design](./voice-commerce.md) define how they work. Keep those documents aligned when a milestone changes.

[Runtime and scaling design](./scaling.md) adds proposed PostgreSQL accounting/feature storage, separate temporary Redis before voice replicas, private object storage, and independent worker pools at the milestones that need them. These design defaults do not change implementation status or require all infrastructure in M1.

## Release sequence

Each milestone should produce a usable demonstration before the next feature group begins. The early text and manual-voice milestones are integration releases; **M3 is the complete core voice MVP**. Milestone numbers are delivery order, not package versions or promised dates.

| Milestone | Scope mapping | Usable result | Status |
| --- | --- | --- | --- |
| M1: Foundation and text agent | Core foundation | Type a question, receive an answer, and see calculator/time tool activity | Not started; M1-A is next |
| M2: Manual voice | Core integration baseline | Record a question manually, see its transcript, and hear/read the answer | Not started |
| M3: Hands-free voice MVP | Complete core | Speak, pause for about 3 seconds, receive a reply, and interrupt by speaking again; Urdu and contextual delivery work | Not started |
| M4: Everyday controls and budgets | Phase 1 | Correct transcripts, adjust answers/playback, manage pronunciation, and see/enforce usage limits | Not started |
| M5: Familiar voices and selected memory | Phase 2 | Use an authorized AI voice profile and privately save chosen memories/original messages | Not started |
| M6: Documents, practice, and routines | Phase 3 | Ask source-backed document questions, practise a topic, and run editable saved prompts | Not started |
| M7: Reminders and action approval | Phase 4, foundations | Manage reminders/agenda and review exact external actions before authorized execution | Not started |
| M8: Voice shopping and tickets | Phase 4, commerce | Review a cart/itinerary, use a supported checkout path, and see truthful order/booking status | Not started |
| M9: Explanatory videos | Phase 5 | Request a verified narrated walkthrough with captions and visual steps | Not started |

Already completed planning work:

- [x] Document product scope, architecture, browser audio boundaries, and development conventions.
- [x] Select ElevenLabs as the initial STT/TTS provider, with LangChain retained as the text agent.
- [x] Record the approved extension phases and shopping/booking design.

## M1: Foundation and text agent

Work in three reviewable batches:

- [ ] **M1-A — Working scaffold:** create Bun workspaces for `apps/web`, `apps/server`, and `packages/shared`; select the API framework and compatible dependency versions; configure React + TypeScript and the proposed Vite tooling; preserve strict typing; add runnable development, typecheck, build, and relevant test scripts. Establish atomic UI components and feature hooks as needed for a working conversation screen, plus thin API handlers and shared validated contracts.
- [ ] **M1-B — Text conversation:** choose the agent model provider; check the selected dependencies' official documentation; implement LangChain `createAgent`, server-owned credentials, authenticated HTTP/SSE/WebSocket session contracts, isolated bounded history, final-answer commit, and public progress events. Implement the baseline HTTP/SSE transport now and its streaming WebSocket extension in M3. Document actual setup commands, provider retention, and placeholder-only environment variables.
- [ ] **M1-C — Tools and turn reliability:** register validated calculator/current-time tools; implement one active turn per session, cancellation/reset, deadlines, overload limits, duplicate-submission handling, and read-only event reconnection. Keep history temporary and operational logs sanitized.

**Demo:** type a calculation, see the real tool activity and answer, ask a follow-up, then cancel an unfinished turn.

**Exit gate:** strict TypeScript and affected builds pass; focused checks cover arithmetic overflow/invalid inputs, session isolation, stale history/event rejection, duplicate admission and expired-retry rejection, delayed/duplicate cancellation against a newer turn, expired-replay status recovery without another agent run, and controlled model/tool failures. No provider secrets enter the browser bundle. Commands in the README run as documented.

## M2: Manual voice baseline

- [ ] Select ElevenLabs transcription/synthesis models, their supported API endpoints, and an initial voice; verify audio formats, Bun/SDK compatibility, account access, provider retention and Urdu recognition/output support separately. Keep STT and TTS behind backend adapters; temporary app history does not imply provider zero retention.
- [ ] Add native MediaRecorder capture, manual finish/cancel controls, transcription, final text publication before synthesis, and audio playback. Keep text input usable without microphone permission.
- [ ] Add visible recording/processing/playback states, immediate playback stop, provider cancellation where supported, late-result rejection, and cleanup of tracks/audio resources.
- [ ] Preserve finalized text on synthesis/playback failure; speech retry uses committed text without rerunning the agent or tools.

**Demo:** manually record a question, read its transcript/answer, hear the reply, and stop playback while preserving the answer.

**Exit gate:** the complete flow passes with mocked providers, including denied permission, empty recording, failed transcription/synthesis, and late cancelled work. With configured providers, verify real microphone/playback and representative Urdu recordings. A mock-only demonstration is not verified live voice support. This milestone remains the manual fallback, not the completed core MVP.

## M3: Hands-free voice MVP

- [ ] Keep listening during an enabled voice session. Finalize only nonempty speech after about 3 seconds of continuous detected silence; resumed speech resets the timer; await final STT and invoke the agent once.
- [ ] Confirmed new speech immediately silences playback and cancels older unfinished work. Atomically replace the backend turn, bound pending capture, preserve the start of the new utterance, and retain committed answers.
- [ ] Add the streaming target: STT during recording and TTS after final-answer commit, using supported endpoints. Follow the AudioWorklet/Dedicated Worker boundaries, pooled typed transfers, generation checks, bounded buffers, and cleanup in the browser audio design; retain the native fallback.
- [ ] Detect language per finalized utterance and reply in that language unless explicitly overridden. Verify Urdu, language changes, mixed-language input, and uncertain/unsupported-language recovery. Publish the verified model/language combinations rather than promising every world language.
- [ ] Add bounded contextual delivery presets and manual overrides; keep language, voice identity, and delivery metadata separate from answer text. Unsupported delivery falls back to neutral in the same language when supported.

**Demo:** speak a question, pause, hear a reply, then interrupt it with another question. Repeat in Urdu and with a language change; request a story and override its delivery style.

**Exit gate:** all [core MVP criteria](./architecture.md#mvp-completion-criteria) pass. Verify silence-boundary races, no empty/duplicate turns, background noise, speaker echo, interruption during processing/playback, stale audio rejection, and session disposal. Profile native versus streaming paths on representative devices for long tasks, frame delivery, underruns, bounded memory/buffers, stop responsiveness, and provider latency. Record measured stage latency before choosing targets.

## M4: Everyday controls and budgets — Phase 1

- [ ] Add short/detailed answers, supported pacing, replay, and transcript correction as a fresh linked turn.
- [ ] Add supported pronunciation preferences with preview/removal and usable fallback.
- [ ] Add estimated usage, configurable budgets, warnings, atomic server-side reservations, and text fallback within the remaining model allowance. Distinguish estimates from confirmed provider usage/charges. Before offering account-wide/persistent budgets, introduce stable ownership and the minimal durable PostgreSQL reservation/usage ledger; restart must not reset potentially billable spend. Label any earlier session-only meter accurately.

**Demo:** correct a transcript, replay a committed answer without another tool call, and reach a configured speech limit while text remains available within its model budget.

**Exit gate:** the [Phase 1 acceptance expectations](./feature-ideas.md#phase-1-core-usability) pass, including correction provenance, replay of an older retained answer under a fresh playback attempt without agent/tool reruns, model-capability checks, concurrent budget enforcement, cancellation costs and reservation recovery after restart. Persist preferences only when chosen. Basic request/concurrency limits already belong in M1; this phase adds the user-facing budget feature.

## M5: Familiar voices and selected memory — Phase 2

- [ ] Establish or reuse stable authenticated ownership and the proposed PostgreSQL feature store/private object storage, with explicit retention/removal policies. Extend the minimal M4 accounting store when present; keep all retained data separate from temporary conversation/event state.
- [ ] Add deliberate authorized voice enrollment/import, private preview/selection, separate persona settings, a clear AI label, and profile revocation/removal with accurate provider-cleanup status.
- [ ] Add the original-message library and explicit selected memories with provenance, edit/delete/expiry controls, and grounded retrieval. Uploading an original recording does not itself authorize cloning.

**Demo:** choose an authorized profile, explicitly save a sourced memory, ask about it, then remove it and revoke the profile.

**Exit gate:** the [Phase 2 acceptance expectations](./feature-ideas.md#phase-2-personal-companion-and-memory) pass: ownership isolation, current provider enrollment/sharing eligibility, grounded memories, revocation during synthesis, private access, and local/provider deletion failures. Do not promise enrollment paths that have not been verified.

## M6: Documents, practice, and routines — Phase 3

- [ ] Add private PDF/note upload, bounded parsing/indexing, owner-filtered retrieval, source citations, and deletion of originals plus derived data.
- [ ] Add visible tutor/language/interview practice modes with topic/difficulty controls and an ordinary-mode exit.
- [ ] Add create/preview/edit/run/delete controls for one-tap routines; each run uses the existing turn and permission rules.

**Demo:** ask a cited question about an uploaded document, run a practice session, and launch an edited saved routine.

**Exit gate:** the [Phase 3 acceptance expectations](./feature-ideas.md#phase-3-retrieval-practice-and-routines) pass, including unreadable/missing evidence, cross-owner isolation, source deletion during retrieval/indexing, malicious instructions in supplied content, mode exit, and one run per admitted routine turn. Saved prompts do not authorize future external actions.

## M7: Reminders and action approval — Phase 4 foundations

Deliver scheduling and action infrastructure as separate batches:

- [ ] **Scheduling:** add timers/reminders, agenda, review/snooze/cancel, IANA timezones, recurrence/expiry, and a verified notification channel. Durable reminders survive conversation reset and app closure; accurately label delivery that requires the app to remain open.
- [ ] **Actions:** build exact versioned review cards and server-enforced owner/payload/scope approval, atomic approval consumption, idempotency, and uncertain-outcome reconciliation. Validate with a controlled mock integration before enabling a concrete supported adapter.

Use durable schedule/occurrence and operation/job/outbox records, bounded claims, and independent notification/reconciliation entry points as defined in the runtime design. Duplicate delivery and expired worker leases must not duplicate a reminder or blindly repeat a submitted purchase.

**Demo:** schedule and snooze a reminder, verify its stated delivery mode, then edit and approve a mock action; the old approval must fail and repeated confirmation must not repeat execution.

**Exit gate:** the [Phase 4 expectations](./feature-ideas.md#phase-4-reminders-and-external-actions) pass for scheduling and approval: restart/overdue handling, timezone boundaries, revoked/denied notifications, deduplicated delivery, edited/expired approval, unknown results, and externally confirmed outcomes after voice interruption. Mock action validation is not a claim of a working live connector.

## M8: Voice shopping and tickets — Phase 4 commerce

Keep grocery/shopping and ticket booking as separately tracked capabilities, even when they share action infrastructure.

- [ ] **M8-A — Drafts and supported handoff:** resolve item/travel ambiguities and prepare reviewable carts/itineraries. Verify the intended provider's discovery/quote/handoff capabilities, show freshness/terms/known totals and unsupported steps, and provide a labelled supported checkout path. Before secure provider login/payment input, pause capture/transcription forwarding and playback, invalidate pending input/silence timing, and resume listening only through a valid user action. Flipkart, Blinkit, Zepto, and IRCTC/authorized partners remain targets, not verified integrations.
- [ ] **M8-B — First direct integration:** when authorized buyer/booking access is verified, implement one supported adapter end to end. Recheck quotes, enforce exact current approval, use secure provider account/payment input, pause capture/transcription/playback during secure input, and retain minimal owned operation/receipt status independently of voice generations.
- [ ] **M8-C — Additional capabilities/providers:** add shopping and ticketing adapters one at a time. Verify search, quote, cart/hold, checkout, tracking, cancellation, and refund separately; each external mutation has its own required approval. Keep unsupported direct operations visibly planned or offered through a supported handoff.

**Demo:** describe an order or trip, resolve missing details, review the proposal, and complete a supported handoff or verified integration. Show handoff status honestly; claim confirmed purchase/booking only from verified evidence, with seat/payment/refund states separate.

**Exit gate:** relevant [commerce acceptance checks](./voice-commerce.md) pass: changing price/stock/seats, substitutions, ambiguous dates, edited/expired approval, duplicate callbacks, revoked access, secure input, uncertain outcomes, and interruption. Validate with mocks before an explicitly authorized live transaction. A successful handoff completes M8-A only; direct checkout and other providers remain open until separately verified. An unavailable provider can be marked blocked with its required access, while the supported slice proceeds to M9.

## M9: Explanatory videos — Phase 5

- [ ] Add source-checked scripts and genuine sanitized tutorial assets based on a finalized answer.
- [ ] Add ElevenLabs narration, readable captions/highlights, and a renderer after verifying its runtime/deployment fit; Remotion remains the initial candidate.
- [ ] Add separate bounded background jobs with progress/cancellation, source-answer binding, private expiring input/output, and user-initiated playback. Isolate rendering/narration capacity from interactive voice; keep durable queue metadata free of full narration/transcripts. Durable saving remains an explicit choice.

**Demo:** ask for a how-to explanation, receive text/speech promptly, then open the completed narrated walkthrough.

**Exit gate:** the [Phase 5 expectations](./feature-ideas.md#phase-5-explanatory-videos) pass: current source accuracy, genuine visual steps, caption readability and synchronization, Urdu right-to-left rendering, provider retention, failed jobs, stale completion, cancellation, and private expiry. Video failures preserve the original answer and completion never autoplays or replaces a newer answer.

## How we deliver and track progress

1. Work on one active milestone and one reviewable batch at a time. For the first implementation session, start with M1-A; no voice cloning, checkout, or video infrastructure is needed for that batch.
2. Use **Not started → In progress → Verified** for each batch. Mark **Blocked** only with a concrete missing prerequisite and its resolution. Keep blocked provider capabilities separate from usable completed work.
3. For each batch, implement the working slice, run its relevant checks, document commands/behavior and remaining limits, then prepare it for review. A milestone becomes **Verified** only after its demo and exit gate pass; deployment/release status is recorded separately.
4. Update this roadmap's status and checkboxes, the README's project status, and affected design/scope documents together. Record verification evidence with the change. Do not tick features because interfaces, mocks, or diagrams exist.
5. Keep future features planned until their own gates pass. Revisit dates after measuring M1–M3 work; there are no delivery-date commitments yet.

This plan preserves every approved phase. The separately labelled [further suggestions](./feature-ideas.md#further-suggestions-not-yet-approved)—conversation search, extra accessibility settings, and a pre-session preview/microphone check—remain outside the approved milestones. Semantic controls and keyboard access remain core requirements. Scaling to multiple replicas is also a measured infrastructure decision, not a prerequisite for this first roadmap; follow the architecture's shared-state gates before adding replicas.
