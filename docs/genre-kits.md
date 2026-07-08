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

All kit validation lives in the same fail-closed hook chains (`Intent.WK_Fire`, `Intent.SK_Cast`, `Intent.Checkpoint`, `Intent.Interact`, `Intent.Settings_Set`) — prepend your own rules at lower priority numbers without forking the kit. Beyond the combat kits: [QuestKit](cookbook.md#track-quests-from-things-that-happen) (declarative objectives off bus topics), [InteractionKit](cookbook.md#let-players-interact-with-the-world) (exploit-proof prompts), and [NPCKit](cookbook.md#give-npcs-brains-aim-skill-and-movesets) (behavior lists, difficulty presets, player-unified movesets).

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
