# ChloeKernel

Roblox game framework with an OS-style architecture: a genre-agnostic kernel (scheduler, processes, services, IPC, hooks) plus server-side drivers for networking, persistence, replication, anti-exploit, and commerce. Game code registers as services and communicates through kernel primitives; the kernel contains no game rules.

- Priority-scheduled work under a frame budget; buffer-packed serialization at every boundary ([benchmarks](#benchmarks)).
- Server-authoritative: clients send intents, never state; validation chains are fail-closed; per-channel rate limits; lag-compensated hit validation; movement monitoring.
- **499 specs** run on every Studio play-test boot. Verification status is documented per feature.

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
| [Bind abilities to movement, not just keys](#bind-abilities-to-movement-not-just-keys) | [Fight hand to hand, parries included](#fight-hand-to-hand-parries-included) |
| [Give players an inventory](#give-players-an-inventory) | [Split players into teams](#split-players-into-teams) |
| [Keep global top-100 boards](#keep-global-top-100-boards) | [Roll loot with pity timers](#roll-loot-with-pity-timers) |
| [Craft items at stations](#craft-items-at-stations) | [Sell things for coins](#sell-things-for-coins) |
| [Summon familiars and pets](#summon-familiars-and-pets) | [Fuzz every channel with hostile payloads](#fuzz-every-channel-with-hostile-payloads) |
| [Apply buffs and debuffs](#apply-buffs-and-debuffs) | [Watch kernel health in live servers](#watch-kernel-health-in-live-servers) |
| [React to players and NPCs entering areas](#react-to-players-and-npcs-entering-areas) | [Write tests](#write-tests) |
| [Show stats on the leaderboard](#show-stats-on-the-leaderboard) | [Benchmark your own systems](#benchmark-your-own-systems) |
| [Run code when a player joins or leaves](#run-code-when-a-player-joins-or-leaves) | [Reuse instances instead of churning them](#reuse-instances-instead-of-churning-them) |
| [Schedule recurring or background work](#schedule-recurring-or-background-work) | [Survive game updates gracefully](#survive-game-updates-gracefully) |
| [Run long-lived behaviors you can pause or kill](#run-long-lived-behaviors-you-can-pause-or-kill) | [Run heavy CPU work in parallel](#run-heavy-cpu-work-in-parallel) |
| [Add keybinds players can remap](#add-keybinds-players-can-remap) | [Queue players into matches](#queue-players-into-matches) |
| [Decouple systems with events](#decouple-systems-with-events) | [Watch everything live with the debug panel](#watch-everything-live-with-the-debug-panel) |
| [Give NPCs brains, aim skill, and movesets](#give-npcs-brains-aim-skill-and-movesets) | [Stress-test the kernel with simulation modes](#stress-test-the-kernel-with-simulation-modes) |
| [Find paths four different ways](#find-paths-four-different-ways) | [Save player settings and keybinds](#save-player-settings-and-keybinds) |
| [Track quests from things that happen](#track-quests-from-things-that-happen) | [Ship gameplay analytics](#ship-gameplay-analytics) |
| [Let players interact with the world](#let-players-interact-with-the-world) | [Audit the security surface in one call](#audit-the-security-surface-in-one-call) |
| [Scope replicas to zones](#scope-replicas-to-zones) | [Swing hair, capes, and tails](#swing-hair-capes-and-tails) |
| [Ragdoll bodies that replicate for free](#ragdoll-bodies-that-replicate-for-free) | [Preload assets behind a loading screen](#preload-assets-behind-a-loading-screen) |
| [Run the whole soundstage](#run-the-whole-soundstage) | [Scale effects to the device](#scale-effects-to-the-device) |
| [Animate rigs and blend inverse kinematics](#animate-rigs-and-blend-inverse-kinematics) | |

**Reference:** [API reference](#api-reference) · [Hook points & bus topics](#built-in-hook-points--bus-topics) · [Benchmarks](#benchmarks) · [Project layout](#project-layout) · [Roadmap](#roadmap)

---

## Getting started

```bash
aftman install     # once: pinned rojo/stylua/selene
argon serve        # then connect the Argon plugin in Studio
```

Boot scripts are pre-wired: pressing Play runs the spec suites and boots both kernels. Game code goes in the `Bootstrap` modules (`src/Server/Bootstrap.luau`, `src/Client/Bootstrap.luau`) — each receives its kernel after boot, before start. See [Run code when a player joins or leaves](#run-code-when-a-player-joins-or-leaves) for the basic shape.

```
[TestKit] PASS — 175/175 tests across 23 specs
[TestKit] PASS — 259/259 tests across 23 specs
[ChloeKernel 0.5.0] Server booted 0 services in 0.01ms
[ChloeKernel 0.5.0] Client booted 0 services in 0.01ms
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

### Give players an inventory

InventoryKit is the item model everything else was missing — a definition registry, slotted per-player bags with stacking, equip slots, and use handlers, persisted through the profile:

```lua
local Inventory = InventoryKit.attach(Kernel, {
	MaxSlots = 30,
	Items = {
		Sword = { Stack = 1, Equip = "Primary", Meta = { Damage = 25 },
			OnEquip = function(session, item) giveTool(session, "Sword") end },
		Potion = { Stack = 10, OnUse = function(session)
			healCharacter(session, 25)
			return true -- consumed: one leaves the stack
		end },
	},
})
Inventory:grant(session, "Potion", 3) -- loot drops, shop purchases, quest rewards
Inventory:take(session, "Potion", 1) -- crafting costs, trade escrow (all-or-nothing)
```

Grants and takes are **server-API-only** — no client message can mint an item. Clients get four fail-closed intents (`INV_Move`/`INV_Equip`/`INV_Unequip`/`INV_Use`) whose validators re-derive legality from the server's books, and the owner receives an `INV_Sync` state push after every mutation — inventory UI is purely a render problem. Stacking tops up piles before opening slots, `move` is the drag-and-drop primitive (swap or merge, equip references follow), equipping displaces the incumbent through its `OnUnequip`, and draining a stack auto-unequips. Hook points `Inventory.CanEquip`/`Inventory.CanUse` (fail-open) carry game rules like tournament locks. `Persist = false` keeps a bag session-only (battle-royale loadouts). Bus: `Inventory.Granted/Taken/Equipped/Unequipped/Used/Changed`.

### Roll loot with pity timers

LootKit rolls weighted tables server-side. Entries yield items, nested tables, or nothing; `PityAfter = N` force-hits an entry on the Nth consecutive miss, with counters persisted per player in the profile:

```lua
local Loot = LootKit.attach(Kernel, {
	Inventory = Inventory,
	Tables = {
		Chest = { Rolls = 2, Entries = {
			{ Weight = 70, Id = "Coin", Count = { 5, 20 } },
			{ Weight = 25, Table = "Herbs" }, -- nested table rolled in place
			{ Weight = 5, Id = "RareRelic", PityAfter = 30 }, -- guaranteed within 31 opens
		} },
		Herbs = { Entries = { { Weight = 1, Id = "Sage" }, { Weight = 1, Id = "Nightshade" } } },
	},
})
local Granted, Overflow = Loot:award(session, "Chest") -- rolls + grants through InventoryKit
```

`roll(name)` rolls with no player (display, previews) and never advances pity. `rollFor(session, name)` rolls against the player's persisted counters. Nesting caps at depth 8 and unknown table references fail at attach. Bus: `Loot.Rolled(tableName, items, session?)`, `Loot.Awarded(session, tableName, granted, overflow)`.

### Craft items at stations

CraftingKit is recipes over InventoryKit — inputs check atomically before any take, timed crafts queue with cancel refunds, and station recipes gate on where the player crafts:

```lua
local Crafting = CraftingKit.attach(Kernel, {
	Inventory = Inventory,
	Recipes = {
		HealthPotion = { Inputs = { Herb = 2, Water = 1 }, Output = { Potion = 1 }, Seconds = 3 },
		IronSword = { Inputs = { Iron = 3 }, Output = { Sword = 1 }, Station = "Forge" },
	},
})
```

Clients ride `CR_Craft {recipeId, station}` and `CR_Cancel` through the fail-closed chain; the claimed station verifies in the `Craft.CanCraft` hook (check proximity to the tagged forge there — InteractionKit's zone or a distance check). One active craft per session; `canCraft` returns reasons (`MissingInputs`, `Busy`, `WrongStation`, `Vetoed`). Output that cannot fit publishes `Craft.Overflow`. Bus: `Craft.Started/Completed/Cancelled/Overflow`.

### Sell things for coins

ShopKit is soft-currency shops over InventoryKit (Commerce owns Robux; this owns coins). The default currency adapter reads and writes a profile field; custom adapters plug in for anything else:

```lua
local Shop = ShopKit.attach(Kernel, {
	Inventory = Inventory,
	CurrencyField = "Coins",
	Shops = {
		Apothecary = {
			Listings = { Herb = { Price = 5, SellPrice = 2 }, Cauldron = { Price = 250 } },
			Rotation = { Every = 3600, Size = 3, Pool = { "RareSeed", "Moonstone", "Vial", "Charm" }, Seed = 7 },
		},
	},
})
```

Rotation picks derive from `Seed + floor(clock / Every)` — every server lists the same rotating stock with zero coordination. `Stock = N` caps buys per player per window. Buys grant first and charge for what fit (a partial grant charges the partial price; a full inventory charges nothing). Clients ride `SH_Buy`/`SH_Sell` fail-closed with counts capped at 99. Bus: `Shop.Bought/Sold/Rejected`.

### Summon familiars and pets

CompanionKit gives sessions an owned NPC follower built from NPCKit archetypes — the witchcraft familiar, the lobby pet:

```lua
local Companions = CompanionKit.attach(Kernel, {
	Npcs = Npcs, Effects = Buffs,
	Companions = {
		BlackCat = { Archetype = "Cat", FollowDistance = 8, TeleportDistance = 60, Aura = "CatLuck" },
	},
})
Companions:summon(session, "BlackCat") -- or wire it to an InventoryKit item's OnUse
```

One companion per session (a new summon replaces the old). The follow loop walks the companion at `FollowDistance`, pivots it beside the owner past `TeleportDistance`, and a dead companion clears with `Companion.Died`. `Aura` holds an Effects buff on the OWNER while summoned and removes it on dismiss or death. Gate ownership in the `Companion.CanSummon` hook. Clients ride `CP_Summon`/`CP_Dismiss` fail-closed. Bus: `Companion.Summoned/Dismissed/Died`.

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
Kernel.Bus:subscribe("Net.FloodDetected", function(_, player, bytes, payloadBytes, budgetBytes)
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
	ClientSchemas = { ["Emote.Play"] = { "NumberU8" } },
	RateLimit = 20,                              -- per-player upward budget
})
Kernel.Bus:publishRemote("Server.Announcement", "Double XP weekend") -- any topic, explicit

-- client publishes arrive validated, with the session prepended:
Kernel.Bus:subscribe("Emote.Play", function(_, session, emoteId) ... end)
```

```lua
-- Client (Bootstrap)
BusBridgeClient.attach(kernel, {
	ClientSchemas = { ["Emote.Play"] = { "NumberU8" } },
})
kernel.Bus:subscribe("Round.*", function(topic, ...) ... end) -- server events, locally
kernel.Bus:publishRemote("Emote.Play", 4)                     -- rides the intent pipeline up
```

Upward publishes pass the standard gates: per-player rate limit, exact-name whitelist (no wildcards up, fail-closed), typed schemas per topic, and the `Intent.BusPublish.<Topic>` hook chain for game validators. Use one mechanism per topic — a topic in `ServerTopics` that is also `publishRemote`d forwards twice.

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

### Split players into teams

TeamKit is one source of truth wired into every combat gate — no kit needs to know teams exist:

```lua
local Teams = TeamKit.attach(Kernel, {
	Teams = {
		Raiders = { Color = Color3.fromRGB(200, 60, 60), Capacity = 8 },
		Guards = { Color = Color3.fromRGB(60, 60, 200), Capacity = 8 },
	},
	SyncRoblox = true, -- player-list colors via real Team instances
})
```

New sessions auto-balance onto the emptiest team (capacity-weighted, so a 2-cap duelist corner fills proportionally against 8-cap armies); `assign`/`teamOf`/`players`/`sameTeam` cover manual control. **Friendly fire is off by default** and enforced where damage actually happens: the kit registers vetoes on `Weapon.CanDamage` and `Melee.CanHit`, and npc handles resolve through their `Faction` — Raiders-faction NPCs and Raiders-team players read as teammates with zero extra wiring. Spawn ownership is map-authored: tag parts `TeamSpawn` with a `Team` attribute and characters pivot to an owned spawn (anti-exploit pardoned). Bus: `Team.Assigned(player, team, previous?)`.

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

### Bind abilities to movement, not just keys

MoveKit answers "jump pressed WHILE AIRBORNE" instead of just "jump pressed" — the double-jump/air-step/dash family, with charges that refill on landing:

```lua
-- Client
local Moves = MoveKit.new(Kernel, { Net = Net })
Moves:define("DoubleJump", MoveKit.recipes.doubleJump({ Power = 1.2 }))
Moves:define("AirStep", MoveKit.recipes.airStep({
	Steps = 3, -- anime stair-running: three footholds per fall
	OnStep = function(position, useIndex) spawnGlowPad(position) end,
}))
Moves:define("AirDash", {
	Input = "Dash", -- an InputDriver action; "Jump" rides the jump request itself
	When = "Airborne", -- or "Grounded"/"Rising"/"Falling"/function(context)
	Charges = 1, Cooldown = 1.5,
	Run = function(context)
		context.Root.AssemblyLinearVelocity = context.Root.CFrame.LookVector * 90
	end,
})
```

```lua
-- Server: the meter. Unknown abilities reject, spent charges reject, and the
-- pool refills only when the SERVER's own ground probe sees feet touch down.
MoveKit.server(Kernel, {
	Abilities = {
		DoubleJump = { Charges = 1, Cooldown = 0.2 },
		AirStep = { Charges = 3, Cooldown = 0.15 },
		AirDash = { Charges = 1, Cooldown = 1.5, Forgive = true }, -- horizontal burst: pardon the anti-cheat sample
	},
})
Kernel.Bus:subscribe("Move.Used", function(_, session, name, position)
	-- replicate the VFX to everyone else here
end)
```

The ownership guarantee holds: the client executes (it owns its character's physics — the velocity writes are legitimate), the server meters through the fail-closed `CKMove` intent with its own charge/cooldown books. A hacked client can spam requests; it cannot make the server bless a fourth air-step. `Run` receives `{Humanoid, Root, State, AirTime, Rising, UseIndex, ChargesLeft}` — `UseIndex` is how the second air-step gets a different sound than the first. Triggers debounce the takeoff press (`MinAirTime`, default 0.1s), and `kit:trigger(name)` fires abilities from your own code. Client bus: `Move.Triggered(name, context)`; server bus: `Move.Used(session, name, position)` / `Move.Rejected(player, name, reason)`.

### Fight hand to hand, parries included

MeleeKit is the timing game, server-authoritative: windup, hit frame, recovery — with combos that cancel recovery, blocks that chip, guard-breaks that shatter blocks, and timed parries that beat everything:

```lua
local Melee = MeleeKit.attach(Kernel, {
	Moveset = {
		Jab = { Damage = 8, Range = 7, Arc = 120, Windup = 0.15, Recovery = 0.25, ComboNext = { "Cross" } },
		Cross = { Damage = 12, Windup = 0.2, Recovery = 0.3, ComboNext = { "Uppercut" } },
		Uppercut = { Damage = 20, Windup = 0.3, Recovery = 0.5 },
		Haymaker = { Damage = 25, Windup = 0.6, Recovery = 0.5, GuardBreak = true },
	},
	ParryWindow = 0.18, StaggerSeconds = 1.1,
})
Kernel.Bus:subscribe("Melee.Parried", function(_, victim, attacker, attackName)
	-- clang VFX + the attacker is Staggered: the punish window is open
end)
```

**The triangle**: parry beats attack (the attacker staggers — a free punish window), block beats attack (damage scales to chip), guard-break beats block (full damage, the guard drops) but loses to parry HARDER (`GuardBreakStaggerScale`, default 1.5x stagger). Every timing runs on the server clock, and parry windows carry `LatencySlack` so honest defenders get their frames back — spam still can't cover a swing because attempts consume `ParryCooldown` whether they connect or not.

Players ride three fail-closed intents (`ML_Attack`/`ML_Block`/`ML_Parry`, auto-wired per session): "attack while staggered" dies in validation, never in a handler. Combos are declared on the moveset (`ComboNext` cancels recovery along the chain only; `Melee.Combo(attacker, chain)` counts the string). And it's UNIFIED with NPCs the same way everything else is: register an npc handle and its behaviors call `Melee:attack(npc, "Jab")`, while npc VICTIMS with no manual parry automatically roll their own skill-scaled `npc:defend()` — a Perfect duelist reads your haymaker like a book, a Bad one eats it. Hook point `Melee.CanHit` (fail-open) vetoes friendly fire. Bus: `Melee.Hit/Blocked/GuardBroken/Parried/Staggered/Combo/State`.

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

Effects is a full status system. `TickSeconds` + `OnTick` fire periodic ticks (poison damage, regen heals) with bounded catch-up after a stall. `Category = "Curse"` makes an effect dispellable: `Buffs:cleanse(session, "Curse")` removes every matching effect, `cleanse(session)` removes every categorized one, and uncategorized effects (VIP flags) never cleanse. `Immune = { "Curse" }` on an effect blocks applies of the covered categories while it holds — wards are effects, so an anti-curse ward is one `define` with a duration. Owners receive a `CKEffects` state push after every change (`{ [name] = { Stacks, Remaining? } }`) for status-icon UI. Bus adds: `Effects.Blocked(player, name, byEffect)`, `Effects.Cleansed(player, category?, count)`.

```lua
Buffs:define("Poison", { Duration = 6, Category = "Toxin", TickSeconds = 1,
	OnTick = function(session) damage(session, 2) end })
Buffs:define("Ward", { Duration = 30, Immune = { "Curse", "Toxin" } })
Buffs:cleanse(session, "Toxin") -- the antidote
```

### React to players and NPCs entering areas

```lua
local Regions = Zones.attach(Kernel)

-- Per-zone handle: no name-filtering global topics
local Safe = Regions:add("SafeZone", workspace.SafeZonePart, { -- any invisible part as the volume
	OnEnter = function(player) Buffs:apply(Kernel:getSession(player), "Protected") end,
})
Safe.Left:connect(function(player) Buffs:remove(Kernel:getSession(player), "Protected") end)

-- Or listen globally on the Bus (handlers get the zone name first)
Kernel.Bus:subscribe("Zone.Entered", function(_, name, player) ... end)
Kernel.Bus:subscribe("Zone.Left", function(_, name, player) ... end)
```

Zones aren't players-only — track NPCs, props, or vehicles and they fire their own events:

```lua
Regions:track(BossModel)   -- one entity; returns an untrack function
Regions:trackTag("Enemy")  -- every CollectionService-tagged instance, live

Kernel.Bus:subscribe("Zone.EntityEntered", function(_, name, entity)
	if name == "Objective" then Alarm:trip(entity) end
end)
Regions:entitiesIn("Objective") -- plus :playersIn(name) and :isInside(name, occupant)
```

Zero-code zones straight from Studio — tag parts and each becomes (or joins) a zone:

```lua
Regions:addTagged("CKZone") -- zone name = "ZoneName" attribute, falling back to the part's Name
-- Parts resolving to the same name union into one zone, same as Regions:add("L", { PartA, PartB })
```

Volume queries on a cadence, not `Touched` — slow walks can't slip through, ragdoll limbs don't double-fire, and leaves always fire before enters on a swap. Destroying (or untagging) a tag-tracked entity fires its leaves immediately; a manually tracked entity that despawns leaves on the next sweep.

### Scope replicas to zones

```lua
local Regions = Zones.attach(Kernel)
Regions:add("Dungeon", workspace.DungeonVolume)

local DungeonState = Replicas:create("DungeonState", {
	Schema = { BossHealth = "NumberU32", Phase = "String" },
	Data = { BossHealth = 5000, Phase = "Idle" },
})
DungeonState:bindZone(Regions, "Dungeon") -- returns an unbind function
```

Players outside the dungeon pay **zero bytes** for its state. Entering delivers the full snapshot the moment `Zone.Entered` fires — no waiting for the replica's next interest scan — and leaving unsubscribes the same way. The scan stays wired underneath (via `isInside`), so anyone already in the zone when the bind lands still converges.

### Give NPCs brains, aim skill, and movesets

NPCKit is **helpers, not an AI runtime** — the kernel ships syscalls, and chase/cover/flank policy is game rules. The kit owns a thin spawn/despawn lifecycle plus per-npc state (skill model, senses memory, cooldowns); YOUR loop makes the decisions:

```lua
local Paths = Pathfinding.attach(Kernel)
local Arena = Pathfinding.grid({ Origin = Vector3.new(-200, 0, -200), Width = 100, Height = 100, CellSize = 4 })
local Npcs = NPCKit.attach(Kernel, { Pathfinding = Paths, Grid = Arena, Projectiles = Projectiles, Zones = Regions })

Npcs:define("Cultist", {
	Model = ServerStorage.NPC.Cultist,
	Difficulty = "Medium", -- Perfect | HumanPeak | ReallyGood | Good | Medium | Novice | Bad — or a custom table
	Moveset = {
		-- The SAME projectile definition players fire — one server code path,
		-- two kinds of trigger finger. The npc handle rides as the owner session.
		Firebolt = { Kind = "Projectile", DefId = 2, Speed = 120, Cooldown = 3 },
	},
})

Npcs:spawn("Cultist", CFrame.new(10, 5, -40))
Npcs:spawn("Cultist", CFrame.new(30, 5, -60), { Difficulty = "HumanPeak" }) -- per-spawn skill override

-- The game's brain tick: schedule it yourself, compose the helpers however
-- your combat works. This one chases on sight and casts in range.
Kernel.Scheduler:every(0.15, function()
	Npcs:update() -- squad director + due melee windups
	for _, Npc in Npcs:all() do
		Npc:updateTarget(80, 120, { RequireSight = true, MemorySeconds = 3 })
		local Target = Npc.Target
		if Target == nil then
			if Npc.State.HeardPosition then
				Npc:moveTowards(Npc.State.HeardPosition) -- investigate the noise
			end
			continue
		end
		local Position = Npc:targetPosition(Target)
		if (Position - Npc:position()).Magnitude > 40 then
			Npc:moveTowards(Position)
		else
			Npc:faceTowards(Position)
			if Npc:canReact() and Npc:canSee(Target) then
				Npc:act("Firebolt", Target)
			end
		end
	end
end, Kernel.Priority.Low)
```

The handle's helpers are the vocabulary: `updateTarget` (acquisition, sight memory, graded detection), `canSee`/`sweep` (cone-gated eyes, pie-slice corner fans), `aimAt` (the full skill model), `act` (movesets), `defend` (reaction rolls), `moveTowards`/`faceTowards` (movement + body tracking), `kit:findCover`/`kit:findPeekPoint` (cover scoring), `kit:all()` (the roster), `kit:update()` (director + windups). NPC-versus-NPC combat is opt-in: mark an archetype `Targetable = true` and every other npc's ordinary acquisition can hunt it (self-targeting excluded), or override `GetTargets` for full roster control.

Difficulty is nine knobs, not a vibe: **AimErrorDegrees** (per-axis spray), **ReactionSeconds** (delay between noticing and acting), **LeadSkill** (0..1 of the ideal moving-target intercept), **CooldownMultiplier** (attack pace), **HearingMultiplier**, **HearingBlurStuds**, **PathSkill** (routing quality), **DefenseSkill** (0..1 parry/deflect success), and **FieldOfViewDegrees** (the vision cone — `Perfect` scans 360, `Bad` sees 100 in front). `Perfect` is an aimbot, `HumanPeak` is plausible-human-limit, `Bad` can't hit a barn — and a custom table slots anywhere between. The same knobs shape guns, wizard projectiles, and forest creatures alike: `Kind = "Melee"` gates on range and swings `Damage`/`OnHit`, and `Kind = "Custom"` receives a pre-aimed direction so you can point it at the same server function a WeaponKit or SpellKit handler already calls. With `Zones` attached, spawned models fire `Zone.EntityEntered/Left`. Bus: `Npc.Spawned/Died/Action/Heard/Suspicious/Damaged/Reload/Windup`.

NPCs also have **ears**. Player noise alerts them to a location, with accuracy scaled by skill — `Perfect` pinpoints the exact position, `Bad` hears a vague 30-stud blur:

```lua
local Npcs = NPCKit.attach(Kernel, {
	Pathfinding = Paths, Grid = Arena,
	Sounds = {
		Occlusion = "Path", -- sound goes AROUND obstacles, maze-correct; walls block it.
		-- "Blocked" = walls stop sound dead; "Through" (default) = walls ignored
		Topics = { ["Weapon.Fired"] = 60, ["Spell.Cast"] = 30 }, -- player actions that make noise
	},
})
Npcs:emitSound(explosionPosition, { Range = 80, Loudness = 2 }) -- or emit manually
```

The heard fix lands in `State.HeardPosition/HeardAt/HeardSource` for your loop to consume — walk there, comb the area, forget on your own schedule (a fresh sound resets the search scratch fields so your hunt restarts at the new spot). The ear model itself is the standalone **`Senses`** module: `Senses.hear(listenerPosition, attunement, sound, { Distance?, Rng? })` with named **attunement levels** (`Keen` hears half again as far and pinpoints; `Oblivious` hears half as far and vaguely; any custom `{ Range, BlurStuds }` slots between) and a `Distance` override so route-measured occlusion plugs straight in — hand-rolled agents and kit npcs share one set of ears. `Senses.inCone`/`Senses.canSee` are the matching vision gates.

Delivery mechanics all read the same skill model. `Kind = "Hitscan"` resolves instantly (aim error only, no lead) and publishes `Npc.Hitscan(npc, origin, hitPosition, victim?)` tracers; `Kind = "Projectile"` leads moving targets by `LeadSkill`; adding `Gravity` to the action solves the **ballistic arc** (matching the projectile def's gravity), so lobbers loft over distance and hopeful 45-degree at max range. Routing is skill-tuned too: the new `PathSkill` knob picks any-angle Theta* re-pathed eagerly at the top of the ladder and stale-path A* staircases at the bottom, and every computed route publishes `Npc.Path(npc, waypoints)` for visualization. `npc:setDifficulty("HumanPeak")` swaps the whole model live.

**Sight is a cone, not a sphere.** `npc:canSee(target, maxRange?, fovOverride?)` gates on range, then the skill's `FieldOfViewDegrees` against the rig's actual facing (a cheap dot compare — the raycast only runs for things in frame), then line of sight. Walk past a `Bad` guard's back and it never knows; a `HumanPeak` one covers 270 degrees like it's paid to. Sight-gated targeting (`RequireSight` on `updateTarget`) tracks a briefly-occluded target for `MemorySeconds`, then DROPS it and hands the last seen position to the hearing memory — your loop hunts the spot like a soldier instead of pathing omnisciently through walls. Hearing stays omnidirectional, so noise still covers the rear. Call `npc:faceTowards` whenever an npc holds position with a live target: a stopped body must keep turning or its own frozen cone loses a target circling behind it.

**Detection charges; pain registers; magazines run dry.** Add `DetectionSeconds` to `updateTarget` and acquisition becomes graded awareness instead of an instant lock: seeing a shape charges an accumulator — faster up close and against sprinters, decaying while nothing is visible — where half charge publishes `Npc.Suspicious` and points the ears at the glimpse (your investigate logic walks over for a look), and full charge acquires. Re-sighting a target after real occlusion re-imposes **half the reaction time**; the reappearance is a fresh stimulus, not a free instant re-engage. Getting hit registers through `npc:notifyDamage(attacker?)` — automatic for kit `Hitscan`/`Melee` victims and Humanoid damage on spawned rigs, published as `Npc.Damaged` — which blooms aim for 1.5s (read `State.SuppressedUntil` in your loop to duck an exposed peek) and drops an unseen attacker's position into the hearing memory: shooting a guard in the back turns it around. Ammo is real when you want it: `Magazine`/`ReloadSeconds` on an action force the reload pause players push into (`Npc.Reload`); melee `WindupSeconds` telegraphs the swing with an honest dodge window (`Npc.Windup`; the landing re-checks the arc); and ranged actions in a squad **hold fire when a squadmate blocks the lane** (`FriendlyFire = true` opts out). Aim error itself is a correlated drift plus near-Gaussian jitter that tightens with time-on-target — bursts walk like a hand, not a random number generator — and target leading solves the true intercept, not a distance/speed estimate.

**Cover scoring and defense.** `kit:findCover(npc, threat, { SearchRadius?, SpotTag?, BackAway? })` returns the nearest position the threat can't see — grid cells, or map-authored **cover nodes**: tag parts `SpotTag`, optionally with a `Direction` attribute (the facing of the wall protecting them) so a node only counts when the threat is on its covered side; squadmates already camping a node push the score down (spread out, don't stack), and `BackAway` weights spots that give ground. `kit:findPeekPoint(spot, threat)` finds the **corner beside the spot** with eyes on the threat — peeking means leaning to THAT, not strolling at the enemy until sight happens; your loop decides when to hold, peek, duck (`State.SuppressedUntil`), and relocate. Before committing past an angle, **slice the pie** — `npc:sweep(direction, arc?, rays?, range?)` fans short raycasts across the unseen angle. Defense is per-archetype `Reactions`: `Reactions = { Deflect = { Against = { "Hitscan", "Spell" }, Cooldown = 0.8, Difficulty = "HumanPeak", Run = counterVfx } }` — kit `Hitscan`/`Melee` actions roll the victim's matching reaction automatically (attempts consume the cooldown, success rolls `DefenseSkill`, `Npc.Defense(npc, name, success, context)` publishes either way), and `npc:defend(context)` runs the same gauntlet for your own combat code. Per-action `Difficulty` overrides let one body carry Perfect defense and Novice offense.

**Squads: a shared blackboard and a director.** Give archetypes `Faction = "Raiders"` (faction-mates never target each other) and `Squad = "Fireteam"` (auto-join on spawn, or `kit:squad(name):add(npc)`). Members report what their senses actually produced — sight fixes exact, heard positions **blurred by the listener's skill** — into the squad blackboard (`squad:lastKnown(target)`, `npc:lastIntel(target)`); nobody learns anything a squadmate didn't honestly sense, and intel stale past 30s expires. Each `kit:update()` the director fields a real fireteam per engaged target: the closest member holds **Pressure**, and the rest alternate **Flank** — sides split via `Squad.FlankSigns` so the pincer closes from BOTH directions — and **Suppress** (base of fire while the flanks move), publishing `Npc.Tactic(npc, role)` so clients play the dive/peek/sprint polish (the server only moves roots). Your loop reads `squad:role(npc)` and acts it out: pour `npc:act` at `npc:lastIntel(target).Position` to pin, curve to an enfilade point off the pressure axis to flank. All of it is combat-agnostic — the same roles drive gunfights, wizard duels, and wolf packs.

**Behavior trees for the strategic layer.** When ad-hoc loop logic stops being enough ("am I suppressed? is a squadmate distracting them? was that corner cleared?"), `BehaviorTree` composes `Selector`/`Sequence` over `Condition`/`Action` leaves with real `Running` semantics — composites remember the running child and resume it next tick instead of re-planning from the top, plus `Invert`/`Succeed`/`Cooldown` decorators and `reset()` to abort a plan. Combat brains root on **`ReactiveSelector`**, which re-evaluates from the first child every tick so a higher-priority branch coming alive preempts a Running lower one (its resume memory resets — "enemy appeared" interrupts "investigate" mid-plan). Build one tree per npc (they carry resume memory) and tick it from your loop — `BehaviorTree.tick(root, npc, dt, Npcs.Clock)` — with the kit clock so `Cooldown` nodes stay deterministic under spec; call `reset` when the target changes, because half a plan against the old target is a bug against the new one.

### Find paths four different ways

```lua
local Paths = Pathfinding.attach(Kernel)
local Grid = Pathfinding.grid({ Origin = Vector3.new(-200, 0, -200), Width = 100, Height = 100, CellSize = 4 })

local Ok, Waypoints = Paths:find({ Method = "AStar", Grid = Grid, Start = A, Goal = B }) -- complete grid search
Paths:find({ Method = "ThetaStar", Grid = Grid, Start = A, Goal = B }) -- any-angle: best straight lines
Paths:find({ Method = "Direct", Start = Eye, Goal = Target, Exclude = { Npc.Model } }) -- one raycast: "can I walk at it"
Paths:find({ Method = "Roblox", Start = A, Goal = B, AgentParams = { AgentRadius = 3 } }) -- engine navmesh; yields
```

One handler, four methods, each **lazily required on first use** — a game that never calls `AStar` never loads it. AStar/ThetaStar run on a rasterized grid with a `MaxExpansions` budget so a sealed-off goal can't eat the frame; failures return reasons (`GoalBlocked`, `NoPath`, `BudgetExhausted`), and `grid:refresh()` re-rasterizes after the map changes — or `grid:refreshRegion(minWorld, maxWorld)` drops only the cells a moving wall or gate swept, so a course that never stops moving stays cheap. `Roblox` wraps `PathfindingService` when you want the engine navmesh.

Grids account for the real world: the default rasterizer raycasts for **ground** (Terrain, parts, and model geometry all count — `CanCollide = false` decor never rasterizes as floor) and requires the agent-height column above it clear of CanCollide obstacles. Two things deliberately do NOT rasterize: **bodies** (anything under a Humanoid model — players and npcs are transient, and a cell probed while someone stood on it would cache blocked forever, including the asker's own start cell) and **steppable clutter** (the clearance column starts at step height, so a 1-stud lip or shin-high debris beside the cell center doesn't dead-strip the ring of cells around it). Non-block shapes get a **narrow phase**: the broad-phase overlap query matches bounding boxes, and a wedge ramp's bounding box is a full-height slab that would block every cell of its own slope — wedges, cylinders, meshes, and unions confirm against real geometry, so ramps rasterize as the walkable slopes they are. For user-driven queries, `grid:nearestWalkable(position, maxCells?)` is the forgiving front door: it clamps off-grid points into bounds and snaps clicks on top of walls to the nearest standable cell instead of failing on a technicality. Cells carry their ground height, so **waypoints follow elevation**, adjacent cells rising more than `MaxStepHeight` (default 4) read as cliffs, and line of sight refuses to cross walkable high ground — a wall's top surface still occludes. Inject `IsWalkable` (optionally returning a ground Y) for RTS maps, dungeons, or deterministic specs.

**Feet and photons are different queries.** `grid:lineOfSight` answers "can I WALK straight there" — it gates consecutive cells along the line by `MaxStepHeight`, so a Theta* shortcut can never walk an agent off a sheer cliff a stepwise search would refuse — while `grid:sightLine` answers "can I SEE there": valleys and pits that block feet stay transparent to eyes, and only ground rising past the height band (or an eye-level obstruction) occludes. NPCKit vision rides `sightLine`; movement rides `lineOfSight`. `grid:addLink(fromWorld, toWorld, cost?)` authors **off-mesh jump/vault edges** (ledges, window sills) that A*/Theta* traverse — never shortcut across — and humanoid movers hop for waypoints rising past step reach. `Pathfinding:distanceField({Grid, Origin, MaxDistance})` Dijkstra-floods route distances from one origin in a single search — it backs `emitSound`'s Path occlusion, so a gunshot near fifty listeners costs one flood, not fifty searches. Movement is a helper too, never a baked-in loop: `Pathfinding.follower({ Paths, Grid, Method?, RepathDistance? })` wraps re-pathing and waypoint-following in a per-agent handle YOUR loop steps — `follower:step(position, goal)` returns the next point plus `{ Path?, Jump, Stuck }` — with goal-drift and `Grid.Version` re-paths (a moved gate invalidates cached routes), a 1s backoff after failed searches (an unreachable goal never re-burns the full budget every tick), a stuck watchdog after a second of zero ground covered, jump hints for waypoints above step reach, and `Method = "Roblox"` computing off-thread so a yielding navmesh search never stalls the caller. `npc:moveTowards` is a thin wrapper over one of these.

### Track quests from things that happen

```lua
Kernel:registerService(QuestKit.service({
	Quests = {
		WolfCull = {
			AutoAssign = true,
			Objectives = { { Topic = "Combat.Kill", Count = 5 } },
			OnComplete = function(session) session.Profile.Data.Coins += 500 end,
		},
		Summit = { AutoAssign = true, Objectives = { { Topic = "Zone.Entered", Match = "Peak" } } },
		DailyObby = { AutoAssign = true, Repeatable = true, Objectives = { { Topic = "Obby.Checkpoint", Count = 10 } } },
	},
}))

Kernel.Bus:subscribe("Quest.Progress", function(_, player, questId, index, count, required) ... end)
Kernel.Bus:subscribe("Quest.Completed", function(_, player, questId) ... end)
```

A quest is data: objectives count **bus topics the server publishes** ([the catalog](#built-in-hook-points--bus-topics) is the menu), so quests can't be spoofed — there is no client input to validate. By default an event counts for a session when any published arg is that session or its player; `Match` additionally pins an arg (a zone name, a weapon id), and `Filter = fn(session, topic, ...)` takes full control (required for actor-less topics like `Round.PhaseChanged`). Progress persists via `session.Profile` when a DataDriver is attached; `Repeatable` resets to fresh on completion (dailies). `assign`/`abandon`/`progress` on the service handle the rest.

### Let players interact with the world

```lua
Kernel:registerService(InteractionKit.service({
	Interactions = {
		OpenChest = {
			Tag = "Chest", -- tag parts in Studio; prompts appear (and follow the tag) automatically
			ActionText = "Open",
			HoldDuration = 0.5,
			MaxDistance = 10,
			Cooldown = 3,
			LineOfSight = true,
			OnInteract = function(session, chest) Loot:roll(session, chest) end,
		},
	},
}))

-- Game rules prepend below the kit's gate (priority 50):
Kernel.Hooks:on("Intent.Interact", function(context)
	if context.Id == "OpenChest" then return context.Session.Data.HasKey == true end
end, 10)
```

A prompt trigger is **client input** — exploit tooling fires prompts from across the map — so every trigger re-validates server-side: distance (with latency slack), line of sight (server raycast, not the prompt's client-side flag), and per-player cooldown, all through the fail-closed `Intent.Interact` chain before `OnInteract` runs. Bus: `Interact.Triggered(session, id, instance)`, `Interact.Rejected(player, id)`.

### Swing hair, capes, and tails

```lua
-- CLIENT: bones are visual, so the sim runs where the frames are
local Bones = BonePhysics.attach({ MaxDistance = 100 })

-- Scan a character for boned accessories (layered hair, capes) and bind them
local Handles = Bones:bindCharacter(Players.LocalPlayer.Character, {
	Damping = 0.92, Stiffness = 0.1, -- floppiness vs snap-back-to-pose
})

-- Or a single mesh, or a plain part chain for stud-style tails
Bones:bind(capeMeshPart, { GravityScale = 1.4 })
Bones:bindParts({ TailRoot, Seg1, Seg2, Seg3 }, { Stiffness = 0.05 })
```

A lean verlet bone simulator built for what SmartBone-style modules get wrong: **fixed-timestep** solving (no frame-rate-dependent droop), flat arrays instead of per-bone objects, camera-distance **culling** that snaps back to the pose on wake, a **teleport guard** so respawns don't whip chains across the map, and a leak-free lifecycle — roots auto-unbind on `Destroying`, and poses are written through `Bone.Transform` (the animation slot), never `WorldCFrame`, so `destroy()` hands the skeleton back exactly as it found it. Chain roots stay rig-owned; physics only swings the children.

**Boneless accessories move too.** Most catalog hair has no bones — `bindAccessory` (run automatically by `bindCharacter`) models the accessory as a swing-clamped pendulum hinged at its joint and steers whichever joint kind holds it on: Weld/Motor6D offsets rotate, **RigidConstraint** accessories steer the handle-side Attachment CFrame (nothing swapped or disabled), and WeldConstraints swap for an equivalent local Weld restored exactly on release. `AmbientWind = Vector3` gives the whole sim a procedural gusting breeze with zero wiring — hair drifts, capes breathe. For hanging things (tails, chains), bind with `Stiffness = 0` and let gravity own them; stiffness is pose-hold strength, not springiness.

### Ragdoll bodies that replicate for free

```lua
-- server
local Ragdolls = Ragdoll.attach(Kernel, { MaxActive = 16, AutoDeath = true })
Ragdolls:enable(character, { Duration = 4, Impulse = hitDirection * 40 })
Ragdolls:release(character) -- restores every joint, socket tune, collision flag, and humanoid state

-- client bootstrap (player characters need the owner-side half)
RagdollClient.listen(netClient)
```

Rig-agnostic and build-once: classic rigs swap every enabled `Motor6D` for a `BallSocketConstraint` with per-joint swing/twist limits (necks stay necks, shoulders flop); NEW-format rigs (`AnimationConstraint` joints) just cut the animated joint and **tune the rig's own sockets** — friction (`FrictionTorque`, default 10: loose enough to drape, damped enough to settle) and WIDE anatomical cones (shoulders 120, hips 90 — tight limits read as a stiff mannequin; `Options.Limits` overrides any joint's `{Upper, Twist}` per game) go on for the ride and come off on release, because a frictionless free-spinning joint reads as a seizure, not a body. No automatic force: the body crumples on its own the moment the joints cut; pass `Impulse` for hit flings (applied per part by the owning machine). Components build ONCE per model and toggle on re-ragdolls (nothing new replicates), and limbs ride an auto-registered **self-blind collision group** (`CKRagdollLimbs` collides with the world, never with itself — jointed-pair no-collides alone miss hand-vs-hip and arm-vs-leg contacts, whose depenetration jitter is the classic residual spazz) plus per-pair `NoCollisionConstraint`s. Limbs gain world collision while the root goes ghost — but the **root joint itself never gets cut**: the `HumanoidRootPart` is the camera subject, and orphaning a massless non-colliding ghost sends it (and your camera) through the floor; left attached, it rides the torso, so the camera lies down with the body. `release()` puts back **exactly** what it found.

Player characters get the one thing property replication cannot do: humanoid state is client-authoritative, so the server pushes the `CKRagdoll` state channel and `RagdollClient.listen(netClient)` answers on the owning machine — it enters `Physics` and **latches it** (auto-getup off plus a `StateChanged` re-force: ground impacts knock the state machine into `Landed`/`GettingUp`, whose standing forces wrestling the sockets are the seizure), silences the `Animate` script (frozen tracks aren't enough; it keeps starting new ones), and applies any explicit `Impulse` fling per part — with rigid joints cut every limb is its own assembly, so an impulse on the massless ghost root would move nothing. Release **waits for the server's joint re-enables to replicate** (standing up while joints are still cut is where broken recoveries come from), kills all momentum, stands the root upright with its yaw kept, re-enables `Animate` fresh, and gets up — no slanted-walk hangover, no snap. Server-simulated rigs get the same momentum-kill and upright on release. Replication costs nothing extra: the swap is server-side and the engine streams the resulting physics like any assembly (the kernel never takes ownership of player characters). `MaxActive` releases the oldest when a pile-up would hurt the server. Bus: `Ragdoll.Started/Ended(model)`.

### Run the whole soundstage

```lua
-- CLIENT (server-attached AudioKit also works: world sounds replicate)
local Audio = AudioKit.attach(Kernel)
Audio:registerBank({
	Shot = { Id = "rbxassetid://1", Bus = "SFX", Priority = 3, Pooled = true },
	CaveTheme = { Id = "rbxassetid://2", Bus = "Music", Looped = true, Cues = { Drop = 32.5 } },
	GuideLine1 = { Id = "rbxassetid://3", Bus = "Voice", Subtitle = "Head for the summit." },
})

Audio:play("Shot") -- 2D
Audio:playAt("Shot", muzzlePosition, { Occlusion = "Path", Alert = { Range = 70 } }) -- 3D
Audio:playMusic("CaveTheme", { Crossfade = 3 })
Audio:speak("GuideLine1") -- ducks Music/SFX, queues lines, publishes Audio.Subtitle

local Layers = Audio:playLayers({ Calm = "rbxassetid://4", Combat = "rbxassetid://5" }, { StartLayer = "Calm" })
Layers:setWeights({ Calm = 1, Combat = 0.7 }, 2) -- stems MIX on one locked timeline

Audio:reverbZone(workspace.CaveVolume, Enum.ReverbType.Cave)
Audio:soundscape({ Bed = "WindLoop", Chirps = { "Bird1", "Bird2" }, Interval = { 4, 12 } })
Audio:bindSettings(Prefs, { Music = "MusicVolume", SFX = "SfxVolume" }) -- persisted volumes
```

Everything BetterSound-style modules do, rebuilt on kernel primitives: bus SoundGroups auto-wire, ONE scheduler stepper drives every fade/crossfade/duck (no TweenService churn, no per-sound Heartbeat connections — and the whole engine is deterministic under spec), sounds and 3D emitter parts pool, and channel pressure evicts by priority so a grenade never loses its bang to thirty footsteps. The kernel-only tricks: `Alert` rings NPCKit ears (players' sounds are NPC intel), `Occlusion = "Path"` muffles by the pathfinding route around geometry (the next corridor sounds far, not crystal-clear through the wall), cue points fire loop-aware `Audio.Cue` bus events, and volumes persist through the validated Settings service instead of a raw DataStore. Server triggering: `AudioKit.server(kernel)` gives `broadcast/playFor/broadcastAt`, and clients `audio:listen(netClient)`.

### Animate rigs and blend inverse kinematics

```lua
local Anims = AnimKit.attach(Kernel)
Anims:registerBank({
	Walk = { Id = "rbxassetid://10", Group = "Locomotion" },
	Run = { Id = "rbxassetid://11", Group = "Locomotion" },
	Swing = { Id = "rbxassetid://12", Markers = { "Impact" }, Priority = Enum.AnimationPriority.Action },
})

local Rig = Anims:attachRig(character)
Rig:play("Walk")
Rig:play("Run", { Fade = 0.3 }) -- same Group: Walk crossfades out, no T-pose blink
Kernel.Bus:subscribe("Anim.Marker", function(_, model, animName, marker, param) ... end)

-- IK blends over the keyframes: look at a target, reach for a handle
local Aim = Rig:ik("Aim", { Type = "LookAt", ChainRoot = waist, EndEffector = head, Target = enemyHead, FadeIn = 0.4 })
Aim:setWeight(0.5, 0.3)
Rig:ik("Grab", { Type = "Reach", EndEffector = rightHand, Target = Vector3.new(3, 4, 2) }) -- Vector3 spawns a movable helper
```

Tracks cache per rig, exclusive groups keep one locomotion clip alive, markers forward to the bus, and the IK layer wraps `IKControl` with the same fade stepper as AudioKit — weights blend in/out instead of snapping, `release()` restores the skeleton, and everything a rig created dies with its model. `Properties` passes raw IKControl overrides for anything the options don't cover, and `assetIds()` feeds the preloader.

### Preload assets behind a loading screen

```lua
local Warmup = Preload.run(Kernel, {
	Banks = { Audio, Anims }, -- anything with :assetIds()
	Assets = { "rbxassetid://999", workspace.Map }, -- extra ids or whole Instances
})
Kernel.Bus:subscribe("Preload.Progress", function(_, loaded, total, assetId)
	LoadingBar.Size = UDim2.fromScale(loaded / total, 1)
end)
Kernel.Bus:subscribe("Preload.Done", function(_, loaded, total, failed) LoadingGui:Destroy() end)
Warmup:await() -- or block a flow until warm
```

The framework already knows every asset the banks registered — one call warms them in batches, narrates progress on the bus for whatever loading UI you feed it, and collects failures without aborting the pass.

### Scale effects to the device

```lua
-- CLIENT, during the loading screen: Preload warms assets on the network,
-- the bench uses the idle CPU
local Bench = DeviceBench.run({ BudgetMs = 90, Bus = Kernel.Bus })
print(Bench.Quality) -- continuous 0.25..4, decimals allowed (1 = mid-range reference)
print(Bench.Axes) -- e.g. { Compute = 2.1, Churn = 3.4, Resume = 1.6 }

-- Budgets are funded PER AXIS: the workload a device is good at pays for the
-- systems that stress it. Strong Churn buys particles and vfx density; strong
-- Compute buys bone chains and projectile visuals; strong Resume buys channels.
local Quality = DeviceBench.profile() -- computed from the benched axes
Emitter.Rate = BaseRate * Quality.VfxDensity
Lighting.GlobalShadows = Quality.ShadowsEnabled

-- Map the continuous quality onto your own knobs (log-mapped: every doubling
-- of device quality buys the same slice of the range) — or ask for any exact
-- quality point, decimals included
local ViewDistance = DeviceBench.scale(80, 500)
local Handcrafted = DeviceBench.profile(1.35)
local MaxRagdolls = DeviceBench.pick({ Low = 4, High = 16 }) -- coarse tiers remain a convenience

-- Stability outranks beauty: hold 60fps, nudging the effective quality down
-- ~20% on sustained misses and gently back up with headroom, never past the bench
DeviceBench.governor({ TargetFps = 60, Bus = Kernel.Bus, OnChange = function(quality, profile)
	applyQuality(profile)
end })
```

A one-shot **mini benchmark** answers "how fast is this device" so quality stops being one-size: three time-boxed Luau workloads — compute, allocation churn, and coroutine resumes (how quick the client's threads actually run) — each become a continuous **0.25..4 axis multiplier** against a mid-range reference, decimals fully allowed, and their geometric mean is the overall `Quality`. Time-boxed means a weak device runs fewer iterations in the same ~90ms instead of hitching longer. The result caches (`Force` re-measures), optionally publishes `Device.Benchmarked`, and includes `Platform` plus an optional median `FrameMs`.

`DeviceBench.profile()` computes **granular budgets from the axes** — no tier cliffs, and each knob scales by the axis that pays its cost, so a device that benched 3.4x on churn but 1.2x on compute gets near-max particle budgets while bone chains stay modest. It accepts a `Result`, any quality number (`profile(1.35)` is a real point, not a rounded tier), or a tier name for coarse use; `Low`/`Medium`/`High`/`Ultra` remain as a convenience over the continuous scale for `pick()`. Framework client modules consume the profile when you don't configure them: BonePhysics defaults its cull `MaxDistance` to `BoneDistance` and its simultaneous chain budget `MaxChains` to `BoneChains`, AudioKit its `MaxChannels` to `AudioChannels`, ProjectileClient its `MaxVisuals` cap to `ProjectileVisuals` — every default is an option you can override, and the caps are purely cosmetic (a capped chain freezes at its pose). Projectiles obey a **fairness rule** on top: the cap only decides who gets the FULL visual (trails, particles) — overflow renders as a minimal pooled tracer instead of nothing, and a shot on course to pass within `ThreatRadius` (default 15) of the local character always renders full, cap or no cap. No quality level ever hides an incoming round from its target. The **governor** keeps it honest at runtime: it watches p95 frame times against `TargetFps` (default 60 — hold that before beauty), multiplies the effective quality by 0.8 after ~3s of sustained misses and by 1.15 after ~10s of clear headroom — granular nudges, not tier jumps — never past what the bench measured, and its profiles preserve the device's axis ratios, so a churn-strong device dialed down still favors its particles. Changes arrive via `OnChange(quality, profile)` and `Device.QualityChanged`. All of it is a **client measurement**: perfect for cosmetics, never authority.

### Save player settings and keybinds

```lua
-- Server: the schema is the allowlist — clients cannot write anything else
local Prefs = Settings.attach(Kernel, {
	Schema = {
		MusicVolume = { Kind = "number", Min = 0, Max = 1, Default = 0.5 },
		ShowHints = { Kind = "boolean", Default = true },
		DashKeys = { Kind = "strings", MaxItems = 2, MaxLength = 20, Default = { "Q" } },
	},
})
Prefs:get(session, "MusicVolume") -- server-side read with defaults

-- Client: fetch once, set optimistically, react to confirmed changes
local MyPrefs = SettingsClient.new(Net)
MyPrefs:fetch()
MyPrefs:set("MusicVolume", 0.8)
MyPrefs:onChanged(function(key, value)
	if key == "DashKeys" then Input:rebind("Dash", toKeyCodes(value)) end
end)
```

Writes ride a rate-limited intent through the fail-closed chain: unknown keys reject, numbers **clamp** into `Min/Max`, strings cap at `MaxLength`, and string arrays are **rebuilt clean** so hidden hash keys never reach storage. Values persist via the DataDriver profile (pre-profile writes merge in, player's choice wins) and `Settings_Sync` pushes the authoritative value back so an out-of-range optimistic set self-corrects. Bus: `Settings.Changed(session, key, value)` — wire it to InputDriver `rebind` and remaps survive rejoins.

### Ship gameplay analytics

```lua
local Events = Analytics.attach(Kernel, {
	MaxPerMinute = 120, -- protects Roblox's custom-event budget
	Sample = { PositionPing = 0.1 }, -- keep 10% of chatty events
	WatchErrors = true, -- bridge Kernel.ScriptError automatically
})
Events:track("RoundFinished", winner, roundNumber)
Events:track("BossKilled", killer, 1, { CustomField01 = bossName })
```

`track()` is a queue insert; a background flush batches to `AnalyticsService:LogCustomEvent` — funnels and retention with zero external infrastructure — or to an injected `Destination` (webhook, warehouse). Over-cap and sampled-out events **drop and are counted** in `Stats`, never silently; a crashing destination loses only its own batch.

### Audit the security surface in one call

```lua
Kernel:securityAudit()
-- [High]   UnguardedChannel — intent Grant: has a handler but no validator ...
-- [High]   FailOpenGate — hook Intent.Buy: guards a channel but is FailOpen ...
-- [Medium] OpenChannel — intent ReadyUp: any client payload reaches the handler ...
-- [Info]   DefaultRateLimit — intent Chat: rides the default rate limit (30/s) ...

local Findings = Kernel:securityAudit({ Silent = true }) -- CI-style gate
assert(#Findings == 0, "ship blocked: security audit found issues")
```

One post-boot sweep over every assumption the security model makes: handler-bearing channels with no validator, `Open = true` escape hatches (by design, but confirm each survives hostile input), gate hook points flipped `FailOpen` (a crashing validator would PASS payloads), and channels still on the default rate limit. Sorted most severe first; `auditValidators()` remains the narrow unguarded-only sweep.

### Fuzz every channel with hostile payloads

The audit reads configuration; the fuzzer exercises handling. `Fuzz.run` builds hostile argument sets per schema type (empty and 10KB strings, format specifiers, control bytes, 0, -1, type maxima, NaN, infinities, NaN vector components, nil and nested tables for Any) and drives each registered intent and request through the real pipeline — hook chain first, handler on pass:

```lua
local Report = Fuzz.run(Kernel, { Session = session, CasesPerChannel = 64 })
assert(#Report.HandlerErrors == 0) -- a boot-time sanity gate
```

A payload that passes validation and then crashes the handler is a bug in one of the two — every such case lands in `Report.HandlerErrors` with the channel, arguments, and error. Validators that throw count as rejects (fail-closed holds under fuzzing) and surface as kernel warnings. Deterministic per `Seed`; filter with `Channels`; pass a real session for accurate results. The wire's typed serialization bounds what real clients can send — the fuzzer sends past those bounds on purpose.

### Show stats on the leaderboard

```lua
Leaderstats.attach(Kernel, {
	Coins = { Field = "Coins" },                            -- from session.Profile.Data
	Stage = { Field = "BestStage" },
	Title = { Field = "Title", Kind = "StringValue", From = "Session" },
})
```

That's the whole integration — values track the data automatically.

### Keep global top-100 boards

Leaderstats is per-server; Leaderboards is the game-wide top-N on OrderedDataStores, with the persistence layer's budget discipline:

```lua
local Boards = Leaderboards.attach(Kernel, {
	Boards = {
		Kills = {}, -- descending, keep-best, top 100, 60s cache
		BestTime = { Ascending = true }, -- speedruns: smaller wins
	},
})
Boards:submit("Kills", session, killCount) -- queue: instant, allocation-cheap
local Top = Boards:top("Kills", 10) -- {Key, Value, Rank} from cache
```

Submits queue and flush on a cadence — the newest value per player per window costs ONE write no matter how fast scores change, `KeepBest` boards only overwrite improvements (atomic `UpdateAsync`, no read-modify race), failed writes stay queued for the next window instead of vanishing, and `WritesPerFlush` caps each window so provider budgets survive spikes. `top()` serves a cached page so a hundred UI readers cost one `GetSortedAsync` per window, and a failed refresh keeps serving the previous page. The backend is a **first-class seam** — Roblox OrderedDataStores by default, custom stores (external DBs over HttpService, MemoryStores) as equals: implement `submit(board, key, value, keepBest, ascending)` and the keep-best rule arrives declaratively for your engine to apply atomically its own way (SQL upsert, Redis script, conditional PUT), or implement DataStore-shaped `set(board, key, updater)`; both need `sorted(board, ascending, count)`. Bus: `Leaderboard.Updated(name, entries)` on refresh.

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

All kit validation lives in the same fail-closed hook chains (`Intent.WK_Fire`, `Intent.SK_Cast`, `Intent.Checkpoint`, `Intent.Interact`, `Intent.Settings_Set`) — prepend your own rules at lower priority numbers without forking the kit. Beyond the combat kits: [QuestKit](#track-quests-from-things-that-happen) (declarative objectives off bus topics), [InteractionKit](#let-players-interact-with-the-world) (exploit-proof prompts), and [NPCKit](#give-npcs-brains-aim-skill-and-movesets) (behavior lists, difficulty presets, player-unified movesets).

> **ProceduralKit** rides Roblox's ProceduralModels (Studio beta; works in live games when the beta ships): parameter-driven Models whose server-side Generator ModuleScript rebuilds their geometry when Size or an attribute changes. The peer that changes a parameter generates — server writes generate server-side and replicate the results, so generators stay in ServerScriptService where exploiters can't decompile them. The kit validates and clamps every parameter (unknown names and wrong types reject outright), buffers writes behind a per-model cooldown so a spammed customization intent costs one rebuild per window instead of one per message, and ships an ExampleGenerator demonstrating the two generator survival habits: `params:Pause()` in loops (the engine kills generators that work too long without yielding) and deriving ALL randomness from a Seed attribute (every rebuild reruns the generator — unseeded randomness reshuffles the model on every parameter change).

> **Status:** WeaponKit and SpellKit have run live in Studio play sessions (fire, cast, interrupt) on top of spec-verified validators. CheckpointKit's validators are spec-verified, but it needs a `workspace.Checkpoints` folder with numbered pads to wire up — its `Touched` path warns at boot when the folder is missing. WeaponKit damage-on-hit deserves a two-player live test — solo Studio has nothing to shoot.

### ProceduralKit: parametric geometry end to end

Everything below is the complete path — generator, archetypes, spawning, runtime parameter driving, player customization, and observation. Each step is real code against the shipped API.

**1. Write a generator module** — a ModuleScript in ServerScriptService (server-side on purpose: the peer that changes a parameter generates, so server-driven models never expose your generator to decompilers). This is the engine contract every generator must satisfy:

```lua
-- ServerScriptService.Generators.Fence (ModuleScript)
local Fence = {}

-- Attributes declare every parameter AND its type (the kit rejects writes
-- whose typeof() differs from the default's). These become real Roblox
-- attributes on the spawned ProceduralModel.
Fence.Attributes = {
	Posts = 6,
	PlankColor = Color3.fromRGB(140, 100, 60),
	Style = "Solid", -- validated against OneOf below
	Seed = 0,
}

-- Called by the ENGINE whenever Size or any attribute changes. Rules:
--   * Build ONLY into targetContainer — never touch workspace or the model
--     directly; the engine migrates the container's contents in afterward.
--   * The whole folder is rebuilt from scratch every time — manual edits to
--     generated content are lost by design, so generate everything.
--   * params.Size is the model's normalized bounds, centered on its pivot.
--   * Derive ALL randomness from a Seed parameter. The generator reruns on
--     every parameter change; math.random would reshuffle the fence each
--     time someone recolors it.
--   * Call params:Pause() inside loops — the engine kills generators that
--     work too long without yielding ("Script Timeout"); Pause is nearly
--     free and usually returns immediately.
function Fence.OnGenerate(params, targetContainer)
	local Attrs = params.Attributes
	local Size = params.Size
	local Rng = Random.new(Attrs.Seed)
	local Spacing = Size.X / math.max(1, Attrs.Posts - 1)

	for Index = 1, Attrs.Posts do
		local Post = Instance.new("Part")
		Post.Anchored = true
		Post.Size = Vector3.new(0.4, Size.Y, 0.4)
		-- lay posts along X in normalized space: -Size.X/2 .. +Size.X/2
		Post.CFrame = CFrame.new(
			-Size.X / 2 + Spacing * (Index - 1),
			0,
			(Rng:NextNumber() - 0.5) * 0.1 -- seeded jitter: stable per Seed
		)
		Post.Parent = targetContainer
		params:Pause()
	end

	local Planks = if Attrs.Style == "Gapped" then 2 else 3
	for Row = 1, Planks do
		local Plank = Instance.new("Part")
		Plank.Anchored = true
		Plank.Color = Attrs.PlankColor
		Plank.Size = Vector3.new(Size.X, Size.Y / (Planks * 2), 0.2)
		Plank.CFrame = CFrame.new(0, Size.Y * (Row / (Planks + 1)) - Size.Y / 2, 0)
		Plank.Parent = targetContainer
		params:Pause()
	end
end

return Fence
```

**2. Register archetypes** — the kit's config is where parameters become *enforced*:

```lua
local ProceduralKit = require(ServerScriptService.ChloeKernelServer.Kits.ProceduralKit)

Kernel:registerService(ProceduralKit.service({
	Archetypes = {
		Fence = {
			Generator = ServerScriptService.Generators.Fence,
			-- Defaults define the parameter set: a set() for any name not in
			-- here is rejected outright (fail-closed — attackers probe
			-- attribute names), and each default's type is the enforced type
			Defaults = {
				Posts = 6,
				PlankColor = Color3.fromRGB(140, 100, 60),
				Style = "Solid",
				Seed = 0,
			},
			Size = Vector3.new(16, 4, 1), -- default model bounds at spawn
			Validate = {
				Posts = { Min = 2, Max = 24 }, -- numbers CLAMP into range
				Seed = { Min = 0, Max = 2 ^ 31 },
				Style = { OneOf = { "Solid", "Gapped" } }, -- non-members REJECT
				-- PlankColor has no rule: still type-checked (must be a Color3)
			},
		},
	},
	-- Minimum seconds between applied parameter batches PER MODEL. Writes
	-- inside the window buffer and flush together as ONE rebuild — a player
	-- spamming a customization UI costs one regeneration per window, not
	-- one per click. Generation creates and replicates real instances, so
	-- this is the knob that keeps it affordable.
	RegenSecondsPerModel = 0.25,
}))
```

**3. Spawn and drive from server code** — every option and every handle method:

```lua
local Procedural = Kernel:getService("ProceduralKit")

local Handle = Procedural:spawn("Fence", {
	CFrame = CFrame.new(0, 2, -20), -- pivot; generated content centers on it
	Size = Vector3.new(24, 5, 1), -- overrides the archetype default bounds
	Params = { Posts = 10, Seed = 42 }, -- validated; garbage here errors LOUDLY
	Parent = workspace, -- default
	Owner = session, -- optional: the fence is destroyed when this player leaves
})

-- set(): validated then applied — or buffered if inside the regen cooldown.
-- Returns (false, reason) for unknown names / wrong types / OneOf misses;
-- numbers clamp instead of failing.
local Ok, Reason = Handle:set("Posts", 999) -- applies as 24 (clamped to Max)
Handle:set("Style", "Lasers") -- (false, "not one of the allowed values")

-- patch(): several parameters, still ONE rebuild when the batch flushes
Handle:patch({ PlankColor = Color3.fromRGB(60, 60, 70), Style = "Gapped" })

-- resize(): Size changes regenerate too, so they ride the same cooldown
Handle:resize(Vector3.new(32, 5, 1))

-- waitForGeneration(): yields until the engine finishes the current rebuild.
-- (false, err) surfaces the generator's own error (GenerationError) — a
-- crashing generator leaves a placeholder, not geometry, so check this in tooling.
local Built, Err = Handle:waitForGeneration()

-- regenerate(): force a rebuild without changing anything (beta API; returns
-- false where the engine stubs it). Rarely needed — parameter writes are the trigger.
Handle:regenerate()

Handle:destroy() -- removes the model; safe to call twice. The kit also
-- cleans its state up if game code Destroy()s Handle.Model directly.
```

**4. Let players customize it — fail-closed.** The kit deliberately owns no wire channel; you wire an intent and make `service:validate` the validator, so the kit's rules ARE the gate:

```lua
local Net = Kernel:net()
-- parameter name + value as a string (parse server-side); keep payloads
-- typed-narrow — never accept arbitrary tables from clients
Net:defineIntent("FenceStyle", { "String", "String" }, { RateLimit = 5 })

-- the validator: resolve the player's own fence, translate the payload into
-- a params table, and let the kit decide. Returning false rejects — and a
-- channel with a handler and NO validator rejects everything by default.
Kernel.Hooks:on("Intent.FenceStyle", function(context)
	local Fence = context.Session.Data.Fence -- the Handle stored at spawn
	if not Fence then
		return false
	end
	local Param, Raw = context.Args[1], context.Args[2]
	local Value: any = Raw
	if Param == "Posts" or Param == "Seed" then
		Value = tonumber(Raw)
		if Value == nil then
			return false
		end
	end
	local Ok, CleanedOrReason = Procedural:validate("Fence", { [Param] = Value })
	if not Ok then
		return false -- unknown name, wrong type, OneOf miss — all die here
	end
	-- pass the CLAMPED values forward by replacing the args the handler sees
	context.Args[3] = CleanedOrReason
end)

Net:onIntent("FenceStyle", function(session, _param, _raw, cleaned)
	-- only reached when every validator passed; apply the cleaned batch.
	-- The kit's per-model cooldown still applies underneath — a spammed
	-- intent inside the window batches into one rebuild.
	session.Data.Fence:patch(cleaned)
end)
```

**5. Watch it on the Bus** — every lifecycle edge publishes:

```lua
Kernel.Bus:subscribe("Procedural.Spawned", function(_, name, model) end)
Kernel.Bus:subscribe("Procedural.Updated", function(_, name, model, applied)
	-- applied = the exact parameter batch that just flushed (post-clamp)
end)
Kernel.Bus:subscribe("Procedural.Rejected", function(_, name, param, reason)
	-- a validated write failed — wire this to your anti-exploit strikes:
	-- honest clients never send unknown params or wrong types
end)
Kernel.Bus:subscribe("Procedural.Removed", function(_, name, model) end)
```

> **Cost model, so the knobs make sense:** a rebuild reruns the whole generator and replaces the whole `GeneratedFolder`; if the server did it, every generated instance replicates to every client. `RegenSecondsPerModel` bounds how often that can happen per model, intent `RateLimit` bounds how often a client may even ask, and `params:Pause()` bounds how long a single rebuild can hog the frame. Size the three to your geometry: a 12-part fence can afford a 0.25s window; a 500-part castle wants seconds.

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

`InputDriver.new(kernel)` · `:bindAction(name, {Keyboard?, Gamepad?, TouchButton?, Handler?})` · `:rebind(name, "Keyboard"|"Gamepad", keys?)` · `:unbindAction(name)` · `:getBindings()` · `:destroy()`. Emits `Input.<Action>` on the bus. `Keyboard`/`Gamepad` take one `Enum.KeyCode` or an array of them; snapshots mirror the shape (name or array of names).

`MoveKit.new(kernel, {Net?, Clock?, Humanoid?, Root?, SkipCharacterTracking?}?)` — `:define(name, {Input?, When?="Airborne", Charges?=1, Cooldown?=0, MinAirTime?=0.1, Announce?=true, Run})` · `:trigger(name) → bool` · `:destroy()` · `MoveKit.jumpVelocity(humanoid)` · recipes `MoveKit.recipes.{doubleJump({Power?, Charges?, Cooldown?}?), airStep({Steps?=3, Boost?=0.75, Cooldown?, OnStep?}?)}`. Server: `MoveKit.server(kernel, {Abilities = {[name] = {Charges?, Cooldown?, Forgive?}}, IsGrounded?, Clock?, CooldownSlack?=0.05})` — fail-closed `CKMove` intent metering with server-side ground truth; `Input = "Jump"` rides `UserInputService.JumpRequest`, other strings ride `Input.<Action>` bus events. Bus: client `Move.Triggered(name, context)`, server `Move.Used(session, name, position)` / `Move.Rejected(player, name, reason)`.

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

`InventoryKit.attach(kernel, {Items, MaxSlots?=30, Persist?=true, Field?="Inventory"})` — `:grant(session, id, count?) → granted` · `:take(session, id, count?) → bool` (all-or-nothing) · `:count(session, id)` · `:move(session, from, to)` · `:equip(session, slotIndex)` · `:unequip(session, slotName)` · `:equipped(session, slotName) → (item?, index?)` · `:use(session, slotIndex)` · `:items(session)`. Items: `{Name?, Stack?=1, Kind?, Equip?, OnUse?, OnEquip?, OnUnequip?, Meta?}`. Client intents `INV_Move/Equip/Unequip/Use` fail-closed; owner state push `INV_Sync`; hooks `Inventory.CanEquip/CanUse` (fail-open); Bus `Inventory.Granted/Taken/Equipped/Unequipped/Used/Changed`.

`LootKit.attach(kernel, {Tables, Inventory?, Seed?, Rng?})` — `:roll(table) → items` · `:rollFor(session, table) → items` (pity persists in profile `Data.LootPity`) · `:award(session, table) → (granted, overflow)`. Entries: `{Weight, Id?, Count?|{min,max}, Table?, Nothing?, PityAfter?}`; tables: `{Rolls?|{min,max}, Entries}`; nesting caps at 8; unknown references fail at attach. Bus `Loot.Rolled/Awarded`.

`CraftingKit.attach(kernel, {Recipes, Inventory, TickSeconds?=0.25, Clock?, Intents?=true, SkipLoop?})` — `:craft(session, recipeId, station?) → (ok, reason?)` · `:canCraft(...)` · `:cancel(session)` (refunds) · `:activeCraft(session)` · `:destroy()`. Recipes: `{Inputs, Output, Seconds?=0, Station?, OnComplete?}`; intents `CR_Craft`/`CR_Cancel`; hook `Craft.CanCraft` (fail-open); Bus `Craft.Started/Completed/Cancelled/Overflow`.

`ShopKit.attach(kernel, {Shops, Inventory, Currency?, CurrencyField?="Coins", Clock?, Intents?=true})` — `:buy(session, shop, item, count?) → (ok, reason?)` · `:sell(...)` · `:listings(shop)`. Shops: `{Listings? = {[id] = {Price?, SellPrice?, Stock?}}, Rotation? = {Every, Size, Pool, Seed?, Listing?}}`; rotation is deterministic per window across servers; Currency adapter `{Get, Take, Give}`; intents `SH_Buy`/`SH_Sell` (count <= 99); Bus `Shop.Bought/Sold/Rejected`.

`CompanionKit.attach(kernel, {Npcs, Effects?, Companions, TickSeconds?=0.25, Intents?=true, SkipLoop?})` — `:summon(session, id) → npc?` · `:dismiss(session)` · `:companionOf(session) → (npc?, id?)` · `:destroy()`. Companions: `{Archetype, FollowDistance?=8, TeleportDistance?=60, Aura?, OnSummon?, OnDismiss?}`; one per session; intents `CP_Summon`/`CP_Dismiss`; hook `Companion.CanSummon` (fail-open); Bus `Companion.Summoned/Dismissed/Died`.

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
| `Pool.new({Create, InitialSize?})` · `:acquire()` · `:release(instance)` · `:idleCount()` · `:destroy()` | |
| `ErrorWatch.attach(kernel?, {WindowSeconds?=30}?)` · `:counts()` | Bus `Kernel.ScriptError(message, trace, source, count)` |
| `LiveConfig.new(kernel, {Defaults, PollSeconds?=30})` · `:get(key)` · `:set(key, value)` | Bus `Config.Changed(key, value)`; `set(key, nil)` deletes the override and every server reverts to its default |
| `Pathfinding.attach(kernel?)` · `:find({Method?="Direct", Start, Goal, Grid?, MaxExpansions?, Exclude?, AgentParams?}) → (ok, waypointsOrReason)` · `:distanceField({Grid, Origin, MaxDistance?}) → {[cellKey] = studs}?` · `Pathfinding.follower({Paths, Grid, Method?="ThetaStar", RepathDistance?, ArriveDistance?, Clock?}) → follower` · `follower:step(position, goal) → (nextPoint, {Path?, Jump, Stuck})` · `follower:reset()` · `Pathfinding.grid({Origin, Width, Height, CellSize?=4, AgentHeight?=6, MaxStepHeight?=4, IsWalkable?}) → grid` · `grid:refresh()` · `grid:refreshRegion(minWorld, maxWorld)` · `grid:addLink(fromWorld, toWorld, cost?)` · `grid:groundY(x, z)` · `grid:nearestWalkable(position, maxCells?=8) → Vector3?` · `grid:lineOfSight(x0, z0, x1, z1)` (feet) · `grid:sightLine(x0, z0, x1, z1)` (eyes) | Methods `AStar`/`ThetaStar`/`Direct`/`Roblox`, lazily required; failures return reasons (`GoalBlocked`, `NoPath`, `BudgetExhausted`, ...); `Roblox` yields (the follower runs it off-thread). Default rasterizer raycasts ground (Terrain + CanCollide parts), waypoints ride elevation, `MaxStepHeight` refuses cliffs per step AND along straight lines; walkable high ground occludes sight, valleys only block feet. `refresh` bumps `grid.Version` so followers re-path |
| `Settings.attach(kernel, {Schema, Field?="Settings", RateLimit?=10})` · `:get(session, key)` · `:all(session)` · `:set(session, key, value)` · rule: `{Kind = "number"/"boolean"/"string"/"strings", Default, Min?, Max?, MaxLength?=64, MaxItems?=8}` | Allowlisted client writes: clamp/cap/rebuild through `Intent.Settings_Set`; persists via profile; Bus `Settings.Changed`. Client: `SettingsClient.new(net)` · `:fetch()` · `:get` · `:set` · `:onChanged(fn)` |
| `Analytics.attach(kernel, {FlushSeconds?=10, MaxPerMinute?=120, Sample?, Destination?, WatchErrors?, Seed?}?)` · `:track(name, player?, value?, fields?) → bool` · `:stats via .Stats` · `:destroy()` (flushes) | Batched to `AnalyticsService:LogCustomEvent` or an injected destination; sampled/over-cap drops counted, never silent |
| `BonePhysics.attach({Gravity?, Wind?, FixedHz?=60, MaxSubsteps?=3, MaxDistance?=100, MaxChains?=20, ShouldSimulate?}?)` · `:bind(part, {Damping?=0.92, Stiffness?=0.1, GravityScale?, WindScale?}?) → handle?` · `:bindParts({parts}, settings?)` · `:bindCharacter(model, settings?) → {handles}` · `:step(dt)` · `handle:destroy()` · `:destroy()` | Client-side verlet bone chains (shared module, `ReplicatedStorage.ChloeKernel.BonePhysics`); fixed timestep, culling with pose snap-back, chain budget (past `MaxChains` the nearest to camera win), teleport guard, Transform-slot writes, Destroying auto-unbind; unconfigured `MaxDistance`/`MaxChains` default from the device profile on clients |
| `Ragdoll.attach(kernel, {MaxActive?=16, AutoDeath?, CollisionGroup?, FrictionTorque?=60}?)` · `:enable(model, {Duration?, Impulse?}?) → bool` · `:release(model)` · `:isRagdolled(model)` · `:destroy()` — clients: `RagdollClient.listen(netClient)` | Build-once/toggle: Motor6D→limited BallSocket swap or AnimationConstraint cut with native-socket friction/limit tuning, NoCollision limb pairs, full restore; owner-client `CKRagdoll` push flips humanoid state, silences Animate, applies impulses, uprights on release; Bus `Ragdoll.Started/Ended` |
| `AudioKit.attach(kernel, {Buses?, MaxChannels?=32, Parent?, Npcs?, Occlusion = {GetListener?, IsBlocked?, Pathfinding?, Grid?}?}?)` · `:register/registerBank` · `:play(name, opts?) → handle` · `:playAt(name, target, opts?)` · `:playMusic` · `:playLayers → {setWeights, resync, stop}` · `:speak` · `:duck(bus, scale) → release` · `:setBusVolume` · `:bindSettings(prefs, map)` · `:reverbZone(part, type)` · `:soundscape` · `:assetIds()` · `:destroy()`; `AudioKit.server(kernel) → {broadcast, playFor, broadcastAt}` · `audio:listen(netClient)` | Bank config: `{Id, Bus?, Volume?, Speed?, Looped?, Priority?, Pooled?, RollOff*, Cues?, Subtitle?, Duck?}`; handle: `stop(fade?)/setVolume/setSpeed`. Bus events `Audio.Cue/Subtitle/SubtitleEnded` |
| `AnimKit.attach(kernel, {LoadTrack?}?)` · `:register/registerBank` (`{Id, Priority?, Speed?, FadeIn?, Looped?, Group?, Markers?}`) · `:attachRig(model) → rig` · `:assetIds()` · `:destroy()`; rig: `:play(name, {Fade?, Speed?, Weight?})` · `:stop/stopAll/setSpeed` · `:ik(name, {Type = "LookAt"/"Reach"/"Transform"/"Rotation", Target, ChainRoot?, EndEffector?, Weight?, FadeIn?, Properties?}) → {Control, setWeight, setTarget, release}` · `:destroy()` | Exclusive `Group`s crossfade; markers publish `Anim.Marker`; IK weights blend on the kit stepper; rigs die with their model |
| `Preload.run(kernel, {Assets?, Banks?, BatchSize?=16}?) → {Done, Loaded, Total, Failed, await()}` | Batched `PreloadAsync`; Bus `Preload.Started/Progress/Done`; failures collect, never abort |
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
