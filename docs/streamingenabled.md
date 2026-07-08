## StreamingEnabled

Supported. The architecture is server-authoritative, and the server sees the full world under instance streaming — every system that decides anything (Zones sweeps, Replica interest, NPC senses and pathfinding grids, anti-exploit, kit validation, MoveKit's grounded probe, hit detection) runs server-side and is unaffected by what any client has streamed in.

What streaming actually touches is client-side presentation:

- **InteractionKit** prompts are server-created and parented to their tagged parts — they stream in and out with the part, and triggers re-validate distance server-side either way.
- **ProjectileClient**, **AudioKit** emitters, and **RagdollClient** operate on wire data, pooled local parts, and the player's own character — no dependency on streamed world geometry.
- **BonePhysics** binds bones on models the client can see. Player characters persist under streaming by default. For NPC or prop rigs a client binds chains on, set `ModelStreamingMode = Persistent` (or handle rebind on stream-in) — a model that streams out mid-simulation takes its bones with it.
- **NPC models** may stream out on distant clients while their server brains keep acting. That is cosmetic: hits, hearing, and pathfinding never depended on the client copy.

The one rule: anything your CLIENT code holds a long-lived reference to (BonePhysics binds, custom viz attached to world models) should be `Persistent` or rebound on `Zone.Entered`/stream-in. Server code needs no changes.
