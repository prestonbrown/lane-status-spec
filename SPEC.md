# RFC 0001: Lane Status Surface

| | |
|---|---|
| **Status** | Draft for discussion |
| **Version** | 0.1 |
| **Created** | 2026-10-27 |
| **Authors** | Preston Brown (HelixScreen) |
| **Discussion** | Pull request and Discussions in this repository |
| **Detail** | [REFERENCE.md](REFERENCE.md): the `afc` and `lane_data` field tables, command mappings, the Happy Hare bridge, the spool-tracker survey, and a source for every claim |

"Lane Status Surface" is a working name. Nothing here is endorsed yet by the AFC, Happy Hare,
Moonraker, Mainsail, Fluidd or OrcaSlicer projects; this draft exists to get their input. The
key words MUST, SHOULD and MAY are used as in RFC 2119.

## Summary

A small, shared way for Klipper filament systems to publish what is loaded in each lane, so that
web UIs, touchscreens and slicers can all read it. It keeps today's Mainsail and Fluidd panels
working, tells consumers whether real hardware is behind the data, and gives OrcaSlicer enough to
pick the right filament profile for each lane.

## Motivation

Multi-material printers keep lane state in many different shapes, and nothing outside the
vendor's own screen can see what is loaded. The web UIs understand two systems, AFC and Happy
Hare, so firmware modders impersonate AFC: paxx (Snapmaker U1), zmod_afclite and KILN_AFCLite each
ship a module that reverse-engineers the Mainsail and Fluidd panels. They disagree with each
other and with AFC, and a screen can't tell them from real hardware without guessing.

Slicers get little. OrcaSlicer reads six fields from Moonraker's `lane_data` namespace, and as
of 2026-10-26 can match an exact filament preset from a `filament_id` there. Its own docs say
"the schema is not standardized across the Klipper ecosystem." It writes nothing back.

This RFC writes down the contract those modules never had:

- one surface any filament system can publish and any UI, screen or slicer can read;
- today's Mainsail and Fluidd panels keep working unchanged;
- a producer says whether it drives hardware or only reports it, and refuses what it can't do;
- enough per-lane data for a slicer to pick the right profile and map tools.

Not in scope: replacing AFC or Happy Hare, or driving hardware the producer doesn't own.

## Specification

### 1. Two halves

| Half | Where | Read by | Changes |
|---|---|---|---|
| Live state: presence, active lane, errors | Klipper printer objects | UIs and screens (subscribe) | constantly |
| Filament identity: profile, temps, spool, tags | Moonraker database, namespace `lane_data` | slicers (poll over HTTP) | when a spool changes |

Both are needed: UIs read only Klipper objects, and Moonraker cannot add Klipper objects. A
producer is therefore a small Klipper module plus a `lane_data` writer.

### 2. Live state

#### 2.1 `lane_status`

Every producer MUST publish a Klipper object named `lane_status`:

| Key | Type | Meaning |
|---|---|---|
| `spec_version` | string | `"0.1"` |
| `producer` | object | `{name, version, url}` |
| `kind` | string | `status_only` (reports another system) or `controller` (drives the hardware) |
| `wraps` | string or null | For `status_only`: the native system read, e.g. `zmod_ifs`, `snapmaker_u1` |
| `capabilities` | list | Commands actually implemented (section 3); empty for status-only |
| `automap` | string | How the producer maps slicer tools to lanes by itself at print start: `none`, `filament_name`, `material`, `closest_color` or `color` (Happy Hare's AUTOMAP strategies, less `spool_id`, which Happy Hare lists but rejects as not yet supported) |
| `compat_profiles` | list | Extra surfaces published: `afc`, `slicer` |
| `units`, `lanes` | lists | Names; lanes in display order |
| `current_lane` | string or null | Lane feeding the active extruder |
| `state` | string | `Idle`, `Loading`, `Unloading`, `Ejecting`, `Error`, ... (AFC's words, capitalized) |
| `error` | object | `{active, message, type}` |

#### 2.2 Rules for every object

- **Every key from the first status.** A producer MUST return every key it will ever return
  from its first `get_status()`, with `null` for unknown. Klipper fixes a subscription's field
  list at its first query, so a key that appears later is missed until something re-subscribes.
- **Register early.** Objects SHOULD exist before `klippy:ready`; UIs read the object list once.
- **Lane names** match `[A-Za-z0-9_]+` and are unique. Colors are `#RRGGBB`.

#### 2.3 Compatibility profile `afc`

Mainsail and Fluidd draw their filament panel when a Klipper object named `AFC` exists. A
producer that wants those panels MAY publish `AFC`, one `AFC_unit <unit>` per unit and one
`AFC_lane <lane>` per lane. These names are a wire format the UIs require, not a brand; the
producer's own name, config and docs MUST NOT present it as AFC.

A status-only producer of this profile:

- MUST NOT publish `AFC` when one already exists;
- MUST report empty `extruders`, `hubs` and `buffers`, use unit type `unit`, and never publish
  `AFC_extruder`, `AFC_hub`, `AFC_buffer` or `AFC_stepper`;
- MUST send `current_state` capitalized (`Idle`, not `idle`);
- MUST report `weight` greater than 0 (a configurable default if unmeasured). Otherwise the
  panels' lane edit keeps Save disabled until the user types a weight, and a producer that
  refuses `SET_WEIGHT` then ends every such save in an error;
- SHOULD add `"lane_status_kind": "status_only"` to `AFC` (section 4).

Field tables: REFERENCE.md, Part A §5.

### 3. Commands

A producer lists each command it implements in `capabilities`. The commands, with their AFC and
Happy Hare spellings and what each existing module maps them to, are in REFERENCE.md Part A
§6.2 and Part B §7a.

- **Never fail silently.** Klipper answers an unregistered command with an info line and no
  error, and the script carries on, so a missing command is a silent no-op. A producer of the
  `afc` profile MUST register the lane and dialog commands the panels send (REFERENCE.md Part A
  §6.3 C1 lists them). One it doesn't implement MUST raise
  `<COMMAND> is not supported by <producer> on <wraps>: <reason>`.
- **Never shadow.** A producer MUST NOT register a command name that already exists.
- **Safe by default.** Load, unload and eject MUST refuse during a print unless the native
  system supports it.

### 4. Telling a status-only producer from real hardware

A consumer reads `lane_status.kind`. If it is `status_only` and the consumer has its own backend
for `wraps`, it SHOULD use that backend. A consumer that reads only `AFC` looks for
`lane_status_kind`; real AFC never sends it, so absent means real hardware. Guessing from which
other objects are present stays only as a fallback for producers that predate this RFC.

### 5. Slicer data: `lane_data`

`lane_data` is the Moonraker database namespace OrcaSlicer reads. The name comes from Moonraker
PR #1022 (jimmyjon711), which proposed a `lane_data` component as "a common API for filament
changers", with a documented field table. Moonraker's maintainer suggested the database
instead, and the PR is still open, so writers have no published schema; OrcaSlicer documents
only the fields its reader takes. This section proposes one, building on that table and on
OrcaSlicer's reader.

- **One writer per machine.** A producer MUST set `writer` and MUST NOT overwrite records another
  live producer wrote.
- **One record per lane**, kept in step with the loaded spool. An empty lane keeps its keys, with
  `null` values.
- **Ignore the key.** Writers disagree (`lane0`, `lane1`, `T0`, `"0"`); consumers MUST read
  `lane`.

| Field | Type | Req | Meaning |
|---|---|---|---|
| `lane` | string (int) | MUST | Slot index the slicer lays out. PR #1022's table described it as a tool mapping (`T0`); REFERENCE.md Part B §7 reconciles the two |
| `material` | string | MUST | Matchable type, e.g. `PLA`, `PETG-CF` |
| `color` | `#RRGGBB` | MUST if loaded | |
| `nozzle_temp`, `bed_temp` | int, C | SHOULD | |
| `filament_id` | string | SHOULD | OrcaSlicer's filament id, its preset match key. It names a filament product, not a preset: every per-printer variant of a product, and every user preset derived from one, inherits the product's id. Copy the exact id from Orca's filament library when the spool is in it; otherwise MAY mint one. A producer MUST NOT write an id that more than one product carries, and MUST NOT use it to select a particular preset (that is `slicer_preset`). REFERENCE.md Part B §3 and §7. A numeric value means a Spoolman id (Happy Hare's usage) |
| `lss_version`, `writer` | string | MUST | `"0.1"`; producer name |
| `lane_name` | string | MUST | Joins the record to `lane_status` |
| `loaded` | bool | MUST | Explicit presence |
| `extruder_index` | int | SHOULD | 0-based extruder the lane feeds |
| `vendor_name`, `name` | string | SHOULD | Manufacturer and product name (Happy Hare's keys) |
| `spool_id` | int or null | SHOULD | Spoolman spool |

The first five rows (six fields) are what OrcaSlicer reads today. Optional fields (multi-color,
temperature ranges, weights, pressure advance, tags) are in REFERENCE.md Part B §7.

**Writes from a slicer** are G-code through Moonraker, and only commands in `capabilities`:

- map tools before the print: `SET_MAP LANE=<lane> MAP=T<n>` per tool (Happy Hare:
  `MMU_TTG_MAP`), all before the print starts;
- or no write at all: the producer reads Moonraker's per-file metadata (`referenced_tools`,
  `filament_colors`, `filament_type`, `filament_name`) at print start and maps by its `automap`
  strategy.

### 6. Spools and trackers

A lane points at a spool; it does not copy a tracker's data model.

- **Reference, don't model.** A record MAY carry `spool: {tracker, id}`, where `tracker` is a
  short name (`spoolman`, `filaman`, `simplyprint`, ...) and `id` is that tracker's spool id as a
  string (REFERENCE.md Part B §7). `spool_id` stays for Spoolman compatibility; when
  `spool.tracker` is `spoolman`, the two MUST agree.
- **The tracker wins on identity; the producer wins on presence.** When a lane references a
  spool, the producer SHOULD copy the tracker's identity fields into the record (`material`,
  `color`, `vendor_name`, `name`, temperatures, weights), as AFC does today. Only the producer
  sets `loaded`.
- **Tracker internals stay outside this RFC.** Anything tracker-specific copied into a record
  MUST be vendor-prefixed (section 7).
- **Tags are physical identity.** An NFC/RFID reading goes in `tag` (`{format, uid, data}`). A
  tag MAY resolve to a tracker spool, for example through Spoolman's tag table. Where a tag and
  the tracker disagree, the tracker wins, because the user edited it after the tag was written.

The trackers, lane-to-tracker links and tag formats in use today are surveyed in REFERENCE.md
Part C.

### 7. Versioning

`spec_version` and `lss_version` are `major.minor`; a minor version adds optional keys or
capabilities only.
Consumers ignore unknown keys. Vendor-specific keys MUST be prefixed `<vendor>_`.

## Rationale and alternatives

- **Why not a Moonraker component alone?** Moonraker passes object lists, queries and
  subscriptions through to Klipper and cannot add objects, and the UIs read only Klipper objects.
- **Why reuse AFC's object names instead of new ones?** Apart from Happy Hare's `mmu` (next
  point), they are the only filament-system objects current Mainsail and Fluidd draw a panel
  from. A neutral `lane_status` object sits beside them for consumers that adopt it.
- **Why not imitate Happy Hare's `mmu` object?** It would make every consumer treat the printer
  as Happy Hare, draw a selector panel with servo, encoder and sync state a status-only producer
  would have to fake, and send it `MMU_*` commands. Happy Hare is instead the precedent for the
  `lane_data` field names, the automap strategies and the per-file metadata.
- **Why `lane_data` for slicer data, not a new endpoint?** OrcaSlicer already reads it with one
  HTTP request, it persists across Klipper restarts, and any process can write it. A
  product-specific endpoint would need product-specific code in every slicer.

## Compatibility

- **Current Mainsail and Fluidd:** unchanged; the `afc` profile is what they already read.
- **Real AFC:** unaffected. A producer stands down when an `AFC` object already exists.
- **Happy Hare:** unaffected. REFERENCE.md Part B §7a maps lanes, presence, filament fields,
  tool maps and commands to its `mmu` fields and names the gaps (lane to extruder, runout
  groups), so a consumer can read most of a Happy Hare printer through this RFC's terms.
- **OrcaSlicer as shipped:** reads the six fields in section 5 today. Everything else is ignored
  until it opts in.
- **Existing `lane_data` writers:** records without `writer` are treated as legacy and still read.

## Prior art

AFC-Klipper-Add-On; Happy Hare (`mmu` object, `lane_data` writer, Moonraker PR #962 file
metadata); paxx's U1 AFC-Lite; zmod_afclite and KILN_AFCLite; Moonraker PR #1022; OrcaSlicer's
Moonraker agent and its `lane_data` documentation; spoolman-lane-sync's `source` key; Spoolman's
native tags (v0.27.0). Each is described from source in REFERENCE.md.

## Unresolved questions

1. The name of this RFC, and of the `afc` compatibility profile. ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/2))
2. Should a status-only producer publish the `afc` profile at all, given every panel button it
   can't serve must refuse? Or publish only `lane_status`? ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/3))
3. When a consumer has a native backend for what a status-only producer wraps, should the native
   backend always win (section 4)? ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/4))
4. How `writer` relates to spoolman-lane-sync's existing `source` key, and how ownership of
   `lane_data` is handed over. ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/5))
5. Should a `lane_data` key style be required, or is "ignore the key" enough? ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/6))
6. Where the `lane_data` schema should live long term, for example Moonraker's
   `integrations.md`. ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/7))
7. Whether `spool: {tracker, id}` is the right shape for non-Spoolman trackers. ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/8))
8. The open questions listed in REFERENCE.md Part C (tag formats and tracker APIs). ([discussion](https://github.com/prestonbrown/lane-status-spec/discussions/9))

## Feedback wanted

| From | On |
|---|---|
| AFC maintainers | The `afc` profile's accuracy, naming, and section 2.3's rules |
| Happy Hare maintainers | The bridge in REFERENCE.md Part B §7a and the `lane_data` fields borrowed from Happy Hare |
| Moonraker maintainers | `lane_data` as a documented namespace (question 6) |
| OrcaSlicer maintainers | Section 5 and REFERENCE.md Part B §11 |
| Shim authors (paxx, zmod_afclite, KILN_AFCLite) | Sections 2.3 and 3 against your modules |
| Spool tracker authors | Section 6 and REFERENCE.md Part C |
| Mainsail, Fluidd, KlipperScreen and touchscreen authors | Sections 2 and 4 |
