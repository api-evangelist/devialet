---
name: Tune sound on a Devialet system
description: >-
  Set volume, equalizer, and night mode on a Devialet system, respecting the firmware feature
  gates, the system-leader requirement for writes, and the gain rounding rules.
api: openapi/devialet-ip-control-openapi.yml
operations:
  - getSystem
  - getSystemVolume
  - setSystemVolume
  - systemVolumeUp
  - systemVolumeDown
  - getSystemEqualizer
  - setSystemEqualizer
  - getSystemNightMode
  - setSystemNightMode
generated: '2026-08-04'
method: generated
source: openapi/_original/devialet-ip-control-r1.pdf
---

# Tune sound on a Devialet system

Sound lives at the **system** level — the set of speakers that always share playback state. All
devices in a system share one volume, because a system can only combine devices of the same family
and power rating. Playback commands live at the *group* level; do not look for them here.

Run `devialet-discover-installation` first.

## Two gates before you write anything

**Firmware.** Volume works from DOS 2.14. The equalizer and night mode require **DOS 2.16**. Read
`release.version` from `getDevice`, and read `availableFeatures` from `getSystem` — it lists
`equalizer` and/or `nightMode` when the system supports them. Calling an unsupported endpoint on
older firmware returns HTTP 404; that is the documented feature-detection behaviour, not a bug.

**System leader.** One device in the system hosts its settings. You can **read** settings from any
device in the system even with the leader offline, but **writing** them requires the leader to be
reachable. If it is not, the call reports `error.code` = `SystemLeaderAbsent` and nothing changes.
The leader is elected by firmware and cannot be chosen.

## Volume

Read with `getSystemVolume` (`systemId` = `current`). Returns `{"volume": <0-100>}` as a percentage.

Write with `setSystemVolume`, POSTing `{"volume": 35}`.

- Values outside 0-100 report `InvalidValue`. A fractional number is rounded to the nearest
  integer rather than rejected.
- On DOS 2.14.x, `InvalidValue` is known **not** to be implemented for invalid numerical values.
  Validate client-side; do not rely on the device to reject bad input. Fixed in DOS 2.16.x.
- Every volume command unmutes the current source. It never changes `playingState`.

`systemVolumeUp` and `systemVolumeDown` move by one step of **5%** of the range. The step is not
configurable. Overshooting clamps to 100% or 0%, and calling at the boundary still succeeds.

**Prefer `setSystemVolume` for anything automated.** The relative steps are not idempotent away from
the boundaries — a retry after an ambiguous timeout will move the volume a second time. An absolute
set is safe to repeat.

### Group volume

Group volume is an aggregate of the individual system volumes. Setting one system's volume does not
change the others, though it may change the aggregate. Setting a group's volume can move every
system at once — except a system already at zero, which is left alone. Revision 1 specifies only the
`/systems/...` volume endpoint; the group form is named in the sample implementation but never
specified, so this skill uses the system form. See `referenced_but_unspecified` in
`conventions/devialet-conventions.yml`.

## Equalizer

Requires DOS 2.16 and `equalizer` in `availableFeatures`.

Read with `getSystemEqualizer`. You get back:

- `enabled` — **read-only**. False means the equalizer is inactive because of a special audio
  processing mode. You can still change settings, but they will have no audible effect. Devialet
  recommends disabling equalizer controls in your UI when this is false. You cannot re-enable it
  through this endpoint.
- `preset` — `flat`, `custom`, or `voice`
- `currentEqualization` — per-band `frequency` (hertz, may be rounded, may be absent) and `gain`
  for the active preset
- `customEqualization` — the gains stored for the `custom` preset
- `gainRange` — `min`, `max`, and `stepPrecision`
- `availablePresets` — the authoritative preset list for this system

**Band labels are dynamic.** Documented examples are `low` and `high`, but different systems expose
different labels. Read them from `currentEqualization` — never hard-code them.

Write with `setSystemEqualizer`. Send `preset`, and optionally `customEqualization`:

```json
{"preset": "custom", "customEqualization": {"low": {"gain": 3.0}, "high": {"gain": 3.0}}}
```

Or switch preset only, leaving stored gains untouched:

```json
{"preset": "flat"}
```

Rules that matter:

- Gains are rounded by the device to the nearest multiple of `gainRange.stepPrecision` (typically
  0.1, 0.25, 0.5, or 1), so the value you read back may differ slightly from what you sent. Round
  in your UI to match.
- Out-of-bounds gains report `InvalidValue`. Clamp to `gainRange.min`..`gainRange.max`.
- Stored custom gains are independent of the selected preset. You can edit them while `flat` is
  active, and switching preset never rewrites them.
- Sending several bands together guarantees they are applied at the same instant. Sending them one
  by one is also allowed.
- Omitting `customEqualization` leaves stored gains unchanged.
- `enabled`, `currentEqualization`, `gainRange`, and `availablePresets` are GET-only and are
  ignored if you send them.
- If the parameters already match the current state, the call succeeds — this write is idempotent.

## Night mode

Requires DOS 2.16 and `nightMode` in `availableFeatures`.

Read with `getSystemNightMode`, write with `setSystemNightMode`, POSTing `{"nightMode": "on"}` or
`{"nightMode": "off"}`. Any other value reports `InvalidValue`. Setting the value it already has
succeeds, so this write is idempotent.

## Request mechanics

POST requests require `Content-Type: application/json` — omitting it returns HTTP 415. A successful
command returns `{}`. Application errors arrive as HTTP **200** with an `error` object in the body,
so parse the body and test for an `error` key rather than trusting the status code. Full list in
`errors/devialet-error-codes.yml`.

## Errors you will actually hit

| `error.code` | Cause | Response |
|---|---|---|
| `SystemLeaderAbsent` | Leader offline on a settings write | Nothing changed; retry when the leader is back |
| `InvalidValue` | Out-of-range volume or gain, or a bad enum | Clamp to `gainRange` / 0-100 and re-send |
| `NoCurrentSource` | No source selected on the system | Select a source before volume operations |
| `UnreachableDevices` | Some system devices unreachable | Partial application possible — re-query |
