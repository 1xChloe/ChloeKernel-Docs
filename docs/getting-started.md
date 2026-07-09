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

### Starter templates

[templates/](templates/) holds three complete minimal games as reference code — a team gun arena (WeaponKit, lag compensation, TeamKit, rounds, global boards), a wizard duel (SpellKit, MeleeKit parries, MoveKit double jumps, curse/ward Effects), and a checkpoint obby (CheckpointKit, air steps, kill-brick zones, speedrun boards). Each is one server module and one client module; copy the bodies into your Bootstraps. Map requirements sit at the top of each server file.

### Releases

Tagged versions ship a built `.rbxm` on the [GitHub Releases page](https://github.com/1xChloe/ChloeKernel/releases) — insert it into a place to pin an exact framework version instead of syncing `main`. Or build locally: `rojo build build.project.json -o ChloeKernel.rbxm`. Pushing a `v*` tag builds the artifact and cuts the release from the matching CHANGELOG section automatically.
