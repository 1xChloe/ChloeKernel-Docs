# ChloeKernel

> Documentation for [1xChloe/ChloeKernel](https://github.com/1xChloe/ChloeKernel) — the framework source, changelog, and releases live there. This repo carries the docs only; framework access is private, granted individually by Chloe.

Roblox game framework with an OS-style architecture: a genre-agnostic kernel (scheduler, processes, services, IPC, hooks) plus server-side drivers for networking, persistence, replication, anti-exploit, and commerce. Game code registers as services and communicates through kernel primitives; the kernel contains no game rules.

- Priority-scheduled work under a frame budget; buffer-packed serialization at every boundary ([benchmarks](#benchmarks)).
- Server-authoritative: clients send intents, never state; validation chains are fail-closed; per-channel rate limits; lag-compensated hit validation; movement monitoring.
- **290 specs** run on every Studio play-test boot. Verification status is documented per feature.

Requirements: [aftman](https://github.com/LPGhatguy/aftman) (pinned rojo/stylua/selene) and the Argon or Rojo Studio plugin. License: see [LICENSE.md](LICENSE.md) — access is **private**, granted individually by Chloe; use in games requires attribution; sharing or redistribution is not permitted.

---

## Quick navigation

**Guides:** [Getting started](#getting-started) · [Core concepts](#core-concepts) · [Architecture & security](#architecture--the-security-model) · [Genre kits](#genre-kits)

**Cookbook — I want to…**

| Networking & state | Data | Combat & security |
|---|---|---|
| [Send a player action to the server](#send-a-player-action-to-the-server) | [Save player data](#save-player-data) | [Catch speedhackers and teleporters](#catch-speedhackers-and-teleporters) |
| [Define channels once for both sides](#define-channels-once-for-both-sides) | [Back up player data](#back-up-player-data) | |
| [Push updates to clients](#push-updates-to-clients) | [Sell a developer product safely](#sell-a-developer-product-safely) | [Validate hits with lag compensation](#validate-hits-with-lag-compensation) |
| [Replicate live state with deltas](#replicate-live-state-with-deltas) | [Trade items between players](#trade-items-between-players) | [Teleport players without tripping the anti-cheat](#teleport-players-without-tripping-the-anti-cheat) |
| [Ask the server a question](#ask-the-server-a-question) | [Share state across servers](#share-state-across-servers) | [Punish network spammers](#punish-network-spammers) |
| [Bridge bus events across the network](#bridge-bus-events-across-the-network) | [Send events across servers](#send-events-across-servers) | |

| Gameplay systems | Infrastructure |
|---|---|
| [Run a round-based match loop](#run-a-round-based-match-loop) | [Log with scopes and sinks](#log-with-scopes-and-sinks) |
| [Fire projectiles with travel time](#fire-projectiles-with-travel-time) | [Capture and deduplicate script errors](#capture-and-deduplicate-script-errors) |
| [Make abilities feel instant](#make-abilities-feel-instant) | [Tune values live without redeploying](#tune-values-live-without-redeploying) |
| [Apply buffs and debuffs](#apply-buffs-and-debuffs) | [Watch kernel health in live servers](#watch-kernel-health-in-live-servers) |
| [React to players entering areas](#react-to-players-entering-areas) | [Write tests](#write-tests) |
| [Show stats on the leaderboard](#show-stats-on-the-leaderboard) | [Benchmark your own systems](#benchmark-your-own-systems) |
| [Run code when a player joins or leaves](#run-code-when-a-player-joins-or-leaves) | [Reuse instances instead of churning them](#reuse-instances-instead-of-churning-them) |
| [Schedule recurring or background work](#schedule-recurring-or-background-work) | [Survive game updates gracefully](#survive-game-updates-gracefully) |
| [Run long-lived behaviors you can pause or kill](#run-long-lived-behaviors-you-can-pause-or-kill) | [Run heavy CPU work in parallel](#run-heavy-cpu-work-in-parallel) |
| [Add keybinds players can remap](#add-keybinds-players-can-remap) | [Queue players into matches](#queue-players-into-matches) |
| [Decouple systems with events](#decouple-systems-with-events) | [Watch everything live with the debug panel](#watch-everything-live-with-the-debug-panel) |
| | [Stress-test the kernel with simulation modes](#stress-test-the-kernel-with-simulation-modes) |

**Reference:** [API reference](#api-reference) · [Benchmarks](#benchmarks) · [Project layout](#project-layout) · [Roadmap](#roadmap)

---

## Getting started

```bash
aftman install     # once: pinned rojo/stylua/selene
argon serve        # then connect the Argon plugin in Studio
```

Boot scripts are pre-wired: pressing Play runs the spec suites and boots both kernels. Game code goes in the `Bootstrap` modules (`src/Server/Bootstrap.luau`, `src/Client/Bootstrap.luau`) — each receives its kernel after boot, before start. See [Run code when a player joins or leaves](#run-code-when-a-player-joins-or-leaves) for the basic shape.

```
[TestKit] PASS — 118/118 tests across 18 specs
[TestKit] PASS — 131/131 tests across 14 specs
[ChloeKernel 0.2.0] Server booted 0 services in 0ms
[ChloeKernel 0.2.0] Client booted 0 services in 0ms
```

## Core concepts

**1. Kernel** — boots once per machine (server and client each run one). Owns the scheduler, hooks, bus, and services. `ServerKernel` adds sessions, networking, and everything privileged.

```lua
-- src/Server/Main.server.luau already does this; shown for orientation
local ServerKernel = require(game:GetService("ServerScriptService").ChloeKernelServer)
local Kernel = ServerKernel.boot()
-- ...Bootstrap registers services here...
Kernel:start()

-- anywhere else on the server, after boot:
local Kernel = ServerKernel.current()
```

**2. Services** — your game's systems, registered before `start()`, booted in dependency order. Every `init` runs before any `start`, so `start` can safely use any dependency. Cycles crash loudly with the full chain (`A -> B -> C -> A`).

```lua
Kernel:registerService({
	Name = "Quests",
	Dependencies = { "Inventory" },
	init = function(self, kernel)
		self.Kernel = kernel
		self.Active = {}
	end,
	start = function(self)
		self.Inventory = self.Kernel:getService("Inventory") -- guaranteed initialized
	end,
	stop = function(self) end, -- reverse boot order on shutdown
})
```

**3. Sessions** — one per player, created on join, destroyed on leave. Anything bound to a session is torn down automatically.

```lua
Kernel:onSession(function(session)
	session.Data.Streak = 0                                      -- per-visit scratch
	session:bind(session.Player.CharacterAdded:Connect(onSpawn)) -- auto-disconnected on leave
	session:bind(function() print("left:", session.Player.Name) end)
end)
```

**4. Hooks** — named validation chains. Handlers mutate a context or cancel by returning `false`. **Fail-closed:** a crashing validator rejects the action. Every network intent passes through one.

```lua
Kernel.Hooks:on("Intent.BuyItem", function(context)
	if context.Session.Profile.Data.Coins < Prices[context.Args[1]] then
		return false -- rejected; the handler never runs
	end
end, 10) -- lower numbers run earlier

local Passed, Context = Kernel.Hooks:fire("Intent.BuyItem", { Session = s, Args = { itemId } })
```

**5. The Bus** — topic pub/sub so systems communicate without importing each other.

```lua
Kernel.Bus:subscribe("Combat.Kill", function(topic, killer, victim) end)
Kernel.Bus:subscribe("Combat.*", function(topic, ...) end) -- whole family
Kernel.Bus:publish("Combat.Kill", killerSession, victimSession)
```

Security model: the client sends intents, never state; everything privileged lives in ServerScriptService where it cannot be decompiled; nothing the client says is trusted. Details in [Architecture & security](#architecture--the-security-model).

---

## Cookbook

Complete, copyable examples per task. `Kernel` is the booted server kernel unless stated otherwise; channel schemas use Packet type names (`"NumberU16"`, `"Vector3F24"`, …) — keep them in one shared module so both sides agree.

Entries that depend on Roblox services Studio can't fully exercise (DataStores, MemoryStores, teleports, real purchases, real latency) carry a **Status** line stating exactly what is proven and what needs a published place. Everything without one has been verified live in Studio play sessions.

### Send a player action to the server

Client requests, server validates through a fail-closed hook chain, server acts. The example exercises every part of the feature: a typed schema, a rate limit, two independent validators, and the handler.

```lua
-- Server (Bootstrap or a service)
local Net = Kernel:net()
Net:defineIntent("UseItem", { "NumberU16" }, { RateLimit = 5 }) -- schema: one u16 arg; ≤5 sends/sec/player

-- Validator 1: argument validation (priority 10 runs first)
Kernel.Hooks:on("Intent.UseItem", function(context)
	if ItemCatalog[context.Args[1]] == nil then
		return false -- unknown id: rejected, the handler never runs
	end
end, 10)

-- Validator 2: state gate — server-side state, never client claims
Kernel.Hooks:on("Intent.UseItem", function(context)
	local LastUse = context.Session.Data.LastUse or 0
	if os.clock() - LastUse < ItemCatalog[context.Args[1]].Cooldown then
		return false
	end
end, 20)

-- Handler: runs only after every validator passed
Net:onIntent("UseItem", function(session, itemId)
	session.Data.LastUse = os.clock()
	applyItem(session, itemId)
end)
```

```lua
-- Client
local Net = NetClient.new()
local UseItem = Net:intent("UseItem", { "NumberU16" })
UseItem.fire(3)
```

Rejected intents cost the client nothing and the server ~150ns through the chain. A crashing validator counts as a rejection (fail-closed). Lower priority numbers run first.

**Fail-closed means absence too, not just errors.** An intent or request that runs a handler with **no validator registered** rejects every payload — a missing validator is not permission. This closes the classic footgun: an unguarded `GrantCoins`/`SellItem` intent where the handler trusts a client-supplied amount. If a channel genuinely needs no authorization (a payload-less ready-up, an idempotent toggle), acknowledge it explicitly with `{ Open = true }`; declaring `Validate` rules counts as a validator. `Net:auditValidators()` lists any handler-bearing channel still missing both — call it after boot for a proactive sweep, and watch for the one-time `[NetDriver] ... fail-closed` warning (and the `Unguarded` verdict in the NET panel) when a channel rejects for this reason.

### Define channels once for both sides

A single shared module is the source of truth for every channel — schemas cannot drift between client and server, and declarative argument rules compile into the fail-closed hook chains:

```lua
-- ReplicatedStorage.Game.Channels (a module your game owns)
local Registry = require(ReplicatedStorage.ChloeKernel.Net.Registry)
return Registry.define({
	UseItem = {
		Kind = "Intent",
		Schema = { "NumberU16" },
		RateLimit = 5,
		Validate = { [1] = { Range = { 1, 999 } } }, -- rejects out-of-range ids before any game code
	},
	KillFeed = { Kind = "State", Schema = { "String", "String" } },
	GetStock = { Kind = "Request", Schema = { "NumberU16" }, Response = { "NumberU8" }, RejectValue = 0 },
	Aim = { Kind = "Unreliable", Schema = { Pitch = "NumberF16", Yaw = "NumberF16" } },
	Dash = { Kind = "PredictedIntent", Schema = {}, Open = true }, -- any client may dash; reconciled, not gated per-arg
})
```

```lua
-- Server (Bootstrap): every channel defined + validators installed in one call
local Net = Registry.server(Kernel, Channels)
Net.UseItem.on(function(session, itemId) ... end)
Net.KillFeed.broadcast(killerName, victimName)
Net.GetStock.handle(function(session, itemId) return Stock[itemId] or 0 end)

-- Client
local Net = Registry.client(Channels)
Net.UseItem.fire(3)
Net.KillFeed.on(function(killer, victim) ... end)
local Stock = Net.GetStock.invoke(3)
local Dash = Net.Dash.predict({ Predict = dashLocally }) -- Prediction.wrap underneath
```

Rules: `Range = {min, max}` (numbers, inclusive), `OneOf = {...}` (allowed values), `MaxLength = n` (strings). They install at priority 5, ahead of game validators, and REJECT — type coercion already clamps values into the wire type; rules express game limits the schema cannot. `Registry.define` shape-checks the table at load, so a typo'd `Kind` fails the boot instead of a random send later.

### Push updates to clients

For fire-and-forget notifications (kill feed, toasts, round events):

```lua
-- Server
Net:defineState("KillFeed", { "String", "String" }) -- killer, victim
Net:broadcastState("KillFeed", killerName, victimName)   -- everyone
Net:sendState("KillFeed", somePlayer, killerName, victimName) -- one player
```

```lua
-- Client
Net:onState("KillFeed", { "String", "String" }, function(killer, victim)
	addFeedLine(`{killer} eliminated {victim}`)
end)
```

For *state that changes over time* (scores, health bars, timers), use a [replica](#replicate-live-state-with-deltas) instead — it handles late joiners and only sends changes.

### Replicate live state with deltas

Server-owned data that clients watch. Late joiners get a snapshot automatically; changes go out as field deltas (one changed `NumberU16` = 4 bytes):

```lua
-- Server
local Replicas = ReplicaService.new(Kernel)
local Match = Replicas:create("Match", {
	Schema = { TimeLeft = "NumberU16", RedScore = "NumberU8", BlueScore = "NumberU8" },
	Data = { TimeLeft = 300, RedScore = 0, BlueScore = 0 },
	TickRate = 4, -- scoreboard doesn't need 60Hz
})
Match:set("RedScore", Match:get("RedScore") + 1)
```

```lua
-- Client
local Replicas = ReplicaClient.new()
Replicas:listen("Match", { TimeLeft = "NumberU16", RedScore = "NumberU8", BlueScore = "NumberU8" }, function(match)
	Scoreboard.update(match.Data)
	match.OnChange:connect(function(key, value)
		Scoreboard.update(match.Data)
	end)
end)
```

To show different players different data (team-only info, proximity), pass `Interest = function(player) return ... end` — players entering interest get the snapshot, leaving gets a remove.

**Slimming the wire.** Per-recipient deltas are already small (the NET window's fan-out totals are `per-message bytes x recipients`, not one buffer); the levers that shrink them further, cheapest first:

- **Pick the narrowest type that fits.** Every field costs exactly its width: `NumberU8` 1B, `NumberU16` 2B, `NumberU24` 3B, `NumberU32` 4B, `Vector3S16` 6B (integer studs), `Vector3F24` 9B, `Vector3F32` 12B, `Boolean1` packs 8 flags into 1 byte. A score that can't exceed 16M in `NumberU24` instead of `NumberU32` saves a byte on every delta forever.
- **Quantize chatty fields.** `Quantize = { Position = 0.1 }` deadbands a field: writes that moved less than the threshold from the *last replicated* value update server data but skip the wire entirely. Drift is bounded by the threshold (slow movement still replicates once it accumulates), so pick the error nobody can see. A 20Hz position wiggling by hundredths of a stud goes from 20 deltas/s to zero.
- **Lower `TickRate`.** Dirty fields coalesce until the replica's own cadence — a 4Hz scoreboard batches everything that changed in 250ms into one delta envelope instead of fifteen.
- **Split replicas by audience and cadence.** One fast small replica (positions, 20Hz, `Interest`-scoped) plus one slow wide one (stats, 2Hz, broadcast) beats a single replica ticking everything at the fastest field's rate.

### Ask the server a question

Request/response with a timeout and a safe rejection value:

```lua
-- Server
Net:defineRequest("GetShopStock", { "NumberU16" }, { "NumberU8" }, {
	RejectValue = 0,
	Handler = function(session, itemId)
		return Stock[itemId] or 0
	end,
})
```

```lua
-- Client
local GetStock = Net:request("GetShopStock", { "NumberU16" }, { "NumberU8" })
local Stock = GetStock.invoke(itemId) -- yields; 0 on rejection or timeout
```

### Save player data

```lua
local PlayerData = DataDriver.new({
	Name = "PlayerData",
	Defaults = { Coins = 0, Level = 1, Inventory = {}, Settings = { Music = true } },
	SchemaVersion = 1,
	Validate = function(data)
		return type(data.Coins) == "number" and data.Coins >= 0
	end,
})
PlayerData:attach(Kernel)
```

`attach` wires the full lifecycle: load on join into `session.Profile`, save on leave, autosave every 60s, flush on shutdown. From any service:

```lua
Kernel:onSession(function(session)
	-- session.Profile.Data is persistent; session.Data is per-visit scratch
	session.Profile.Data.Coins += 10
end)
```

Built-in protections: session locks prevent rejoin-dupes, a failed load **kicks instead of substituting defaults** (defaults overwriting a real record is how data wipes happen), writes are atomic with retries, and unserializable or `Validate`-failing data refuses to save. For schema changes, add `Migrations = { [1] = function(data) ... return data end }` and bump `SchemaVersion`.

Backends: `"DataStore"` (default), `"Memory"` (tests/Studio), `"Http"` for your own database — note the `Http` backend has never been pointed at a real API and its read-modify-write isn't atomic like `UpdateAsync` (the session lock makes collisions rare; an API wanting hard guarantees should add revision checks — see the module header).

For smaller records, add `Codec = Serde.schema({...})` and profile data stores as packed buffers — or `Codec = "Auto"`, which infers a typed schema from `Defaults` (safe wide widths) **with extras enabled**, so fields added later persist adaptively before they are typed. See the [API reference](#datadriver) and [Serde](#serde--base64).

> **Status:** the loss-hardening logic (locks, retries, migrations, fail-safe loads, codecs) is fully spec-verified against the in-memory backend. To hit real DataStores in Studio, enable *Studio Access to API Services* in Game Settings. Two things only a live game can prove: DataStore throttling under real player counts, and the `BindToClose` flush during an actual server shutdown.

### Back up player data

`Config.Backup` enables rotated backups with automatic disaster recovery:

```lua
local PlayerData = DataDriver.new({
	Name = "PlayerData",
	Defaults = { Coins = 0, Inventory = {} },
	Backup = {
		IntervalSeconds = 86400, -- one backup per key per day (the default)
		Keep = 3,                -- rotate the last 3 dailies per key
		Mirror = true,           -- ALSO copy every save to a "live" slot
		-- Backend = "Http", BackendConfig = { BaseUrl = ... }  -- external DB as the backup target
	},
})
```

- **Exactly once, game-wide, no coordination.** Backups are written only by the server that holds the profile's session lock, and the schedule (`BackupAt`) is stored inside the record itself — so a hundred servers can't produce a hundred dailies, and a player hopping servers doesn't reset the clock. A key that isn't loaded anywhere doesn't get re-backed-up because its data isn't changing — the last backup is still true.
- **Rotation, not growth.** `Keep = 3` means each key cycles through 3 slots; storage per player is bounded forever.
- **`Mirror`** additionally copies *every* save to a `live` slot, so the fallback is never older than one autosave — combine both for "fresh copy + daily history".
- **Automatic restore.** If a record exists but its data won't decode or migrate, `load()` restores the newest readable backup, flags `profile.RestoredFromBackup`, and publishes **`Kernel.ProfileRestored`** on the bus (tell the player, ping your webhook, grant make-goods). Restore **never** runs when the profile is locked by another server (corrupt ≠ unlocked) and never when the backend is down — loading without a lock is a dupe vector, so those still fail closed.
- **Ops tooling:** `driver:backupNow(profile)` forces a copy, `driver:listBackups(key)` shows slots + ages, `driver:peekBackup(key, slot?)` reads a backup **without locking anything** — safe while the player is online elsewhere.

The backup target is any backend: a second DataStore (default, named `{Name}_Backups`), or `"Http"` to keep copies entirely off Roblox. Roblox's own point-in-time versioning (`ListVersionsAsync`) still works underneath as a third layer, since the driver only ever writes through `UpdateAsync`.

### Sell a developer product safely

Receipts are acknowledged only after the grant has persisted — grant, persist, *then* acknowledge:

```lua
local Shop = Receipts.attach(Kernel, {
	GetProfile = function(player)
		local Session = Kernel:getSession(player)
		return Session and Session.Profile
	end,
})

Shop:onProduct(1234567890, function(session, receiptInfo)
	session.Profile.Data.Coins += 500
end)
```

Duplicate receipts are detected via a ledger in the profile; a failed save defers the acknowledgement so Roblox retries, and the retry grants exactly once.

> **Status:** the exactly-once grant logic (dedup, failed-save deferral, ledger capping) is spec-verified with fake receipts. Wire a real ProductId and use Studio's test-purchase flow to exercise the full `ProcessReceipt` path — Studio purchases are free and hit this code for real.

### Trade items between players

`Transactions.atomic` is the escrow: mutate deep copies, abort freely, commit only when both profiles save:

```lua
local Result = Transactions.atomic(SellerProfile, BuyerProfile, function(seller, buyer)
	local Index = table.find(seller.Inventory, ItemId)
	if not Index or buyer.Coins < Price then
		return false -- abort: both untouched
	end
	table.remove(seller.Inventory, Index)
	table.insert(buyer.Inventory, ItemId)
	buyer.Coins -= Price
	seller.Coins += Price
	return true
end)

if not Result.Ok then
	warn("trade failed:", Result.Reason)
end
```

Pass a transaction id and identical replays (retried intents, double-clicked accepts) are rejected instead of applied twice:

```lua
Transactions.atomic(SellerProfile, BuyerProfile, mutator, { Id = tradeOfferId })
-- Reason = "DuplicateTransaction" on replay, "Busy" if either profile is
-- already inside another atomic (per-profile locks: concurrent trades over
-- the same inventory cannot interleave — the classic same-server dupe)
```

If the second save fails the first is compensated back. Cross-server trades are rejected by design — that's how dupes are born.

> **Status:** fully spec-verified including the rollback paths (abort, mutator crash, failed second save). Inherits the DataDriver's status above for the underlying saves.

### Run code when a player joins or leaves

```lua
-- src/Server/Bootstrap.luau receives the kernel after boot, before start
return function(kernel)
	kernel:onSession(function(session)
		session.Data.JoinedAt = os.clock()

		session:bind(session.Player.CharacterAdded:Connect(onSpawn)) -- auto-disconnected
		session:bind(function()
			print(session.Player.Name, "played for", os.clock() - session.Data.JoinedAt)
		end)
	end)
end
```

`onSession` also fires for players already in the game, so late-registering systems never miss anyone. Everything passed to `session:bind` — connections, instances, functions — is cleaned up on leave.

### Schedule recurring or background work

```lua
-- Background priority: runs only on spare frame budget
Kernel.Scheduler:every(30, function()
	for _, Session in Kernel.Sessions do
		periodicCheck(Session)
	end
end, Kernel.Priority.Background)

-- one-shot, next frame, high priority
Kernel.Scheduler:schedule(rebuildNavMesh, Kernel.Priority.High)
```

The scheduler drains queues in priority order under a per-frame budget. On the **server** the budget is frame-aware by default: each step spends the slack left under a 16.5ms frame target after the engine's measured share of the frame (`Stats.HeartbeatTimeMs`), so an idle server drains backlog fast and a physics-heavy one backs off automatically — never below a 1.5ms floor. The **client** defaults to a fixed 1.5ms slice (the render thread's cost is invisible to the frame stat, so "fill the frame" would starve rendering). `Background` runs only on spare budget; `Kernel` priority can't be starved. Task errors are isolated and reported on `Scheduler.OnError`.

### Run long-lived behaviors you can pause or kill

A `Process` is a coroutine with a lifecycle (PID, state machine, exit signal) stepped by the scheduler — for any logic that runs across frames:

```lua
local Process = require(ReplicatedStorage.ChloeKernel.Process)

local Proc = Kernel:spawnProcess(function()
	while true do
		stepWork()
		Process.yield() -- resumes on a later scheduler step; never blocks the frame
	end
end, { Name = "Worker" }) -- the name appears in the hot-task profiler as "Process Worker"
Proc:start()

Proc:suspend() -- stops stepping; state is kept
Proc:resume()
Proc:kill("owner released") -- closes the coroutine, fires OnExit
```

`Proc.OnExit` fires with the final state (`Completed`/`Crashed`/`Killed`); crashes carry a traceback in `.Result`.

### Add keybinds players can remap

```lua
-- Client
local Input = InputDriver.new(kernel)
Input:bindAction("Sprint", {
	Keyboard = Enum.KeyCode.LeftShift,
	Gamepad = Enum.KeyCode.ButtonL3,
	Handler = function(state)
		setSprinting(state == Enum.UserInputState.Begin)
	end,
})

-- bind one action to several physical inputs: pass an array per device
Input:bindAction("Cast", { Keyboard = { Enum.KeyCode.E, Enum.KeyCode.F } })

-- settings menu
Input:rebind("Sprint", "Keyboard", Enum.KeyCode.LeftControl)
local Snapshot = Input:getBindings() -- serialize into the player's profile
```

Actions also publish `Input.<Action>` on the client bus, so systems can listen without owning the binding.

### Catch speedhackers and teleporters

```lua
local Monitor = Movement.attach(Kernel) -- defaults: 1.4× speed tolerance, 80-stud teleport

Kernel.Hooks:on("AntiExploit.MovementViolation", function(context)
	-- context: Player, Type ("Speed"|"Teleport"), HorizontalSpeed, Strikes
	if context.Strikes >= 5 then
		context.Player:Kick("Suspicious movement")
	end
end)
```

Detection is the kernel's job; the punishment policy is yours. Rubberbanding kicks in automatically after repeated strikes, and a flagged position never becomes the trusted baseline — exploiters can't launder a teleport through strikes.

### Validate hits with lag compensation

The server keeps a position history ring and validates shots against where targets were at the client's timestamp — high-ping players hit what they aimed at; impossible shots are rejected:

```lua
local Lag = Rewind.attach(Kernel)

-- inside your fire-intent handler:
local Ok, Reason = Lag:validateShot({
	Shooter = session.Player,
	Origin = muzzlePosition,
	Direction = direction,
	Timestamp = clientTimestamp, -- client sends workspace:GetServerTimeNow()
	MaxRange = 300,
})
if not Ok then
	return -- Reason: OriginMismatch, StaleTimestamp, OutOfRange, ...
end
-- server raycasts; the client never reports hits
```

> **Status:** the ring buffer, interpolation, and every reject reason are spec-verified, and the pipeline has run live in Studio play sessions. The caveat is the inverse of usual: solo Studio play has ~0ms latency, so lag compensation passes *trivially* there — the interesting cases (100ms+ players hitting moving targets) only exist on a live server. If you see `OriginMismatch` rejections from honest live players, widen `OriginToleranceStuds` before suspecting them.

### Teleport players without tripping the anti-cheat

Any legit server-initiated move (teleporter pads, blink spells, knockback, cutscenes) should announce itself first:

```lua
Kernel.Bus:publish("AntiExploit.Forgive", player)
character.HumanoidRootPart.CFrame = Destination
```

Without the announcement, the movement monitor treats the server-initiated move as a violation and rubberbands it.

### Punish network spammers

Rate limiting is automatic (per player, per channel, token bucket). What you add is policy:

```lua
Kernel.Bus:subscribe("Net.RateLimited", function(_, player, channelName)
	noteStrike(player, "rate")
end)
Kernel.Bus:subscribe("Net.FloodDetected", function(_, player, bytes)
	-- transport-level flooding (Packet's 8KB/heartbeat cap): heavier hand
	noteStrike(player, "flood")
end)
```

### Share state across servers

MemoryStore with automatic buffer packing — for round state, global events, live leaderboards:

```lua
local GlobalBoss = MemoryDriver.new({
	Name = "GlobalBoss",
	Codec = Serde.schema({ Health = "NumberU32", Phase = "NumberU8" }),
	DefaultTtlSeconds = 300,
})

GlobalBoss:update("current", function(state)
	state = state or { Health = 1000000, Phase = 1 }
	state.Health -= damage
	return state -- return nil to abort the write
end)
```

For leaderboards use `Kind = "SortedMap"` and `getRange()`.

> **Status:** serialization, retries, abort semantics, and the base64 fallback are spec-verified with mock stores. Real MemoryStoreService calls work in Studio with *Studio Access to API Services* enabled (buffer support is feature-detected at runtime, never assumed) — but "cross-server" by definition only demonstrates itself on a published game with multiple servers.

### Queue players into matches

```lua
local Duels = Matchmaking.new(Kernel, "Duels", { TeamSize = 2, PlaceId = ARENA_PLACE_ID })

Duels:enqueue(player)        -- from a lobby pad or UI intent
Duels:dequeue(player)        -- changed their mind

Kernel.Bus:subscribe("Matchmaking.MatchFound", function(_, queueName, members)
	-- members are this server's players in the match; teleport is automatic
end)
```

Any server can assemble a match; members get teleported to a reserved server wherever they are. The queue is shared across all servers (MemoryStore), the match broadcast rides MessagingService, and the queue's read-invisibility window keeps two coordinators from forming the same match twice.

Deployment notes:

- `PlaceId` must be a place **in the same universe** (a `ReserveServer` requirement), booted with its own kernel to receive arrivals.
- **Studio can't teleport**, so the live loop only proves itself on a published place — the orchestration logic is spec-tested with mocks until then.
- A player who queues and then leaves is tombstoned (MemoryStore queues can't remove by value): coordinators on every server filter their reads against the tombstone map, consume reads containing ghosts, and re-queue the live members — a match never reserves a server for fewer real players than `TeamSize`. If the match teleport fails after retries, the stranded members are re-queued and `Matchmaking.TeleportFailed` publishes. Matching is FIFO — for skill brackets, run one queue per bracket (`Matchmaking.new(Kernel, "Ranked_Gold", ...)`).

### Run heavy CPU work in parallel

Parallel Luau with worker readiness, result routing, and timeouts handled by the pool:

```lua
-- PathWorker.luau (ModuleScript): pure function, no shared state
return function(startPos: Vector3, goalPos: Vector3)
	return computePathWaypoints(startPos, goalPos)
end
```

```lua
local Pool = ActorPool.new({ Size = 4, WorkerModule = script.PathWorker })
local Ok, Waypoints = Pool:dispatch(npcPosition, targetPosition) -- yields
```

### Decouple systems with events

```lua
-- the quest system doesn't import the combat system:
Kernel.Bus:subscribe("Combat.Kill", function(_, killerSession, victim)
	advanceQuest(killerSession, "KillEnemies")
end)

-- analytics sees everything in a family without knowing anyone:
Kernel.Bus:subscribe("Combat.*", function(topic, ...)
	track(topic, ...)
end)
```

### Bridge bus events across the network

The Bus's `NetBridge` slot, fulfilled: attach `BusBridge` and marked topics cross the wire as ordinary bus publishes — no channel boilerplate per event family.

```lua
-- Server (Bootstrap)
BusBridge.attach(Kernel, {
	ServerTopics = { "Round.*", "Combat.Kill" }, -- auto-forward to every client
	ClientTopics = { "Emote.Play" },             -- EXACT names clients may publish upward
	RateLimit = 20,                              -- per-player upward budget
})
Kernel.Bus:publishRemote("Server.Announcement", "Double XP weekend") -- any topic, explicit

-- client publishes arrive validated, with the session prepended:
Kernel.Bus:subscribe("Emote.Play", function(_, session, emoteId) ... end)
```

```lua
-- Client (Bootstrap)
BusBridgeClient.attach(kernel)
kernel.Bus:subscribe("Round.*", function(topic, ...) ... end) -- server events, locally
kernel.Bus:publishRemote("Emote.Play", 4)                     -- rides the intent pipeline up
```

Upward publishes pass the standard gates: per-player rate limit, exact-name whitelist (no wildcards up, fail-closed), and the `Intent.BusPublish` hook chain for game validators. Use one mechanism per topic — a topic in `ServerTopics` that is also `publishRemote`d forwards twice.

### Send events across servers

`MessageDriver` completes the infrastructure trio — `MemoryDriver` owns cross-server *state*, `DataDriver` owns *storage*, this owns *events*:

```lua
local Messages = MessageDriver.new() -- or { Codec = Serde.schema({...}) } for packed payloads
Messages:subscribe("LiveOps", function(data, sentAt)
	if data.Event == "BossSpawn" then spawnGlobalBoss(data.BossId) end
end)
Messages:publish("LiveOps", { Event = "BossSpawn", BossId = 3 }) -- every server receives it
```

Payloads are Serde-packed and base64'd; the ~1KB MessagingService ceiling is guarded with a clear error (send a MemoryDriver key instead of a large body). Publishes retry with backoff; decode failures and crashing subscribers are isolated per message.

> **Status:** packing, the size guard, retries, and crash isolation are spec-verified against a fake service. Real MessagingService delivery needs *Studio Access to API Services*, and "cross-server" only proves itself on a published game — same caveat as MemoryDriver.

### Log with scopes and sinks

```lua
local Log = Logger.scope("Shop")
Log.info("purchase", player.Name, itemId)
Log.warn("stock low", itemId)

-- pipe everything to your analytics service without touching call sites:
Logger.addSink(function(entry)
	Analytics:track(entry.Scope, entry.Message, entry.Args)
end)
Logger.setLevel(Logger.Level.Debug) -- in Studio
```

Sinks are crash-isolated — a broken sink can't break the caller.

### Watch kernel health in live servers

```lua
Kernel:enableDiagnostics(5) -- publishes to ReplicatedStorage attributes every 5s

Kernel.Bus:subscribe("Kernel.Overload", function(_, snapshot)
	Log.warn("sustained frame budget overruns", snapshot.Scheduler.TasksDeferred)
end)

print(Kernel:stats()) -- { Scheduler = {...}, ServiceTimings, Sessions, Net }
```

### Write tests

Create `MyThing.spec` ModuleScripts next to what they test (shared specs in `ChloeKernel/Tests`, server-only specs in `ChloeKernelServer/Tests` — those never replicate):

```lua
return function(T)
	local describe, it, expect = T.describe, T.it, T.expect

	describe("Wallet", function()
		it("rejects negative deposits", function()
			expect(function()
				Wallet.new():deposit(-5)
			end).toThrow("negative")
		end)
	end)
end
```

They run on every Studio play-test boot. Matchers: `toBe, toEqual` (deep), `toBeTruthy/Falsy/Nil`, `toBeType`, `toBeCloseTo`, `toBeGreaterThan/LessThan`, `toContain`, `toHaveLength`, `toThrow(pattern?)`, all negatable with `.never`.

### Benchmark your own systems

```lua
-- MySystem.bench ModuleScript in ChloeKernel/Benches
return function(bench)
	bench("Inventory:add", function(i)
		Inventory:add(i % 50, 1)
	end)
end
-- run on demand:
Bench.runAll(ReplicatedStorage.ChloeKernel.Benches) -- prints ops/sec + p50/p99
```

### Survive game updates gracefully

```lua
SoftShutdown.attach(Kernel)
```

When Roblox shuts the server down for an update, players ride a reserved transit server back into a fresh public server instead of being dumped to the app. Combined with the DataDriver's shutdown flush: data saves, players come back, the update deploys.

> **Status:** this one is *entirely* a production feature — it deliberately no-ops in Studio (there's nothing to migrate), and Studio can't teleport anyway. The flow is the standard, widely-deployed transit-server pattern, but treat your first real update deploy as its acceptance test: shut down one server from the dashboard and watch players resurface.

### Run a round-based match loop

The lobby → round → results cycle as a declarative phase list. Durations tick down, `MinPlayers` gates hold, and clients follow via a replica:

```lua
local Replicas = ReplicaService.new(Kernel)
Kernel:registerService(RoundKit.service({
	Replicas = Replicas,
	Phases = {
		{ Name = "Lobby", MinPlayers = 2 },
		{ Name = "Intermission", Duration = 10 },
		{ Name = "Round", Duration = 180 },
		{ Name = "Results", Duration = 8 },
	},
}))

Kernel.Bus:subscribe("Round.PhaseChanged", function(_, name)
	if name == "Round" then teleportToArena() end
	if name == "Results" then crownWinner() end
end)
-- clients: ReplicaClient listen("RoundState", { Phase = "String", TimeLeft = "NumberU16" }, ...)
```

End a round early with `Kernel:getService("RoundKit"):advance()`.

### Fire projectiles with travel time

The server simulates every trajectory (gravity, swept raycasts, lifetime); clients render pooled visuals running the same math:

```lua
-- Server
local PJ = Projectiles.attach(Kernel, { Rewind = Lag })
PJ:define(1, {
	Speed = 90, Gravity = 40, MaxLifetime = 4,
	OnHit = function(ownerSession, result, victimPlayer)
		if victimPlayer then dealDamage(victimPlayer, 25) end
	end,
})
-- inside your throw-intent handler:
PJ:fire(session, 1, muzzle, direction, timestamp) -- lag-comp validated
```

```lua
-- Client: visuals only, pooled
ProjectileClient.new({
	[1] = {
		Gravity = 40, -- match the server definition
		Create = function() return makeProjectilePart() end,
		OnImpact = function(position) playImpactVfx(position) end,
	},
})
```

Keep ids and physics params in one shared module so server sim and client visuals can't drift.

### Make abilities feel instant

Optimistic prediction: the client acts immediately, the server validates, rejections roll back. No round-trip lag on button presses, no trust in the client:

```lua
-- Server: a predicted intent acks every sequence number
Net:definePredictedIntent("Dash", {}, {
	Handler = function(session) session.Data.LastDash = os.clock() end,
})
Kernel.Hooks:on("Intent.Dash", function(context)
	return os.clock() - (context.Session.Data.LastDash or 0) >= 2 -- cooldown
end)
```

```lua
-- Client
local Dash = Prediction.wrap(Net, "Dash", {}, {
	Predict = function()
		local Before = Root.CFrame
		Root.CFrame += Root.CFrame.LookVector * 12 -- instantly
		return function() Root.CFrame = Before end -- rollback if the server says no
	end,
})
Dash.fire()
```

Predict only local/cosmetic state. No ack within the timeout (dropped packet, rate-limited spam) also rolls back.

### Apply buffs and debuffs

```lua
local Buffs = Effects.attach(Kernel)
Buffs:define("Haste", { Duration = 5, WalkSpeed = 1.5 })
Buffs:define("Chill", { Duration = 3, MaxStacks = 3, WalkSpeed = 0.85 }) -- stacks multiply
Buffs:define("Poison", {
	Duration = 6,
	OnApply = function(session) startPoisonTick(session) end,
	OnExpire = function(session) stopPoisonTick(session) end,
})

Buffs:apply(session, "Haste")
if Buffs:has(session, "Chill") then ... end
```

WalkSpeed recomputes from the humanoid's base across all active effects, so overlapping buffs and removals always land on the right value — and the movement anti-exploit reads live WalkSpeed, so hasted players are never false-flagged.

### React to players entering areas

```lua
local Regions = Zones.attach(Kernel)
Regions:add("SafeZone", workspace.SafeZonePart) -- any invisible part as the volume

Kernel.Bus:subscribe("Zone.Entered", function(_, name, player)
	if name == "SafeZone" then Buffs:apply(Kernel:getSession(player), "Protected") end
end)
Kernel.Bus:subscribe("Zone.Left", function(_, name, player) ... end)
```

Volume queries on a cadence, not `Touched` — slow walks can't slip through, ragdoll limbs don't double-fire, and leaves always fire before enters on a swap.

### Show stats on the leaderboard

```lua
Leaderstats.attach(Kernel, {
	Coins = { Field = "Coins" },                            -- from session.Profile.Data
	Stage = { Field = "BestStage" },
	Title = { Field = "Title", Kind = "StringValue", From = "Session" },
})
```

That's the whole integration — values track the data automatically.

### Capture and deduplicate script errors

```lua
local Watcher = ErrorWatch.attach(Kernel)

Kernel.Bus:subscribe("Kernel.ScriptError", function(_, message, trace, source, count)
	Analytics:track("script_error", { message = message, source = source, count = count })
end)
```

Identical errors log once per window with an occurrence count instead of flooding — a crash loop becomes one line with `(x4000)`, not four thousand lines.

### Tune values live without redeploying

```lua
local Config = LiveConfig.new(Kernel, {
	Defaults = { BossHealth = 100000, DoubleXP = false, NewShopEnabled = true },
})

local Health = Config:get("BossHealth")
Kernel.Bus:subscribe("Config.Changed", function(_, key, value)
	if key == "DoubleXP" then announceEvent(value) end
end)

-- from any server (or the dev console) — reaches the whole fleet within a poll:
Config:set("DoubleXP", true)
-- mid-incident kill switch:
Config:set("NewShopEnabled", false)
```

> **Status:** logic spec-verified with mocks; backed by MemoryStore, so the same notes as [sharing state across servers](#share-state-across-servers) apply.

### Reuse instances instead of churning them

```lua
local Casings = Pool.new({
	Create = function() return makeShellCasing() end,
	InitialSize = 16,
})
local Casing = Casings:acquire()
Casing.Parent = workspace
task.delay(2, function() Casings:release(Casing) end)
```

`Instance.new`/`Destroy` churn fragments memory and stutters frames; pools turn both into table operations. ProjectileClient uses one internally.

### Watch everything live with the debug panel

> **Try it live:** [ChloKernel Test](https://www.roblox.com/games/81098553592999/ChloKernel-Test) is the framework's public test place with the debug panel enabled for every joiner — press **F8** in-game to open the CLIENT / SERVER / NET windows on a real server.

An immediate-mode (ImGui-style) debug overlay, split into **three windows — CLIENT, SERVER, and NET** — so each machine's stats read separately and the wire has its own view. Client: FPS, ping, network kbps, physics rate, memory breakdown, scheduler health with hot-task profiling, kernel internals, live log tail. Server (mirrored via diagnostics attributes): sessions, services, scheduler health, hot tasks, net counters. NET: live kernel channel traffic from both machines. All windows drag independently and tooltip to their own right side. **Studio-only by design** — it auto-attaches in Studio and refuses to exist in production unless ReplicatedStorage has the `ChloeKernelDebug` attribute explicitly set to `true`. On a live server, also set `ChloeKernelDebugUserIds` (comma-separated UserIds) to restrict the whole surface to those users: the panel only attaches for them, the NET/log mirrors fire per-recipient (never broadcast — they carry every player's decoded traffic and the full server log), and the simulation controls reject everyone else. **F8** toggles everything (F9 is Roblox's own dev console); title bars drag (dragging never collapses — only a clean click does).

**Reading the panel:**

- **Colors are health grades** — <span>green</span> = good, orange = worth watching, red = act now. Every graded stat has thresholds tuned for its meaning (e.g. `Deferred` is green only at 0; memory bands account for Studio's editor overhead).
- **The Frame sections decompose the whole frame.** Both windows split measured frame compute into *engine + other scripts* (heartbeat phase minus the kernel), *physics*, the *kernel share*, and (client) *render CPU* — when a frame is slow, the red row names WHOSE problem it is before you reach for a profiler. The server budget bar then shows the slack that composition leaves the kernel (frame-aware budgeting).
- **Bars show consumed vs permitted.** The *frame budget* bar in both scheduler sections reads `used / budget ms (%)` — at 100% the step stops and remaining tasks defer to the next frame, so the bar tells you how close the kernel is to deferring *before* `Deferred` goes nonzero. The server's *Total* memory bar reads against Roblox's 6.25GB hard cap (the server crashes at the cap). Hover any bar for what the limit is and what happens at it.
- **The NET window is a live wire inspector.** Every kernel channel message from BOTH machines (the server's entries arrive on a debug-only mirror): direction arrows (↑ client→server, ↓ server→client), channel, decoded arguments, payload bytes computed from the schema, processing time server-side / round-trip time for requests, and the verdict — **Rejected** (validator said no), **RateLimited** (token bucket), **Dropped** (Packet's 8KB/heartbeat flood cap or the ~900-byte unreliable limit). **Pause** freezes the tail for inspection, **Clear** resets it, and a checkbox hides replica noise. **Click any row to pin it in the Inspector**, which shows the message both ways: the *decoded* argument values and the *serialized* payload as hex — the exact bytes the Packet type engine writes for those values. Totals up top: messages/s and KB/s each direction plus running verdict counters.
- **The Engine section works where memory tracking does not.** Roblox disables engine memory tracking in ALL production builds and its switch (`Stats.MemoryTrackingEnabled`) is read-only — no script, setting, or flag re-enables it, so total/category memory exists only in Studio. The SERVER window compensates with ungated engine counters published by diagnostics: *replication* and *physics repl* bandwidth (the suspect when frame compute reads green but the server still lags — replication CPU has no public timing stat), *primitives* (total and awake), and *contacts*. Lua heap, GC, and Instance rows work everywhere.
- **Hover any stat for a diagnostic tooltip** — not just what the number means, but *where to look when it's wrong*. Hover `Lua heap` and it tells you the usual leak suspects (insert-only arrays, undisconnected connections, per-player tables not cleared — and that `session:bind()` exists for exactly this); hover `Instances` and it points at unpooled projectiles/VFX and respawn clones; hover `Queues`/`Recurring`/`Processes`/`Hook points` and each names its own leak pattern.
- **Hot tasks name the exact offender.** Both scheduler sections list the heaviest task origins (ms/s) by their defining `script:line` — the kernel profiles every scheduled function and attributes the time, so "something is lagging" becomes "`Systems.Combat:131` is eating 4.2ms/s, open it." Processes show as **`Process <name>`** (their work runs inside their own coroutine, so the time belongs to the process — not to the kernel's resume line). Hover an entry for the full path and call count. (`Scheduler:topTasks(n)` / `:resetProfile()` expose the same data; `Scheduler.setTaskOrigin(fn, label)` names your own functions.)
- **GC rows tell leaks and churn apart.** Roblox exposes no GC pause timings, so the panel shows the signals that drive GC cost instead (via `GcWatch`): **GC alloc** is the Lua allocation rate — the lever; cut allocation and you cut GC time — and **GC cycles** shows sweep cadence and reclaim sizes. The diagnostic combo: heap *climbing* + reclaims near zero = a leak (the GC has nothing to free); heap *stable* + busy cycles = garbage churn (tables/strings created per frame — pool or hoist them).
- **Leak-hunting workflow:** watch the Memory section while playing — when a number ratchets up and never comes back, hover it, and do the thing the tooltip says. The *Dump kernel stats* button drops snapshots into the log tail (hover a tail row for the full untruncated line) AND prints to the output console, where the text can be selected and copied.

```lua
-- already wired in Main.client.luau; server side feeds it with:
Kernel:enableDiagnostics(1)
```

Build your own debug windows with the same library:

```lua
local ImGui = require(ReplicatedStorage.ChloeKernel.Debug.ImGui)
local Window = ImGui.window("Combat", { Parent = screenGui })
-- re-render as often as you like; instances are reused, not churned
Window:render(function(ui)
	ui:header("Weapons")
	ui:value("Active projectiles", count)
	ui:sparkline("damage/s", DamageHistory, 500)
	if ui:button("Give rifle") then ... end
	DebugAimLines = ui:checkbox("Draw aim lines", DebugAimLines)
end)
```

### Stress-test the kernel with simulation modes

A built-in load harness (`Simulation.server.luau` + `ChloeKernelServer/Simulation`) drives **real kernel codepaths** at game-shaped intensities, so the debug panel and hot-task profiler show exactly where things strain *before* your game does. Set one attribute and press Play:

```lua
-- in the command bar (or the Properties panel on ReplicatedStorage):
game.ReplicatedStorage:SetAttribute("ChloeKernelSimulation", "HighIntensity")
```

| Mode | Shape | What it exercises |
|---|---|---|
| `HighIntensity` | busy shooter-scale server | 200 entity Processes, 30 projectile arcs/s, 60 effect ops/s, 500 bus msgs/s, 300 hook chains/s, 8 replicas × 30 viewers with delta encoding, Serde round trips, pooled part churn, and a deliberate 20ms/s CPU burn. A modern server clears all of it in ~1-2ms/frame — HighIntensity reads GREEN under frame-aware budgeting, and that is the point; use Overload to see the red states |
| `SmallGame` | social/obby-scale baseline | the same rigs at ~1/10 intensity, no burn — compare panel readings against HighIntensity |
| `Overload` | deliberate torture test | ~20ms of scheduled burn demand per frame (8 tasks, every step, Low/Background mix) plus 500 entities — exceeds the whole frame target so the budget pegs, `Deferred` goes nonzero, and the red panel states show. Not a model of anything real |
| `Showcase` | narrated feature tour | cycles 6 acts every 8s: orbiting Processes, **real haste/slow effects on present players**, projectile fans, replica delta sizes, anti-dupe trade replay rejection, Serde-vs-JSON byte counts — narration lands in the console and the panel log tail |

- **Change the attribute mid-game to hot-swap modes** — the old mode tears down completely (processes killed, recurring tasks cancelled, parts destroyed) before the new one starts. An empty string stops simulation.
- **Or click instead:** the **SIM CONTROL** window (`Debug/SimPanel`, wired through this place's client Bootstrap) switches server modes live via a Studio-only remote AND runs **client-side loads** on your machine (`Debug/SimLoad` — the same rigs built on shared primitives only), so you can stress both sides at once and watch the CLIENT/SERVER panel windows react. Server and client loads start/stop independently.
- **Studio-only** unless `ChloeKernelDebug` is flagged, same rule as the debug panel — and the SimControl intent additionally honors the `ChloeKernelDebugUserIds` allowlist, so flagging a live server does not hand load controls to every player.
- Open the panel (**F8**) while it runs: hot tasks name the simulation's own origins (`Simulation:375` for the burn, `Process SimBolt` / `Process SimEntity` for the rigs), Deferred/queue grading reacts to the burn, and the 10s heartbeat logs achieved rates plus GC cost (`hooks/s=300 bus/s=500 … gcAlloc=3800KB/s gcCycles=252/min gcReclaim=900KB`) so you can confirm the load is real — and see what it costs the collector.
- Load modes use **fake packet factories on purpose** — simulated channels never touch the real wire, so a connected client's Packet stream can't be corrupted by simulation traffic. Everything else (scheduler, processes, effects sweeps, zone diffs, replica interest scans + struct deltas, hook chains, bus fan-out, pools) is the genuine subsystem under genuine load.

> **Status:** all three modes verified in live play tests — HighIntensity sustained ~17k tasks/sec at 1.3ms steps with zero deferrals, hot-task attribution named the burn line exactly, hot-swapping between all modes left no residue (workspace folder, processes, and recurring tasks all confirmed gone), and the Showcase's haste/slow act was verified against a real player's Humanoid (16 → 25.6 → 8 → 16 WalkSpeed).

---

## Genre kits

Pre-built userspace services in `ChloeKernelServer/Kits` — register, configure, extend through hooks. Each config field maps to a kit-enforced feature:

```lua
Kernel:registerService(WeaponKit.service({
	Weapons = {
		-- per-id config; the kit enforces FireRate server-side and rejects
		-- shots past MaxRange before any raycast happens
		[1] = { Name = "Rifle", Damage = 34, FireRate = 8, MaxRange = 300 },
	},
	Rewind = Lag, -- optional: shots validate against rewound positions (lag compensation)
	Ammo = { -- optional gate; point Get/Take at session.Profile.Data to persist ammo
		Get = function(session) return session.Data.Ammo end,
		Take = function(session) session.Data.Ammo -= 1 end,
	},
}))
```

| Kit | Wire channel | Owns | Bus events |
|---|---|---|---|
| **WeaponKit** | `WK_Fire(weaponId, dir, timestamp)` | fire rate + ammo gates, lag-comp validation, rewound-hitbox hit detection (per-weapon capsule sizes), damage | `Weapon.Fired/Hit/ShotRejected` |
| **SpellKit** | `SK_Cast(spellId)` | mana + regen, cooldowns, channelled casts as killable Processes, damage interrupts | `Spell.Cast/Interrupted` |
| **CheckpointKit** | server-side `Touched` | stage sequencing, minimum-time validation, profile persistence | `Obby.Checkpoint` |
| **RoundKit** | — (replica out) | phase cycle, duration timers, MinPlayers gates, countdown replication | `Round.PhaseChanged` |
| **ProceduralKit** | — (wire yours through `service:validate`) | runtime ProceduralModel spawning, fail-closed parameter validation (type/Min/Max/OneOf), per-model regeneration rate limiting with batched flushes, session-bound lifecycle | `Procedural.Spawned/Updated/Rejected/Removed` |

All kit validation lives in the same fail-closed hook chains (`Intent.WK_Fire`, `Intent.SK_Cast`, `Intent.Checkpoint`) — prepend your own rules at lower priority numbers without forking the kit.

> **ProceduralKit** rides Roblox's ProceduralModels (Studio beta; works in live games when the beta ships): parameter-driven Models whose server-side Generator ModuleScript rebuilds their geometry when Size or an attribute changes. The peer that changes a parameter generates — server writes generate server-side and replicate the results, so generators stay in ServerScriptService where exploiters can't decompile them. The kit validates and clamps every parameter (unknown names and wrong types reject outright), buffers writes behind a per-model cooldown so a spammed customization intent costs one rebuild per window instead of one per message, and ships an ExampleGenerator demonstrating the two generator survival habits: `params:Pause()` in loops (the engine kills generators that work too long without yielding) and deriving ALL randomness from a Seed attribute (every rebuild reruns the generator — unseeded randomness reshuffles the model on every parameter change).

> **Status:** WeaponKit and SpellKit have run live in Studio play sessions (fire, cast, interrupt) on top of spec-verified validators. CheckpointKit's validators are spec-verified, but it needs a `workspace.Checkpoints` folder with numbered pads to wire up — its `Touched` path warns at boot when the folder is missing. WeaponKit damage-on-hit deserves a two-player live test — solo Studio has nothing to shoot.

## Architecture & the security model

```
ReplicatedStorage.ChloeKernel        replicates — generic primitives only
ServerScriptService.ChloeKernelServer   NEVER replicates — everything privileged
```

- **The client is a renderer with opinions.** It sends intents, never state. The server validates every intent through fail-closed hook chains and is the only thing that mutates state.
- **Code placement = attack surface.** Validators, rate limits, drop tables, AI logic, data: ServerScriptService. The client holds only what it must execute — input, prediction, VFX.
- **Fail-closed everywhere.** A crashing validator rejects the action. A failed data load kicks rather than risking a wipe. A schema-violating save is refused. An unmigratable record won't load.
- **Your character stays yours.** The kernel never reassigns a character's network ownership and never anchors character parts — the client keeps simulating its own character at all times, so movement always feels local. Anti-exploit corrections are plain CFrame writes (ownership intact), and movement abilities go through [Prediction](#make-abilities-feel-instant): the client moves itself instantly, the server validates, rejections roll back.
- **Explicit non-goals:** client code is always readable by exploiters — the defense is never trusting it, not hiding it. Cross-server trades can't be made safe without a coordinator, so they're refused. Obfuscation is not security.

## Project layout

```
src/
├── Shared/                       → ReplicatedStorage
│   └── ChloeKernel/               microkernel: Scheduler, Process, ServiceManager,
│                                 Logger, GcWatch, Serde, Base64, InputDriver, IPC/, Hooks/,
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
│                                 Simulation/, Matchmaking, SoftShutdown,
│                                 ActorPool/, Tests/
└── Client/
    ├── Main.client.luau          boot → Bootstrap → start
    └── Bootstrap.luau            YOUR client wiring registers here
```

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
| `BusBridge.attach(kernel, {ServerTopics?, ClientTopics?, RateLimit?=20}?)` · `BusBridgeClient.attach(kernel, netClient?)` | Bus topics across the network; upward publishes are whitelisted + rate-limited |
| `MessageDriver.new({Codec?, BackoffSeconds?}?)` · `:publish(topic, data) → (ok, err?)` · `:subscribe(topic, fn(data, sentAt?))` | Cross-server events over MessagingService, Serde-packed, 1KB-guarded |
| `Net:defineUnreliableState(name, structSchema)` · `:sendUnreliableState(name, player, data)` · `:broadcastUnreliableState(name, data)` | Lost > late for high-frequency cosmetic data; ≤900 bytes |
| `NetClient:onUnreliableState(name, structSchema, fn(data))` | |
| `Net.Stats` | `IntentsAccepted/Rejected/RateLimited, RequestsRejected` |
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

`InputDriver.new(kernel)` · `:bindAction(name, {Keyboard?, Gamepad?, TouchButton?, Handler?})` · `:rebind(name, "Keyboard"|"Gamepad", keys?)` · `:unbindAction(name)` · `:getBindings()` · `:destroy()`. Emits `Input.<Action>` on the bus. `Keyboard`/`Gamepad` take one `Enum.KeyCode` or an array of them; snapshots mirror the shape (name or array of names).

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
| `ActorPool.new({Size, WorkerModule, Parent?, TimeoutSeconds?=10})` · `pool:dispatch(...) → (ok, result|err)` (yields) · `:destroy()` | WorkerModule returns a pure `function(...) → result` |

```lua
local Arena = Matchmaking.new(Kernel, "Arena", { TeamSize = 4, PlaceId = ARENA_PLACE })
Arena:enqueue(player)

SoftShutdown.attach(Kernel) -- one line; transit-server migration on updates

local Pool = ActorPool.new({ Size = 4, WorkerModule = script.ChunkWorker })
local Ok, Chunk = Pool:dispatch(chunkX, chunkZ) -- yields until the worker replies
```

### Kits

`WeaponKit.service({Weapons, Rewind?, Ammo?})` · `SpellKit.service({Spells, MaxMana?=100, ManaRegenPerSecond?=5})` · `CheckpointKit.service({FolderName?="Checkpoints", MinimumLegitSeconds?=3, StageField?="BestStage"}?)` · `ProceduralKit.service({Archetypes, RegenSecondsPerModel?=0.25})` — each returns a service definition for `kernel:registerService`. ProceduralKit: `spawn(name, {CFrame?, Size?, Params?, Parent?, Owner?}) -> handle` (`handle:set/patch/resize/regenerate/waitForGeneration/destroy`), `validate(name, params) -> (ok, cleanedOrReason)` for wiring client customization intents.

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
| `ProjectileClient.new({[id] = {Gravity?, Create, OnImpact?}})` | Pooled client visuals, same math as the server |
| `Effects.attach(kernel)` · `:define(name, {Duration?, MaxStacks?=1, Stats? = {[stat]=multiplier}, OnApply?, OnExpire?})` · `:apply(session, name, {Duration?}?) → stacks` · `:remove` · `:has` · `:stacks` · `:statMultiplier(session, stat) → number` | Humanoid stats auto-applied; Bus `Effects.Applied/Expired` |
| `Prediction.wrap(net, name, schema, {Predict?, TimeoutSeconds?=2}) → {fire → seq, OnResolved}` | Pairs with `Net:definePredictedIntent(name, schema, opts)` |
| `Zones.attach(kernel, {IntervalSeconds?=0.25}?)` · `:add(name, part)` · `:remove(name)` · `:playersIn(name)` | Bus `Zone.Entered/Left` (leaves before enters) |
| `Leaderstats.attach(kernel, {Display = {Field, From?, Kind?}}, opts?)` | Values track data on a 1s sweep |
| `Pool.new({Create, InitialSize?})` · `:acquire()` · `:release(instance)` · `:idleCount()` · `:destroy()` | |
| `ErrorWatch.attach(kernel?, {WindowSeconds?=30}?)` · `:counts()` | Bus `Kernel.ScriptError(message, trace, source, count)` |
| `LiveConfig.new(kernel, {Defaults, PollSeconds?=30})` · `:get(key)` · `:set(key, value)` | Bus `Config.Changed(key, value)`; `set(key, nil)` deletes the override and every server reverts to its default |

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

## Benchmarks

Measured via `Bench.runAll` on kernel hot paths:

| Operation | Throughput | p50/op |
|---|---|---|
| TokenBucket:take | 17.0M ops/s | 58ns |
| Hooks:fire (3-validator chain) | 6.6M ops/s | 148ns |
| Bus:publish | 5.0M ops/s | 198ns |
| Signal:fire (1 handler) | 4.5M ops/s | 203ns |
| Scheduler schedule+step (production) | 2.2M ops/s | 354ns |
| Scheduler schedule+step (Studio, MicroProfiler zones) | 2.1M ops/s | 388ns |
| Serde schema encode (3 fixed fields, fast engine) | 3.5M ops/s | 285ns |
| Serde schema decode (3 fixed fields, fast engine) | 2.7M ops/s | 367ns |
| Serde struct delta encode (2 fields) | 3.7M ops/s | 270ns |
| Serde struct delta decode (2 fields) | 3.4M ops/s | 293ns |

The fast engine's same-machine A/B against the wire-engine path: fixed-schema encode 463ns to 285ns (38% faster), decode 514ns to 367ns, dynamic schemas 22-27% faster — identical bytes either way.

The full intent security pipeline (rate limit + session check + 3 validators) costs well under a microsecond per packet. The always-on hot-task profiler (per-task timing attributed to script:line) costs **~33ns per task** — origins are cached by function identity, and MicroProfiler zones run only in Studio unless a live server opts in via `Scheduler.setMicroProfilerZones(true)`.

## Roadmap

- **Live-server soak testing** — SoftShutdown migration, shutdown data flushes, and cross-server matchmaking can only be fully proven on a published place with real players; everything else is verified in Studio.
- **API doc site** — Moonwave comments → generated reference once the surface stabilizes; this README is the reference meanwhile.

## License & contributions

ChloeKernel is **private-access** under its own license ([LICENSE.md](LICENSE.md)) — access is granted individually by Chloe (CryptedChloes on Roblox, 1xChloe on GitHub) and is revocable. The short version, for those granted access:

- **Allowed:** use it in your Roblox games, commercial or not; modify it freely inside your own game's codebase
- **Allowed:** contribute improvements via pull requests to this repository
- **Required:** credit `CryptedChloes` in your game's description — for any game using ChloeKernel
- **Not allowed:** sharing the source with anyone or granting access on Chloe's behalf — only Chloe grants access
- **Not allowed:** redistributing it as your own framework — no forks published as separate projects, free or paid

The wire transport (`ChloeKernel/Net/Packet`) is a kernel-maintained port of Packet v1.7 by Suphi Kaner (5uphi); the original work remains credited to its author (see its `CREDITS.md` for attribution and the marked local fixes). Kernel code style is enforced by StyLua + Selene in CI — run `aftman install` to match locally.
