# XferHelper: waiting lobby + relay-aware MOTD

**Date:** 2026-07-20
**Status:** Approved

## Overview

Two related additions to XferHelper (`mliem2k/XferHelper`), designed together since both
extend existing hooks rather than adding new ones, and both are things only a Paper
plugin can do (unlike the real-time player-event tracking covered in a separate
mliem-landing-side design, which moved to a plain Bun log watcher instead):

1. **Waiting lobby**: a player who joins while the relay isn't ready yet gets frozen in
   a small dedicated room with a waiting message, instead of playing freely on the
   catcher path during the wait. Chat still works.
2. **Relay-aware MOTD**: the server list ping shows a different MOTD depending on
   whether the fast route (relay) currently has anyone on it, extending the existing
   `onPing` hook that already rewrites the player sample.

## Context: what exists today

`join_transfer_watcher.ts` (Bun, home server) tails the server log for `"X joined the
game"`, and for a genuine new join (not a rejoin within `COOLDOWN_MS` of its own prior
transfer), polls relay reachability every 5s for up to 2 minutes. The instant it's
reachable, it shells out to `transfer_one.ts`, which logs into PufferPanel's local API
(`http://localhost:8080`, credentials at `/root/pufferpanel-admin-pass.txt`) and POSTs
`xfer <player> <host> <port>` to the server console. XferHelper's existing `/xfer`
command handler (`notifyAndTransfer`) then freezes the player (cancels movement,
interaction, block break/place, damage — chat is untouched), shows a title/subtitle and
a live countdown action bar for `delay-ticks` (default 40 = 2s), then sends the real
Transfer packet.

Today, a player who joins while the relay isn't ready yet just plays normally on the
catcher path for however long the 2-minute polling window takes — nothing restricts
them, and nothing tells them a faster connection is on its way until the last 2 seconds
right before the actual transfer.

## Goals

1. Freeze a player (movement/interaction/combat/building, chat unaffected) the moment
   `join_transfer_watcher.ts` determines the relay isn't immediately ready, not just in
   the final 2-second countdown before the real transfer.
2. Teleport them to a small, purpose-built waiting room rather than freezing wherever
   they happened to spawn.
3. Release them cleanly either when the real transfer fires (seamless handoff into the
   existing countdown) or when the relay never becomes ready within the timeout
   (release back to normal play on catcher, not stuck frozen forever).
4. Show a different MOTD depending on whether the relay currently has any player on it.

## Non-Goals

- Building the waiting room itself is not this plugin's job — the room's location is
  config, construction is a one-time in-game task for whoever administers the server
  (FastAsyncWorldEdit is already installed and active for this).
- No change to the real-time player-event tracking design (separate spec, separate
  repo side) — this design doesn't touch or depend on it.
- No persistence of "who is currently in the lobby" across a plugin/server restart —
  if the server restarts mid-wait, the player simply reconnects and the join watcher's
  logic runs fresh for them, same as any other join.
- No queueing/rate-limiting of the MOTD's relay-state check — it's computed fresh on
  every ping from already-in-memory state (`Bukkit.getOnlinePlayers()` plus the
  existing `playerPath` map), which is cheap enough to not need caching.

## Waiting lobby design

### Config (`config.yml`, new section)

```yaml
lobby:
  # Leave world empty ("") to disable teleporting and just freeze players in place
  # (matches the pre-transfer countdown's existing behavior) until you've built a
  # room and filled these in.
  world: ""
  x: 0.0
  y: 64.0
  z: 0.0
  yaw: 0.0
  pitch: 0.0
  title: "&b&lFinding you a faster route"
  subtitle: "&fHang tight, this won't take long"
  waiting-message: "&7Please wait, you're free to chat in the meantime."
  released-message: "&7Connection didn't improve in time — you're free to move now."
```

### XferHelper changes (Java)

**New command** `/xferlobby <player> <start|cancel>`:

- `start`: if `lobby.world` is non-empty and resolves to a loaded world, teleport the
  player there first; then freeze them (reuses the existing `frozen` set, so all the
  current movement/interact/break/place/damage cancellation already applies with zero
  new code there) and show `lobby.title`/`lobby.subtitle` via `showTitle` with a long
  duration (no numeric countdown — the wait length is unknown), plus a one-time chat
  message using `lobby.waiting-message`.
- `cancel`: unfreeze the player (reuses the existing `unfreeze(uuid)`) and send
  `lobby.released-message` to chat.

**Fix needed in `freeze()`**: it currently does `countdownTasks.put(uuid, ...)` without
cancelling any task already in the map for that `uuid` first. This was harmless before
(freeze was only ever called once per transfer), but the lobby flow means `freeze()` can
now be called a second time for the same player (once for `/xferlobby start`'s
indefinite wait, again for `/xfer`'s final short countdown) — without a fix, the old
lobby task would keep running and fight the new countdown's action-bar updates. Fix:
`freeze()` cancels any existing `countdownTasks` entry for the `uuid` before starting a
new one, same as `unfreeze()` already does.

**`onQuit`**: add `unfreeze(uuid)` if not already effectively covered (it already calls
this today — confirmed no change needed here beyond what exists).

### `join_transfer_watcher.ts` changes (Bun, `infra/home-server/`)

Restructured `handleJoin`: check relay reachability once *before* entering the polling
loop. If it's already reachable, transfer immediately exactly as today (no lobby —
there's no meaningful wait to show one for). If not, fire `/xferlobby <player> start`
before starting the polling loop; on success inside the loop, the existing `/xfer` call
naturally supersedes the lobby state (per the `freeze()` fix above); on exhausting the
loop without success, fire `/xferlobby <player> cancel` instead of just logging and
returning as today.

```ts
async function sendConsoleCommand(command: string): Promise<void> {
  // same PufferPanel-login-then-POST-/console pattern transfer_one.ts already uses
}

async function handleJoin(player: string) {
  if (recentlyTransferred(player)) {
    logger(`join_transfer_watcher: ${player} rejoined within cooldown of its own transfer, skipping (not a new session)`);
    return;
  }
  logger(`join_transfer_watcher: ${player} joined, checking relay readiness`);

  const firstIp = await resolveBackend();
  if (firstIp && isReachable(firstIp)) {
    await doTransfer(player, firstIp);
    return;
  }

  await sendConsoleCommand(`xferlobby ${player} start`);

  for (let i = 0; i < 23; i++) {
    await Bun.sleep(5000);
    const ip = await resolveBackend();
    if (ip && isReachable(ip)) {
      await doTransfer(player, ip);
      return;
    }
  }

  logger(`join_transfer_watcher: relay never became reachable within 2min for ${player}, releasing from lobby`);
  await sendConsoleCommand(`xferlobby ${player} cancel`);
}
```

(`doTransfer` factors out the existing `lastTransferred.set(...)` + `transfer_one.ts`
invocation + logging, unchanged in behavior, just named so both call sites — immediate
and post-poll — share it instead of duplicating it.)

## Relay-aware MOTD design

### Config (`config.yml`, new keys alongside the existing `motd`-adjacent settings)

```yaml
dynamic-motd:
  enabled: true
  relay-active: "&a&lFast route active&r &7- you're all set"
  relay-inactive: "&e&lWaking up the fast route...&r &7- playable now, faster soon"
```

### XferHelper changes (Java)

Extend the existing `onPing` handler (`PaperServerListPingEvent`): after the current
player-sample rewrite logic, if `dynamic-motd.enabled`, compute whether the relay is
"active" by checking if any *currently online* player's `playerPath` entry equals
`transfer-path-label` (`Bukkit.getOnlinePlayers().stream().anyMatch(p ->
transferLabel.equals(playerPath.get(p.getUniqueId())))`) — deliberately scoped to
online players only, not just "does the map contain the label anywhere," since
`playerPath` entries aren't cleaned up on quit today and checking only online players
sidesteps that staleness without needing to add cleanup logic. Set
`event.motd(...)` (Adventure Component, same `legacy()` helper already used for
titles) to `dynamic-motd.relay-active` or `dynamic-motd.relay-inactive` accordingly.

This only affects the MOTD Geyser/vanilla clients see in the multiplayer server list
via SLP — it has no interaction with the `expose-player-metadata` player-sample
rewrite already happening in the same handler.

## Testing

- Manual only for both features (matches this plugin's existing testing posture, no
  test suite in this repo).
- Waiting lobby: join while relay is cold, confirm teleport + freeze + waiting message
  + chat still works; confirm seamless handoff into the existing short countdown once
  the relay becomes ready; confirm release-with-message if forced to wait the full 2
  minutes (can be tested by temporarily lowering the loop count for a dry run).
- MOTD: check the server list entry (from a real client, or a raw SLP status query)
  both while nobody is on the relay path and while someone is, confirm it differs.
