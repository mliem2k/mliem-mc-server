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
  server's real ones. Also includes `home-server/mc_status_events_watcher.ts`
  (added 2026-07-20): tails this server's own log for real-time join/leave events
  and pushes them to mliem-landing's `/mc` status page, replacing an older
  polling-based approach that could miss short sessions. Runs as a user-level
  systemd service on the home server (no root needed), independent of the
  relay-transfer watcher it shares a log-tailing pattern with.

## The stack, briefly

Purpur 1.21.4 (`purpur.yml`), `online-mode=false` (auth is handled in-plugin, see
below), behind [XferHelper](https://github.com/mliem2k/XferHelper) (v1.0.2 as of
2026-07-22; also adds an open-to-everyone `/relaystatus` command that reports live
relay/catcher/home-server/DNS status in chat) for the
catcher/relay handoff and Geyser + Floodgate + ViaVersion + TransferTool for
Bedrock/cross-version support, with Geyser's `auth-type` set to `offline` so
Bedrock/PE players can join without a linked Microsoft/Java account, matching the
server's cracked-friendly Java side. Auth is **AuthMe** (not LibreLogin): both
LibreLogin builds (0.23.1, pinned specifically to dodge a known upstream
PacketEvents bug, and 0.24.0) sit disabled in `plugins/` as of 2026-07-18 and were
never actually re-enabled after that fix was pinned, so AuthMe (kept originally as
a fallback) has been the real, live auth plugin since. Confirmed working as of
2026-07-19; deliberately left as-is rather than switching back, to not risk
retriggering that crash.

Gameplay-wise this is a modded survival server: LuckPerms for permissions (three
roles as of 2026-07-19: `default`/Member, `moderator`, `admin`), TAB for
tablist/nametag rendering and EssentialsX's chat format for role-colored player
names (aqua/yellow/red respectively, sourced from each LuckPerms group's prefix),
plus a personalized TAB header/footer (server name, real-time clock, real domain,
replacing stock placeholder text). **No anticheat is
currently active** (GrimAC and its dependency ProtocolLib both sit disabled in
`plugins/backup/`, confirmed via the live boot log). WorldGuard/WorldEdit and
GriefPrevention are also disabled; FastAsyncWorldEdit (active) provides
WorldEdit's functionality on its own. MythicLib/MMOItems (custom items and skills), AuraSkills (RPG-style leveling), and
BetonQuest (quest scripting) are present but disabled, their jars sit in
`plugins/backup/`, not `plugins/`, so Paper never loads them; config trees are left
on disk, not deleted. Full list with versions and source links in `plugins.md`.

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
(initial pull 2026-07-18, most recently corrected/updated 2026-07-20), not a
source of truth for what's running right now.
