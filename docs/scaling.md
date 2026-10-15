# Runtime, recovery, and scaling design

Proposed [full-product design](./architecture.md#architecture-for-the-full-product), not installed infrastructure. No server, store, broker, worker pool or deployment exists. Add components only for the active [milestone](./roadmap.md).

## Deployment stages

| Stage | Add when | Proposed shape / gate |
| --- | --- | --- |
| M1–M3 core | First assistant | Static React, one Bun/TypeScript backend, bounded in-memory sessions; verify text/manual/hands-free behavior and isolation |
| M4 accounting | Persistent budgets/preferences | Stable authenticated owner and PostgreSQL reservation/usage ledger; verify concurrency/restart and opt-in retention |
| M5–M6 saved features | Profiles, selected memories, documents/routines | PostgreSQL, private object storage and bounded indexing; verify ownership/revocation/removal |
| M7–M8 schedules/actions | Reminders and approved commerce | Durable schedules/operations/outbox, notification/reconciliation processes; verify exact approval, deduplication and unknown outcomes |
| M9 video | Requested rendering | Separate bounded render pool, expiring private inputs/results; protect interactive capacity |
| M10 learning | After M1–M9 | Reuse coordinator, stores/accounting and provider gates; bounded branches, one final commit, private evals |
| Replicas | Measured capacity/availability need | HTTPS edge, shared temporary Redis, drained live streams; verify shared fencing/replay and provider-wide limits |

Temporary history/event/ownership stores have no persistence or backups unless retention is explicitly chosen. Durable owned records, accounting and broker metadata stay separate; private objects have retention/expiry/removal policies. Hosting remains open: measure region/provider routes and require long-lived WebSocket/SSE support. Isolate CPU-heavy parsing/rendering in separate processes; Kubernetes, per-feature services, extra brokers and multi-region active voice state are not initial requirements.

## Interactive turn path

1. Authenticate/validate and atomically admit against session generation/owner fence. Typed input goes directly to text; M2/M3 adds audio. Bound audio duration/bytes and reserve extensions before more paid capture.
2. Forward admitted audio with credits/bounded queues. The [audio contract](./browser-audio.md#audio-and-transport-contract) makes `end_input` an epoch/last-sequence barrier: validate and forward all required packets in order before provider commit. Fail gaps/stalls; partial transcripts stay draft.
3. Resolve final language and snapshot authorized context/preferences. Acquire stage capacity; retrieval filters owner/source and bounds results, while initial indexing is a job.
4. Run LangChain on an isolated snapshot with allowlisted public progress. Proposals do not execute external actions; approval/execution has its own owned workflow.
5. Conditionally commit final answer/replay metadata under current ownership; publish text before synthesis. Apply validated speech settings and current profile authorization; revocation stops pending/active use.
6. Release after actual completion. Preserve text on audio failure; new speech mutes locally and replaces unfinished work. Secure input uses the capture-pause barrier; interruption preserves submitted external outcomes.

M1 needs deadlines/concurrency limits, not speech or an account ledger. M4 adds conservative atomic reservations **before every paid stage**, including STT. Reuse existing turn reservations without double accounting. Refusal produces a usable fallback only within remaining allowance. Interactive waits are bounded; ordinary turns never enter a durable background queue.

## Ownership and recovery

| Identity/version | Invalidated by | Preserves |
| --- | --- | --- |
| Session generation/store incarnation | Reset, replacement as required, expiry/authority loss | Saved records, operations, budgets |
| Turn ID/renewable owner fence | Cancel, replacement, lease loss | Committed answers/confirmed effects |
| Capture epoch/sequence | New capture, pause, discontinuity/disposal | Independent jobs/reminders |
| Playback attempt | Stop, speech, retry, profile revocation/disposal | Canonical answer/receipt |
| Source/profile version | Edit, removal, revocation | Other authorized sources |
| Job ID/generation/lease fence | Cancel, source change, applicable reset/disposal, lease loss | Unrelated turns; results attach to original source |
| Action ID/version/attempt | Proposal edit/expiry/rejection; submission-state transitions | Verified outcome; cancellation/refund is a new authorized action |
| Schedule version/occurrence ID | Edit/snooze/cancel/delivery transitions | Reset, microphone stop, app closure |

Every write/publication checks the relevant owner/fence/version; lease renewal cannot restore revoked access. Controls atomically compare target identity and expected version and deduplicate commands, preventing old cancellation from affecting a successor. Automatic speech requires current-turn authority. Historical replay creates a new authorized attempt against retained committed text/current profile access; the original turn need not remain active.

Before replicas, put history, active ownership, cancellation, deduplication and replay in shared expiring state. Retain minimal deduplication independently of history/event eviction for a server-verifiable retry horizon; reuse the same turn ID for uncertain submissions and reject expired IDs. Explicit retries after terminal failure/cancellation use new IDs; speech retries never rerun the agent/tools.

Redis scripts/fences close local races, but [failover may lose recent ownership](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/). Fail closed on authority uncertainty: invalidate old store incarnations/sessions, reject stale work and never rebuild authority from client data. A lease does not guarantee exactly-once external effects.

Recover crash-after-commit delivery from surviving authoritative status/replay without another model run. Replay expiry alone requires status lookup: recover committed text or observe the same running turn. Initial in-memory restart loses status/history/replay and ends that session. Broken live audio is not transparently migrated; recover visibly. Durable saved features/operations follow their separate retention policy.

## M10 bounded multi-agent execution

[M10](./agent-learning.md#m10-bounded-multi-agent-learning) adds bounded asynchronous branches on the direct interactive path, not a new conversation authority or early worker pool. Bind child ID/attempt to parent session/turn/generation/fence and source versions; recheck at admission, join, final commit, events and speech. Cancellation rejects late branches while preserving committed text and independent external effects. Only the coordinator commits/publishes/speaks; supervisor/verifier cannot grant permission or expand limits.

Apply [finite team ceilings](./agent-learning.md#limits-cancellation-and-injection-defenses) across fan-out/depth, context/results, calls, duration and concurrency. Charge all paid supervisor/specialist/verifier/evaluation stages through existing accounting without double reservation; cancelled upstream work remains counted. Reconnect observes the same run; replicas require shared parent/result checks rather than local task maps.

Use minimal child snapshots, existing owner/source deletion rules and no global specialist history. Private expiring traces contain route/version, counts/timings and sanitized outcomes, excluding prompts, raw findings/tool payloads and reasoning. Evals have separate bounded capacity. Optional MCP has explicit capabilities, credential isolation, limits and cleanup.

## Durable work and background pools

PostgreSQL is the proposed durable authority. Transactionally record an owned change/approval consumption, operation state and outbox intent before dispatch. Claim bounded batches with lease/fence in short transactions; do network/CPU work outside locks. [`FOR UPDATE SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html) is a queue-claim candidate, not general consistency.

| Pool | Required boundaries |
| --- | --- |
| Indexing | File/page/CPU/memory bounds; owner/source checks before publication; revoke/delete purges derived data |
| Rendering | Asset/duration/resolution/concurrency bounds, separate narration/model allowance, source binding/private expiry |
| Notifications | Unique schedule/version/occurrence, current cancellation/expiry checks, verified channel and truthful delivery |
| Reconciliation | Authenticated callbacks/status, durable deduplicated events/provider references, bounded backoff |
| Dispatch/cleanup | Fenced bounded polling, expired-claim recovery and tracked incomplete provider cleanup |

Use durable unique keys/conditional updates for at-least-once dispatch. Callbacks wake reconciliation, never a new conversation. A sweeper handles missed dispatch and cleanup. Retry reversible jobs only after source/access/version/attempt checks. An expired action lease after possible submission becomes uncertain: retain the original idempotency reference and reconcile before retry; unsupported status/idempotency may require user resolution.

Reminders use durable schedules/occurrences, not browser timers. Store local schedule, IANA timezone, recurrence/expiry, resolved instant and version; handle daylight-saving ambiguity/overdue policy. Recheck cancel before send. Accepted notifications cannot be recalled by later snooze/cancel; ambiguous sends remain uncertain.

Queue only owner/feature/job/operation identifiers, version, deadline, attempt and private expiring input reference. Narration/transcripts stay out of durable payloads; expired inputs fail, never refetch deleted sources. A later broker uses transactional outbox dispatch, acknowledges after durable transition and tolerates duplicates; keep its persistence/backups separate from temporary Redis.

## Capacity, latency, and overload

One admission policy covers runtimes/pools, provider-wide rates/connections/tokens/audio and per-owner budgets. Reserve interactive capacity and bound background queues; more replicas cannot create provider quota. Bound browser pools/socket bytes/playback, sessions/turns/context/deadlines, database connections/query time/upload bytes, and job age/depth/attempts/CPU/memory/output. Refuse predictably; approval/accounting fails closed. Reuse bounded clients/pools.

Cancellation removes publication rights immediately, not upstream cost/concurrency. Hold slots/reservations until completion or conservative reconciliation, including after crashes. M4's durable PostgreSQL ledger reserves before paid stages and reconciles reported usage; local estimates/limits do not guarantee provider invoices. Redis can accelerate limiting but is not financial authority. Lost limiter state stops new paid admission until current usage/windows are conservatively reconstructed.

Measure speech-end/seal/STT/commit/first-audio stages, interrupt-to-mute, admission wait, p50/p95/error rate, event-loop lag, buffer peaks/underruns, provider saturation, queue age, uncertain operations, reminder lag and database wait. Use local monotonic durations and correlated IDs, not unsynchronized clock subtraction. Set targets from M1–M3/concurrency measurements. Scale within provider/database ceilings using connection/latency headroom or runnable-job age, drain streams on scale-down and avoid outage-amplifying retries.

## Private access, caching, and deletion

Derive owners server-side; filter every saved query. Source versions/tombstones prevent late or restored publication. Validate/bound uploads/parsers; authorize private objects. Expiring signed URLs may remain usable until expiry: immediate revocation needs a current-access proxy/check. Distinguish expiry/access removal from physical, backup and provider deletion.

Public caches contain static UI/immutable public assets only. Any approved private cache keys include owner/scope, source/profile version, language/model/settings and expiry, with delivery reauthorization/removal eviction. Replay reuses retained committed text/audio, not the agent. Quotes require provider freshness/expiry and revalidation.

Telemetry contains identifiers/counts/timings/sanitized errors, not raw audio, transcripts, tool payloads or private dead-letter content. Avoid unbounded session/turn metric labels. Document durable backup retention/deletion reconciliation; incomplete provider cleanup remains visible.

## Rollout and verification gates

- **Core:** typing/builds, authenticated routes, temporary isolation, final commit, targeted cancellation/reset, bounded deduplication/status/replay, controlled failures and no browser secrets. M2 adds mocked then real configured microphone/STT/TTS/Urdu checks; M3 adds silence/interruption, noise/echo, language/delivery, continuity and native/streaming profiling.
- **Durable features/jobs:** cross-owner access, concurrent reservations, source/profile revocation/deletion, restart, expired inputs/leases, duplicate dispatch, crash after claim/result, bounded queues/retries and private dead letters.
- **Actions/schedules:** edited/expired/duplicate approval, crash around submission, unknown outcomes/callback order, reminder timezone/restart/snooze/cancel races and truthful receipts/delivery.
- **Replicas:** stale fences, state loss/failover, reconnect without reruns, commit/publication crash gap, slow clients, shared quota and graceful draining.
- **M10:** finite branch/cost limits, parent/source isolation, malformed/injected results, verifier non-authority, cancellation/duplicate joins, one final answer/audio, trace privacy and measured baseline comparison; optional MCP gates remain separate.

Use capability flags, compatible contracts/migrations and a small observed rollout. Back up only approved durable data. **After restoration, suspend external dispatch/notification retries** until ownership, versions, post-snapshot deletions and possible prior effects are reconciled. Invalidate stale approvals, retain operation/idempotency references, resolve unknown effects, and resume schedules only under a reviewed overdue policy. Unresolved operations/occurrences cannot repeat automatically; restored deleted assets remain unavailable. This design establishes no measured capacity guarantee.
