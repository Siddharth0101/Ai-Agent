# Ai-Agent

A planned voice and text assistant built with React, TypeScript, Bun, LangChain, and ElevenLabs.

## Status

Only the Bun + TypeScript starter exists. The frontend, backend, agent, tools, and speech integrations are not implemented or installed. Next: **M1-A — scaffold the apps and shared contracts**.

## Core scope

- Spoken and typed questions, visible answers, and spoken replies.
- Calculator/time tools and temporary, isolated session context.
- Hands-free submission after about 3 seconds of silence; new speech interrupts older work.
- Same-language responses, including Urdu, with model-specific speech support verified during integration.
- Contextual speech delivery, manual overrides, cancellation, and text fallback.

M1 delivers the text foundation, M2 the manual voice baseline, and **M3 the complete core voice MVP**. Approved later work covers controls/budgets, authorized voice profiles and selected memory, documents/practice/routines, reminders/reviewed actions, shopping/booking, videos, and bounded multi-agent learning. All remain planned; see the [roadmap](./docs/roadmap.md) and [feature scope](./docs/feature-ideas.md).

Conversation history is temporary by default. Saving personal data requires an explicit choice; provider retention is separate. External changes require approval for the exact action.

## Run the starter

```bash
bun install
bun run index.ts
```

This prints a greeting. Frontend/backend development, build, and test commands will be added with their implementation. Keep provider credentials on the server.

## Documentation

- [Architecture](./docs/architecture.md) and [interactive diagram](./docs/architecture.html): system boundaries and proposed structure.
- [Roadmap](./docs/roadmap.md): delivery order, batches, and completion gates.
- [Feature scope](./docs/feature-ideas.md): approved features and separately unapproved suggestions.
- [Browser audio](./docs/browser-audio.md): capture, streaming, interruption, and cleanup.
- [Runtime and scaling](./docs/scaling.md): ownership, storage, recovery, and capacity.
- [Voice commerce](./docs/voice-commerce.md): provider access, approval, checkout, and reconciliation.
- [Agent learning](./docs/agent-learning.md): bounded M10 collaboration, evaluation, and optional MCP.
- [Development rules](./AGENTS.md): implementation and validation requirements.
