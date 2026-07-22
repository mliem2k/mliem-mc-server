# Installed plugins

Pulled from the live server (`/var/lib/pufferpanel/servers/mliem/plugins/`) on 2026-07-18,
with corrections and config changes applied through 2026-07-20 (activation-status audit,
Geyser/TAB/LuckPerms/Essentials changes) — see individual rows and the root `README.md`
for what changed and when.
Versions below are only listed where the jar filename or a matching entry in Paper's
plugin-remap cache (`plugins/.paper-remapped/index.json`) confirmed one; plugins
without a top-level jar in `plugins/` are marked unconfirmed rather than guessed
(their jar may be loaded from elsewhere, this wasn't tracked down).

| Plugin | Version | Source | Notes |
|---|---|---|---|
| Purpur | 1.21.4 | https://purpurmc.org | Server core, downgraded from 1.21.10 during the LibreLogin/PacketEvents fix, see the root `README.md` |
| AuthMe | 5.7.0 | https://www.spigotmc.org/resources/authmereloaded.6269/ | **Active auth plugin** (corrected 2026-07-19; previously documented as disabled/replaced, which was wrong). Confirmed via live `/plugins` list and in-game login flow |
| LibreLogin | 0.23.1 | https://github.com/kyngs/LibreLogin/releases | **Disabled** (`.jar.disabled`), confirmed 2026-07-19. 0.24.0 also present, also disabled. Pinned to 0.23.1 to dodge a known PacketEvents bug, but never actually re-enabled after that fix; AuthMe has been the real auth plugin since. See root README |
| LuckPerms | 5.5.59 | https://luckperms.net | Permissions, storage-method h2 (local, no remote DB). Three groups as of 2026-07-19: `default` (base survival, aqua `§b` name color), `moderator` (inherits default + kick/mute/tp, yellow `§e`), `admin` (inherits moderator + wildcard `*` permission, red `§c`). Group data lives in LuckPerms' own H2 database, not a file in this repo |
| EssentialsX (+ AntiBuild, Chat, Protect, Spawn) | 2.22.0 | https://essentialsx.net | |
| FastAsyncWorldEdit | 2.15.3 (Paper build) | https://github.com/IntellectualSites/FastAsyncWorldEdit | |
| Geyser-Spigot | unpinned, updated 2026-07-18 | https://geysermc.org | Bedrock protocol translation. `auth-type` fixed 2026-07-19: was `online` (required a linked Java/Microsoft account, defeating the point of Floodgate), now `offline` so any Bedrock/Java client can join regardless of account ownership, matching this server's cracked-friendly auth model. Also: `advanced.cache-images` 0 → 7 (days) so skins survive the scale-to-zero relay's reconnects, `gameplay.nether-roof-workaround` false → true |
| Floodgate | unpinned, updated 2026-07-18 | https://github.com/GeyserMC/Floodgate | Bedrock auth bypass, companion to Geyser |
| ViaVersion | unpinned, updated 2026-07-18 | https://github.com/ViaVersion/ViaVersion | Multi-version support |
| SkinsRestorer | unpinned, updated 2026-07-10 | https://skinsrestorer.net | |
| XferHelper | 1.0.5 | https://github.com/mliem2k/XferHelper (this account's own plugin) | Exposes the Java Transfer packet as a console command, see `infra/` submodule. **History**: a 1.0.1 fix (onPing crashing with `IllegalArgumentException` for any player once the combined "name \| path \| address" profile string exceeded 16 characters, effectively guaranteed for real usernames) was pushed to GitHub 2026-07-17 but never actually deployed, the live server silently kept running the old broken 1.0.0 build for 3 days. Caught and properly redeployed 2026-07-20 alongside a new `/relaystatus` command (open to everyone, reports live relay/catcher/home-server/DNS status in chat by querying the same API the `/mc` web page uses). **1.0.2 (2026-07-22)**: fixed a real, live-confirmed bug where `Player#transfer()` doesn't close the old connection server-side, leaving an unbounded race window in which the reconnecting client's own login got rejected as a duplicate ("The same username is already playing on the server!"), sometimes fully disconnecting the player instead of landing them on the fast route. Now explicitly kicks the old connection synchronously right after firing the transfer. **1.0.3 (2026-07-22)**: added a waiting lobby (`/xferlobby <player> <start\|cancel>`, freeze and optional teleport to a configured room while the relay isn't ready yet, driven by `join_transfer_watcher.ts`) and a relay-aware dynamic MOTD (server list description changes depending on whether the fast route currently has anyone on it). **1.0.4 (2026-07-22)**: a real waiting room was built (world/0.5/251/0.5, high in the sky above spawn, 7x7 footprint with a glass band and glowstone ceiling light), config now points there so `/xferlobby start` actually teleports instead of just freezing in place. **1.0.5 (2026-07-22)**: fixed a real bug where a player teleported to the lobby rejoined still standing in it after a successful transfer (entity position persists through the same-JVM reconnect); now restores their original position before the transfer fires and on lobby timeout. |
| TransferTool | 3 (release-3 asset) | https://github.com/onebeastchris/TransferTool/releases | A **Geyser extension**, lives in `plugins/Geyser-Spigot/extensions/`, not a Bukkit plugin |
| WorldGuard / WorldEdit | **disabled** | https://enginehub.org/worldguard | Confirmed inactive 2026-07-19: jars live in `plugins/backup/`, not `plugins/`. FastAsyncWorldEdit (above) provides WorldEdit's functionality instead |
| GriefPrevention | **disabled** | https://www.spigotmc.org/resources/griefprevention.1884/ | Confirmed inactive 2026-07-19: jar lives in `plugins/backup/`. Data/config dir named `GriefPreventionData`, left in place |
| Vault | 1.7.3-b131 | https://www.spigotmc.org/resources/vault.34315/ | Was sitting disabled in `plugins/backup/` until 2026-07-19 despite being required for Essentials' chat-prefix/group features (Essentials logs an explicit warning without it) — moved into `plugins/` and confirmed enabled |
| PlaceholderAPI | **disabled** | https://www.spigotmc.org/resources/placeholderapi.6245/ | Confirmed inactive 2026-07-19: jar lives in `plugins/backup/`. Config dir exists but plugin never loads |
| ProtocolLib | **disabled** | https://www.spigotmc.org/resources/protocollib.1997/ | Confirmed inactive 2026-07-19: jar lives in `plugins/backup/`. Dependency for GrimAC, which is also disabled |
| GrimAC | **disabled** | https://github.com/GrimAnticheat/Grim | Confirmed inactive 2026-07-19: jar lives in `plugins/backup/`. No anticheat is currently active on the server |
| TAB | 6.1.0 (Paper 1.20.5-1.21.4 build) | https://github.com/NEZNAMY/TAB | Was sitting disabled in `plugins/backup/` as v4.1.8 until 2026-07-19. Moved into `plugins/`, then upgraded to 6.1.0 after 4.1.8 logged "No fallback solution was found" NMS compatibility errors against Purpur 1.21.4; 6.1.0's paper_1_21_4-specific build loads with a real native implementation instead. MySQL storage present but disabled (unused stock config) |
| AuraSkills | 2.2.3, **disabled** | https://github.com/Archy-X/AuraSkills | RPG skills. Jar lives in `plugins/backup/`, not `plugins/`, so Paper never loads it; confirmed inactive via the boot log (zero "Enabling AuraSkills" lines). Config dir left in place, not deleted |
| MythicLib / MMOItems / MMOInventory | 1.6.2 / 6.10 / 1.8.1, **disabled** | https://gitlab.com/phoenix-dvpmt/mythiclib , https://www.mmoitems.net | Same as AuraSkills: all three jars live in `plugins/backup/`, confirmed inactive via the boot log. Config trees (the bulk of `plugins/MMOItems/`) left in place |
| BetonQuest | **disabled** | https://betonquest.org | Same as above, jar in `plugins/backup/`, confirmed inactive |
| Multiverse-Core / -NetherPortals / -Portals | unconfirmed | https://github.com/Multiverse | Multi-world management |
| My_Worlds / BKCommonLib | unconfirmed | https://www.spigotmc.org/resources/my-worlds.39594/ | Per-world inventories, requires BKCommonLib |
| Codex | unconfirmed | https://www.spigotmc.org/resources/codex.99043/ | MySQL storage present but disabled (unused stock config) |
| Chunky | unconfirmed | https://github.com/pop4959/Chunky | Pregeneration |
| MiniMOTD | unconfirmed | https://github.com/jmp-lockheed/minimotd | Server list MOTD |
| TreeAssist | unconfirmed | https://www.spigotmc.org/resources/treeassist.6934/ | |
| KnockbackSync | unconfirmed | https://www.spigotmc.org/resources/knockbacksync.91527/ | |
| LushRewards | unconfirmed | https://polymart.org/resource/lushrewards | Daily/playtime rewards, JSON storage (MySQL present but disabled) |
| Prism | unconfirmed | https://github.com/Prism-Data/Prism | Block-change logging, **actually uses local MySQL** (`127.0.0.1`), password redacted in `plugins/Prism/config.yml` |
| bStats | n/a | https://bstats.org | Metrics library bundled by several plugins above, not a standalone plugin |

## Not included in this repo

- Plugin jars themselves (binary, some are paid/licensed resources, not meaningfully
  version-controllable).
- World data (`world/`, `world_nether/`, `world_the_end/`, `limbo/`), `logs/`,
  `libraries/`, `versions/` (Purpur/Paper server jar cache).
- Player-identifying files: `usercache.json`, `whitelist.json`, `ops.json`,
  `banned-players.json`, `banned-ips.json`, `version_history.json`.
- Per-player data: `plugins/*/playerdata`, `*/userdata`, FastAsyncWorldEdit session
  files.
- `plugins/floodgate/key.pem` (Floodgate's Bedrock auth encryption key), any other
  `.pem`/`.key`/`.jks`/`.p12` files.
- Backup directories (`*-backup-*`, `*.jar.bak*`) and Paper's plugin-remap cache
  (`.paper-remapped/`).

## Redacted values

Two live secrets were found in the pulled configs and replaced with placeholders
before anything was committed:

- `server/server.properties`: `management-server-secret` (auto-generated by the
  server; the management-server feature itself is disabled, `management-server-enabled=false`).
- `plugins/Prism/config.yml`: the local MySQL password for Prism's block-logging
  database (`127.0.0.1`-only, but a real active credential, unlike the other
  plugins' unused stock-default DB passwords which were left as-is since they're
  identical across every fresh install of those plugins and the storage backends
  using them are disabled).
