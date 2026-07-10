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

Rules: `Range = {min, max}` (numbers, inclusive), `OneOf = {...}` (allowed values), `MaxLength = n` (strings). They install at priority 5, ahead of game validators, and REJECT. Channel integers WRAP into the wire type (a 300 sent as `NumberU8` arrives as 44 — replica Serde fields clamp instead), so rules are the layer that catches out-of-range payloads. `Registry.define` shape-checks the table at load, so a typo'd `Kind` fails the boot instead of a random send later.

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

- **Pick the narrowest type that fits.** Every field costs exactly its width: `NumberU8` 1B, `NumberU16` 2B, `NumberU24` 3B, `NumberU32` 4B, `Vector3S16` 6B (integer studs), `Vector3F24` 9B, `Vector3F32` 12B, `Boolean8` 1B (channel schemas additionally offer `Boolean1`, packing 8 flags into 1 byte — replica schemas do not). A score that can't exceed 16M in `NumberU24` instead of `NumberU32` saves a byte on every delta forever.
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
local Input = InputDriver.attach(kernel)
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

### Record exploiters for forensic replay

Forensics is the anti-exploit flight recorder: every player's kinematics roll through a ring buffer, and the moment a watched topic flags someone, the window freezes into a capture and keeps recording through the tail:

```lua
Forensics.attach(Kernel, {
    Destination = { Url = "https://my-tool.example/captures" }, -- POSTs the JSON
})

-- Studio: walk any capture as a ghost rig
local Capture = Recorder:captures()[1]
Forensics.replay(Capture, { Loop = true })
```

Frames are flat arrays `[t, px, py, pz, lx, ly, lz, vx, vy, vz, humanoidState]` at `RecordHz` (default 10) covering `WindowSeconds` before the flag (default 20) plus `TailSeconds` after (default 5) — the movement before, during, and after the violation as reconstructable data. A web tool can rebuild the run frame by frame from the JSON; `replay()` does it in Studio with an interpolated ghost. Re-flags during the tail append reasons to the same capture, leavers finalize immediately, delivery never blocks the sweep (function destinations and `{ Url }` POSTs both fail into the in-memory ring, last `MaxCaptures` always readable). Captures hold movement, reasons, and the user id only. Bus: `Forensics.Captured(player, capture)`.

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

### Adapt replication to each player's connection

NetGovernor grades every client Good, Strained, or Poor from live link health and drives replication down to what the connection can carry:

```lua
-- Server
NetGovernor.attach(Kernel)

-- Client (bootstrap)
GovernorClient.attach() -- readiness handshake, probe echoes, device hint

Kernel.Bus:subscribe("Net.TierChanged", function(_, player, tier, stats)
	print(player.Name, "->", tier, stats.PingMs, stats.Loss)
end)
```

Measurement is threefold: ping median and jitter from `GetNetworkPing` samples, unreliable packet loss from sequence-numbered echo probes (probes hold until the client's readiness handshake, so join silence never reads as loss), and the client's DeviceBench quality hint (clamped, floor-to-Strained only — a weak device never grades Poor by itself). The worst axis wins. Two systems respond automatically: ReplicaService sends deltas to Strained clients every 2nd replica tick and Poor every 4th (skipped fields bank per subscriber and merge into their next send, so discrete state never goes missing — snapshots and removals always send, phases stagger per player), and unreliable state channels marked `Priority = "Low"` (the default) shed entirely to Poor clients while `"High"` traffic always flows. Shed counts land in `Governor.Stats`; thresholds and divisors are configurable. Bus: `Net.TierChanged(player, tier, stats)`.

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

Fork-join turns one batch into balanced parallel work — 50 NPC cover matrices while pathfinding recomputes a region, without choking a single thread:

```lua
-- Worker fn runs per item: return function(request, context) ... end
local Ok, Results = Pool:forkJoin(CoverRequests, { Context = { Grid = GridVersion } })
-- Results[i] corresponds to CoverRequests[i]; compose phases by forking again
```

The batch splits into chunks (default 4 waves per actor) and every actor pulls the next chunk the moment it finishes its last, so uneven items self-balance — a slow chunk never idles the rest of the pool. Results stitch back in item order and the calling coroutine resumes once with `(true, results)` or `(false, firstError)`; a failed item reports its absolute index, new sends stop, and outstanding chunks drain.

### Recycle scratch tables in hot loops

Per-frame sweeps that build temporary arrays feed the garbage collector. TablePool hands the same tables back instead:

```lua
local Scratch = TablePool.acquire()
CharacterIndex:radiusInto(origin, 40, Scratch) -- Spatial's Into variants append, zero allocation
consume(Scratch)
TablePool.release(Scratch) -- cleared, capacity kept for the next acquire
```

NPCKit target prefilters, Projectiles candidate sets, and NetGovernor stats already ride the pool internally. Rules: a released table must not be held, double release fails loud, never release a table that escaped to a caller.

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

### Group players into parties

PartyKit is same-server parties: invites with expiry, one party per player, leader flow, and roster pushes so party UI is a render problem:

```lua
local Parties = PartyKit.attach(Kernel, { MaxSize = 4, InviteTtlSeconds = 60 })
Kernel.Bus:subscribe("Party.Joined", function(_, party, player) ... end)
```

Inviting while partyless creates the party on the first accept with the inviter as leader. Accepting leaves the current party. Leaders kick and promote; a leaving leader promotes the longest-standing member; the last member out disbands. Session end leaves the party and clears the player's invites in both directions. Clients drive everything through six fail-closed intents (`PT_Invite/Accept/Decline/Leave/Kick/Promote`, UserIds on the wire) and receive `PT_Sync` roster pushes plus `PT_Invited` notifications. Parties are NOT teams — they never touch the combat gates; `sameParty(a, b)` exists for games that want party friendly-fire, and `members(player)` feeds Matchmaking group queues (a partyless player is a party of one). Gate invites in the `Party.CanInvite` hook (blocklists, level requirements). Bus: `Party.Created/Disbanded/Joined/Left/Kicked/Invited/Promoted`.

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
local Moves = MoveKit.attach(Kernel, { Net = Net })
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

### Author attachments in the map

Mounts turns Studio tagging into engine configuration. Tag an instance `CKMount`, set attributes, and the systems wire themselves at level load:

```lua
-- Server (Bootstrap): Zone mounts feed the zones service
Mounts.attach(Kernel, { Zones = Regions })

-- Client: the same tagged map wires audio emitters and physics chains
Mounts.attach(Kernel, {
	AudioKit = Audio,
	BonePhysics = Bones,
	Handlers = {
		Torch = function(instance, attributes) -- custom types are one function
			local Flame = spawnFlame(instance, attributes.Color)
			return function() Flame:Destroy() end -- cleanup on unmount
		end,
	},
})
```

A waterfall part with `MountType = "Sound"` and `SoundName = "Waterfall"` becomes a positional emitter; a doorway with `MountType = "Zone"` and `ZoneName = "Vault"` feeds `Zones:addPart`; a chain bridge part with `MountType = "BoneChain"` binds its bones into BonePhysics with per-instance `Damping`/`Stiffness`. Built-ins mount only when their system is passed, so the server attaches with Zones and the client with AudioKit and BonePhysics off one tagged map. Tag add/remove signals mount and unmount live, which makes StreamingEnabled maps work for free: what streams in mounts, what streams out cleans up. Bus: `Mount.Added(instance, mountType)`, `Mount.Removed(instance, mountType)`.

### Query neighbors without touching the DataModel

Spatial is a flat spatial-hash index: services register ids at positions and broad-phase queries replace full-roster scans and speculative raycasts:

```lua
local Index = Spatial.new({ CellSize = 8 })
Index:set(entity, position) -- same-cell moves are just a write
local Nearby = Index:radius(origin, 40)
local Visible = Index:cone(eye, facing, 60, 120) -- radius + horizontal FOV dot
local Closest, Distance = Index:nearest(origin)

-- Kernel-wired: player character roots refresh at 10Hz
local Characters = Spatial.characters(Kernel)
Projectiles.attach(Kernel, { Spatial = Characters }) -- per-projectile radius instead of full scans
NpcKit = NPCKit.attach(Kernel, { Spatial = Spatial.new() }) -- update() refreshes it, target scans broad-phase through it
```

The grid is pointer-free (packed integer cell keys, exact-filtered queries, coordinates clamp at half a million studs) and every query touches only the cells the volume covers — with hundreds of entities moving, sensor sweeps and threat scans read the index and spend engine raycasts on the survivors only. NPCKit pads its prefilter by one refresh of drift so a stale index never hides a real candidate, and Projectiles reads fresh root positions for the selected candidates so hit math stays exact.

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

NPCs also have **ears**. Player noise alerts them to a location, with accuracy scaled by skill — `Perfect` pinpoints the exact position, `Bad` hears a vague 32-stud blur:

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

### Rig hair automatically and make it sway

HairKit gives boneless catalog hair a real skeleton at runtime — EditableMesh in, hardware-skinned mesh out:

```lua
-- Client bulk manager: rigs every character's hair once, sway rides BonePhysics
local Bones = BonePhysics.attach()
HairKit.attach(kernel, { BonePhysics = Bones })

-- Or rig server-side so the boned mesh replicates to everyone once
HairKit.server(kernel)
```

The pipeline is write-once, read-many: `plan()` partitions the mesh vertices into radial sectors and depth bands and emits a bone chain per sector (defaults: 3 sectors x 3 segments + root = 10 bones), every vertex weighted to at most 2 bones (`MaxInfluences`, clamped to 4); `rig()` injects the skeleton through `EditableMesh:AddBone`/`SetVertexBones`/`SetVertexBoneWeights`, registers the content into the DataModel with `AssetService:CreateDataModelContentAsync(Content.fromObject(mesh))`, bakes through `CreateMeshPartAsync`, swaps the skinned result into the MeshPart in place, and creates the Bone instance skeleton the mesh links to by name — the GPU deforms it natively and the baked editable stays rooted but is never mutated again. Sway is BonePhysics driving those Bone instances: one fixed-timestep stepper for every character, camera-distance culling, and the DeviceBench `MaxChains` budget, with zero per-accessory scripts and zero per-frame network traffic in either mode. Bus: `Hair.Rigged(character, meshPart, boneNames)`.

Two independent permission gates apply. The experience setting **Allow Mesh & Image APIs** (Game Settings > Security) gates every EditableMesh call; `supported()` probes it with a real `AddBone` call and caches the result. `CreateEditableMeshAsync` additionally requires experience-owned or creator-owned meshes — third-party catalog hair rejects with "no permission to load asset" and `rig()` returns `(false, reason)` for that accessory only. Either failure keeps the `bindAccessory` pendulum fallback, so use owned hair assets for full vertex-skinned sway.

### Dissolve things into drifting voxels

Dissolve is a vertex-sampled voxelizer with noise-advected scatter (client VFX):

```lua
local Vfx = Dissolve.new()
Vfx:play(banishedNpc.Model, { Duration = 1.5, Drift = Vector3.new(0, 8, 0), Spin = true })

-- Manual mode: drive erosion yourself (proximity dissolves, breaking wards)
local Controller = Vfx:play(wardWall, {})
Controller:setProgress(0.5) -- half the surface eroded, noise-thresholded
```

MeshParts sample EditableMesh vertex positions (plain parts and denied meshes sample a surface shell — Ball and Cylinder filter to their true silhouette and interior cells drop, so nothing dissolves as a filled box), points snap to a `DotSize` voxel grid and dedupe per cell, and each dot advects through a 3D Perlin field (`noiseVelocity = field sample * ScatterSpeed + Drift`) while fading out. `Reverse = true` assembles the target out of drifting voxels instead. Home positions ride the target's anchor part (PrimaryPart or the part itself), so reassembly and morph landings follow a moving target — a player who walks off mid-effect reforms where they are, not where they stood.

Rest state is free. Settled dots, fully faded dots, and landed morph dots skip their frame entirely (a standing manual controller costs comparisons, not allocations), and a manual controller fully at rest for `HibernateSeconds` (default 2) releases its dots, trims the pool to `PoolKeep`, and shows the real target again — the next `setProgress` above zero re-materializes instantly from the cached sample. A proximity ward costs zero instances and zero allocation until someone walks up to it. Dots above the erosion threshold ease back to their surface cell and resolidify (`ReturnSpeed`), so a manual controller restructures the target when its driver recedes — walk away from the proximity ward and it knits itself shut. Performance follows the framework rules: one Heartbeat stepper for every active effect, pooled dots written through `workspace:BulkMoveTo`, and a `MaxPoints` budget that scales 500..5000 by DeviceBench quality when unconfigured. `play()` hides the target locally only — broadcast the trigger through a state channel and play on every client for shared VFX.

Morphs turn one thing into another through the same particles — avatars included:

```lua
-- Disintegrate an avatar and reform it. Chunks keep the original detail
Vfx:play(character, { Duration = 1.6, Spin = true, DotSize = 0.3, Chunks = true, OnDone = reform })

-- Morph an avatar into a statue, then back, then reform the statue
Vfx:morph(character, golemStatue, { Duration = 1.6, OnDone = function()
    Vfx:morph(golemStatue, character, { OnDone = function()
        Vfx:play(golemStatue, { Reverse = true }) -- the statue was the return flight's source
    end })
end })
```

`morph(source, destination)` voxelizes the source and flies every dot into the destination's sampled shape: pairs assign bottom-up (`pairPoints`, every destination cell used), launches stagger by dot key (`Stagger`), flights arc through Perlin turbulence (`Turbulence`) and lerp source color to destination color, then the destination reveals (`Reveal`) and the source stays hidden. Character targets hide Decals and Textures with their parts, so faces and clothing vanish cleanly. Positions sample once at call time.

Avatars are catalog meshes the experience cannot editable-load, so their voxel sampling falls back to bounding-box grids — blocky silhouettes. `Chunks = true` fixes that on both `play()` and `morph()`: every source part clones as a full-detail flying chunk (meshes, textures, faces, SurfaceAppearance survive; joints, welds, sounds, and emitters strip) that rides the same noise field or morph flight while shrinking into the dust. Detail stays perfect at launch and melts into particulate instead of starting squared. Chunks respect the erosion threshold, so partial dissolves and restructures work on them too. On meshes the experience owns, vertex sampling already yields detailed voxel shells without chunks.

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
	if key == "DashKeys" then Input:rebind("Dash", "Keyboard", toKeyCodes(value)) end
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
