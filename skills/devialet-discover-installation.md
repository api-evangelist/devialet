---
name: Discover a Devialet installation
description: >-
  Find every Devialet device on the local network, resolve the device, system, and group topology,
  and read the current playback state — the mandatory first step before any other Devialet skill.
api: openapi/devialet-ip-control-openapi.yml
operations:
  - getDevice
  - getSystem
  - listGroupSources
  - getGroupCurrentSource
  - getSystemVolume
generated: '2026-08-04'
method: generated
source: openapi/_original/devialet-ip-control-r1.pdf
---

# Discover a Devialet installation

Devialet's IP Control API has no cloud endpoint and no directory service. Every device serves the
API itself on the local network, and the topology (which speakers form a system, which systems
form a group) exists only as identifiers you read back from the devices. You must build the
picture before you can act on it. This skill follows the sample implementation Devialet publishes
in its reference documentation.

## Before you start

- **Base URL** is `http://<device-address>/ipcontrol/v1`. Plain HTTP, port 80, no TLS.
- **No authentication.** Any client on the LAN can call anything. There is no token to acquire.
- **Timeout:** allow 1000 ms. Devialet permits up to 500 ms of on-device processing plus network
  time.
- **Error checking is not the HTTP status.** Application errors come back as HTTP 200 with an
  `error` object in the body. Always parse the body and test for an `error` key. See
  `errors/devialet-error-codes.yml`.

## Steps

### 1. Discover device addresses over mDNS

Browse for the `_http._tcp` service type. Keep only instances whose TXT record contains **both**
`manufacturer=Devialet` and `ipControlVersion=1`. Use the advertised `address`, `port`, and
`path` values — do not hard-code `/ipcontrol/v1`, because Devialet reserves the right to change it.

Do not filter on, or otherwise use, the mDNS hostname. It does not change when a device is renamed
and is not a stable contract. Do not use the mDNS service name for display either; mDNS conflict
resolution can alter it.

If mDNS is unavailable, fall back to fixed IP addresses configured on the devices or reserved on
the router. This is what Devialet recommends for custom-install deployments.

### 2. Identify each device

Call `getDevice` on **every** discovered address, with `deviceId` set to `current`.

`current` always means "the device that received this request", so calling it on each address is
how you enumerate the installation.

Record from each response:

- `deviceId`, and — for speakers only — `systemId` and `groupId`
- `release.version`, the firmware version. **This gates everything else.** DOS 2.14 gives you
  sources, playback, and volume. DOS 2.16 adds the equalizer, night mode, Bluetooth pairing, and
  power commands.
- `model`, `serial`, `role` (`FrontLeft`, `FrontRight`, or `Mono`), and `deviceName`

A response with **no** `systemId` and **no** `groupId` is an accessory (Arch or Dialog), not a
speaker. Accessories belong to no system and no group. Calling any `/systems/...` or `/groups/...`
path against one returns HTTP 404 — that is expected, not a fault.

### 3. Build the system and group lists

Group the speakers you found by `systemId`, then group the systems by `groupId`. Every speaker
belongs to exactly one system and every system to exactly one group at any moment.

### 4. Name each system

Call `getSystem` once per system, against **one** device from that system, with `systemId` set to
`current`.

Take `systemName` — this is the name to show a user ("Dining room", "Kitchen"), not the mDNS name.
Take `availableFeatures` too: it is your feature test for `equalizer` and `nightMode` on DOS 2.16+.

### 5. Enumerate sources per group

Call `listGroupSources` once per group, against one device from that group, with `groupId` set to
`current`.

Each entry has `sourceId`, the `deviceId` of the host, and a `type`. Keep the `sourceId` values —
they are the only way to select a source with `playGroupSource`.

Use `deviceId` to tell left from right on a stereo pair, since both speakers can host the same
physical input type.

### 6. Read current playback state per group

Call `getGroupCurrentSource` once per group, with `groupId` set to `current`.

Record `playingState` (`playing` or `paused`), `muteState` (`muted` or `unmuted`), `metadata` when
present, and — critically — `availableOperations`. That array is the only reliable signal for
whether `next` and `previous` will work on the current source.

If the body carries `error.code` = `NoCurrentSource`, the group simply has nothing selected. That
is a normal state, not a failure.

### 7. Read the volume per system

Call `getSystemVolume` with `systemId` set to `current`, on one device per system. Volume is
shared by every device in a system and is returned as a percentage, 0-100.

## Keeping the picture fresh

There are no webhooks, no streaming, and no subscriptions — Devialet states plainly that
asynchronous notifications are not available yet. Poll these:

- `listGroupSources` — the source list changes when devices go offline or an Arch input is
  reconfigured
- `getGroupCurrentSource` — playback state and available operations
- `getSystemVolume` — volume

Re-run discovery from step 1 whenever a device stops responding, and after any AirPlay 2 or Roon
Ready session, because those restructure group membership and can change `groupId`.

## Failure handling

| `error.code` | What it means here | What to do |
|---|---|---|
| `UnreachableDevices` | Some devices could not be reached | State of non-dispatcher devices is undefined — re-query them |
| `Timeout` | Request did not complete in time | Resulting state undefined; re-query before retrying |
| `NoCurrentSource` | Group has no source selected | Normal; offer the user the list from `listGroupSources` |

An HTTP 404 means either a genuinely unknown endpoint, a `/systems` or `/groups` call against an
accessory, or an endpoint from a newer API revision that this firmware does not have. Check
`release.version` before assuming a fault.
