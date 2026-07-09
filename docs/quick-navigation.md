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
| [Group players into parties](#group-players-into-parties) | [Rig hair automatically and make it sway](#rig-hair-automatically-and-make-it-sway) |
| [Dissolve things into drifting voxels](#dissolve-things-into-drifting-voxels) | |
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
