# Approved feature roadmap and further ideas

Status: approved planned scope. The user has approved all previously suggested features below, including reviewable external actions. None is implemented; the repository still contains only the Bun + TypeScript starter. [The delivery roadmap](./roadmap.md) tracks implementation milestones and completion gates; [README.md](../README.md) summarizes them, and [the architecture](./architecture.md) defines the required boundaries. The final section contains new suggestions that have not been approved.

Complete the core React/TypeScript, Bun, LangChain, and ElevenLabs voice flow first: spoken and typed turns, calculator/time tools, hands-free submission after about 3 seconds of silence, interruption, same-language replies including Urdu, and contextual delivery. The phases below organize the approved additions; they are implementation order, not delivery dates or claims that every extension belongs in the first milestone.

Delivery mapping: M1–M3 build the core; M4 implements Phase 1; M5 implements Phase 2; M6 implements Phase 3; M7–M8 split Phase 4 into scheduling/action foundations and commerce; M9 implements Phase 5. Provider-dependent capabilities remain separately planned until verified, even when an initial supported handoff is delivered. The feature scope below remains approved in full.

## Familiar-voice companion requirements

Let a user talk with an AI assistant using an authorized loved one's voice. Keep the selected voice, conversational style, and optional saved memories independently configurable. Clearly label the interaction as AI-generated. Use user-supplied stories, letters, and recordings as cited context; do not invent personal memories or claim to speak the real person's beliefs.

Enrollment is deliberate: review voice-use rights and provider eligibility, explain provider storage, upload approved samples or select an available shared voice, preview the result, and choose who can use it. Provide app-access revocation and profile removal, plus provider deletion for clones the account owns where supported. See [the companion architecture](./architecture.md#familiar-voice-companion-planned-extension).

Current ElevenLabs rules distinguish cloning methods. Instant cloning requires rights/consent confirmation; Professional cloning must be created and verified by the voice owner, who can then share it. For memorial use, review the current provider policy and applicable rights before enrollment rather than assuming archived recordings qualify for verification. Voice cloning samples require provider storage and are excluded from its Zero Retention Mode.

- [Instant Voice Cloning](https://elevenlabs.io/docs/eleven-creative/voices/voice-cloning/instant-voice-cloning)
- [Professional cloning ownership rules](https://elevenlabs.io/docs/help-center/product/voices/voice-cloning/can-i-create-a-professional-voice-clone-of-someone-elses-voice)
- [Sharing supported voices](https://elevenlabs.io/docs/help-center/product/voices/my-voices/how-do-i-share-a-voice)
- [Provider use policy](https://elevenlabs.io/use-policy)
- [Provider retention](https://elevenlabs.io/docs/eleven-api/resources/zero-retention-mode)

## Phase 1: core usability

Add these controls while stabilizing the core conversation. Effort estimates are design judgments; validate model/SDK support and measured usage during implementation.

| Planned feature | Acceptance expectations | Effort |
| --- | --- | --- |
| Answer and playback controls | Offer concise answers or more detail, supported speech pacing, and replay. Keep canonical answer text unchanged by playback settings. Replay retained audio when available; speech regeneration uses committed text without another agent run. Interruption still stops audio immediately. | Small |
| Transcript correction | Show recognized text for editing and allow submission of the corrected question as a new turn. Cancel unfinished work from the replaced turn, preserve already committed answers, and show the correction relationship without silently rewriting history or generating duplicate replies. | Small to medium |
| Pronunciation preferences | Let users specify pronunciations for names and Urdu terms, preview them, and remove preferences. Map them only to supported model controls; keep pronunciation markup out of visible text/history. Report unsupported preferences and retain usable text rather than changing the requested language. | Small to medium |
| Usage meter and budget | Show estimated speech/model usage with its units and estimate basis, plus configurable limits and warnings. Enforce admission limits on the server before starting further paid work. When only the speech allowance is exhausted, allow text conversation within the remaining model allowance; when the model allowance is exhausted, keep existing answers readable and explain how to resume. | Medium |

## Phase 2: personal companion and memory

Implement the familiar-voice enrollment requirements above before enabling a cloned profile. Voice selection, persona, delivery, and saved knowledge remain separate controls.

| Planned feature | Acceptance expectations | Effort |
| --- | --- | --- |
| Authorized voice profiles and persona settings | Provide deliberate enrollment/import, a private preview, voice selection, and separate conversational-style presets. Keep the selected authorized voice stable across delivery styles. Clearly label generated conversation as AI. Verify ownership/version before synthesis; revocation stops active and pending use. Distinguish app-profile removal from provider deletion and show unresolved cleanup. | Medium to large |
| Original-message library | Allow explicit uploads of authentic recordings, letters, and family stories with provenance and private ownership. Clearly distinguish original playback/quotations from generated narration. Cite supplied sources when answering about personal memories; say when the library lacks support. Give each asset retention and removal controls, and do not treat an upload as permission to clone its speaker. | Medium |
| Explicit saved memory | Save only user-selected facts through an explicit “remember this” or save action. Let users inspect, edit, delete, expire, and clear saved entries, with provenance and an explained retention choice. Later answers use current authorized entries; removed/expired memories no longer enter future retrieval. Saving a fact does not save the full conversation or raw audio. | Medium |

Persistent profiles, memories, and library assets require a stable authenticated owner. Test separate-user access, unsupported enrollment, revoked profiles during synthesis, grounded memory answers, and app/provider deletion failures before enabling this phase.

## Phase 3: retrieval, practice, and routines

These features reuse the text agent and speech adapters. Retrieved files, library content, and saved prompts are data; they cannot grant tool permissions or override server authorization.

| Planned feature | Acceptance expectations | Effort |
| --- | --- | --- |
| Personal document answers | Accept supported PDFs/notes within stated size limits and ownership/retention choices. Return source passages and page/section references when available; explain missing or unreadable evidence instead of inventing an answer. Removed documents and their derived retrieval data stop appearing in future answers. Test access isolation and instructions embedded in uploaded content. | Medium to large |
| Tutor and practice mode | Offer language practice, quizzes, and interview rehearsal with a chosen topic and difficulty. Give understandable corrections/feedback and allow difficulty or language changes, including Urdu, without forcing English. Keep practice context session-scoped unless the user explicitly saves it; exit the mode without leaving hidden instructions in later ordinary turns. | Small to medium |
| One-tap routines | Let users create, preview, edit, run, and delete prompts for planning, stories, or study. Starting a routine produces one admitted turn with visible context and the existing cancellation rules. Editing a routine affects future runs. A routine that proposes an external action must still use the action approval flow below. | Small |

## Phase 4: reminders and external actions

Introduce scheduling and external integrations after the core cancellation and authorization boundaries are verified. Implement concrete integrations as working features rather than empty connectors.

| Planned feature | Acceptance expectations | Effort |
| --- | --- | --- |
| Timers and reminders | Create, review, snooze, and cancel timers/reminders and display an agenda with clear dates/timezones. Explain whether delivery requires an open app or uses a configured notification channel, and verify that mode before claiming background delivery. Handle notification denial, restart, overdue reminders, and cancellation without duplicate alerts. Creating a scheduled reminder is an explicit choice to retain that reminder until its stated expiry/removal. | Medium to large |
| Reviewable action cards | Show the exact proposed message or account change, destination, and relevant parameters before execution. Allow edit, approve, or reject; editing invalidates the previous approval. The server verifies owner, action version, and explicit authorization, then reports the confirmed result or uncertain failure. Duplicate approvals/retries must not repeat an action that may already have completed. | Large |
| Voice shopping and grocery ordering | Accept products, variants, quantities, location, and a total budget; compare supported options and show a draft cart with known charges/terms. Initial targets are Flipkart, Blinkit, and Zepto, subject to verified supported access. Obtain explicit exact-cart approval, re-quote changes, use secure checkout/handoff, and verify the receipt. Cover substitutions, out-of-stock items, revoked accounts, duplicate requests, and unknown outcomes. | Large |
| Voice ticket booking | Resolve absolute dates/timezone, route, passengers, class/seats, budget, and fare/refund preferences. Show and confirm the exact itinerary, total, and terms; use an authorized ticketing adapter or labelled handoff. Distinguish issued/waitlisted/confirmed seat status and payment status. Cover changed fares/seats, ambiguous dates, expired approval, duplicate callbacks, secure verification, and uncertain results. | Large |

Starting a voice session, choosing a routine, or approving this roadmap does not authorize a later external action. Interruption cancels unfinished generation but cannot undo a completed change; the UI preserves its actual outcome and offers an undo only where the integration supports one.

Follow [the voice-commerce design](./voice-commerce.md) for shopping and booking. These are approved planned features, not permission to perform a real purchase now. Search, cart changes, booking, payment, cancellation, and refund are separate capabilities. Public consumer checkout access for the named stores remains unverified; begin with one verified integration or a reviewed provider handoff. Keep monetary purchase budgets separate from AI usage budgets, and keep credentials/payment verification outside model conversation.

## Phase 5: explanatory videos

On-demand explanatory videos remain a separate approved phase after the core assistant. Follow [the video architecture](./architecture.md#explanatory-videos-planned-extension).

Acceptance requires a finalized, source-checked explanation; genuine sanitized interface assets for tutorials; ElevenLabs narration in the resolved reply language/voice; readable captions and highlights; and synchronized playback, including Urdu right-to-left text. Return the text/spoken answer promptly, then render under a separate bounded job with progress, cancellation, and a private expiring result. Bind the video to its source answer; late completion cannot overwrite a newer answer or autoplay. Preserve the text answer on rendering failure, and make durable video saving a separate retention choice. Validate procedure freshness, failed jobs, cancellation, expired results, and provider retention before private uploads.

## Storage and completion expectations

All features in Phases 1–5 are approved planned scope. Select concrete models, limits, storage adapters, notification channels, and action integrations as each phase is implemented; document working commands and verified behavior then. Follow the [proposed storage/runtime defaults](./scaling.md): durable accounting before account-wide budgets, owned saved storage before profiles/documents, and separate bounded processes for indexing, notifications/reconciliation and rendering. A feature is complete only when its acceptance expectations and relevant failure/cancellation paths have been checked. No app build or feature tests exist yet.

Conversation history stays temporary by default. Persistent voice profiles, selected memories, original recordings, saved documents/routines, and scheduled reminders are deliberate storage exceptions with their own ownership, retention, and removal decisions. State app and provider retention separately; expiry does not by itself define backup deletion. Uploading one asset does not opt the user into saving all conversations or raw microphone audio. Keep retrieval content as data and maintain the existing server-enforced tool permissions.

## Further suggestions: not yet approved

These are new ideas for consideration, separate from the approved roadmap above.

| Suggested feature | Expected benefit and boundaries | Effort |
| --- | --- | --- |
| Searchable conversation history | Search the current session and any conversations the user explicitly saves. Saving is opt-in with expiry, delete, and clear-all controls; temporary turns are not silently indexed into durable history. Results link to the original turn and enforce ownership. | Medium |
| Accessibility controls | Add user-selectable high contrast, larger captions/text, and text-only presentation. Keep all controls keyboard-accessible with visible focus and clear labels; text-only use must work without microphone permission or automatic audio. Semantic controls and basic keyboard access remain required for the core UI. | Small to medium |
| Pre-session language/voice preview and microphone check | Before enabling listening, let users choose or preview language/voice settings and explicitly run a short microphone check. Explain detected devices/capture failures and unsupported combinations. Discard check recordings by default, keep preview use within usage limits, and leave manual/text fallback available. | Small to medium |
