## API reference

Conventions: `Kernel.Priority = { Kernel=1, High=2, Normal=3, Low=4, Background=5 }` (lower runs first). Bus handlers receive `(topic, ...)`. Hook handlers receive a mutable `context` and cancel by returning `false`. Channel schemas are arrays of Packet type-name strings.

### Kernel (`ReplicatedStorage.ChloeKernel`)

| Member | Description |
|---|---|
| `Kernel.boot(config?) → kernel` | Once per VM. Server budgets are frame-aware by default (16.5ms frame target, 1.5ms floor); `config.BudgetSeconds` switches to a fixed slice, `config.TargetFrameSeconds` tunes the frame-aware target |
| `Kernel.current() → kernel` | The booted instance |
| `kernel:shutdown()` | Full teardown (schedulers, bus, services, diagnostics) and clears the boot singleton — a fresh `boot()` works after it |
| `kernel.Scheduler / .Hooks / .Bus / .Services` | Core subsystems |
| `kernel:registerService(def)` | `def = { Name, Dependencies?, init(self, kernel)?, start(self)?, stop(self)? }` |
| `kernel:start()` | Two-phase boot of registered services |
| `kernel:getService(name)` | After boot |
| `kernel:spawnProcess(fn, {Name?, Priority?}?) → Process` | |
| `kernel:phaseScheduler("PreSimulation"|"PostSimulation"|"PreRender") → Scheduler` | Physics/camera-phase work; PreRender is client-only |
| `kernel:stats() → snapshot` | Scheduler health + service timings |
| `kernel:enableDiagnostics(interval?)` | Diag attributes + `Kernel.Overload` bus alerts |
| `kernel:api(extensions?) → frozen table` | Restricted userspace surface |

```lua
local Kernel = require(ReplicatedStorage.ChloeKernel).boot({ BudgetSeconds = 0.002 })
Kernel:registerService(MyService)
Kernel:start()
print(Kernel:stats().Scheduler.LastStepSeconds)
```

### ServerKernel (`ServerScriptService.ChloeKernelServer`)

Everything above, plus:

| Member | Description |
|---|---|
| `kernel:onSession(fn) → connection` | Fires for current AND future sessions |
| `kernel:getSession(player) → Session?` | |
| `kernel:net() → NetDriver` | Lazy singleton |
| `kernel:stats()` | Adds `Sessions` count and `Net` counters |
| `kernel:securityAudit({Silent?}?) → {AuditFinding}` | One pre-ship sweep: unguarded channels, `Open` escape hatches, fail-open gate hooks, defaulted rate limits — prints a summary (unless Silent) and returns `{Severity, Kind, Name, Detail}` findings, most severe first |

**Session**: `.Player`, `.Data` (per-visit scratch), `.Profile` (persistent; set by `DataDriver:attach`), `.Active`, `.OnEnd`; `session:bind(x)` auto-tears-down connections, instances, functions, and disconnectables.

```lua
local Kernel = ServerKernel.current()
Kernel:onSession(function(session)
	session.Data.Combo = 0
	session:bind(function() saveCombo(session) end)
end)
local Session = Kernel:getSession(somePlayer)
```

### Scheduler

| Member | Description |
|---|---|
| `scheduler:schedule(fn, priority?, ...) → handle` | One-shot next step; `handle:cancel()` |
| `scheduler:every(seconds, fn, priority?) → handle` | Recurring |
| `scheduler:step(budget?)` | Manual drive (tests) |
| `scheduler.OnError` | Signal; task crashes are isolated and land here |
| `scheduler.Stats` | `LastStepSeconds, TasksRun, TasksDeferred` |
| `scheduler:topTasks(n)` / `:resetProfile()` | Hot-task profiler readout (what the panel shows) |
| `Scheduler.setTaskOrigin(fn, label)` | Name a function in the profiler instead of its `script:line` (Process does this per process) |

Work scheduled during a step runs next step. `Kernel` priority ignores the budget.

```lua
local Handle = Kernel.Scheduler:every(0.5, updateMinimap, Kernel.Priority.Low)
Kernel.Scheduler:schedule(function(a, b) print(a + b) end, nil, 2, 3) -- args pass through
Handle:cancel()
Kernel.Scheduler.OnError:connect(function(err) Log.error("task crashed", err) end)
```

### Process

`Process.new(fn, {Scheduler, Name?, Priority?})` · `proc:start(...)` · `:suspend()` · `:resume()` · `:kill(reason?)` · `Process.yield()` (inside) · `Process.get(pid)`. Fields: `.Pid`, `.State` (`Created|Running|Suspended|Completed|Crashed|Killed`), `.Result`, `.OnExit(state, result)`.

```lua
local Proc = Kernel:spawnProcess(function(target)
	while not reached(target) do
		stepToward(target)
		Process.yield()
	end
	return "arrived"
end, { Name = "Escort" })
Proc:start(destination)
Proc.OnExit:connect(function(state, result) print(state, result) end)
```

### Signal / Bus / HookRegistry

| Member | Description |
|---|---|
| `Signal.new()` · `:connect(fn) → conn` · `:once` · `:wait` · `:fire(...)` · `:destroy()` | Thread-reusing, O(1) disconnect |
| `Bus.new()` · `:subscribe(topic, fn) → conn` · `:publish(topic, ...)` | `"Family.*"` wildcards |
| `HookRegistry.new({WarnOnError?})` · `:definePoint(name, {FailOpen?})` | Points default fail-closed |
| `hooks:on(name, fn, priority?) → disconnectFn` | Default priority 100; ties keep registration order |
| `hooks:fire(name, context?) → (passed, context)` | |
| `hooks:listPoints() → { {Name, FailOpen, Handlers} }` | Point snapshot for audits/debug tooling |

```lua
local Sig = Signal.new()
local Conn = Sig:connect(print)
Sig:fire("hello")
Conn:disconnect()

Kernel.Hooks:definePoint("Telemetry.Event", { FailOpen = true }) -- logging mustn't block gameplay
local Off = Kernel.Hooks:on("Intent.Jump", function(context)
	return context.Session.Data.Stamina > 0
end)
Off() -- handlers return their own disconnect function
```

### Built-in hook points & bus topics

The two systems answer different questions. **Hooks** ask permission: handlers run in priority order, share one mutable `context` table, and any `false` return (or error, unless the point is `FailOpen`) cancels the action. **Bus topics** announce facts: handlers get `(topic, ...)`, run independently, and cannot cancel anything. Both registries create points and topics on first use, so your own names cost nothing to add.

**Hook points fired by the framework** (all server-side):

| Point | Context | Mode | Fired |
|---|---|---|---|
| `Intent.<Channel>` | `{Name, Player, Session, Args, Seq?}` | fail-closed | Before every intent handler. Validators may rewrite `Args` (clamp, normalize) — the handler receives the mutated copy |
| `Request.<Channel>` | `{Name, Player, Session, Args}` | fail-closed | Before every request handler; a rejection answers the client with the channel's `RejectValue` |
| `Intent.BusPublish.<Topic>` | `{Name, Player, Session, Args}` | fail-closed | Client-to-server bus publish (BusBridge), after the exact-name whitelist and rate limit |
| `Session.Start` / `Session.End` | `{Session, Player}` | observational | Player join/leave. The kernel ignores the verdict, but handlers run synchronously before the matching bus topic — seed per-player state here and bus subscribers already see it |
| `Net.RateLimited` | `{Player, Channel}` | fail-open | A channel's token bucket rejected a message |
| `Net.FloodDetected` | `{Player, Bytes, PayloadBytes, BudgetBytes}` | fail-open | The transport dropped a client that blew its byte budget |
| `AntiExploit.MovementViolation` | `{Player, Session, Type, HorizontalSpeed, Displacement, Strikes}` | fail-open | A movement sample failed validation |
| `Intent.Interact` | `{Session, Player, Id, Instance, Definition}` | fail-closed | Every prompt trigger (InteractionKit); the kit's distance/line-of-sight/cooldown gate sits at priority 50 |

Kit channels are ordinary intents, so their points follow the same shape — `Intent.WK_Fire`, `Intent.SK_Cast`, `Intent.Checkpoint`, `Intent.Settings_Set` — and you prepend rules at lower priority numbers without forking the kit.

**Bus topics published by the framework.** Handlers receive `(topic, ...)`; the args below follow the topic name.

| Topic | Args | Published when |
|---|---|---|
| `Kernel.SessionStart` / `Kernel.SessionEnd` | `session` | Join/leave; `SessionEnd` fires before the session object is destroyed, so bound state is still readable |
| `Kernel.ProfileLoaded` | `session, profile` | DataDriver finished loading a player's profile |
| `Kernel.ProfileRestored` | `session, profile` | A corrupt or unmigratable record was restored from backup — tell the player, ping your webhook |
| `Kernel.ScriptError` | `message, trace, source, count` | ErrorWatch captured (or re-counted) a script error |
| `Kernel.Overload` | `statsSnapshot` | Diagnostics saw deferred work three windows in a row — the scheduler is saturated |
| `Kernel.SoftShutdown` | `playerCount` | Update migration is about to teleport everyone out |
| `Intent.<Channel>` | `session, ...validatedArgs` | AFTER an accepted intent ran — the observational twin of the hook chain |
| `Net.IntentRejected` | `player, channelName` | A validator, missing guard, or crashing handler rejected an intent |
| `Net.RateLimited` | `player, channelName` | Token bucket rejection |
| `Net.FloodDetected` | `player, bytes, payloadBytes, budgetBytes` | Transport-level flood drop |
| `Zone.Entered` / `Zone.Left` | `zoneName, player` | A player crossed a zone boundary (leaves fire before enters) |
| `Zone.EntityEntered` / `Zone.EntityLeft` | `zoneName, entity` | A tracked entity (NPC, prop, vehicle) crossed a zone boundary |
| `Effects.Applied` | `player, effectName, stacks` | Effect applied or stacked |
| `Effects.Expired` | `player, effectName` | Effect expired or was removed |
| `Round.PhaseChanged` | `phaseName, duration` | RoundKit advanced a phase |
| `Projectile.Hit` | `ownerSession, victim, defId, position` | A server-simulated projectile landed |
| `Config.Changed` | `key, value` | LiveConfig saw a new value (local `set` or remote poll) |
| `Obby.Checkpoint` | `player, stage` | CheckpointKit accepted a checkpoint |
| `Spell.Cast` / `Spell.Interrupted` | `session, spellId` | SpellKit outcome |
| `Weapon.Fired` / `Weapon.Hit` | `(session, weaponId, origin, direction)` / `(session, victim, weaponId, position)` | WeaponKit accepted a shot / landed a hit |
| `Weapon.ShotRejected` | `player, reason` | WeaponKit refused a shot |
| `Procedural.Spawned/Updated/Rejected/Removed` | see [ProceduralKit](#proceduralkit-parametric-geometry-end-to-end) | Runtime geometry lifecycle |
| `Quest.Progress` | `player, questId, objectiveIndex, count, required` | An objective advanced |
| `Quest.Completed` | `player, questId` | Every objective met (Repeatable quests reset after) |
| `Interact.Triggered` / `Interact.Rejected` | `(session, id, instance)` / `(player, id)` | A prompt interaction passed / failed validation |
| `Npc.Spawned` / `Npc.Died` | `npc` | NPCKit lifecycle (`Died` precedes the despawn delay) |
| `Npc.Action` | `npc, actionName, target` | A moveset action executed |
| `Npc.Heard` | `npc, heardPosition, source?` | A sound survived range/occlusion for this npc (position pre-blurred by its skill) |
| `Npc.Idle` / `Npc.IdleEnded` | `(npc, actionName, duration)` / `(npc, actionName)` | An idle action (sleeping, eating) started / finished — hook animations here |
| `Npc.Path` | `npc, waypoints` | A new route was computed (full list, own position first) — hook for path visualization |
| `Npc.Hitscan` | `npc, origin, hitPosition, victim?` | A hitscan action fired (hit or miss) — hook for tracers |
| `Npc.Defense` | `npc, reactionName, success, context` | A defensive reaction attempt (deflect flash or fumble — hook VFX here) |
| `Npc.Tactic` | `npc, tactic` | Director/behavior state flip: `Pressure`/`Flank`/`Suppress`/`Cover`/`Peek`/`Flee` — hook client animation polish |
| `Ragdoll.Started` / `Ragdoll.Ended` | `model` | A ragdoll engaged / restored |
| `Audio.Cue` | `soundName, cueName, handle` | Playback crossed a registered cue time (loop-aware) |
| `Audio.Subtitle` / `Audio.SubtitleEnded` | `(text, duration)` / `()` | Dialogue line started / finished — wire your subtitle UI here |
| `Anim.Marker` | `model, animName, markerName, param` | An animation marker fired on a rig |
| `Preload.Started` / `Preload.Progress` / `Preload.Done` | `(total)` / `(loaded, total, assetId, ok)` / `(loaded, total, failed)` | Asset warm-up lifecycle — wire your loading screen here |
| `Settings.Changed` | `session, key, value` | A setting was written (client intent or server `set`) |
| `Matchmaking.Queued` / `Dequeued` | `player, queueName` | Queue membership changed |
| `Matchmaking.MatchFound` / `TeleportFailed` | `queueName, members` | Match lifecycle |
| `Commerce.PurchaseGranted` | `player, productId, purchaseId` | A receipt was processed (exactly once per purchase) |
| `AntiExploit.MovementViolation` | `player, violationType, strikes` | A movement strike was recorded |
| `Input.<Action>` | `state, inputObject` | **Client-side** — an InputDriver action changed state |

One topic is consumed rather than published: publish **`AntiExploit.Forgive`** with `(player)` right before a legit server-side teleport and the next movement sample is pardoned.

### Logger

`Logger.scope(name) → { debug, info, warn, error }` · `Logger.setLevel(Logger.Level.X)` · `Logger.addSink(fn) → removeFn` (crash-isolated) · `Logger.setConsoleEnabled(bool)` / `Logger.isConsoleEnabled()`.

```lua
local Log = Logger.scope("Matchmaking")
Log.info("queued", player.Name)
Log.debug("hidden unless setLevel(Logger.Level.Debug)")
local Remove = Logger.addSink(function(entry)
	if entry.Level >= Logger.Level.Error then alertWebhook(entry) end
end)
```

### GcWatch

GC statistics from sampling `gcinfo()` on a Background task (a counter read — effectively free). Roblox exposes **no GC pause timings** (`collectgarbage` is limited to `"count"`), but the signals that matter are all readable off the heap curve, and **allocation rate is the lever**: every KB allocated must be marked and swept by Luau's incremental GC, and allocation spikes trigger GC assists inside your frames.

| Member | Description |
|---|---|
| `GcWatch.attach(kernel, {SampleSeconds?, WindowSeconds?}?) → watch` | Default 0.1s samples, 5s rate windows |
| `watch:stats() → { HeapKb, PeakHeapKb, AllocKbPerSec, CyclesPerMin, Cycles, LastReclaimKb, AvgReclaimKb, ReclaimedKb }` | |
| `watch:destroy()` | Cancels the sampler |

```lua
local Gc = GcWatch.attach(Kernel)
local Stats = Gc:stats()
-- heap CLIMBING + reclaims ~0  = a leak (the GC has nothing to free)
-- heap STABLE  + busy cycles   = garbage churn (cut per-frame allocation)
print(`alloc {Stats.AllocKbPerSec} KB/s, {Stats.CyclesPerMin} GC cycles/min`)
```

`kernel:enableDiagnostics()` attaches one automatically and publishes `ChloeKernelDiag{Role}GcAllocKbS` / `GcCyclesMin` / `GcReclaimKb` attributes; the debug panel grades them in both windows.

### Serde / Base64

| Member | Description |
|---|---|
| `Serde.schema(spec, {Version?, Extras?}?) → codec` | Spec: type-name strings, `{elementSpec}` lists, nested dicts, `Serde.optional(spec)` |
| `codec:encode(value) → buffer` / `codec:decode(buffer)` | Version byte checked when configured |
| `Serde.encode(value)` / `Serde.decode(buffer)` | Adaptive: numbers auto-packed, 1 type byte per value |
| `Serde.infer(sample) → spec` | Typed spec from a plain table — safe wide widths, never narrowed from the sample's magnitude |
| `Serde.struct(spec) → struct` | `encodeFull/decodeFull/encodeDelta/decodeDelta` — powers replication |
| `Base64.encode(buffer) → string` / `Base64.decode(string) → buffer` | RFC 4648, buffer-native |

Type menu: `NumberU8/16/24/32`, `NumberS8/16/24/32`, `NumberF16/24/32/64`, `NumberVlq` (variable-length: 1 byte under 128, 2 under 16384, up to 2^53), `String(Long)`, `Buffer(Long)`, `Boolean8`, `Vector2/3` (S16/F24/F32), `CFrameF24U8`–`CFrameF32U16`, `Color3`, `BrickColor`, `EnumItem`, sequences, `Characters`, **`Any`**. Storage codecs reject `Instance`.

Schema and struct codecs compile to a **fast engine** (techniques from [light/holy](https://github.com/hardlyardi/light) by @hardlyardi, MIT): byte widths resolve at compile time, encode allocates once at the exact size, and writers run with zero capacity checks — 25-40% faster than the wire engine path with **byte-identical output** (spec-asserted). Schemas using types the fast engine doesn't cover (`Any`, CFrames, sequences, `Extras`) fall back automatically; the format never changes either way.

**Anything non-typed gets the best-fitting representation automatically:**

- **`"Any"` in a schema** — that field stores whatever it holds (nil, numbers at the smallest width that fits, strings of any length, whole tables, vectors) at 1 type byte per value. Unstorable values error with a clear message instead of corrupting.
- **`Extras = true`** — fields present in the data but missing from the schema are preserved adaptively instead of silently dropped (the default drops them — typed fields are zero-overhead because the schema is the contract). New fields survive before you've typed them; tighten later.
- **`Serde.infer(sample)`** — builds a typed spec from plain data. Integers infer `NumberU32`/`NumberS32` and floats `NumberF64` **deliberately**: inference never narrows from a sample's magnitude, because samples lie (`Coins = 0` today is not Coins' maximum). Edit the returned spec wherever you know the real bounds.

```lua
local Codec = Serde.schema({
	Coins = "NumberU32",
	Position = "Vector3F24",
	Inventory = { "NumberU16" },          -- list of item ids
	Title = Serde.optional("String"),     -- 1 byte when absent
	Pouch = "Any",                        -- untyped: best-fit per value
}, { Version = 2, Extras = true })        -- un-schema'd fields ride along
local Packed = Codec:encode(data)         -- buffer, 5-20 bytes vs ~80 of JSON
local Back = Codec:decode(Packed)         -- errors on version mismatch

local Spec = Serde.infer(defaults)        -- typed spec from a plain table
local Auto = Serde.schema(Spec)           -- tighten Spec by hand where you know better

local Blob = Serde.encode({ anything = true, nested = { 1, 2.5, "x" } }) -- schemaless
local AsText = Base64.encode(Blob)        -- for stores that need strings
```

### NetDriver (server) / NetClient

| Member | Description |
|---|---|
| `Net:defineIntent(name, schema, {RateLimit?=30, Burst?, Handler?, Open?}?)` | Handler + no validator + not `Open` → fail-closed (rejects) |
| `Net:onIntent(name, fn(session, ...))` | |
| `Net:defineState(name, schema)` · `:sendState(name, player, ...)` · `:broadcastState(name, ...)` | |
| `Net:defineRequest(name, schema, responseSchema, {Handler, RejectValue?, RateLimit?, Open?}?)` · `Net:onRequest(name, fn(session, ...))` | same fail-closed rule; unguarded → `RejectValue` |
| `Net:auditValidators() → { {Name, Kind} }` | Handler-bearing channels with no validator and no `Open` (i.e. currently rejecting) |
| `Registry.define(defs)` · `Registry.server(kernel, defs) → handles` · `Registry.client(defs, netClient?) → handles` | Single-source channels; `Validate` rules (Range/OneOf/MaxLength) compile into hook chains; `Open = true` per def for no-auth channels |
| `BusBridge.attach(kernel, {ServerTopics?, ClientTopics?, ClientSchemas?, RateLimit?=20}?)` · `BusBridgeClient.attach(kernel, netClient?, {ClientSchemas?}?)` | Bus topics across the network; upward publishes are whitelisted, typed, and rate-limited |
| `MessageDriver.new({Codec?, BackoffSeconds?}?)` · `:publish(topic, data) → (ok, err?)` · `:subscribe(topic, fn(data, sentAt?))` | Cross-server events over MessagingService, Serde-packed, 1KB-guarded |
| `Net:defineUnreliableState(name, structSchema)` · `:sendUnreliableState(name, player, data)` · `:broadcastUnreliableState(name, data)` | Lost > late for high-frequency cosmetic data; ≤900 bytes |
| `NetClient:onUnreliableState(name, structSchema, fn(data))` | |
| `Net.Stats` | `IntentsAccepted/Rejected/RateLimited, RequestsRejected, TransportFloods/Bytes` |
| `NetClient.new()` · `:intent(name, schema) → {fire}` · `:onState(name, schema, fn)` · `:request(name, inSchema, outSchema, {RejectValue?}?) → {invoke}` | A timed-out `invoke` resumes with `RejectValue`, same as a rejection |

Hook points: `Intent.<name>`, `Request.<name>` (fail-closed); `Net.RateLimited`, `Net.FloodDetected` (fail-open). Bus topics: `Intent.<name>`, `Net.IntentRejected`, `Net.RateLimited`, `Net.FloodDetected`.

```lua
-- Server
local Net = Kernel:net()
Net:defineIntent("PlantSeed", { "NumberU8", "Vector2S16" }, { RateLimit = 10 })
Net:onIntent("PlantSeed", function(session, seedId, plot) ... end)
Net:defineState("WeatherChanged", { "NumberU8" })
Net:broadcastState("WeatherChanged", WEATHER_RAIN)

-- Client
local Net = NetClient.new()
Net:intent("PlantSeed", { "NumberU8", "Vector2S16" }).fire(seedId, plotXY)
Net:onState("WeatherChanged", { "NumberU8" }, setWeatherVfx)
```

### ReplicaService (server) / ReplicaClient

| Member | Description |
|---|---|
| `ReplicaService.new(kernel)` | |
| `service:create(name, {Schema, Data, TickRate?=10, Interest?, Quantize?}) → replica` | `Quantize = {[field] = threshold}` deadbands numeric/vector fields against the last replicated value |
| `replica:set(key, value)` · `:patch(table)` · `:get(key)` · `:destroy()` | Dirty fields coalesce per tick |
| `replica:bindZone(zones, zoneName) → unbind` | Zone-scoped interest: `Zone.Entered/Left` flips subscription instantly (snapshot on entry, remove on exit); the interest scan agrees via `isInside` |
| `ReplicaClient.new()` · `:listen(name, schema, fn(replica))` | Early packets buffered until listen |
| client replica: `.Data`, `.OnChange(key, new, old)`, `.OnRemove` | |

```lua
-- Server: per-player private replica (only its owner sees it)
local Wallet = Replicas:create(`Wallet_{player.UserId}`, {
	Schema = { Coins = "NumberU32", Gems = "NumberU16" },
	Data = { Coins = 0, Gems = 0 },
	Interest = function(viewer) return viewer == player end,
})
Wallet:set("Coins", 150)

-- Client
ReplicaClient.new():listen(`Wallet_{LocalPlayer.UserId}`, WalletSchema, function(wallet)
	wallet.OnChange:connect(function(key, value) Hud[key].Text = value end)
end)
```

### InputDriver (client)

`InputDriver.attach(kernel)` · `:bindAction(name, {Keyboard?, Gamepad?, TouchButton?, Handler?})` · `:rebind(name, "Keyboard"|"Gamepad", keys?)` · `:unbindAction(name)` · `:getBindings()` · `:destroy()`. Emits `Input.<Action>` on the bus. `Keyboard`/`Gamepad` take one `Enum.KeyCode` or an array of them; snapshots mirror the shape (name or array of names).

`MoveKit.attach(kernel, {Net?, Clock?, Humanoid?, Root?, SkipCharacterTracking?}?)` — `:define(name, {Input?, When?="Airborne", Charges?=1, Cooldown?=0, MinAirTime?=0.1, Announce?=true, Run})` · `:trigger(name) → bool` · `:destroy()` · `MoveKit.jumpVelocity(humanoid)` · recipes `MoveKit.recipes.{doubleJump({Power?, Charges?, Cooldown?}?), airStep({Steps?=3, Boost?=0.75, Cooldown?, OnStep?}?)}`. Server: `MoveKit.server(kernel, {Abilities = {[name] = {Charges?, Cooldown?, Forgive?}}, IsGrounded?, Clock?, CooldownSlack?=0.05})` — fail-closed `CKMove` intent metering with server-side ground truth; `Input = "Jump"` rides `UserInputService.JumpRequest`, other strings ride `Input.<Action>` bus events. Bus: client `Move.Triggered(name, context)`, server `Move.Used(session, name, position)` / `Move.Rejected(player, name, reason)`.

### DataDriver

```lua
DataDriver.new({
	Name, Defaults,
	Backend? = "DataStore" | "Memory" | "Http" | instance,
	BackendConfig?,        -- { Scope? } or { BaseUrl, Headers } for Http
	Backup? = {            -- see "Back up player data" in the cookbook
		Backend? = "DataStore" | "Memory" | "Http" | instance, -- default: DataStore "{Name}_Backups"
		BackendConfig?,
		IntervalSeconds? = 86400, -- once a day per key, exactly once game-wide
		Keep? = 3,             -- rotation slots per key
		Mirror? = false,       -- ALSO copy every save to a "live" slot
		Fallback? = true,      -- restore newest backup when the record is corrupt
	},
	Codec?,                -- Serde schema codec, or "Auto" (inferred from Defaults + extras)
	SchemaVersion?, Migrations?,  -- { [n] = function(data) -> data }
	Validate?,             -- gates every save
	AutosaveSeconds? = 60, LockTtlSeconds? = 180,
	BackoffSeconds? = {1,2,4,8}, KickOnLoadFailure? = true,
})
```

`driver:attach(kernel)` (full lifecycle) · `driver:load(key) → (Profile?, err?)` · `:saveAll()` · `:releaseAll()` · `:backupNow(profile)` · `:listBackups(key)` · `:peekBackup(key, slot?)` (read-only, lock-free). Profile: `.Data`, `.Meta`, `.Key`, `.Active`, `.RestoredFromBackup`, `:save() → (ok, err?)`, `:release()`.

```lua
-- attached lifecycle (typical):
PlayerData:attach(Kernel) -- session.Profile.Data is wired for every session

-- manual lifecycle (clan banks, shared stashes — anything not player-keyed):
local Bank, Err = ClanData:load(`Clan_{clanId}`)
if Bank then
	Bank.Data.Vault += deposit
	Bank:save()
	Bank:release() -- frees the session lock for other servers
end
```

### MemoryDriver

`MemoryDriver.new({Name, Kind? = "HashMap"|"SortedMap", Codec?, DefaultTtlSeconds?=300})` · `:set(key, value, ttl?)` · `:get(key) → (ok, value, err?)` · `:update(key, transform, ttl?)` (return nil to abort) · `:remove(key)` · `:getRange(direction?, count?)`.

```lua
-- live cross-server leaderboard
local Board = MemoryDriver.new({
	Name = "WeeklyKills",
	Kind = "SortedMap",
	Codec = Serde.schema({ Kills = "NumberU32", Name = "String" }),
	DefaultTtlSeconds = 7 * 24 * 3600,
})
Board:set(tostring(player.UserId), { Kills = kills, Name = player.Name })
local Ok, Top = Board:getRange(Enum.SortDirection.Descending, 10)
```

### Commerce

| Member | Description |
|---|---|
| `Receipts.attach(kernel, {GetProfile, GetPlayer?, LedgerSize?=200, Bind?=true})` | Binds `ProcessReceipt`; only a landed save acknowledges |
| `receipts:onProduct(productId, grantFn(session, receiptInfo))` | |
| `Transactions.atomic(profileA, profileB, mutate, {Id?}?) → {Ok, Reason?}` | Per-profile locks + replay ids. Reasons: `Aborted, MutatorError, SaveAFailed, SaveBFailed, ProfileInactive, SameProfile, Busy, DuplicateTransaction` |

```lua
local Shop = Receipts.attach(Kernel, {
	GetProfile = function(player)
		local Session = Kernel:getSession(player)
		return Session and Session.Profile
	end,
})
Shop:onProduct(GEM_PACK_ID, function(session)
	session.Profile.Data.Gems += 100
end)

local Result = Transactions.atomic(GifterProfile, ReceiverProfile, function(a, b)
	if a.Gems < 50 then return false end
	a.Gems -= 50
	b.Gems += 50
	return true
end)
```

### AntiExploit

| Member | Description |
|---|---|
| `Movement.attach(kernel, {IntervalSeconds?=0.2, SpeedTolerance?=1.4, TeleportStuds?=80, StrikeDecaySeconds?=10, RubberbandStrikes?=2}?)` | |
| `monitor:forgive(player)` / Bus `AntiExploit.Forgive` | Announce server-initiated moves |
| Hook `AntiExploit.MovementViolation` (fail-open) | `{Player, Session, Type, HorizontalSpeed, Displacement, Strikes}` |
| `Rewind.attach(kernel, {SampleHz?=20, HistorySeconds?=1, OriginToleranceStuds?=8, HitToleranceStuds?=6, MaxFutureSeconds?=0.2, ShowHitboxes?, Clock?}?)` | Clock defaults to `workspace.GetServerTimeNow` |
| `lag:getPositionAt(player, timestamp) → Vector3?` | |
| `lag:showHitboxes(enabled)` | Dev visualization: every `castRay` draws the rewound capsules it tested (green = hit, red = miss), replicated to all clients; visual only |
| `lag:validateShot(claim) → (ok, reason?)` | Reasons: `FutureTimestamp, StaleTimestamp, NoShooterHistory, OriginMismatch, OutOfRange, NoTargetHistory, HitMismatch` |
| `lag:castRay(shooter, origin, direction, range, timestamp, hitbox?) → (player?, position?, distance?)` | Lag-compensated hit detection: casts against every tracked player's REWOUND capsule hitbox (`{Radius?=1.6, HalfHeight?=2.4}`); world obstruction is the caller's job |
| `Rewind.rayCapsule(origin, unitDir, range, center, radius, halfHeight) → distance?` | Pure capsule intersection, unit-testable |

```lua
local Monitor = Movement.attach(Kernel, { SpeedTolerance = 1.5 }) -- bunnyhop-friendly game
local Lag = Rewind.attach(Kernel)

Kernel.Hooks:on("AntiExploit.MovementViolation", function(context)
	if context.Strikes >= 4 then context.Player:Kick() end
end)

-- before any server-side teleport:
Kernel.Bus:publish("AntiExploit.Forgive", player) -- or Monitor:forgive(player)
```

### Matchmaking / SoftShutdown / ActorPool

| Member | Description |
|---|---|
| `Matchmaking.new(kernel, queueName, {TeamSize?=2, PlaceId?}?)` · `:enqueue(player)` · `:dequeue(player)` | Bus: `Matchmaking.Queued/Dequeued/MatchFound` |
| `SoftShutdown.attach(kernel)` | Reserved-transit migration; no-op in Studio |
| `ActorPool.new({Size, WorkerModule, Parent?, TimeoutSeconds?=10})` · `pool:dispatch(...) → (ok, result|err)` (yields) · `pool:forkJoin(items, {ChunkSize?, Context?}?) → (ok, results|firstError)` (yields) · `:destroy()` | WorkerModule returns a pure `function(...) → result`; forkJoin calls it as `fn(item, context)` per item, chunks default to 4 waves per actor, idle actors pull the next chunk (self-balancing), results stitch in item order, failures carry the item index |

```lua
local Arena = Matchmaking.new(Kernel, "Arena", { TeamSize = 4, PlaceId = ARENA_PLACE })
Arena:enqueue(player)

SoftShutdown.attach(Kernel) -- one line; transit-server migration on updates

local Pool = ActorPool.new({ Size = 4, WorkerModule = script.ChunkWorker })
local Ok, Chunk = Pool:dispatch(chunkX, chunkZ) -- yields until the worker replies
```

### Kits

`InventoryKit.attach(kernel, {Items, MaxSlots?=30, Persist?=true, Field?="Inventory"})` — `:grant(session, id, count?) → granted` · `:take(session, id, count?) → bool` (all-or-nothing) · `:count(session, id)` · `:move(session, from, to)` · `:equip(session, slotIndex)` · `:unequip(session, slotName)` · `:equipped(session, slotName) → (item?, index?)` · `:use(session, slotIndex)` · `:items(session)`. Items: `{Name?, Stack?=1, Kind?, Equip?, OnUse?, OnEquip?, OnUnequip?, Meta?}`. Client intents `INV_Move/Equip/Unequip/Use` fail-closed; owner state push `INV_Sync`; hooks `Inventory.CanEquip/CanUse` (fail-open); Bus `Inventory.Granted/Taken/Equipped/Unequipped/Used/Changed`.

`LootKit.attach(kernel, {Tables, Inventory?, Seed?, Rng?})` — `:roll(table) → items` · `:rollFor(session, table) → items` (pity persists in profile `Data.LootPity`) · `:award(session, table) → (granted, overflow)`. Entries: `{Weight, Id?, Count?|{min,max}, Table?, Nothing?, PityAfter?}`; tables: `{Rolls?|{min,max}, Entries}`; nesting caps at 8; unknown references fail at attach. Bus `Loot.Rolled/Awarded`.

`CraftingKit.attach(kernel, {Recipes, Inventory, TickSeconds?=0.25, Clock?, Intents?=true, SkipLoop?})` — `:craft(session, recipeId, station?) → (ok, reason?)` · `:canCraft(...)` · `:cancel(session)` (refunds) · `:activeCraft(session)` · `:destroy()`. Recipes: `{Inputs, Output, Seconds?=0, Station?, OnComplete?}`; intents `CR_Craft`/`CR_Cancel`; hook `Craft.CanCraft` (fail-open); Bus `Craft.Started/Completed/Cancelled/Overflow`.

`ShopKit.attach(kernel, {Shops, Inventory, Currency?, CurrencyField?="Coins", Clock?, Intents?=true})` — `:buy(session, shop, item, count?) → (ok, reason?)` · `:sell(...)` · `:listings(shop)`. Shops: `{Listings? = {[id] = {Price?, SellPrice?, Stock?}}, Rotation? = {Every, Size, Pool, Seed?, Listing?}}`; rotation is deterministic per window across servers; Currency adapter `{Get, Take, Give}`; intents `SH_Buy`/`SH_Sell` (count <= 99); Bus `Shop.Bought/Sold/Rejected`.

`CompanionKit.attach(kernel, {Npcs, Effects?, Companions, TickSeconds?=0.25, Intents?=true, SkipLoop?})` — `:summon(session, id) → npc?` · `:dismiss(session)` · `:companionOf(session) → (npc?, id?)` · `:destroy()`. Companions: `{Archetype, FollowDistance?=8, TeleportDistance?=60, Aura?, OnSummon?, OnDismiss?}`; one per session; intents `CP_Summon`/`CP_Dismiss`; hook `Companion.CanSummon` (fail-open); Bus `Companion.Summoned/Dismissed/Died`.

`PartyKit.attach(kernel, {MaxSize?=4, InviteTtlSeconds?=60, TickSeconds?=5, Clock?, Intents?=true, SkipLoop?}?)` — `:invite(from, target) → (ok, reason?)` · `:accept(player, from) → (ok, reason?)` · `:decline(player, from)` · `:leave(player)` · `:kick(leader, member)` · `:promote(leader, member)` · `:partyOf(player) → party?` · `:sameParty(a, b)` · `:members(player) → {Player}` · `:destroy()`. Intents `PT_Invite/Accept/Decline/Leave/Kick/Promote` (fail-closed, UserIds as F64); state pushes `PT_Sync` (roster) and `PT_Invited`; hook `Party.CanInvite` (fail-open); Bus `Party.Created/Disbanded/Joined/Left/Kicked/Invited/Promoted`.

`TeamKit.attach(kernel, {Teams = {[name] = {Color?, Capacity?}}, AutoAssign?=true, FriendlyFire?=false, SpawnTag?="TeamSpawn", UseSpawns?, SyncRoblox?, Seed?})` — `:assign(player, team) → bool` · `:teamOf(subject) → name?` (Player/session/character/npc handle via Faction) · `:sameTeam(a, b)` · `:players(team)` · `:emptiest()` · `:destroy()`. Vetoes `Weapon.CanDamage` + `Melee.CanHit` unless FriendlyFire; tagged `TeamSpawn` parts with a `Team` attribute own spawns; Bus `Team.Assigned(player, team, previous?)`.

`MeleeKit.attach(kernel, {Moveset, BlockScale?=0.25, ParryWindow?=0.18, ParryCooldown?=0.6, StaggerSeconds?=1.1, GuardBreakStaggerScale?=1.5, LatencySlack?=0.075, TickSeconds?=0.05, Intents?=true, Clock?, GetPosition?, GetFacing?, SkipLoop?})` — `:register(entity, {Humanoid?}?)` · `:unregister(entity)` · `:attack(entity, name) → bool` · `:block(entity, down) → bool` · `:parry(entity) → bool` · `:canAttack(entity, name)` · `:state(entity)` · `:destroy()`. Attacks: `{Damage, Range?=7, Arc?=120, Windup?=0.2, Recovery?=0.3, ComboNext?, GuardBreak?, OnHit?}`. Player intents `ML_Attack`/`ML_Block`/`ML_Parry` auto-wire fail-closed; npc victims with `.defend` roll their skill-scaled reaction as the parry. Hook `Melee.CanHit` (fail-open); Bus `Melee.Hit/Blocked/GuardBroken/Parried/Staggered/Combo/State`.

`WeaponKit.service({Weapons, Rewind?, Ammo?})` · `SpellKit.service({Spells, MaxMana?=100, ManaRegenPerSecond?=5})` · `CheckpointKit.service({FolderName?="Checkpoints", MinimumLegitSeconds?=3, StageField?="BestStage"}?)` · `ProceduralKit.service({Archetypes, RegenSecondsPerModel?=0.25})` — each returns a service definition for `kernel:registerService`. ProceduralKit: `spawn(name, {CFrame?, Size?, Params?, Parent?, Owner?}) -> handle` (`handle:set/patch/resize/regenerate/waitForGeneration/destroy`), `validate(name, params) -> (ok, cleanedOrReason)` for wiring client customization intents.

`QuestKit.service({Quests, Field?="Quests"})` — quest: `{Objectives = {{Topic, Count?=1, Match?, Filter?}}, AutoAssign?, Repeatable?, OnComplete?}`; service `:assign(session, id)` · `:abandon` · `:progress(session, id) → {Done, Objectives}?`; Bus `Quest.Progress/Completed`.

`InteractionKit.service({Interactions, GetDistance?, HasLineOfSight?})` — interaction: `{Tag, ActionText?, ObjectText?, HoldDuration?, MaxDistance?=10, Cooldown?, LineOfSight?, OnInteract?}`; prompts follow CollectionService tags; every trigger re-validates through `Intent.Interact` (kit gate at priority 50); Bus `Interact.Triggered/Rejected`.

`NPCKit.attach(kernel, {Pathfinding?, Grid?, PathMethod?, Projectiles?, Zones?, Container?, Seed?, GetTargets?, Clock?, Sounds?, HasLineOfSight?, Raycast?})` — no loop starts: your game schedules its own tick. Kit: `:define(name, {Model?, Health?, WalkSpeed?, Difficulty?="Medium", Moveset?, Reactions?, DespawnSeconds?=3, Targetable?, Faction?, Squad?, GetPosition?, Move?})` · `:spawn(name, cframeOrPosition, {Difficulty?, Squad?}?) → npc` · `:all() → {npc}` · `:update()` (squad director + due windups) · `:squad(name) → squad` · `:emitSound(position, {Range?=40, Loudness?, Source?}?)` · `:findCover(npc, threat, {SearchRadius?=40, SpotTag?, BackAway?}?) → Vector3?` · `:findPeekPoint(spot, threat) → Vector3?` · `:hasLineOfSight(from, to)` · `:count()` · `:destroy()`. `Sounds = {Occlusion? = "Through"/"Blocked"/"Path", Topics? = {[busTopic] = range}}`. Npc: `:position()` · `:facing()` · `:canSee(target, maxRange?, fov?)` · `:sweep(direction, arc?=90, rays?=5, range?=24) → Model?` · `:aimAt(pos, velocity?, speed?, gravity?, skill?) → direction` · `:act(name, target?) → bool` · `:defend(context) → bool` · `:notifyDamage(attacker?)` · `:lastIntel(target)` · `:updateTarget(acquire, giveUp, {RequireSight?, MemorySeconds?, DetectionSeconds?}?)` · `:moveTowards(goal)` · `:faceTowards(point)` · `:canReact()` · `:setDifficulty(nameOrTable)` · `:destroy()`. Squad: `:report(npc, target, position, velocity?)` · `:lastKnown(target)` · `:role(npc)` · `:add/remove(npc)` · `.FlankSigns[npc]`. Actions: `{Kind = "Projectile"/"Hitscan"/"Melee"/"Custom", Difficulty?, Cooldown?, DefId?, Speed?, Gravity?, Range?, Damage?, OnHit?, Run?, Magazine?, ReloadSeconds?, WindupSeconds?, FriendlyFire?}`; Reactions: `{Against?, Cooldown?=1, Difficulty?, Run?}`. Difficulty presets `NPCKit.Difficulties.{Perfect,HumanPeak,ReallyGood,Good,Medium,Novice,Bad}` = `{AimErrorDegrees, ReactionSeconds, LeadSkill, CooldownMultiplier, HearingMultiplier, HearingBlurStuds, PathSkill, DefenseSkill, FieldOfViewDegrees}`. Bus `Npc.Spawned/Died/Action/Heard/Suspicious/Damaged/Reload/Windup/Path/Hitscan/Defense/Tactic`.

`BehaviorTree` (Kits/BehaviorTree) — `Selector(name?, children)` · `ReactiveSelector(name?, children)` (re-evaluates from the top: higher priorities preempt a Running child) · `Sequence(name?, children)` · `Condition(name?, check)` · `Action(name?, run)` (nil return = Success) · `Invert/Succeed(child)` · `Cooldown(seconds, child)` · `tick(root, blackboard, dt, clock?) → "Success"/"Failure"/"Running"` · `reset(root)`. Composites resume `Running` children in place; build one root per agent and tick it from your loop (pass the NPCKit kit clock for deterministic Cooldowns).

`Senses` (ChloeKernelServer/Senses) — `hear(listenerPosition, attunement?, {Position, Range?=40, Loudness?}, {Distance?, Rng?}?) → Vector3?` (anisotropic blur; `Distance` plugs route-measured occlusion in) · `inCone(eye, facing?, fovDegrees, target)` · `canSee(eye, facing?, fovDegrees, target, hasLineOfSight, maxRange?)`. Attunement levels `Senses.Attunement.{Keen,Sharp,Average,Dull,Oblivious}` = `{Range, BlurStuds}`, or any custom table.

```lua
-- WeaponKit: every field is a server-enforced gate
Kernel:registerService(WeaponKit.service({
	Weapons = {
		-- Hitbox sizes the rewound capsule per weapon (defaults 1.6 / 2.4):
		-- widen for forgiving weapons, tighten for precision ones
		[1] = { Name = "Rifle", Damage = 34, FireRate = 8, MaxRange = 300, Hitbox = { Radius = 1.8 } },
	},
	Rewind = Lag, -- claims validate AND hits detect against rewound positions
	Ammo = { -- optional; Get/Take against profile data persists ammo
		Get = function(session) return session.Profile.Data.Ammo end,
		Take = function(session) session.Profile.Data.Ammo -= 1 end,
	},
}))

-- SpellKit: ability casting with resource, cooldown, and channel mechanics
Kernel:registerService(SpellKit.service({
	MaxMana = 150, -- per-session resource pool; regenerates at ManaRegenPerSecond
	Spells = {
		[1] = {
			Name = "ExampleChannelled",
			ManaCost = 30, -- deducted only when validation passes
			Cooldown = 4, -- enforced server-side per session
			ChannelSeconds = 1.2, -- cast runs as a killable Process; taking damage
			-- during the channel interrupts it (Bus: "Spell.Interrupted")
			Cast = function(kernel, session)
				-- effect code; every gate above already passed.
				-- a Cast that moves the player must announce it first:
				-- kernel.Bus:publish("AntiExploit.Forgive", session.Player)
			end,
		},
	},
}))

-- CheckpointKit: stage progression with anti-skip timing
-- expects pads named "1", "2", "3"... under workspace.Checkpoints;
-- MinimumLegitSeconds rejects stage times no honest run can hit
Kernel:registerService(CheckpointKit.service({ MinimumLegitSeconds = 3 }))
```

### Gameplay systems

| Member | Description |
|---|---|
| `RoundKit.service({Phases, Replicas?, TickSeconds?=1})` | Phase: `{Name, Duration?, MinPlayers?}`; service `:advance()`, `:current()`; Bus `Round.PhaseChanged` |
| `Projectiles.attach(kernel, {Rewind?, TickRate?=30}?)` · `:define(id, {Speed, Gravity?, MaxLifetime?=3, Hitbox?, OnHit?, OnExpire?})` · `:fire(session, id, origin, dir, timestamp?) → (ok, reason?)` | Server sim; Bus `Projectile.Hit`. Set `Hitbox = {Radius?, HalfHeight?}` to resolve player hits via capsule hitboxes (same shape/knobs as WeaponKit, honors the Rewind's `ShowHitboxes` display); without it, hits stay raw raycasts against character geometry and `OnHit` gets the engine RaycastResult |
| `ProjectileClient.new({[id] = {Gravity?, Create, OnImpact?}}, {MaxVisuals?, MaxTracers?=300, ThreatRadius?=15}?)` · `.threatens(origin, velocity, position, radius)` | Pooled client visuals, same math as the server; `MaxVisuals` (defaults from the device profile) caps FULL visuals only — overflow renders as minimal pooled tracers, and shots passing within `ThreatRadius` of the local character always render full: no projectile is ever invisible to its target |
| `Effects.attach(kernel)` · `:define(name, {Duration?, MaxStacks?=1, Stats? = {[stat]=multiplier}, Category?, Immune?, TickSeconds?, OnTick?, OnApply?, OnExpire?})` · `:apply(session, name, {Duration?}?) → stacks` (0 = blocked by an immune ward) · `:cleanse(session, category?) → count` · `:remove` · `:has` · `:stacks` · `:statMultiplier(session, stat) → number` | Humanoid stats auto-applied; periodic OnTick with bounded catch-up; owner `CKEffects` state push; Bus `Effects.Applied/Expired/Blocked/Cleansed` |
| `Prediction.wrap(net, name, schema, {Predict?, TimeoutSeconds?=2}) → {fire → seq, OnResolved}` | Pairs with `Net:definePredictedIntent(name, schema, opts)` |
| `Zones.attach(kernel, {IntervalSeconds?=0.25}?)` · `:add(name, part or {parts}, {OnEnter?, OnLeave?}?) → handle` · `:addPart(name, part)` · `:remove(name)` · `:addTagged(tag, {NameAttribute?="ZoneName"}?) → stop` · `:track(entity) → untrack` · `:trackTag(tag) → stop` · `:untrack(entity)` · `:playersIn(name)` · `:entitiesIn(name)` · `:isInside(name, occupant)` | Handle carries `Entered`/`Left` Signals firing `(occupant)`; Bus `Zone.Entered/Left` `(name, player)` for players, `Zone.EntityEntered/EntityLeft` `(name, entity)` for tracked entities (leaves before enters). `addTagged` builds zones from CollectionService tags — same-named parts union into one multi-part zone |
| `Leaderstats.attach(kernel, {Display = {Field, From?, Kind?}}, opts?)` | Values track data on a 1s sweep |
| `Leaderboards.attach(kernel, {Boards = {[name] = {Ascending?, KeepBest?=true, MaxEntries?=100, CacheSeconds?=60}}, FlushSeconds?=15, WritesPerFlush?=30, Backend?, StorePrefix?="CKBoard_", Clock?, SkipLoop?})` · `:submit(board, playerOrKey, value) → bool` · `:top(board, count?) → {{Key, Value, Rank}}` · `:flushNow()` · `:destroy()` | Global top-N on Roblox OrderedDataStores OR custom stores: backends implement declarative `submit(board, key, value, keepBest, ascending)` (external DBs apply keep-best atomically their way) or DataStore-shaped `set(board, key, updater)`, plus `sorted`; queued submits (newest per key per window), per-window write budget, cached reads, failed writes retry; Bus `Leaderboard.Updated` |
| `Fuzz.run(kernel, {Session, Channels?, CasesPerChannel?=32, Seed?=1, IncludeRequests?=true, Driver?, Silent?})` · returns `{Channels, Cases, Passed, Rejected, HandlerErrors = {{Channel, Kind, Args, Error}}}` | Hostile payloads per schema type through the real hook chain + handler pipeline; validator throws count as rejects; deterministic per seed |
| `Pool.new({Create, InitialSize?})` · `:acquire()` · `:release(instance)` · `:idleCount()` · `:trim(keep) → destroyed` · `:destroy()` | `trim` destroys idle instances beyond keep, live instances untouched |
| `TablePool.acquire() → {}` · `TablePool.release(t)` · `TablePool.idleCount()` | Scratch Lua table recycling: release clears, capacity survives, double release fails loud, idle caps at 256; pair with Spatial `radiusInto`/`boxInto`/`coneInto` |
| `Mounts.attach(kernel, {Tag?="CKMount", AudioKit?, Zones?, BonePhysics?, Handlers?}?) → {unmount(instance), destroy()}` | Tagged instances mount by `MountType` attribute: `Sound` (`SoundName`, `Volume?`, `Speed?`) via AudioKit `playAt`, `Zone` (`ZoneName`) via `Zones:addPart`, `BoneChain` (`Damping?`, `Stiffness?`, `GravityScale?`, `WindScale?`) via BonePhysics `bind`; `Handlers` override built-ins and add custom types returning cleanup closures; tag signals mount and unmount live (streaming-safe); bus `Mount.Added/Removed` |
| `ErrorWatch.attach(kernel?, {WindowSeconds?=30}?)` · `:counts()` | Bus `Kernel.ScriptError(message, trace, source, count)` |
| `LiveConfig.new(kernel, {Defaults, PollSeconds?=30})` · `:get(key)` · `:set(key, value)` | Bus `Config.Changed(key, value)`; `set(key, nil)` deletes the override and every server reverts to its default |
| `Pathfinding.attach(kernel?)` · `:find({Method?="Direct", Start, Goal, Grid?, MaxExpansions?, Exclude?, AgentParams?}) → (ok, waypointsOrReason)` · `:distanceField({Grid, Origin, MaxDistance?}) → {[cellKey] = studs}?` · `Pathfinding.follower({Paths, Grid, Method?="ThetaStar", RepathDistance?, ArriveDistance?, Clock?}) → follower` · `follower:step(position, goal) → (nextPoint, {Path?, Jump, Stuck})` · `follower:reset()` · `Pathfinding.grid({Origin, Width, Height, CellSize?=4, AgentHeight?=6, MaxStepHeight?=4, IsWalkable?}) → grid` · `grid:refresh()` · `grid:refreshRegion(minWorld, maxWorld)` · `grid:addLink(fromWorld, toWorld, cost?)` · `grid:groundY(x, z)` · `grid:nearestWalkable(position, maxCells?=8) → Vector3?` · `grid:lineOfSight(x0, z0, x1, z1)` (feet) · `grid:sightLine(x0, z0, x1, z1)` (eyes) | Methods `AStar`/`ThetaStar`/`Direct`/`Roblox`, lazily required; failures return reasons (`GoalBlocked`, `NoPath`, `BudgetExhausted`, ...); `Roblox` yields (the follower runs it off-thread). Default rasterizer raycasts ground (Terrain + CanCollide parts), waypoints ride elevation, `MaxStepHeight` refuses cliffs per step AND along straight lines; walkable high ground occludes sight, valleys only block feet. `refresh` bumps `grid.Version` so followers re-path |
| `Settings.attach(kernel, {Schema, Field?="Settings", RateLimit?=10})` · `:get(session, key)` · `:all(session)` · `:set(session, key, value)` · rule: `{Kind = "number"/"boolean"/"string"/"strings", Default, Min?, Max?, MaxLength?=64, MaxItems?=8}` | Allowlisted client writes: clamp/cap/rebuild through `Intent.Settings_Set`; persists via profile; Bus `Settings.Changed`. Client: `SettingsClient.new(net)` · `:fetch()` · `:get` · `:set` · `:onChanged(fn)` |
| `Analytics.attach(kernel, {FlushSeconds?=10, MaxPerMinute?=120, Sample?, Destination?, WatchErrors?, Seed?}?)` · `:track(name, player?, value?, fields?) → bool` · `:stats via .Stats` · `:destroy()` (flushes) | Batched to `AnalyticsService:LogCustomEvent` or an injected destination; sampled/over-cap drops counted, never silent |
| `BonePhysics.attach({Gravity?, Wind?, FixedHz?=60, MaxSubsteps?=3, MaxDistance?=100, MaxChains?=20, ShouldSimulate?}?)` · `:bind(part, {Damping?=0.92, Stiffness?=0.1, GravityScale?, WindScale?}?) → handle?` · `:bindParts({parts}, settings?)` · `:bindCharacter(model, settings?) → {handles}` · `:step(dt)` · `handle:destroy()` · `:destroy()` | Client-side verlet bone chains (shared module, `ReplicatedStorage.ChloeKernel.BonePhysics`); fixed timestep, culling with pose snap-back, chain budget (past `MaxChains` the nearest to camera win), teleport guard, Transform-slot writes, Destroying auto-unbind; unconfigured `MaxDistance`/`MaxChains` default from the device profile on clients |

| `HairKit.plan(vertices, {Root, Axis?=(0,-1,0), Sectors?=3, Segments?=3, MaxInfluences?=2}) → {Bones, Weights}` · `HairKit.rig(meshPart, {Sectors?, Segments?, MaxInfluences?, Axis?, Root?}?) → (ok, boneNamesOrReason)` · `HairKit.supported()` · `HairKit.attach(kernel, {BonePhysics?, Characters?="All", Rig?, Match?, Settings?, OnRigged?}?)` (client bulk manager) · `HairKit.server(kernel, {Rig?, Match?}?)` (server rig: baked bones replicate once) | Skeleton plan + `AddBone`/`SetVertexBones`/`SetVertexBoneWeights` + `CreateDataModelContentAsync(Content.fromObject)` registration + `CreateMeshPartAsync` bake + in-place `ApplyMesh` + Bone instance skeleton (name-linked); weights cap at `MaxInfluences` (max 4); needs the "Allow Mesh & Image APIs" experience setting (`supported()` probes it) and owned meshes (third-party catalog assets return `(false, reason)`); bus `Hair.Rigged(character, meshPart, boneNames)` |
| `Dissolve.new({MaxPoints?, Parent?, PoolKeep?=64}?) → sim` · `sim:play(target, {DotSize?=0.25, MaxPoints?, Duration?, ScatterSpeed?=14, NoiseScale?=0.08, Drift?=(0,6,0), Reverse?, Spin?, ReturnSpeed?=8, HibernateSeconds?=2, Chunks?, CustomDot?, OnDone?}?) → controller` · `sim:morph(source, destination, {Duration?=1.5, Turbulence?=4, Stagger?=0.35, DotSize?, MaxPoints?, NoiseScale?, Spin?, Chunks?, CustomDot?, Reveal?=true, OnDone?}?) → controller` · `controller:Stop()` · `controller:setProgress(0..1)` when Duration is nil · `sim:destroy()` · pure `Dissolve.noiseVelocity/snap/surfacePoints/gridPoints/sample/pairPoints/morphAlpha` | Vertex-sampled voxels (surface-shell fallback filters Ball/Cylinder to their silhouette, interior cells drop), per-cell dedup, Perlin advection, above-threshold dots ease home and resolidify (restructure), homes ride the target's anchor part so effects follow moving targets, morphs fly dots into the destination shape with bottom-up pairing, staggered launches, turbulence arcs, and color lerp, then reveal the destination; `Chunks` clones source parts as full-detail flying pieces that shrink into the dust (avatar detail without mesh read permission); avatars hide Decals/Textures with their parts; pooled dots via `BulkMoveTo`, one Heartbeat stepper; unconfigured `MaxPoints` scales 500..5000 by DeviceBench quality; local-only VFX |
| `Ragdoll.attach(kernel, {MaxActive?=16, AutoDeath?, CollisionGroup?, FrictionTorque?=60}?)` · `:enable(model, {Duration?, Impulse?}?) → bool` · `:release(model)` · `:isRagdolled(model)` · `:destroy()` — clients: `RagdollClient.listen(netClient)` | Build-once/toggle: Motor6D→limited BallSocket swap or AnimationConstraint cut with native-socket friction/limit tuning, NoCollision limb pairs, full restore; owner-client `CKRagdoll` push flips humanoid state, silences Animate, applies impulses, uprights on release; Bus `Ragdoll.Started/Ended` |
| `AudioKit.attach(kernel, {Buses?, MaxChannels?=32, Parent?, Npcs?, Occlusion = {GetListener?, IsBlocked?, Pathfinding?, Grid?}?}?)` · `:register/registerBank` · `:play(name, opts?) → handle` · `:playAt(name, target, opts?)` · `:playMusic` · `:playLayers → {setWeights, resync, stop}` · `:speak` · `:duck(bus, scale) → release` · `:setBusVolume` · `:bindSettings(prefs, map)` · `:reverbZone(part, type)` · `:soundscape` · `:assetIds()` · `:destroy()`; `AudioKit.server(kernel) → {broadcast, playFor, broadcastAt}` · `audio:listen(netClient)` | Bank config: `{Id, Bus?, Volume?, Speed?, Looped?, Priority?, Pooled?, RollOff*, Cues?, Subtitle?, Duck?}`; handle: `stop(fade?)/setVolume/setSpeed`. Bus events `Audio.Cue/Subtitle/SubtitleEnded` |
| `AnimKit.attach(kernel, {LoadTrack?}?)` · `:register/registerBank` (`{Id, Priority?, Speed?, FadeIn?, Looped?, Group?, Markers?}`) · `:attachRig(model) → rig` · `:assetIds()` · `:destroy()`; rig: `:play(name, {Fade?, Speed?, Weight?})` · `:stop/stopAll/setSpeed` · `:ik(name, {Type = "LookAt"/"Reach"/"Transform"/"Rotation", Target, ChainRoot?, EndEffector?, Weight?, FadeIn?, Properties?}) → {Control, setWeight, setTarget, release}` · `:destroy()` | Exclusive `Group`s crossfade; markers publish `Anim.Marker`; IK weights blend on the kit stepper; rigs die with their model |
| `Preload.run(kernel, {Assets?, Banks?, BatchSize?=16}?) → {Done, Loaded, Total, Failed, await()}` | Batched `PreloadAsync`; Bus `Preload.Started/Progress/Done`; failures collect, never abort |
| `Spatial.new({CellSize?=8}?) → index` · `index:set(id, position)` · `:remove(id)` · `:position(id)` · `:count()` · `:clear()` · `:radius(position, r) → {id}` · `:box(min, max)` · `:cone(origin, direction, range, fovDegrees)` · `:radiusInto/boxInto/coneInto(..., out) → out` (append, zero-alloc) · `:nearest(position, maxRadius?) → (id?, distance?)` · `Spatial.characters(kernel, {CellSize?, Hz?=10}?) → index` (`index:destroy()`) | Flat spatial hash, packed integer cell keys, exact-filtered queries, coordinates clamp at +-524k studs; same-cell set() is a position write; consumers: `Projectiles.attach {Spatial}` per-projectile candidate radius, `NPCKit.attach {Spatial}` update()-refreshed broad-phase for target scans |
| `NetGovernor.attach(kernel, {PingEvery?=1, ProbeHz?=4, Window?=10, LossWindow?=40, Thresholds?, Divisors?={1,2,4}, DeviceStrainQuality?=0.5, PingProvider?, Net?, Clock?, SkipLoops?}?)` · `:tierOf(player)` · `:statsOf(player)` · `:divisorFor(player)` · `:shouldShed(player, priority)` · `:detach()` · pure `NetGovernor.evaluate(stats, thresholds?)` / `pingStats(samples)` — client: `GovernorClient.attach({ReportDevice?=true}?)` | Ping median + jitter (mean absolute deviation) + probe-echo loss + DeviceBench hint grade Good/Strained/Poor (worst axis wins, device floor caps at Strained); probes hold for the NG_Ready handshake; ReplicaService banks skipped deltas per subscriber (owed-merge, staggered phases), unreliable channels shed `Priority = "Low"` to Poor links; `defineUnreliableState(name, schema, {Priority?})`; bus `Net.TierChanged(player, tier, stats)` |
| `Forensics.attach(kernel, {RecordHz?=10, WindowSeconds?=20, TailSeconds?=5, Topics?={"AntiExploit.MovementViolation"}, Destination?, MaxCaptures?=8, Clock?, Sampler?, Http?, SkipLoop?}?)` · `:captures()` · `:flag(player, reason, ...)` · `:export(capture) → json` · `:detach()` · `Forensics.replay(capture, {Parent?, TimeScale?=1, Loop?}?) → {Stop}` · pure `Forensics.frameAt(frames, t)` | Kinematic ring per player (frames `[t, position, look, velocity, humanoidState]`), flag freezes the window + records the tail, re-flags append reasons, leavers finalize; destinations: function or `{ Url }` JSON POST, failures fall into the in-memory ring; replay = interpolated ghost rig; bus `Forensics.Captured(player, capture)` |
| `DeviceBench.run({BudgetMs?=90, Frames?=0, Force?, Clock?, Bus?}?) → {Score, Quality, Axes{Compute, Churn, Resume}, Tier, ComputePerSecond, ChurnPerSecond, ResumePerSecond, Platform, FrameMs?}` · `.profile(resultOrQualityOrTier?) → Profile` · `.quality(result?) → number` · `.axes(measures) → Axes` · `.scale(min, max, result?)` · `.pick(byTier, result?)` · `.tier(result?)` · `.score(measures) → (score, tier)` · `.governor({TargetFps?=60, Result?, Manual?, Clock?, Bus?, OnChange?}?) → {Quality, Tier, sample(dt), profile(), stop()}` | Time-boxed device benchmark (compute + churn + resumes); Quality and each axis are continuous 0.25..4 multipliers vs a mid-range reference (decimals allowed; tiers Low/Medium/High/Ultra remain coarse conveniences); Profile budgets `{ViewDistance, VfxDensity, ParticleBudget, ShadowsEnabled, PostFx, BoneChains, BoneDistance, AudioChannels, ProjectileVisuals}` are funded by the axis that pays each cost (Churn → particles/vfx, Compute → bones/projectile visuals, Resume → channels) and consumed by unconfigured BonePhysics `MaxDistance`/`MaxChains`, AudioKit `MaxChannels`, ProjectileClient `MaxVisuals`; `scale` is log2-mapped (each quality doubling buys the same slice); governor holds TargetFps by nudging effective quality ×0.8 after ~3s of p95 misses / ×1.15 after ~10s of headroom, capped at the benched quality, axis ratios preserved (Bus `Device.QualityChanged(quality, profile)`). Client measurement — cosmetics, never authority |

### TestKit / Bench

`TestKit.runSpecs(container) → Results` over `*.spec` modules returning `function(T)`. `Bench.run(name, fn, {DurationSeconds?}?) → {OpsPerSecond, P50Nanos, P99Nanos}` and `Bench.runAll(container)` over `*.bench` modules.

```lua
-- Inventory.spec (ModuleScript)
return function(T)
	T.describe("Inventory", function()
		T.it("stacks identical items", function()
			local Inv = Inventory.new()
			Inv:add(7, 1)
			Inv:add(7, 2)
			T.expect(Inv:count(7)).toBe(3)
		end)
	end)
end
```
