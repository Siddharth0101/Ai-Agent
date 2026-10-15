# Agent learning and multi-agent design

M10 is approved **planned scope, not implemented**, after M1–M9. Keep LangChain TypeScript, the direct agent path and core voice delivery first; add no speculative services or infrastructure. Learning means evaluating application design, not training on private conversations, automatic retention or autonomous permission changes. Follow [scope](./feature-ideas.md), [delivery order](./roadmap.md) and [runtime boundaries](./scaling.md).

## Learning layers

| Concept | Exercise |
| --- | --- |
| Model, agent and tool | Compare direct answers with validated calculator/time calls. |
| Workflow and routing | Keep capture/cancellation/commit ordering deterministic; compare fixed routes with model-selected tools. |
| Structured outputs | Validate routes, findings and verification reports; valid shape proves neither truth nor permission. |
| Retrieval and memory | Compare cited selected-source answers and minimal context with the direct baseline; reuse authorized M5/M6 sources, not blanket indexing. |
| Collaboration | Combine bounded read-only findings while retaining simple questions on the direct path. |
| Human approval | Edit/reject a proposed action and prove old approval fails through the existing M7/M8 service. |
| Evaluation and private observability | Compare quality, language, latency and usage on controlled fixtures and sanitized metadata. |
| MCP | Optionally compare one verified read-only tool with a normal adapter; connection grants no permission. |

Check current official [LangChain multi-agent](https://docs.langchain.com/oss/javascript/langchain/multi-agent), [structured output](https://docs.langchain.com/oss/javascript/langchain/structured-output) and [workflow](https://docs.langchain.com/oss/javascript/langgraph/workflows-agents) documentation against the installed version before integration; additional orchestration must demonstrate a need.

## M10: bounded multi-agent learning

### Initial team and final-answer owner

Use a fixed registry of LangChain TypeScript agents/tools, without recursive delegation or independent role services.

| Role | Boundary |
| --- | --- |
| App coordinator | Own authentication, admission, access, budgets, cancellation and the **single** history commit, final event and speech dispatch. |
| Supervisor | Route and combine bounded subtasks; cannot expand permissions or commit history. |
| Read-only researcher | Read verified allowed sources or selected authorized existing documents; no arbitrary fetching, private search or writes. |
| Calculator/time specialist | Reuse validated calculator/time tools; no code evaluation, new tools or side effects. |
| Verifier | Flag arithmetic, evidence and consistency issues; advisory only, neither authority nor truth guarantee. |

The coordinator alone publishes final committed text to history/audio. Keep plans, specialist streams, raw payloads and internal reasoning private; expose only allowlisted progress. Use documented direct-path fallback or controlled failure when collaboration cannot finish.

### Routing, structured contracts, and scoped context

Allow only `direct`, `calculator_time`, `selected_retrieval` and `collaboration`; admit available authorized routes server-side. Keep simple requests direct, clarify ambiguity and require source selection before private retrieval. Share DTO/schema definitions in `packages/shared/src/`, keeping authority fields server-created.

| Contract | Required checks |
| --- | --- |
| Task | Owner/session/turn/generation fence, unique child/attempt, fixed role/purpose, granted sources/tools, deadline and reserved allowance. |
| Route | Allowed enum, bounded tasks; reject unknown roles/tools/sources and excessive fan-out. |
| Result | Bounded summary/status, versioned evidence/tool references, uncertainty or controlled error; match child identity and current parent fence. |
| Verifier / candidate | Bounded findings and evidence; no permission fields. Candidate text/language are separate from validated speech settings. |

Give each branch a minimal isolated context. Filter ownership before retrieval; recheck source/profile/access versions before use and commit. Exclude unrelated memories/secrets; honor deletion/revocation of originals, extracts, indexes and caches. Missing evidence stays uncertain.

### Limits, cancellation, and injection defenses

Proposed initial caps: **2 concurrent read-only branches, depth 1, 3 child invocations and 7 total model calls per parent, no automatic delegation retry/repair**. Count SDK retries and repair calls against caps or disable them. Configure finite input/context/result sizes, source/page/tool calls, concurrency and stage/parent deadlines before admission; these caps are measured starting choices, not SDK defaults.

Atomically reserve every paid stage through the M4 ledger without double accounting; share provider quotas and protect ordinary voice capacity. Cancellation retains unsettled costs/capacity. Invalidate all child publication rights on parent cancel/reset/replacement/disposal, abort supported work and reject stale/duplicate results. Any allowed retry needs a new attempt/version and allowance; reconnect observes the same parent run. Local speech interruption remains immediate; only the coordinator emits confirmed terminal cancellation.

Treat user/retrieved/specialist/verifier/MCP content as data. Enforce schemas, source/owner filters, allowed tools, destinations and credentials in code; prompts and verifier votes are not security boundaries. Read-only delegates cannot execute actions. Reuse exact owner/card/version/payload/scope/expiry approval, one-time consumption, idempotency and uncertain-effect recovery through M7/M8; preserve confirmed effects after interruption. Checkpoint/resume cannot bypass approval or authorize persistent history.

### RAG evaluation and private observability

Version public/synthetic or explicitly authorized sanitized fixtures; compare direct, selected-source retrieval and collaboration on the same cases. Measure correctness, relevance, grounded citations, missing-evidence abstention, routing/schema/tool errors, language, latency and usage/cost. Separate deterministic authorization/arithmetic checks from model judging; set non-regression thresholds before comparison and report repeated-run variance.

Keep private bounded expiring metadata: versions, fixture IDs, role counts, timing, estimated charges and sanitized failures/cancellation. No hidden reasoning, secrets, raw audio, full transcripts/prompts or raw tool payloads by default. Real-content traces/evals require owned access, explicit retention/removal and provider review; source deletion applies. Hosted content tracing stays off until nested inputs/outputs, metadata, exceptions, access and deletion are verified; local checks suffice and no vendor is selected.

## M10 delivery batches

All batches are not started. M10-A–C are required; M10-D is optional and does not block their release.

### M10-A: routing, contracts, and baseline

Define roles/schemas/finite limits and verify installed capabilities; implement direct fallback and selected-source retrieval. Record reproducible baseline quality/latency/cost with runnable evaluations. Gate: strict types/builds and route/schema/source-isolation checks pass; unknown, malformed, deleted or cross-owner content fails safely without private-content capture.

### M10-B: supervisor and read-only specialists

Join bounded research/calculation findings with advisory verification under existing admission, budget, fencing, deduplication and cancellation. Gate: branch/call/time/result/cost caps hold; delayed/duplicate branches cannot publish or speak, and one coordinator commit survives failures without granting new action authority.

### M10-C: evaluation and release gate

Compare identical fixtures covering injection, wrong arithmetic/citations, missing/revoked sources, overload, budget exhaustion, cancellation and stale results. Verify trace privacy and unchanged edited/expired approval/uncertain-operation safeguards. Release only a measured useful routing slice; otherwise keep collaboration disabled. Recheck M1–M9 regressions, Urdu/language, final-text-before-TTS and immediate local interruption. Publish activated limits, versions, eval command/evidence and fallback behavior; QA/acceptance remain separate from deployment.

### M10-D: optional verified MCP adapter

Verify one supported read-only server/transport for an existing capability using current [LangChain MCP](https://docs.langchain.com/oss/javascript/langchain/mcp), [MCP authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization) and [security](https://modelcontextprotocol.io/specification/latest/basic/security_best_practices) documentation. Allowlist operator/identity, schema/capabilities, permitted data and bounds; re-review changes. Keep credentials server-side, validate token audience, forbid token passthrough, constrain egress/SSRF and local process/filesystem resources; server annotations do not prove safety. Normalize bounded errors/results, propagate ownership/cancellation and close resources. Gate: denied/wrong-audience auth, schema changes, injection, oversized output, timeouts, cancellation and attempted writes fail safely; verify private-data retention. No arbitrary endpoints, auto-installed servers or permission bypass.
