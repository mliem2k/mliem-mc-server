# mliem-mc-server

The actual configuration for `mc.mliem.com`, a home-hosted Purpur Minecraft server
that supports Java, Bedrock (mobile/console), and cross-play, reachable through a
scale-to-zero relay when nobody's connected from outside the LAN.

This repo is the real thing: actual `server.properties`, actual plugin configs,
actual item/skill/quest definitions, pulled straight from the live box. It's the
companion to [mc-serverless-proxy](https://github.com/mliem2k/mc-serverless-proxy),
which documents the generic wake-on-demand catcher/relay architecture with
placeholder values so it stands alone as a reference implementation. That repo is
included here as the `infra/` submodule.

## Layout

- `server/` — server-root config: `server.properties`, `purpur.yml`, `spigot.yml`,
  `bukkit.yml`, `commands.yml`, `help.yml`, `permissions.yml`, `wepif.yml`,
  `eula.txt`, plus Paper's `config/paper-global.yml` and
  `config/paper-world-defaults.yml`. No per-world overrides exist (each world's
  `paper-world.yml` was just the empty default template, not worth committing).
- `plugins/` — every installed plugin's own config directory (not the jars
  themselves, see `plugins.md` for why and where to get them).
- `plugins.md` — the full plugin list: versions where confirmable, source links,
  and what got deliberately left out (player data, world data, secrets).
- `infra/` — git submodule, [mc-serverless-proxy](https://github.com/mliem2k/mc-serverless-proxy):
  the catcher/relay wake-on-demand infrastructure, the custom UDP relay for Bedrock,
  Terraform/provisioning scripts, all with placeholder values instead of this
  server's real ones.

## The stack, briefly

Purpur 1.21.4 (`purpur.yml`), `online-mode=false` (auth is handled in-plugin, see
below), behind [XferHelper](https://github.com/mliem2k/XferHelper) for the
catcher/relay handoff and Geyser + Floodgate + ViaVersion + TransferTool for
Bedrock/cross-version support. Auth is LibreLogin 0.23.1 (pinned, not the latest,
see `infra/README.md`'s "Bedrock/mobile cross-play" section for the specific
upstream PacketEvents bug that forced that pin). AuthMe is still on disk, disabled,
kept as a fallback rather than deleted.

Gameplay-wise this is a heavily modded survival server: MythicLib/MMOItems for
custom items and skills (the bulk of `plugins/MMOItems/`), AuraSkills for RPG-style
leveling, BetonQuest for quest scripting, GrimAC for anticheat, LuckPerms for
permissions, and the usual EssentialsX/WorldGuard/WorldEdit/Multiverse baseline.
Full list with versions and source links in `plugins.md`.

## Cloning

```bash
git clone --recurse-submodules https://github.com/mliem2k/mliem-mc-server.git
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init
```

## What's not here, on purpose

World data, logs, player-identifying files (whitelist/ops/ban lists, playerdata),
plugin jars, and anything that turned out to be a real credential rather than a
plugin's unused stock-default value. Full breakdown, including the two secrets that
were found and redacted before the first commit, is in `plugins.md`.

This repo is a snapshot, not a live mirror: it won't auto-update when the live
server's configs change. Treat it as documentation of a point in time
(2026-07-18), not a source of truth for what's running right now.
