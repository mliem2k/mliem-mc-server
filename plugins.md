# Installed plugins

Pulled from the live server (`/var/lib/pufferpanel/servers/mliem/plugins/`) on 2026-07-18.
Versions below are only listed where the jar filename or a matching entry in Paper's
plugin-remap cache (`plugins/.paper-remapped/index.json`) confirmed one; plugins
without a top-level jar in `plugins/` are marked unconfirmed rather than guessed
(their jar may be loaded from elsewhere, this wasn't tracked down).

| Plugin | Version | Source | Notes |
|---|---|---|---|
| Purpur | 1.21.4 | https://purpurmc.org | Server core, downgraded from 1.21.10 during the LibreLogin/PacketEvents fix, see the root `README.md` |
| AuthMe | 5.7.0 | https://www.spigotmc.org/resources/authmereloaded.6269/ | Disabled (`.jar.disabled`), replaced by LibreLogin. Kept on disk, not deleted |
| LibreLogin | 0.23.1 | https://github.com/kyngs/LibreLogin/releases | Active auth plugin. 0.24.0 also present but disabled, see root README for why 0.23.1 is pinned |
| LuckPerms | 5.5.59 | https://luckperms.net | Permissions, storage-method h2 (local, no remote DB). Three groups as of 2026-07-19: `default` (base survival, aqua `§b` name color), `moderator` (inherits default + kick/mute/tp, yellow `§e`), `admin` (inherits moderator + wildcard `*` permission, red `§c`). Group data lives in LuckPerms' own H2 database, not a file in this repo |
| EssentialsX (+ AntiBuild, Chat, Protect, Spawn) | 2.22.0 | https://essentialsx.net | |
| FastAsyncWorldEdit | 2.15.3 (Paper build) | https://github.com/IntellectualSites/FastAsyncWorldEdit | |
| Geyser-Spigot | unpinned, updated 2026-07-18 | https://geysermc.org | Bedrock protocol translation |
| Floodgate | unpinned, updated 2026-07-18 | https://github.com/GeyserMC/Floodgate | Bedrock auth bypass, companion to Geyser |
| ViaVersion | unpinned, updated 2026-07-18 | https://github.com/ViaVersion/ViaVersion | Multi-version support |
| SkinsRestorer | unpinned, updated 2026-07-10 | https://skinsrestorer.net | |
| XferHelper | 1.0.1 | https://github.com/mliem2k/XferHelper (this account's own plugin) | Exposes the Java Transfer packet as a console command, see `infra/` submodule |
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
