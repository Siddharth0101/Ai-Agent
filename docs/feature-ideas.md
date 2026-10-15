# Approved feature roadmap and further ideas

Phases 1–5 and the bounded M10 extension below are approved **planned scope, not implemented**. Build the core voice assistant first (M1–M3); follow [delivery order and gates](./roadmap.md), [architecture](./architecture.md), and [storage boundaries](./scaling.md). Provider-dependent capabilities require verification before release.

## Familiar-voice companion requirements

Keep authorized voice identity, conversational style, delivery and saved memories separate; identify generated speech as AI. Ground personal answers in supplied sources, without invented memories or claims about a real person's beliefs. Enrollment needs rights/consent, current provider eligibility, private preview, an explained provider-storage choice and supported sharing. Professional clones must be created and verified by their owner. Check current rules for memorial use; cloning samples are not covered by zero-retention settings. Provide access revocation, profile removal and supported deletion of owned provider clones; report incomplete cleanup. See [companion boundaries](./architecture.md#familiar-voice-companion-planned-extension).

## Phase 1: core usability

Delivered in M4, after the core voice flow.

| Planned feature | Acceptance |
| --- | --- |
| Answer and playback controls | Concise/detailed answers, supported pacing and replay; canonical text stays unchanged. Replay or regenerate speech from committed text without rerunning the agent/tools; interruption stops audio immediately. |
| Transcript correction | Submit edited recognition as a fresh turn with visible provenance; cancel replaced unfinished work, preserve committed answers and prevent duplicates. |
| Pronunciation preferences | Preview/remove name and Urdu pronunciation preferences; apply only supported controls, exclude markup from history and explain unsupported settings without changing language. |
| Usage meter and budget | Explain estimated units/basis, limits and warnings; reserve/enforce paid work server-side across concurrent requests and restarts. Cancellation may cost money. Speech exhaustion permits text only within remaining model allowance; model exhaustion preserves readable answers. |

## Phase 2: personal companion and memory

Delivered in M5. Require authenticated ownership and private durable storage before enabling saved content.

| Planned feature | Acceptance |
| --- | --- |
| Authorized voice profiles and persona settings | Deliberate enrollment/import, private preview and independent style controls; recheck authorization/version before synthesis. Revocation stops active/pending use; distinguish app removal from provider deletion. |
| Original-message library | Explicitly upload authentic recordings, letters and stories with provenance, retention/removal controls and source citations. Distinguish original playback/quotation from generation; uploading does not authorize cloning. |
| Explicit saved memory | Save selected facts only by explicit action; inspect/edit/delete/expire/clear them with provenance. Removed/expired entries cannot reappear in retrieval; saving a fact does not retain the whole conversation or raw audio. |

## Phase 3: retrieval, practice, and routines

Delivered in M6. Retrieved content and saved prompts are data, never permission to change tools or server authorization.

| Planned feature | Acceptance |
| --- | --- |
| Personal document answers | Bounded supported PDF/note uploads with ownership and retention choices; cite passages/pages and explain missing evidence. Delete originals and derived retrieval data; reject cross-owner, deleted-source and injected instructions. |
| Tutor and practice mode | Language practice, quizzes and interview rehearsal with visible topic/difficulty, useful feedback and language changes including Urdu. Keep context temporary unless saved; exiting restores ordinary behavior. |
| One-tap routines | Create/preview/edit/run/delete prompts; each run admits one cancellable turn. Edits affect future runs; external actions still require separate approval. |

## Phase 4: reminders and external actions

Delivered in M7–M8. Scope approval, a voice session or a routine never authorizes an external action.

| Planned feature | Acceptance |
| --- | --- |
| Timers and reminders | Review/snooze/cancel and show an agenda with dates/IANA timezones, recurrence and expiry. Retain schedules deliberately; verify the notification channel before claiming closed-app delivery. Handle denial, restart, overdue work and duplicates. |
| Reviewable action cards | Show exact destination/payload/terms; edit/approve/reject. Server approval binds owner, version, payload, scope and expiry; edits invalidate it. Consume once, reconcile uncertain effects before retries and preserve confirmed outcomes after interruption. |
| Voice shopping and grocery ordering | Resolve product/variant/quantity/location/budget; review exact cart, charges and terms before approval. Target Flipkart/Blinkit/Zepto only through verified supported access or labelled handoff; handle substitutions, changed stock/prices, revoked access, duplicates and receipts. |
| Voice ticket booking | Resolve absolute travel dates/timezone, route, passengers, class/seats, budget and fare/refund preferences; approve exact itinerary/total/terms through supported access or handoff. Distinguish booking/seat/payment states; handle changed fares, expiry, secure verification and uncertain outcomes. |

Follow [voice commerce](./voice-commerce.md): buyer access remains unverified; begin with one verified integration or reviewed handoff. Keep purchases separate from AI budgets; credentials, OTPs and payment authorization stay outside model conversation. Cancellation/refunds need their own approval. Offer undo only where supported.

## Phase 5: explanatory videos

Delivered in M9. Reuse finalized, source-checked answers and resolved language/voice settings; use genuine sanitized tutorial assets, synchronized narration/captions/highlights and Urdu RTL support. Return answers promptly; render in a separate bounded cancellable job with progress and private expiry. Late completion cannot replace newer answers or autoplay; failures preserve text. Durable saving is explicit. Verify source freshness, rendering/cancellation, expiry and provider retention; follow [video boundaries](./architecture.md#explanatory-videos-planned-extension).

## Approved agent learning extension — M10

After M1–M9, add structured routing, bounded read-only supervisor/research/calculator-time/verifier collaboration, one app-owned final answer, baseline evaluations, private sanitized traces and existing exact human approval. Simple questions stay direct; no recursive autonomy, permission changes, private-conversation training or automatic retention. [M10 contracts and batches](./agent-learning.md) define finite limits, cancellation/source ownership and required gates. One verified read-only MCP adapter is optional; it grants no new authority.

## Storage and completion expectations

History stays temporary by default. Saved profiles, selected memories/recordings/documents, routines and reminder schedules require explicit retention/removal choices and authenticated ownership; app and provider retention are separate. Use durable accounting for persistent budgets, separate owned storage from expiring sessions, and remove derived retrieval data with sources. Complete each feature only after its roadmap acceptance and relevant failure, isolation, revocation and cancellation checks pass.

## Further suggestions: not yet approved

- **Searchable conversation history:** search current or explicitly saved conversations with ownership, expiry/delete/clear controls; no silent durable indexing.
- **Extra accessibility settings:** selectable high contrast, larger text/captions and text-only presentation. Semantic controls, keyboard access and visible focus are already core requirements.
- **Pre-session language/voice preview and microphone check:** explicit bounded preview/check with device and compatibility feedback; discard recordings by default and retain manual/text fallback.
