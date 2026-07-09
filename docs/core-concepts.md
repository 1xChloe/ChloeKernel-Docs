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
