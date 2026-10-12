# Runtime, recovery, and scaling design

Status: proposed design for the [full product architecture](./architecture.md#architecture-for-the-full-product). No server, data store, broker, worker pool, or deployment exists yet. This document makes the approved roadmap operational; it adds no new product features or delivery-date commitments.

## Deployment stages

| Stage | Add when | Deployment shape | Required gate |
| --- | --- | --- | --- |
| Core: M1–M3 | First working assistant | Static React app, one Bun/TypeScript backend, bounded in-memory session state, provider adapters | Text/manual/hands-free flows, cancellation/isolation and measured audio behavior |
| Accounting: M4 | Account-wide budgets or retained preferences | Stable authenticated owner and PostgreSQL accounting/preferences; temporary history stays in memory | Concurrent reservations, restart recovery and explicit preference retention |
| Saved features: M5–M6 | Profiles, selected memories, originals/documents/routines | PostgreSQL plus private object storage; indexing entry point as needed | Owner isolation, source/profile revocation, removal and bounded indexing |
| Scheduling/actions: M7–M8 | Durable reminders and approved commerce | Durable operations/schedules/job claims/outbox; notification and reconciliation processes | Deduplication, unknown results, restart, exact approval and secure input |
| Rendering: M9 | Requested explanatory videos | Separate bounded render pool and expiring media inputs/results | Narration capacity isolation, private expiry and stale-job cancellation |
| Voice replicas | Measured single-runtime capacity or availability need | HTTPS edge, multiple voice runtimes, shared temporary Redis and media delivery | Shared ownership/replay tests and provider-wide limits; drain live streams |

Start in one region near the intended users and providers after measuring routes; hosting remains open. Keep CPU-heavy entry points in separate processes so parsing/rendering cannot block live voice callbacks. Scale those pools independently. A container/managed process deployment is a candidate; choose a host that supports the required long-lived WebSocket/SSE traffic. Kubernetes, service-per-feature decomposition, multi-region active voice state, and another broker are not initial requirements.

## Interactive turn path

1. Authorize the session, validate format/payload/limits, and atomically admit its turn against the current session generation and ownership fence. Typed input goes directly to the text path. Before opening paid streaming or batch transcription, reserve its conservative estimated allowance and acquire STT capacity; bound recording duration/bytes and reserve any extension before more paid capture. Streaming input holds its STT slot while the provider is active.
2. Stream admitted audio with packet credits and bounded queues. `end_input` carries the capture epoch and last sequence; validate all required packets, forward them in order through the adapter, then send provider commit on that same ordered stream, using provider acknowledgements when supported. Merely enqueueing packets locally does not satisfy finalization. Bound forwarding stalls and fail discontinuities visibly. Partial transcripts are draft input.
3. Resolve the finalized language, snapshot current authorized context and preferences, reserve the model/tool-stage estimate or consume its existing turn reservation without double accounting, and acquire its capacity. Read-only retrieval runs with owner/source filters and explicit size/time bounds; expensive initial indexing is a separate job.
4. Run LangChain against the isolated snapshot. Map tool lifecycle to allowlisted public progress; do not speak intermediate model text. A tool may return an action proposal, while its later approval/execution remains a separate owned operation.
5. Conditionally commit the finalized answer and replay metadata while ownership/version checks hold. Publish text, then acquire/reserve synthesis capacity and stream audio for that committed answer and validated speech settings. Check current profile authorization before dispatch and during delivery; revocation cancels pending/active use.
6. Release resources after real completion, preserving committed text on audio failure. New confirmed speech stops local audio immediately and replaces unfinished voice work. Secure login/payment uses the capture-pause barrier; a voice interruption does not erase a submitted external operation.

Reserve each paid stage before dispatch; estimates already reserved for a turn cover those stages without double charging. If a stage is refused, report the corresponding usable text/error fallback within remaining allowance. Bounded waiting for interactive admission is allowed; do not place ordinary conversation turns in a durable background queue or retain an unbounded backlog of replacement utterances.

## Ownership and recovery

| Identifier/version | Invalidated by | Does not invalidate |
| --- | --- | --- |
| Session generation and store incarnation | Reset, turn replacement as required, expired/lost temporary authority | Explicitly saved reminders, assets, operations and budgets |
| Turn ID and renewable owner fence | Cancellation, replacement, lost/expired lease | A final answer already committed or confirmed external effect |
| Capture epoch and packet sequence | New capture, secure pause, discontinuity/disposal | An independent render job or saved reminder |
| Playback-attempt epoch | Stop, new speech, retry, profile revocation/disposal | Canonical committed answer or retained external receipt |
| Source/profile version | Edit, removal, access revocation | Other authorized sources/profiles |
| Job ID, generation and lease fence | Explicit cancellation, relevant source change or video reset/disposal, expired job lease | Unrelated new voice turns; worker result always attaches to its original source |
| Action ID/version and execution attempt | Proposal edit, expiry or rejection before submission; attempt state changes after submission | Verified provider outcome; cancellation/refund needs its own approved operation |
| Reminder schedule version and occurrence ID | Snooze/edit/cancel and delivery-state transitions | Conversation reset, microphone stop or app closure |

Every write/publication validates its relevant owner and current fence/version. Keep voice, source, job, action and schedule lifecycles independent; one global epoch would either publish stale work or cancel unrelated durable features. Expiry renewals cannot resurrect revoked access.

Control commands atomically compare target identity and expected generation/version before applying cancellation or replacement; deduplicate them so an old command cannot invalidate a successor. Automatic turn speech uses the current turn fence, while explicit replay authorizes a fresh playback attempt against a retained committed answer and current session/profile access. Its original turn need not remain active. Stop playback affects only its attempt, and new speech invalidates foreground playback immediately.

Shared Redis admission/commit scripts and bounded replay close local process races, but asynchronous replica failover can lose recent ownership data. Fail closed when authority is uncertain; abandon affected old incarnation/session capabilities rather than reconstructing a running turn from client data. Never treat a Redis lease as proof that an external purchase ran exactly once. [Redis documents this failover caveat](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/).

Crash after final-answer commit is recovered from authoritative turn status/replay, not by running the model again. A broken live stream ends with a recoverable error; a subsequent turn may use another healthy replica, but live audio is not transparently migrated. Temporary state loss may lose conversation context by design. PostgreSQL recovery is independent: reminders and minimal action/receipt state survive under their stated durable policy.

The event replay buffer may expire before the retained turn status/history. Query that status to recover a committed answer or observe the same running turn; request a fresh attempt only for a confirmed eligible failure. If the entire session authority has expired or been lost, do not rebuild its history from untrusted client state or repeat operations automatically.

## Durable work and background pools

Use PostgreSQL job/operation rows as the initial durable authority. A transaction records an approved owned change and its outbox/dispatch record together. Workers claim a bounded batch in a short transaction, record a lease/fence, then do network/CPU work outside database locks. `FOR UPDATE SKIP LOCKED` is a candidate for queue-table claims, not general consistency; [PostgreSQL documents this use and its limitations](https://www.postgresql.org/docs/current/sql-select.html).

| Pool | Work | Correctness and limits |
| --- | --- | --- |
| Indexing | Validate/parse approved uploads, create search data | File/page/CPU/memory limits; owner/source-version checks before publishing; delete/revoke cancels and purges derived data |
| Rendering | Storyboard validation, narration, captions, assembly | Per-job asset/duration/resolution/concurrency bounds; separate TTS/model allowance; source-answer binding and private expiry |
| Notifications | Claim due occurrences and call verified channel | Unique schedule/version/occurrence key; expiry and cancellation rechecked just before dispatch; controlled attempts and truthful delivery state |
| Operation reconciliation | Process callbacks/status lookup and resolve uncertain submitted actions | Provider operation reference, verified callback authenticity, deduplication and bounded backoff; no blind repeat purchase/payment |
| Dispatch/cleanup | Advance outbox/job records and remove expired/revoked inputs/results | Bounded polling/batches, fencing, sweep expired leases, track incomplete provider cleanup |

Delivery is at least once where replay/retries are supported; make consumers idempotent with durable unique keys and conditional updates. Do not promise exactly-once third-party effects. A sweeper recovers expired claims, missed dispatch and terminal cleanup. Callback handlers normalize and validate status, store a deduplicated durable event, and wake reconciliation; they do not run a new conversational turn.

An expired claim for a job with no irreversible effect may be retried after checking source/access/version and attempts. If an action worker may have submitted a purchase, payment or booking, lease expiry moves it to uncertain/reconciling state; another worker cannot blindly submit again. Preserve the recorded idempotency key across valid retry/reconciliation. A provider without idempotency/status support may require explicit user resolution rather than safe automatic retry.

Reminders use durable schedules and unique occurrence rows, not long `setTimeout` calls. Store intended local schedule, IANA timezone, recurrence policy, resolved delivery instant and schedule version; handle daylight-saving ambiguity/overdue policy. Recheck cancellation before send. Once a notification channel has accepted a message, later snooze/cancel cannot recall it. Deduplicate with supported provider keys; an ambiguous send is tracked honestly and reconciled where possible.

Keep queue payloads minimal: owner, feature/job/operation ID, version, deadline, attempt, and private expiring input reference. Expired input is a terminal recoverable failure, not permission to fetch a deleted source. Video narration/source text stays in the expiring job-input class rather than durable outbox payloads. If a broker is later added, use outbox dispatch to bridge the database and broker, acknowledge only after durable state transition, and tolerate duplicate messages. Its persistence/backups must be separate from temporary session Redis.

## Capacity, latency, and overload

Use one admission policy across voice processes and worker pools. Track provider-wide request/token/audio quotas, in-flight connections, rate windows and per-owner budgets, with smaller per-workload ceilings. Reserve interactive capacity; background work waits within its own bounded queue. Adding replicas or render workers does not create additional upstream quota.

| Resource | What is bounded | Overload behavior |
| --- | --- | --- |
| Browser capture/transport/playback | Pool blocks, packet age, credits, socket bytes, buffered audio duration | Fail incomplete input or stop stalled playback; keep typed/manual recovery |
| Runtime | Active sessions/turns, event-loop work, deadlines, history/context, connection limits | Predictable admission refusal and retry guidance; no unbounded queue |
| Provider stage | STT/model/TTS connections, tokens/audio, rate, stage timeout | Protect interactive share, controlled retry for safe reads only |
| Database/object access | Pool size, query time, upload/download bytes, search results | Bounded waits and owner-scoped errors; approval/accounting fails closed |
| Workers | Queue age/depth, attempts, per-job CPU/memory/time and artifact size | Reject new work predictably or leave it visibly queued within limits |

Cancellation invalidates publishing rights immediately, but an upstream request may remain running/billable. Do not release its provider slot or reserved estimate on the client cancel event alone. Settle from known completion/usage or a conservative bounded recovery policy; after a process crash, preserve possible charges and uncertain action state. Persistent/account budgets use a durable PostgreSQL reservation/usage ledger when M4 adds them; session-only estimates must be labelled as such. Reserve atomically before paid work, reconcile provider-reported usage when available, and distinguish local estimates from the provider invoice.

Use PostgreSQL unique/conditional records for hard local admission/reservation limits and exact action approval consumption; these do not guarantee the provider's final charges. Bound paid stages and reserve conservatively. Redis may accelerate provider rate/concurrency coordination but is not the sole financial authority. Lost limiter state must not replenish an unrestricted quota: stop new paid admission until the controller can conservatively rebuild the current window/leases. Clients reuse bounded provider/database connections; avoid one new client/pool per turn.

Measure speech-end → 3-second seal → final STT → final commit → first audible audio, plus local interrupt-to-mute, stage admission wait, safe cancellation completion, p50/p95 latency and error rate. Also track event-loop lag, buffer peaks/underruns, provider saturation/rate limits, queue age, job retry/unknown-operation counts, reminder lag and database pool wait. Use monotonic local durations and correlate distributed spans by identifiers; never subtract unsynchronized clocks. Define service objectives only after M1–M3 baselines and realistic concurrency measurements.

Scale voice replicas by measured active connections/turns, stage waits, event-loop lag and latency headroom. Scale workers by oldest runnable job age and processing capacity within remaining quota. Cap all replicas/pools by database/provider limits; use gradual scale-down and stream draining. Do not scale a provider bottleneck by adding more callers. Avoid aggressive retries that amplify an outage; protect providers independently so failed synthesis leaves finalized text usable.

## Private access, caching, and deletion

All saved records/queries carry an authorized owner; derive ownership on the server. Keep per-source versions and deletion tombstones so late indexing or restored data cannot republish removed items. Put private object references behind owner checks, validate uploads, and bound parser capabilities. Asset URLs expire; short-lived signed URLs may remain usable until expiry, so immediate revocation requires an authorization proxy/current version check. Expiry/access removal is distinct from physical deletion and backup/provider cleanup.

Cache static UI and immutable public assets at the CDN. Keep private model responses, tool output, retrieval and TTS out of shared public caches. Any approved private cache key includes owner/access scope, source/profile version, language/model/settings and expiry; recheck authorization on delivery and evict on removal. Replay is session-authorized retained audio or synthesis of committed text, not another agent run. Do not cache changing shopping prices/seat availability as current quotes; use provider freshness/expiry and revalidate before approval/execution.

Operational telemetry records identifiers, counts, timings and sanitized failure classes. Disable raw payload capture in access logs, tracing, exception reporting and worker dead-letter records; observability must not become a durable conversation archive. Do not use per-session/turn IDs as unbounded metric labels. Saved-feature backups need documented retention/restore policies and deletion reconciliation; provider clone/media cleanup may fail and must remain visible.

## Rollout and verification gates

- Core: strict typing/builds, full mocked flow, configured real microphone/playback, Urdu/mixed-language support, noise/echo, silence and cancellation races, resource cleanup and streaming/native profiling.
- Durable features: cross-owner tests, concurrent budget reservations, source/profile revocation during work, derived-data deletion, account/operation restart recovery and expiring job inputs.
- Jobs: duplicate dispatch, expired leases, process crash after claim/result, bounded retries/queue age, dead-letter privacy, source deletion during indexing/rendering and separate provider capacity under render load.
- Actions/scheduling: duplicate/edited/expired approvals, crash before/after provider submission, unknown outcomes/callback ordering, reminder restart/timezone/snooze/cancel races and truthful delivery/receipt states.
- Replicas: concurrent-session isolation, stale-owner/fence rejection, state-loss/failover handling, reconnect without reruns, final-commit/publication crash gap, slow clients, quota enforcement across pools and graceful rolling deployment.

Feature deployment uses capability flags and compatible shared-contract versions. Roll out to a small test cohort, observe stage/queue metrics, and expand only after gates pass. Database migrations remain compatible with active runtimes/workers during draining. Back up only the data classes approved for durable retention; restoration exercises must include operation reconciliation, reminder deduplication and deletion tombstones. Record measured limits and evidence with the implementation milestone; this design alone does not establish capacity or latency guarantees.

After restoring durable data, suspend external dispatch and notification retries until restored ownership, versions, post-snapshot deletions and possible external effects have been reconciled. A backup may contain an approval or due occurrence whose effect already happened later. Invalidate stale approvals, retain original operation/idempotency references, and resolve unknown effects through verified status or explicit user resolution. Restored schedules resume under a reviewed overdue policy; unresolved operations/occurrences remain blocked from automatic repetition. Restored deleted assets remain unavailable until current authorization/deletion checks pass.
