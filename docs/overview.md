# ChloeKernel

Roblox game framework with an OS-style architecture: a genre-agnostic kernel (scheduler, processes, services, IPC, hooks) plus server-side drivers for networking, persistence, replication, anti-exploit, and commerce. Game code registers as services and communicates through kernel primitives; the kernel contains no game rules.

- Priority-scheduled work under a frame budget; buffer-packed serialization at every boundary ([benchmarks](#benchmarks)).
- Server-authoritative: clients send intents, never state; validation chains are fail-closed; per-channel rate limits; lag-compensated hit validation; movement monitoring.
- **598 specs** run on every Studio play-test boot. Verification status is documented per feature.

Requirements: [aftman](https://github.com/LPGhatguy/aftman) (pinned rojo/stylua/selene) and the Argon or Rojo Studio plugin. License: see [LICENSE.md](LICENSE.md) — access is **private**, granted individually by Chloe; use in games requires attribution; sharing or redistribution is not permitted.

---
