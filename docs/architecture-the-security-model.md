## Architecture & the security model

```
ReplicatedStorage.ChloeKernel        replicates — generic primitives only
ServerScriptService.ChloeKernelServer   NEVER replicates — everything privileged
```

- **The client is a renderer with opinions.** It sends intents, never state. The server validates every intent through fail-closed hook chains and is the only thing that mutates state.
- **Code placement = attack surface.** Validators, rate limits, drop tables, AI logic, data: ServerScriptService. The client holds only what it must execute — input, prediction, VFX.
- **Fail-closed everywhere.** A crashing validator rejects the action. A failed data load kicks rather than risking a wipe. A schema-violating save is refused. An unmigratable record won't load.
- **Your character stays yours.** The kernel never reassigns a character's network ownership and never anchors character parts — the client keeps simulating its own character at all times, so movement always feels local. Anti-exploit corrections are plain CFrame writes (ownership intact), and movement abilities go through [Prediction](cookbook.md#make-abilities-feel-instant): the client moves itself instantly, the server validates, rejections roll back.
- **Explicit non-goals:** client code is always readable by exploiters — the defense is never trusting it, not hiding it. Cross-server trades can't be made safe without a coordinator, so they're refused. Obfuscation is not security.
