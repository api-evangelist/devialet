---
name: Control playback on a Devialet group
description: >-
  Select a source, play, pause, mute, unmute, and skip tracks on a Devialet group, handling the
  asynchronous play semantics, the independent play/mute state machines, and the operations that
  are not safe to retry.
api: openapi/devialet-ip-control-openapi.yml
operations:
  - listGroupSources
  - getGroupCurrentSource
  - playGroupSource
  - pauseGroupPlayback
  - muteGroupPlayback
  - unmuteGroupPlayback
  - nextGroupTrack
  - previousGroupTrack
generated: '2026-08-04'
method: generated
source: openapi/_original/devialet-ip-control-r1.pdf
---

# Control playback on a Devialet group

Playback lives at the **group** level — the set of systems currently playing the same content.
Volume and audio settings live at the *system* level; do not look for playback there.

Run `devialet-discover-installation` first. You need a reachable device address and the group's
`sourceId` values before anything here works.

## Two independent state machines

This is the detail most integrations get wrong. `playingState` (`playing` / `paused`) and
`muteState` (`muted` / `unmuted`) are **independent**. All four combinations are legal. Changing
one never changes the other — with two documented exceptions:

- Every volume command forces `muteState` to `unmuted`, and never touches `playingState`.
- Pausing a source that cannot semantically pause (for example `optical`) **mutes** instead:
  `playingState` stays `playing` and `muteState` becomes `muted`.

Never infer one state from the other. Read both from `getGroupCurrentSource`.

## Selecting and playing a source

1. Call `listGroupSources` (`groupId` = `current`) and pick a `sourceId`.
2. Call `playGroupSource` with that `groupId` and `sourceId`.

If the source is not already current it is selected first, and the previous source is set to
`paused` and unselected. Paused sources keep their context, so you can switch away and pull the
playback back later.

**The response does not tell you it worked.** Devialet states this explicitly: the call may have
asynchronous effects, and a delayed failure produces no notification. After a reported success,
re-read `getGroupCurrentSource` and check `playingState` before you update any UI or report success
to a user.

If the body carries `error.code` = `PlaybackNoStream`, playback could not start — no cable on a
sensing input, no lock on a digital input, an audio session that could not be re-established (a
torn-down AirPlay session, an unreachable Bluetooth or UPnP source, an expired or revoked Spotify
Connect token), or an unsupported format. The designated source **still becomes current**, silently.
Tell the user which physical input or service needs attention rather than retrying.

`playGroupSource` is safe to repeat: if the designated source is already the current, playing
source, the call succeeds.

### The AirPlay 2 / Roon Ready caveat

Selecting an `airplay2` or `raat` source removes the host system from its current group, because
Apple and Roon implement multi-room grouping themselves. If the system was alone in its group it
gets a **new `groupId` on every audio session**. If it held the group master, the group is
destroyed and every remaining system moves to its own group. Re-run discovery after touching these
sources; any cached `groupId` is stale.

## Pause, mute, unmute

- `pauseGroupPlayback` — sets `playingState` to `paused`. Every source accepts it. Succeeds if
  already paused.
- `muteGroupPlayback` — sets `muteState` to `muted`. Succeeds if already muted.
- `unmuteGroupPlayback` — sets `muteState` to `unmuted`. Succeeds if already unmuted.

All three are idempotent by Devialet's own documented contract, so they are safe to re-send after a
`Timeout` or `UnreachableDevices` — though you should still re-query state, because those errors
leave the installation in an undefined condition.

`mute` and `unmute` are **always** available and are deliberately never listed in
`availableOperations`. Do not gate them on that array.

## Skipping tracks — check first, and do not retry

`nextGroupTrack` and `previousGroupTrack` are the only playback operations that are **not**
idempotent. They advance or rewind the playlist every time they land.

Before calling either, read `availableOperations` from `getGroupCurrentSource` and confirm `next`
or `previous` is present. The array changes without the source changing — the last track in a
playlist, or a free Spotify account, will drop `next`. Calling anyway reports
`PlaybackOperationNotAvailable`.

After a timeout or an ambiguous failure on these two, **do not blindly retry.** Re-read
`getGroupCurrentSource` and compare `metadata` to decide whether the skip landed.

On DOS 2.14.x, `availableOperations` is known to be wrong for the `upnp` source — it lists
`previous` and `next` when they may not work, and those operations do not behave as expected. Check
`release.version` from `getDevice`; this is fixed in DOS 2.16.x.

"Restart the current track when the user presses previous mid-track" is not supported by the API.
Implement it client-side if you want it.

## Request mechanics

Every command here is a POST with no parameters. The body may be empty or `{}`, but the header
`Content-Type: application/json` is **mandatory** — omit it and you get HTTP 415. A successful
command returns `{}`.

Check the 200 body for an `error` key before treating any call as successful. The full code list is
in `errors/devialet-error-codes.yml`.

## Errors you will actually hit

| `error.code` | Cause | Response |
|---|---|---|
| `NoCurrentSource` | No source selected on the group | Call `listGroupSources` and select one |
| `PlaybackNoStream` | Source cannot produce audio | Source is now current but silent — surface the cable/service problem |
| `PlaybackOperationNotAvailable` | `next`/`previous` not offered | Re-read `availableOperations` |
| `UnreachableSource` | The device hosting the source is offline | Re-query `listGroupSources` |
| `UnreachableDevices` | Some group devices unreachable | Command may have partially applied — re-query |
