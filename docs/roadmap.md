# Delivery roadmap

**Status:** Bun + TypeScript starter only; every milestone is **Not started**. **Next: M1-A, scaffold the working apps.** M1 and M2 are integration milestones; **M3 completes the core voice MVP**.

This document owns delivery order and completion gates. Technical details live in the [architecture](./architecture.md), [browser audio](./browser-audio.md), [runtime/recovery](./scaling.md), [commerce](./voice-commerce.md), and [agent learning](./agent-learning.md) documents. [Approved feature scope](./feature-ideas.md) owns Phase 1–5 acceptance expectations.

## Release sequence

| Order | Scope | Usable result | Status |
| --- | --- | --- | --- |
| M1 | Foundation and text agent | Typed questions, session context, calculator/time tools | Not started; M1-A next |
| M2 | Manual voice baseline | Record, transcribe, read/hear an answer | Not started |
| M3 | Complete core voice MVP | Hands-free listening, interruption, Urdu, contextual delivery | Not started |
| M4 | Phase 1: everyday controls and budgets | Correction, answer/playback preferences, usage limits | Not started |
| M5 | Phase 2: familiar voices and selected memory | Authorized profiles, original messages, explicitly saved memories | Not started |
| M6 | Phase 3: documents, practice, routines | Cited document answers, practice modes, editable saved prompts | Not started |
| M7 | Phase 4 foundations: reminders and approval | Durable reminders, exact external-action review | Not started |
| M8 | Phase 4 commerce: shopping and ticket booking | Supported handoff, then verified provider execution | Not started |
| M9 | Phase 5: explanatory videos | Source-checked narrated walkthroughs | Not started |
| M10 | Bounded multi-agent learning | Evaluated read-only collaboration; optional verified MCP | Not started |

## M1: Foundation and text agent

- [ ] **M1-A — Working scaffold:** Bun workspaces for `apps/web`, `apps/server`, and `packages/shared`; compatible React/TypeScript/Vite/API dependencies; strict typing; atomic UI, thin handlers, shared validated contracts; runnable development, typecheck, build, and relevant test scripts.
- [ ] **M1-B — Text conversation:** select model/provider and verify current official integration docs; add LangChain `createAgent`, server-only credentials, authenticated HTTP/SSE sessions, isolated bounded history, final-answer commit, and allowlisted public progress. Document setup, placeholder environment variables, and provider retention; WebSocket arrives in M3.
- [ ] **M1-C — Tools and turn reliability:** validated calculator/current-time tools, one active turn, cancellation/reset, deadlines/admission limits, deduplicated submission/control commands, bounded retry records, read-only reconnection, temporary history, and sanitized logs.

**Demo/gate:** calculation, contextual follow-up, and cancellation work. Strict TypeScript/builds and documented commands pass. Check invalid/overflow arithmetic, session isolation, stale history/events, duplicate and expired retries, delayed cancellation of an older turn, expired-replay status recovery without another run, controlled provider/tool failures, and absence of browser secrets.

## M2: Manual voice baseline

- [ ] Select ElevenLabs STT/TTS models, supported endpoints/formats, and voice; verify Bun/SDK/account compatibility, provider retention, and Urdu recognition/synthesis separately. Keep replaceable backend adapters.
- [ ] Add native MediaRecorder, manual finish/cancel, transcription, final text before synthesis, playback/stop, visible states, late-result rejection, and resource cleanup. Preserve typed input and committed text; speech retry never reruns the agent/tools.

**Demo/gate:** record, read/hear the answer, and stop playback without losing text. Mock permission denial, empty audio, STT/TTS failure, and late cancellation; then verify real microphone/playback and representative Urdu with configured providers. Mock success alone is not live voice support; this remains the manual fallback.

## M3: Hands-free voice MVP

- [ ] Finalize nonempty speech after about 3 seconds of continuous silence; resumed speech resets timing. Await final STT and invoke the agent once while listening remains enabled.
- [ ] New confirmed speech immediately silences playback and atomically replaces unfinished server work; bound pending capture, preserve the utterance beginning and committed answers.
- [ ] Stream STT during recording and TTS after final commit using verified endpoints and the [audio boundaries](./browser-audio.md); retain native fallback, pooled buffers, generation checks, and bounded queues.
- [ ] Detect language per utterance, honor overrides, verify Urdu/mixed-language/recovery, and publish tested coverage. Add bounded delivery presets/manual overrides with separate language/voice metadata and neutral fallback.

**Demo/gate:** speak, pause, hear, interrupt, change language, and override delivery. Pass [MVP criteria](./architecture.md#mvp-completion-criteria), silence races, no empty/duplicate turns, echo/noise rejection, interruption, stale audio, and disposal. Profile native/streaming paths on representative devices for frame delivery, underruns, memory, stop response, and stage/provider latency before setting targets.

## M4: Everyday controls and budgets — Phase 1

- [ ] Add answer length, supported pacing/replay, fresh linked transcript corrections, and pronunciation preferences with preview/removal/fallback.
- [ ] Add estimated usage, warnings, atomic paid-stage reservations, budgets, and text fallback within remaining model allowance. Stable ownership and a durable PostgreSQL usage/reservation ledger precede persistent/account-wide budgets; cancellation/restart cannot erase potential charges. Save preferences only by choice.

**Demo/gate:** [Phase 1 checks](./feature-ideas.md#phase-1-core-usability) pass: correct input, replay an older committed answer under a fresh playback attempt without agent/tool reruns, validate model controls, enforce concurrent limits, distinguish estimates from charges, and recover reservations after restart. M1 admission limits remain separate from this feature.

## M5: Familiar voices and selected memory — Phase 2

- [ ] Reuse/establish stable authenticated ownership, durable PostgreSQL feature records, private object storage, and explicit retention/removal; keep retained items separate from temporary sessions.
- [ ] Add authorized enrollment/import, private voice preview/selection, separate persona settings, an AI label, revocation/removal, original-message library, and explicitly saved sourced memories with edit/delete/expiry. Uploading audio does not authorize cloning.

**Demo/gate:** save/retrieve/remove a sourced memory and revoke a profile. Pass [Phase 2 checks](./feature-ideas.md#phase-2-personal-companion-and-memory): verified provider eligibility/sharing, owner isolation, grounded recall, revocation during synthesis, private access, and accurate local/provider cleanup failures.

## M6: Documents, practice, and routines — Phase 3

- [ ] Add private PDF/note upload, bounded parsing/indexing, owner-filtered retrieval/citations, and original/derived deletion; visible tutor/language/interview modes with topic/difficulty and exit; editable previewable one-tap routines using existing turn/permission rules.

**Demo/gate:** cited document answer, practice session, and edited routine pass [Phase 3 checks](./feature-ideas.md#phase-3-retrieval-practice-and-routines), including missing evidence, cross-owner isolation, deletion during work, hostile source instructions, mode exit, and one admitted run. Saved prompts cannot authorize external actions.

## M7: Reminders and action approval — Phase 4 foundations

- [ ] **Scheduling:** timers/reminders, agenda, review/snooze/cancel, IANA timezones, recurrence/expiry, and verified notification delivery. Durable reminders survive reset/app closure; label open-app-only delivery accurately.
- [ ] **Actions:** exact versioned owner/payload/scope review cards, server approval, atomic consumption, idempotency, and uncertain-outcome reconciliation; validate a controlled mock before a supported live adapter.

**Demo/gate:** schedule/snooze a reminder, edit/approve a mock action, reject old approval and duplicates. Pass [Phase 4 checks](./feature-ideas.md#phase-4-reminders-and-external-actions): restart/timezone/notification failures, duplicate occurrences, expired claims/approvals, unknown outcomes, and preserved effects after interruption. Use [durable schedule/operation/outbox records and bounded workers](./scaling.md); mock validation is not a live connector.

## M8: Voice shopping and tickets — Phase 4 commerce

- [ ] **M8-A — Drafts and supported handoff:** resolve item/travel details; verify discovery/quote/handoff support; show reviewable cart/itinerary, freshness, totals/terms, and unsupported steps. Flipkart, Blinkit, Zepto, and IRCTC/authorized partners remain targets until verified.
- [ ] **M8-B — First direct integration:** implement one verified authorized buyer/booking adapter; recheck quotes and exact approval, use secure provider checkout, retain minimal owned operation/receipt status independent of voice generations.
- [ ] **M8-C — Additional capabilities/providers:** add shopping and ticketing separately, one adapter at a time; verify search/quote/cart/hold/checkout/tracking/cancel/refund individually, with fresh authorization for each mutation.

**Demo/gate:** review and complete a supported handoff or verified integration. Follow [commerce checks](./voice-commerce.md): pause capture/transcription/playback during secure input, invalidate pending input/timing, resume only by valid user action; test ambiguity, stock/price/seat changes, substitutions, expired/edited approvals, callbacks, revoked access, uncertain results, and interruption. Mock before authorized live use. Handoff verifies M8-A only; claim purchase/booking from receipts and keep seat/payment/refund states separate. Record unavailable access without blocking the supported slice from M9.

## M9: Explanatory videos — Phase 5

- [ ] Add source-checked scripts and genuine sanitized tutorial assets, ElevenLabs narration, captions/highlights, and a verified renderer/runtime (Remotion is a candidate).
- [ ] Use separately bounded background jobs, source-answer binding, progress/cancel, private expiring assets, and user-started playback; isolate interactive capacity and exclude full narration/transcripts from durable queue metadata. Durable saving is opt-in.

**Demo/gate:** prompt text/speech followed by an opened walkthrough passes [Phase 5 checks](./feature-ideas.md#phase-5-explanatory-videos): source accuracy, readable synchronized captions, Urdu RTL, provider retention, failure/cancellation/stale completion, and expiry. Failure preserves the answer; completion never autoplays or replaces newer work.

## M10: Bounded multi-agent learning

**Not started; after M1–M9.** Follow [learning layers](./agent-learning.md#learning-layers) and [detailed batches](./agent-learning.md#m10-delivery-batches). Retain LangChain TypeScript and the direct path; add orchestration dependencies only for demonstrated need after checking official docs.

### M10-A — Routing, contracts, and baseline

- [ ] Allowlisted routes keep simple questions direct. Runtime-validated specialist results carry summary, evidence, uncertainty/errors and current owner/session/turn/source versions; results remain data. Establish a versioned synthetic or authorized/sanitized direct baseline.

**Batch gate:** reproducible baseline; fixtures reject unknown roles, malformed results, and permission escalation.

### M10-B — Bounded read-only collaboration

- [ ] Add supervisor, read-only research/calculator/time specialists, and verifier using existing authorized capabilities. Bound delegation/fan-out/depth, iterations, context, time, concurrency, and aggregate reservations; propagate cancellation, reject stale work, account for cancelled calls, and provide direct fallback/controlled failure.

**Batch gate:** one coordinator commits the answer/history and dispatches audio once; interruption, stale sources, specialist failures, and limits cannot cause unauthorized effects or late results. A verifier flags concerns; it cannot establish truth or authority.

### M10-C — Evaluation, private observability, and human approval

- [ ] Compare direct/specialist quality, grounding, language, latency/usage, hostile input, cancellation, timeouts, and quota isolation. Keep bounded sanitized private traces; retained real content needs explicit ownership/retention/removal. Reuse M7 exact approval/reconciliation with the controlled integration.

**Batch gate:** measured regressions justify routing limits; no raw transcript/tool-payload retention or exposed reasoning by default. Specialists cannot self-approve, edit approved payloads, replay approval, or repeat uncertain effects.

### M10-D — Optional verified MCP adapter

- [ ] Only for a concrete existing capability: verify one allowlisted server/transport, authentication, capabilities, schemas/size, secrets, origin/egress, cancellation/timeouts, and private data handling. Start read-only; writes retain exact server approval/reconciliation.

**Optional gate:** valid read-only use succeeds; revoked access, unexpected tools/targets, hostile results, and failures stay bounded. M10-D is optional; no arbitrary marketplace, extra infrastructure, or content store is implied.

**Demo/exit gate:** compare an eligible direct/supervised answer, interrupt a specialist run, inspect private metrics, and show exact approval still required. M10-A–C checks and existing voice/budget/retrieval/action recovery regressions pass; document measured trade-offs and controlled fallback. Verify MCP separately only if enabled.

## How we deliver and track progress

Work on one milestone/batch at a time: **Not started → In progress → Verified** only after its demonstration and gates pass. Record a concrete prerequisite for **Blocked**, distinguish provider-specific gaps and deployment status, and align README/affected docs with verified behavior. No delivery dates are committed.

Add stores/processes only when required: proposed PostgreSQL for durable owned features, private object storage for assets, and separate expiring Redis with no persistence/backups before replicas. Pass shared ownership/fencing, replay/status recovery without reruns, quota, bounded-resource, and authority-loss tests before scaling; see [runtime gates](./scaling.md). [Further suggestions](./feature-ideas.md#further-suggestions-not-yet-approved) remain unapproved; semantic controls and keyboard access are core.
