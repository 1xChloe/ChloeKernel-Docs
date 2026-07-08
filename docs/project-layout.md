## Project layout

```
src/
├── Shared/                       → ReplicatedStorage
│   └── ChloeKernel/               microkernel: Scheduler, Process, ServiceManager,
│                                 Logger, GcWatch, Serde, Base64, InputDriver, BonePhysics,
│                                 AudioKit, AnimKit, Preload, IPC/, Hooks/,
│                                 Net/ (NetClient, ReplicaClient, TokenBucket,
│                                 Packet/ — wire transport ported from Packet v1.7
│                                 by Suphi Kaner; CREDITS.md lists local fixes),
│                                 TestKit/ (+Bench), Tests/, Benches/
├── Server/                       → ServerScriptService
│   ├── Main.server.luau          boot → Bootstrap → start
│   ├── Bootstrap.luau            YOUR game's services register here
│   ├── Simulation.server.luau    3-mode load harness (ChloeKernelSimulation attr)
│   └── ChloeKernelServer/         sessions, Net (NetDriver), Replica, BusBridge,
│                                 MessageDriver, Data/ (DataDriver, MemoryDriver,
│                                 backends), AntiExploit/ (Movement, Rewind),
│                                 Commerce/ (Receipts, Transactions), Kits/,
│                                 Pathfinding/ (AStar, ThetaStar, Direct, Roblox),
│                                 Settings, Analytics, Zones, Ragdoll, Simulation/,
│                                 Matchmaking, SoftShutdown, ActorPool/, Tests/
└── Client/
    ├── Main.client.luau          boot → Bootstrap → start
    └── Bootstrap.luau            YOUR client wiring registers here
```
