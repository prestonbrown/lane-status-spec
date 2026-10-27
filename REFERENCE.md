# Lane Status Surface: reference

Companion to [SPEC.md](SPEC.md), draft 0.1, 2026-10-27. SPEC.md is the short contract; this file
holds everything an implementer or reviewer needs behind it.

- **Part A, live state:** transport rules, vocabularies, the full `afc` compatibility field
  tables, the command table with each existing shim's mapping, discovery, and sourcing.
- **Part B, slicer data:** how OrcaSlicer reads printers today, where `lane_data` came from,
  what Happy Hare already does, the full `lane_data` record, read and write flows, the Happy Hare
  bridge, and what OrcaSlicer would need to change.
- **Part C, spool trackers and tag formats:** who tracks spools, how lanes link to them, and
  the RFID/NFC tag formats in use.

Section numbers are local to each part. Tags: **[V]** read in source (revision named),
**[I]** inference.

Status columns use: AFC = AFC-Klipper-Add-On, P = paxx U1 stub, Z = zmod_afclite,
K = KILN_AFCLite, HH = Happy Hare, H = HelixScreen; readers M = Mainsail, F = Fluidd,
H = HelixScreen.

---

# Part A: Live state

## 2. Transport rules (all objects)

**R1.** Every object in this spec is a Klipper printer object with a `get_status(eventtime)`
method, so it appears in `objects/list` and can be subscribed. Moonraker passes object lists and
queries through to Klipper, merges its clients' subscriptions into one Klipper subscription, and
cannot add objects, so a Moonraker-only producer can't satisfy this.

**R2.** A producer MUST return **every key it will ever return** from the first `get_status()`
call on, using `null` (or the type's empty value) when there's no value.
- When a subscriber asks for all fields (`null`), Klipper fixes that subscription's field list
  from the first answer, provided the answer is non-empty. Mainsail and Fluidd subscribe this
  way. HelixScreen names its fields, so the rule doesn't affect it.
- A key that first appears later is not sent on that subscription. Moonraker re-sends its
  merged subscription to Klipper whenever any client subscribes, which fixes the list again, so
  a late key reaches clients only after some client (re)subscribes. The producer has no
  control over when that happens.

**R3.** A producer SHOULD register its objects at config load or `klippy:connect`, before
`klippy:ready`.
- Moonraker clients read `objects/list` once, at connect.
- An object added later appears only after a client reconnects. zmod_afclite's README
  documents exactly this for lanes added late.

**R4.** Colors are strings `#RRGGBB` (uppercase or lowercase hex). Consumers SHOULD accept
`RRGGBB` without `#` and an 8-digit `RRGGBBAA`, ignoring alpha.

**R5.** Lane names MUST be unique across all units. They MUST match `[A-Za-z0-9_]+`, because
they travel as G-code parameter values (`LANE=<name>`).

---
## 4. Vocabularies

### 4.1 System state

Use AFC's words, capitalized:

`Initialized`, `Idle`, `Error`, `Loading`, `Unloading`, `Ejecting`, `Moving`, `Restoring`,
`ToolSwap`, `ToolDock`, `ToolPickup`.

The UIs depend on these spellings [V]:
- Mainsail translates `Panels.AfcPanel.<state>`. Its English locale has no entry for
  `ToolSwap`, `ToolDock` or `ToolPickup`.
- Fluidd translates `app.afc.<state>`, with all eleven entries.
- Fluidd also enables tool selection only when the state is `'Idle'` or `'Error'`.

A producer MUST NOT invent other values in the compatibility profile. Today's shims don't
follow this [V]:
- paxx always sends `idle`. Its `AFCState` also defines `loading`, `unloading`, `tool_change`
  and `error`, but `get_status` never sends them.
- zmod_afclite and KILN_AFCLite send `idle`, `loading`, `unloading` and `error`.
- No shim sends `tool_change`.
- All of these values miss both UIs' translations.

Mainsail and Fluidd show the state only inside an `AFC_extruder` block, which these producers
don't publish [V].

### 4.2 Lane status

AFC's lane `status` values (Fluidd typings): `None`, `Error`, `Loaded`, `Tooled`,
`Tool Loaded`, `Tool Loading`, `Tool Unloading`, `HUB Loading`, `Ejecting`, `Calibrating`.

Status-only producers SHOULD use:
- `Loaded` for filament present,
- `Tool Loaded` for filament in the toolhead,
- `None` for empty.

What shims send today [V]:
- zmod_afclite and KILN_AFCLite send `empty`/`loaded`.
- paxx always sends `empty`, even for a loaded lane.

Mainsail and Fluidd never read this key. HelixScreen reads it to work out slot state [V].

---

## 5. Compatibility profile `afc`

This is the profile that makes **current Mainsail and Fluidd** draw their AFC panels unchanged.
The object names below are fixed by those UIs' source. They are a wire-format requirement, not
a brand. The AFC project is ArmoredTurtle's AFC-Klipper-Add-On. A producer MUST NOT use that
project's product names (AFC, Box Turtle, …) in its own module name, config section,
documentation title or UI strings beyond these object and command names.

### 5.1 Panel trigger

- The panel appears in both UIs exactly when an object named `AFC` exists
  (Mainsail `store/gui/getters.ts`; Fluidd `printer/getters.ts#getSupportsAfc`).
- A producer publishing this profile MUST publish `AFC`, one `AFC_unit <unit>` per unit, and
  one `AFC_lane <lane>` per lane.
- A producer MUST NOT publish `AFC` if another object by that name exists. It MUST check at
  `klippy:connect` and stand down without error. zmod_afclite and KILN_AFCLite instead raise a
  config error, checked at config load. Standing down is the proposal.

### 5.2 `AFC`

Column key:
- **Req**: the minimum for the panel to draw correctly.
- **Pub**: who publishes the key. `AFC` = real AFC (`extras/AFC.py`), `P` = paxx U1 stub,
  `Z` = zmod_afclite, `K` = KILN_AFCLite.
- **Read by**: `M` = Mainsail, `F` = Fluidd, `H` = HelixScreen.

| Key | Type | Req | Meaning | Pub | Read by |
|---|---|---|---|---|---|
| `units` | list[string] | MUST | Each entry is `"<Type> <unit>"`. The UI looks up object `AFC_<Type with '_' removed> <unit>` case-insensitively. Status-only producers MUST use Type `unit`, giving `AFC_unit <unit>`. AFC ships the Types `Box_Turtle`, `HTLF`, `Night_Owl`, `Quattro_Box`, `Toolchanger`, `Claymore`, `EMU`, `ViViD` and `OpenAMS`. The first four get a brand icon in both UIs, which strip `_` and ignore case when picking it. A producer that isn't that product MUST NOT use any spelling of an AFC Type | AFC P Z K | M F H |
| `lanes` | list[string] | MUST | All lane names | AFC P Z K | M F H |
| `extruders` | list[string] | MUST | `AFC_extruder` names. Status-only: `[]` (with an extruder listed, the UI draws sensors the producer can't report) | AFC P Z K | M F H |
| `hubs` | list[string] | MUST | `AFC_hub` names. Status-only: `[]` | AFC P Z K | M F H |
| `buffers` | list[string] | MUST | `AFC_buffer` names. Status-only: `[]` | AFC P Z K | H |
| `current_load` | string\|null | MUST | Lane loaded in the toolhead. Read before `current_lane` | AFC P Z K | M F H |
| `current_lane` | string\|null | MUST | Lane being loaded (AFC); the active lane (shims) | AFC P Z K | M F H |
| `next_lane` | string\|null | SHOULD | | AFC P Z K | H |
| `current_state` | string | MUST | §4.1 | AFC P Z K | M F H |
| `error_state` | bool | MUST | | AFC P Z K | M F H |
| `message` | object | MUST | `{"message": string, "type": "error"\|"warning"\|"info"\|""}`. paxx sends `""`, which both UIs tolerate; producers MUST send the object | AFC Z K (P: string) | M F H |
| `current_toolchange` | int | SHOULD | Index of the current toolchange in the print | AFC P Z K | M F H |
| `number_of_toolchanges` | int | SHOULD | | AFC P Z K | F H |
| `spoolman` | bool\|string | SHOULD | Spoolman integration available. AFC sends the Spoolman URL or null; HelixScreen reads only a string | AFC P Z K | H |
| `td1_present` | bool | SHOULD | Status-only: `false` | AFC P Z K | M F |
| `lane_data_enabled` | bool | MAY | | AFC P Z K | — |
| `bypass_state` | bool | SHOULD | | AFC P Z K | M F H |
| `quiet_mode`, `position_saved` | bool | MAY | | AFC P Z K | H |
| `led_state` | bool | SHOULD | Selects the LED button label | AFC (P Z K send `""`) | M F H |
| `maps` | list[string] | MAY | Tool commands (T0…) | AFC | H |

### 5.3 `AFC_lane <lane>`

Mainsail and Fluidd look up `AFC_stepper <lane>` first and then `AFC_lane <lane>`.
Status-only producers MUST use `AFC_lane`.

| Key | Type | Units | Req | Meaning | Pub | Read by |
|---|---|---|---|---|---|---|
| `name` | string | | MUST | Equals `<lane>` | AFC P Z K | M F |
| `unit` | string | | MUST | Unit name (no Type prefix). No consumer reads it today; units come from `AFC_<Type> <unit>.lanes` | AFC P Z K | — |
| `lane` | int | | MUST | Index within the unit, with AFC's meaning (§8 V2): the number after the colon in the lane's `unit:` option (`unit: Turtle_1:1`), so 1-based in AFC's shipped templates, and 0 when the option gives none. paxx, zmod_afclite and KILN_AFCLite send 0-based indices, so they differ from AFC today. No consumer reads it today | AFC P Z K | — |
| `extruder` | string\|null | | MUST | The extruder this lane feeds. In AFC this is the lane's `AFC_extruder` name, which the UIs match against `AFC.extruders`; shims send the Klipper extruder (`extruder`, `extruder1`, …) | AFC P Z K | M F H |
| `hub` | string\|null | | SHOULD | Status-only: `null` | AFC | H |
| `buffer` | string\|null | | SHOULD | Status-only: `null` | AFC | M F |
| `map` | string\|list[string]\|null | | MUST | Slicer tool(s) this lane serves (`"T0"`). `"NONE"` or null = unmapped | AFC P Z K | M F H |
| `load` | bool | | MUST | Filament at the lane's load sensor | AFC P Z K | M F H |
| `prep` | bool | | MUST | Filament at the prep sensor. Status-only: same as `load`. The UI calls a lane ready when `load && prep` | AFC P Z K | M F H |
| `tool_loaded` | bool | | MUST | This lane's filament is in the toolhead | AFC P Z K | M F H |
| `loaded_to_hub` | bool | | SHOULD | Status-only: `false` | AFC P Z K | H |
| `material` | string | | MUST | Material type; `""` when unknown (**not** `"NONE"`) | AFC P Z K | M F H |
| `color` | string | `#RRGGBB` | MUST | §R4. `""` when unknown | AFC P Z K | M F H |
| `weight` | number | g | MUST | Remaining filament. **MUST be > 0**. Both UIs' filament dialogs start the weight field at this value and keep Save disabled while the field is 0, so with no weight the user has to type one on every edit (§6.3 C2). A producer that doesn't measure weight SHOULD report a configurable default (zmod and Kiln: 1000) | AFC Z K (Z and K only when > 0) | M F H |
| `initial_weight` | number\|null | g | SHOULD | Full spool | AFC | M F H |
| `spool_id` | int\|null | | MUST | Spoolman spool id, null when none (paxx sends `0`) | AFC P Z K | M F H |
| `filament_name` | string | | SHOULD | Display name; `""` when unknown. MUST be present from the first call (§R2); paxx, zmod and Kiln omit it conditionally | AFC P Z K | M F H |
| `extruder_temp` | int\|null | °C | SHOULD | | AFC K (K only when the slot has a temperature) | F H |
| `bed_temp` | int\|null | °C | MAY | | AFC | F H |
| `runout_lane` | string\|null | | SHOULD | Backup lane, `"NONE"`, or (paxx) `"AUTO"` for firmware-managed endless spool | AFC P Z K | M F H |
| `status` | string | | SHOULD | §4.2 | AFC P Z K | H |
| `filament_status`, `filament_status_led` | string | | MAY | | AFC P Z K | H |
| `buffer_status`, `dist_hub`, `remember_spool`, `multi_color_hexes`, `td1_td`, `td1_color`, `td1_scan_time`, `selector`, `endstops` | various | | MAY | Real-hardware fields; status-only producers SHOULD omit them entirely (absent from the first call) | AFC | M F (`dist_hub`, `td1_td`, `td1_color`); H (all) |

### 5.4 `AFC_unit <unit>`

| Key | Type | Req | Meaning | Pub | Read by |
|---|---|---|---|---|---|
| `lanes` | list[string] | MUST | This unit's lanes | AFC P Z K | M F H |
| `extruders`, `hubs`, `buffers` | list[string] | MUST | Status-only: `[]` | AFC P Z K | M F (`hubs`); H (all) |

### 5.5 Objects outside the status-only profile

`AFC_extruder <name>`, `AFC_hub <name>`, `AFC_buffer <name>` and `AFC_stepper <name>` belong to
real hardware (fields in Fluidd's `typings/klipper.d.ts` and AFC's `extras/`). A status-only
producer MUST NOT publish them: the UI would draw sensors and buffers it can't report.

### 5.6 What the profile triggers beyond the panel

Publishing `AFC` also enables:
- Mainsail's AFC section in the print-start dialog (`StartPrintDialog.vue`). It is mounted
  whenever `AFC` exists and draws one row per tool whose `filament_weights` entry is > 0.
- Fluidd's `AfcPrintStartDialog`, shown when `AFC` exists and the file has a
  `filament_weights` entry > 0.

Both send `SET_MAP` for a tool when the user picks a different lane for it; neither sends
anything on its own. A producer of the `afc` profile MUST answer `SET_MAP` (§6.2): either
remap, or raise the standard refusal.

---

## 6. Commands

### 6.1 Status-only tier

A producer with `capabilities: []` registers **no** commands. It is RECOMMENDED that such a
producer publish `lane_status` only, with no `afc` profile. The AFC panel offers Load or
Unload, Eject and the filament edit dialog on every lane whose `load` and `prep` are both true,
so a status-only `afc` producer would face §6.3 on every click.

### 6.2 Command tier

All commands take `LANE=<lane>` (R5). Each row names the capability token that advertises it in
`lane_status.capabilities`.

| Command | Capability | Sent by | Expected behaviour | P | Z | K |
|---|---|---|---|---|---|---|
| `CHANGE_TOOL LANE=` | `load` | M F (lane Load; extruder settings dialog) H. No UI sends it at print start | Load the lane into the toolhead, unloading another first if needed | `AUTO_FEEDING LOAD=1` | `IN_ZCOLOR`/`_T_IN_ZCOLOR` | `IFS_LOAD` (homes first) |
| `TOOL_UNLOAD [LANE=]` | `unload` | M F (always with `LANE`) H (without `LANE` when no lane is picked) | Unload from the toolhead; without `LANE`, the loaded lane | `AUTO_FEEDING UNLOAD=1` | `IN_ZCOLOR NAPR=1` / C5 prompt | `IFS_UNLOAD` (homes first) |
| `LANE_UNLOAD LANE=` | `eject` | M F (Eject) H | Retract the lane's filament out of the unit | `AUTO_FEEDING UNLOAD=1` | as unload | `IFS_EJECT` |
| `SET_COLOR LANE= COLOR=` | `set_color` | M F H | Set lane color. An empty `COLOR` SHOULD clear the color. Real AFC then stores a bare `#`, which HelixScreen reads as no color, and HelixScreen's lane Clear sends `SET_COLOR COLOR=` for exactly that. zmod_afclite keeps the color instead | `SET_PRINT_FILAMENT_CONFIG` | `CHANGE_ZCOLOR` | `IFS_SET_MATERIAL` |
| `SET_MATERIAL LANE= MATERIAL=` | `set_material` | M F H | An empty `MATERIAL` SHOULD clear the material, as in real AFC; HelixScreen's Clear sends `SET_MATERIAL MATERIAL=` | ✓ | ✓ (+ picker) | ✓ (`IFS_SET_MATERIAL`) |
| `SET_WEIGHT LANE= WEIGHT=` | `set_weight` | M F H | Persist the weight. `WEIGHT=0` returns to the default | **raises** | ✓ | ✓ |
| `SET_SPOOL_ID LANE= SPOOL_ID=` | `set_spool_id` | M F H | Bind or clear (empty or 0) a Spoolman spool | (SpoolLink) | ✓ (+`CARD_UID`) | ✓ (+`CARD_UID`) |
| `SET_MAP LANE= MAP=T<n>` | `set_map` | M F (mapping dialog; print-start dialog when the user picks a lane) H | Point slicer tool `T<n>` at the lane | swap | **raises** | ✓ (`IFS_MAP_TOOL`) |
| `SET_RUNOUT LANE= RUNOUT=` | `set_runout` | M F H | Backup lane | — | — | — |
| `SET_VENDOR LANE= VENDOR=` | `set_vendor` | none | Not an AFC command: real AFC registers no `SET_VENDOR`, and no UI sends it. Outside the C1 required set | ✓ | **raises** | **raises** |
| `AFC_CLEAR_MESSAGE` | `clear_message` | M F H | Clear `message` and `error_state` | — | ✓ | ✓ |
| `AFC_RESET`, `RESET_FAILURE` | `reset` | H | Clear an error after the user fixed it | — | ✓ | ✓ |
| `AFC_SELECT_TOOL TOOL=` | `select_tool` | F H | | — | — | — |
| `AFC_SET_SPOOL_TEMP LANE= EXTRUDER_TEMP= [BED_TEMP=]` | `set_temp` | F (only when the command exists) | | — | — | — |
| `AFC_CALIBRATION`, `TURN_ON_AFC_LED`/`TURN_OFF_AFC_LED`, `AFC_BRUSH`, `AFC_PARK`, `AFC_GET_TD_ONE_DATA`, `SET_HUB_DIST`/`SAVE_HUB_DIST`, `SET_BOWDEN_LENGTH`, `UPDATE_TOOLHEAD_SENSORS`, `SAVE_EXTRUDER_VALUES` | — | M F (buttons, settings) H | Real-hardware only; see the note below the table | — | — | — |
| `AFC_QUIET_MODE`, `AFC_LANE_RESET`, `AFC_RESET_MAPPING`/`RESET_AFC_MAPPING`, `AFC_RESET_MOTOR_TIME`, `AFC_SET_EXTRUDER_LED`, `AFC_TEST_LANES`, `AFC_TOGGLE_MACRO`, `AFC_TOOLHEAD_CALIBRATION`, `AFC_CHANGE_BLADE`, `SET_LONG_MOVE_SPEED` | — | H | Real-hardware only. A status-only producer MUST NOT define them | — | — | — |

**Real-hardware buttons in Mainsail and Fluidd [V].** Three groups behave differently:
- **Hidden unless the command exists.** The calibration, LED, TD-1, brush and park buttons
  appear only when a matching command or macro exists. Mainsail needs a `gcode_macro` of that
  name for all of them. Fluidd needs the command in its available-commands list for the
  calibration, LED and TD-1 buttons, and a macro for brush and park. TD-1 also needs
  `td1_present`, and brush and park need `wipe`/`park` in the `[AFC]` config. A producer that
  doesn't define these commands never shows them.
- **Unreachable.** `SET_BOWDEN_LENGTH`, `UPDATE_TOOLHEAD_SENSORS` and `SAVE_EXTRUDER_VALUES`
  live in hub and extruder settings rows, which don't exist when `hubs` and `extruders` are
  empty.
- **Reachable.** `SET_HUB_DIST` and `SAVE_HUB_DIST` stay reachable: the AFC Settings button is
  always shown, and its dialog lists a `dist_hub` row for every lane. A status-only producer of
  the `afc` profile SHOULD therefore register these two with the C1 refusal rather than leave
  them unregistered, which Klipper answers with only an info line (C1).
- **Also sendable, outside C1's required set.** Fluidd's extruder card sends
  `AFC_SELECT_TOOL`, and the hidden buttons above send their commands once those exist.

### 6.3 Unimplemented commands

**C1.** A producer of the `afc` profile MUST register every command in §6.2 down to and
including `AFC_CLEAR_MESSAGE`, except `SET_VENDOR`. For one it doesn't implement, the handler
MUST raise:

```
gcmd.error("<COMMAND> is not supported by <producer.name> on <wraps>: <reason>")
```

It MUST NOT silently succeed or move hardware. A missing command does not fail at all: once
Klippy is ready, Klipper's `cmd_default` only answers `Unknown command:"<COMMAND>"` as an info
line, raises nothing, and runs the rest of the script, so the request returns success. The UI
shows no error, only a console line, and the user's action silently did nothing [V].

**C2.** `SET_WEIGHT` follows C1 like any other command. What the UIs send [V]:
- Mainsail's filament dialog sends one newline-joined script: `SET_COLOR`, `SET_MATERIAL`,
  `SET_WEIGHT`.
- Fluidd's sends `SET_SPOOL_ID` (when clearing), `SET_COLOR`, `SET_MATERIAL`, `SET_WEIGHT`,
  then `AFC_SET_SPOOL_TEMP` (only when that command exists).
- Mainsail adds each line only when its value changed. Fluidd does the same, except that when
  the user clears the spool id it sends `SET_COLOR`, `SET_MATERIAL`, `SET_WEIGHT` (and
  `AFC_SET_SPOOL_TEMP` when it exists) unconditionally.
- Klipper runs the lines in order and stops at the first error. A `SET_WEIGHT` raise therefore
  leaves the color and material applied, shows an error, and skips only what follows it.
- HelixScreen sends `SET_WEIGHT` as a command of its own, so a raise fails only that command.

For a producer that reports a weight > 0 (§5.3), no reason remains to accept `SET_WEIGHT`
without measurement. The dialog then sends it only when the user edits the weight, and a raise
costs an error message after color and material are already saved.

The case still worth avoiding is a producer that reports no weight (paxx today). The dialog's
weight field then starts at 0, so the user must type a weight before Save enables. Every save
then carries a `SET_WEIGHT`, and raising on it ends every successful edit in an error. §5.3's
default weight removes that case.

**C3.** A producer MUST NOT register a command name that already exists. It MUST check before
registering, MUST NOT override a user macro, and SHOULD log the skip.

**C4.** Load, unload and eject MUST refuse while a print is running unless the native system
supports it mid-print. They SHOULD home first when the native load needs it (Kiln does).

**C5.** While an operation runs, `current_state` SHOULD be `Loading`/`Unloading`/`Ejecting`.
A native failure SHOULD set `error_state` and `message`.

---

## 7. Discovery: telling an adapter from real hardware

**Today [V, HelixScreen `printer_discovery_parse.cpp#parse_objects`].** HelixScreen guesses
from co-presence:
- `AFC_unit` beside `filament_detect` means paxx on a U1.
- `zmod_afclite` beside the Creator 5 changer fingerprint means Z-Mod.
- Every other `AFC` object is treated as real AFC. `AFC` also wins over a native `ifs` object.
  That includes zmod_afclite and KILN_AFCLite on an AD5X.
- One exception: the AFC backend still notes `zmod_afclite` on any machine and treats tool
  mapping as per-print firmware state (`ams_backend_afc.cpp#set_discovery`).

**Proposal.**

**D1.** A conforming producer MUST publish `lane_status` with `kind` (SPEC.md §2.1).

**D2.** A consumer that has a native backend for the system named in `wraps` SHOULD prefer that
backend when `kind == "status_only"`. A consumer without one MAY drive the producer through the
command tier, using `capabilities` to enable controls.

**D3.** Inside the `afc` profile, a producer SHOULD also add
`"lane_status_kind": "status_only"` to the `AFC` object. Consumers that read only `AFC` then see
the marker without subscribing a second object. Real AFC never sends the key, and absent means
`controller`.

**D4.** Co-presence heuristics stay as a fallback for producers that predate this spec. They are
retired for a producer once it publishes `lane_status`.

---

## 8. Versioning and extensions

**V1.** `spec_version` is `"<major>.<minor>"`.
- A minor bump adds optional keys or capabilities only.
- A major bump may change meaning.
- Consumers MUST ignore unknown keys and unknown capability tokens.

**V2.** Vendor-specific keys in any object MUST be prefixed `<vendor>_`, for example
`zmod_slot` (already published by zmod_afclite) or `helix_…`.
- Unprefixed keys are reserved for this spec and for the field names the `afc` profile inherits
  from AFC.
- A producer MUST NOT reuse an unprefixed AFC key with a different meaning.

**V3.** New system-state, lane-status or `wraps` values are added only by a spec revision. The
`wraps` vocabulary is open: a producer MAY use a new value, and consumers MUST treat unknown
values as "no native backend".

**V4.** Persistent lane metadata (`lane_data` in Moonraker's database) is the slicer half of
this spec: SPEC.md §5 and Part B.
- A producer that writes it follows Part B §6 S2: it MUST NOT overwrite records another live
  producer wrote.
- HelixScreen's `docs/specs/filament_slots.md` documents HelixScreen's own records.

---

## Appendix A: sourcing

Abbreviations used below:
- **UI** = Mainsail (`03a4d7b`, develop 2026-10-25) and Fluidd (`738ff93`, develop
  2026-10-25).
- **H** = HelixScreen `origin/main` `11cf6f0f29` (2026-10-26).

| Part | Source | Revision |
|---|---|---|
| §2 R1 (objects/list = objects with `get_status`; subscriptions merged) | **Source**: Klipper and Kalico `klippy/webhooks.py#_handle_list`; Moonraker `components/klippy_connection.py#_request_subscripton` | Klipper `ee2c6fb`; Kalico `0057b6a`; Moonraker `b559b9a` |
| §2 R2 (field list fixed at first query; re-subscription) | **Source**: Klipper and Kalico `klippy/webhooks.py#QueryStatusHelper._do_query` (identical); Moonraker `klippy_connection.py#_request_subscripton`; Mainsail `#initSubscripts` and Fluidd `#onPrinterObjectsList` subscribe with `null`; H `ams_backend_afc.cpp#required_status_objects` names fields | as above; UI; H |
| §2 R3 (UIs read objects/list once) | **Source**: Mainsail `store/printer/actions.ts#initSubscripts`, Fluidd `store/printer/actions.ts#onPrinterObjectsList`; zmod_afclite README | UI; `375c86d` |
| §2 R4, R5 | **Proposal** (R4 tolerance mirrors paxx RGBA input and HelixScreen's reader `color_utils.h#parse_hex_color`) | |
| `lane_status` object (SPEC.md §2.1; §7 D1) | **Proposal** | |
| §4.1 state vocabulary | **Source**: AFC `extras/AFC.py` `class State`; Mainsail `locales/en.json` `Panels.AfcPanel.*`, `AfcPanelExtruder.vue#state`; Fluidd `locales/en.yaml` `app.afc.*`, `AfcCardExtruder.vue#state`, `#toolSelectionDisabled`. Shims' values: paxx `AFC.py#get_status`, zmod_afclite `AFCState`, KILN_AFCLite `extras/kiln_afclite.py` `AFCState`, `ACTIVITY_STATES` | AFC `5ea84ea`; UI; paxx `417e639`; Z `375c86d`; K `67aae0d` |
| §4.2 lane status | **Source**: Fluidd `typings/klipper.d.ts#AfcLaneStatus`; shims' `get_status` (`AFCLaneState`); H `ams_backend_afc.cpp#parse_afc_stepper`. The recommendation is a **proposal** | UI; paxx, Z, K as above; H |
| §5.1 panel trigger | **Source**: Mainsail `store/gui/getters.ts`, `mixins/afc.ts#afcExists`; Fluidd `printer/getters.ts#getSupportsAfc`, `views/Dashboard.vue`; zmod_afclite `zmod_afclite.py#AFC.__init__`, KILN_AFCLite `kiln_afclite.py#AFC.__init__` (config errors) | UI; Z; K |
| §5.2-5.4 field tables (keys, who publishes, who reads) | **Source**: AFC `extras/AFC.py`, `AFC_lane.py`, `AFC_unit.py` `get_status`; paxx `31-feature-afc-lite/.../AFC*.py`; zmod_afclite `zmod_afclite.py` `get_status`; KILN_AFCLite `extras/kiln_afclite.py` `get_status`; Mainsail `mixins/afc.ts` + `panels/Afc/*` + `dialogs/Afc*`; Fluidd `mixins/afc.ts` + `widgets/afc/**` + `widgets/status/StatusTab.vue`; H `src/printer/ams_backend_afc.cpp#required_status_objects`, `#parse_afc_state`, `#parse_afc_stepper`, `#parse_afc_unit_object`. "Req" levels are **proposal** | AFC `5ea84ea`; paxx `417e639`; Z `375c86d`; K `67aae0d`; UI; H |
| Unit lookup rule `AFC_<Type> <unit>` and brand icons | **Source**: Mainsail `AfcPanelUnit.vue#unit`, `#type`; Fluidd `AfcCardUnit.vue#unit`, `#unitType`; AFC `AFC.py` `unitdisplay` and the `self.type` defaults in `AFC_BoxTurtle.py`, `AFC_HTLF.py`, `AFC_NightOwl.py`, `AFC_QuattroBox.py`, `AFC_Toolchanger.py`, `AFC_Claymore.py`, `AFC_EMU.py`, `AFC_vivid.py`, `AFC_OpenAMS.py`; Mainsail `AfcPanelUnit.vue#modulIcon`, Fluidd `AfcCardUnit.vue#unitIcon` (icons) | UI; AFC |
| `AFC_lane.lane` meaning | **Source**: AFC `extras/AFC_lane.py#AFCLane.__init__` (`self.index = int(unit.split(':')[1])`, else 0), `templates/AFC_Turtle_1.cfg`, `templates/AFC_NightOwl_1.cfg`; paxx `AFC_lane.py` (`lane_index`); zmod_afclite and KILN_AFCLite `AFCLane.get_status` | AFC `5ea84ea`; paxx; Z; K |
| `weight > 0` for the dialog | **Source**: Mainsail and Fluidd `AfcUnitLaneFilamentDialog.vue#disableSetBtn` (`!material \|\| !weight \|\| !color` disables Save; Fluidd also disables it for a weight below 0 and on its temperature checks), `#currentWeight`, and the open watcher (Mainsail `#onShowDialogChange`, Fluidd `#onShowChange`) | UI |
| §5.6 print-start dialogs | **Source**: Mainsail `StartPrintDialog.vue`, `StartPrintDialogAfc.vue#usedTools`, `StartPrintDialogAfcTool.vue#changeToolMapping`; Fluidd `AfcPrintStartDialogTool.vue#changeLaneMapping`, `mixins/afc.ts#shouldShowAfcDialog` | UI |
| §6.1 lane actions | **Source**: Mainsail `AfcPanelUnitLane.vue` (`laneReady`), `AfcPanelUnitLaneActions.vue`; Fluidd `AfcCardUnitLane.vue`, `AfcCardUnitLaneActions.vue` | UI |
| §6.2 commands and who sends them | **Source**: grep of Mainsail `panels/Afc`, `dialogs/Afc*`, `StartPrintDialogAfc*`, `SpoolmanChangeSpoolDialog*`; Fluidd `widgets/afc/**`; H grep of `src/printer/ams_backend_afc.cpp`; button gating in Mainsail `AfcPanelButtons.vue#macros`, `AfcSettingsDialog*.vue` and Fluidd `AfcCardButtons.vue#commands`, `#macros`, `dialogs/AfcSettingsDialog*.vue`. Producer mappings from paxx `afc.cfg` and `38-feature-spoollink/.../klipper.cfg`, zmod `zmod_afclite.cfg` and `zmod_afclite.py`, Kiln `plugin.cfg` and `extras/kiln_afclite.py`. "AFC has no `SET_VENDOR`" from grep of AFC `extras/`. Capability tokens are **proposal** | UI; H; paxx; Z; K; AFC |
| §6.3 C1-C5 | **Proposal**. The facts C1 and C2 state are **source**: Klipper and Kalico `klippy/gcode.py#GCodeDispatch.cmd_default` (unknown command = `respond_info`, no raise), Mainsail and Fluidd `AfcUnitLaneFilamentDialog.vue#setSpool`, Klipper `klippy/gcode.py#_process_commands`, paxx `afc.cfg` `SET_WEIGHT`, H `ams_backend_afc.cpp`. C3 has no precedent: zmod_afclite and Kiln check object names, not command names | UI; Klipper `ee2c6fb`; Kalico `0057b6a`; paxx; H |
| §6.2 empty `COLOR`/`MATERIAL` | **Source**: AFC `extras/AFC_spool.py#cmd_SET_COLOR` (stores `'#' + COLOR`), `#cmd_SET_MATERIAL`; H `ams_backend_afc.cpp#clear_slot_override`, `#parse_afc_stepper`; zmod_afclite `zmod_afclite.cfg` `[gcode_macro SET_COLOR]`. The SHOULD is **proposal** | AFC; H; Z |
| §7 current heuristic | **Source**: H `src/printer/printer_discovery_parse.cpp#parse_objects`, `src/printer/ams_backend_afc.cpp#set_discovery` | H |
| §7 D1-D4 | **Proposal** | |
| §8 | **Proposal**; `zmod_slot` example from source | |
| Appendix B (Happy Hare) | **Source**: Mainsail `components/mixins/mmu.ts` and grep of `mmu?.` reads; `store/gui/getters.ts` | UI |

Faking Happy Hare's `mmu` panel is covered in Part B, Appendix B.

---

# Part B: Slicer data

## Part B.I: Research

## 1. How OrcaSlicer pulls filament data today

Every path ends in the same place. The agent builds Bambu-shaped tray JSON and hands it to
`DevFilaSystemParser::ParseV1_0`. The shared matcher (`PresetBundle::sync_ams_list`) then resolves
each tray's `tray_info_idx` (an Orca `filament_id`) to a filament preset.
[V `src/slic3r/Utils/MoonrakerPrinterAgent.cpp#build_ams_payload`, `docs/HLSD/filament_id.md`]

| Printer family | Agent | Transport | What it reads | How a lane becomes a preset / tool |
|---|---|---|---|---|
| **Bambu (AMS)** | `BBLPrinterAgent` | MQTT push (`print.push_status`) | Per tray (`DevFilaSystem.cpp` ParseV1_0):<br>• identity: `tray_info_idx` (Bambu `GF*` id), `setting_id`, `tray_uuid`, `tag_uid`, `tray_sub_brands`<br>• material and color: `tray_type`, `tray_color`, `cols` + `ctype` (multi-color)<br>• temps: `nozzle_temp_min/max`, `bed_temp`<br>• spool: `tray_weight`, `tray_diameter`, `remain` (%)<br>• calibration: `k`, `n`, `cali_idx` | `BBLPrinterAgent` translates `GF*` to Orca ids, then the shared matcher runs. The tray position is the AMS slot [V] |
| **Generic Klipper / Moonraker** | `MoonrakerPrinterAgent` | HTTP GET `/server/database/item?namespace=lane_data`, pulled on demand. Fallback: HTTP GET `/printer/objects/query?mmu` | `lane_data`: `lane` (required, 0-based slot; the record key is ignored), `material` (empty means empty lane), `color`, `bed_temp`, `nozzle_temp`, `filament_id` (**string only**: Happy Hare's numeric Spoolman id is treated as absent).<br>Happy Hare: `num_gates`, `gate_status`, `gate_material`, `gate_color`, `gate_temperature` | `filament_id` goes to the matcher unchanged (PR #14423, merged 2026-10-26). Otherwise `filament_id_by_type(material)`. Trays are laid out as 4-slot "AMS Lite" units by `lane` index. There is no tool or extruder mapping [V] |
| **Snapmaker U1** | `SnapmakerPrinterAgent` | HTTP GET `/printer/objects/query?print_task_config&filament_detect` | From `print_task_config`: `filament_exist`, `filament_type`, `filament_sub_type`, `filament_color_rgba`, `filament_vendor`.<br>From `filament_detect.info[i]`: `VENDOR`, `BED_TEMP`, `FIRST_LAYER_TEMP`; the temperatures are applied only when `VENDOR` is neither `"NONE"` nor empty | Sub-type is folded into the type (`PLA`+`SILK` becomes `PLA SILK`). The preset is the visible root preset with the same vendor and type that is nearest in RGB to the spool color, else by type [V] |
| **QIDI Box** | `QidiPrinterAgent` | HTTP GET `/printer/objects/query?save_variables=variables&multi_color_controller=box&box_stepper slot0..15=runout_button`, plus `/server/files/config/officiall_filas_list.cfg` (the material and color dictionary) | `box_count`, `color_slot<i>`, `filament_slot<i>`, `vendor_slot<i>`; presence from `runout_button == 0` | A composed id `QD_<series>_<vendor>_<typeidx>`. No preset carries it, so it falls back to type (filament_id.md table) [V] |
| **Creality K-series CFS** | `CrealityPrintAgent` | Creality's own WebSocket: `{"method":"get","params":{"boxsInfo":1}}` (`CrealityPrint.cpp#query_boxes_info`). Falls back to the Moonraker agent | `boxsInfo.materialBoxs[].materials[]`: `state`, `vendor`, `type`, `name`, `color` (`#0RRGGBB`) | Filters presets to the spool's base type, then scores: +20 for the brand name in the preset name, +10 for the vendor, −5 per preset-name word the spool didn't claim (`match_filament_preset`). Returns the winner's `filament_id`, else `filament_id_by_type` [V] |
| **Anycubic ACE, Prusa MMU, OpenPrintTag, OpenSpool** | none | — | — | Nothing on `main` reads them: no hits for `openprinttag`/`openspool`, and no ACE agent [V by grep; nothing in issue search] |

**A generic schema already exists in Orca, unused [V].** `DevFilaSystemParser::ParseAgentFilament`
arrived with the Python plugin system (commit `ecddf3d18f`, dated 2026-07-02; PR #14530 merged 2026-07-17). It parses an
agent-neutral filament document:

- **Units:** `units[]` with `id`, `extruder`, `type`, `temperature`, `humidity_percent`,
  `dry_time_min`.
- **Slots:** `units[].slots[]` with `index`, `loaded`, `material`, `preset_id`, `color`,
  `colors[]`, `nozzle_temp_min/max`, `remain_percent`, `k`, `diameter_mm`, `weight_g`.
- **External spools:** `external[]`.

On `main` it is declared and defined but **nothing calls it**. It is the shape Orca's own
developers chose for non-Bambu agents, and Part II's lane record is designed to map onto it
one-to-one.

**Open upstream work [V PR bodies; not merged]:**
- #16508 generalises N extruders and AMS units for Moonraker. Its body says "Parse AFC lane/tray
  data and Happy Hare toolhead mappings" and "Group synced device filaments by their associated
  extruder". The diff (not the body) maps AFC's `extruder`, `extruder1`, … names to indexes, reads
  `extruder_index` from `lane_data`, and falls back to `/printer/objects/query?AFC` plus
  `AFC_lane <name>.extruder`.
- #13719 adds vendor-aware name matching from `lane_data`.
- #12724 prefers custom presets when syncing.
- #16802 syncs when a printer connects.
- #16808 syncs only loaded slots.
- #15136 and #15857 read CFS over Moonraker.
- #16208 duplicates the already-merged #14423.

**The J0eB0l fork (a fork of Snapmaker/OrcaSlicer; PR #1 is open against J0eB0l's own `main`) [V doc/AFC.md, AFCProtocol.cpp, AFCManager.cpp]:**
- **Reads:**
  - the AFC Klipper webhook `printer.afc.status` (AFC registers `afc/status`,
    `extras/AFC.py:102`),
  - the `lane_data` namespace through `server.database.get_item`,
  - each lane's Spoolman spool, asking Moonraker's Spoolman proxy and AFC's own Spoolman address
    together and taking the first answer (`AFCManager.cpp#read_flow_k`), including a `flow_k` value
    in the spool's `extra` fields, which it applies as per-slot pressure advance. `flow_k` is
    written by J0eB0l's own AFC fork (`BridgeBox` branch `26c8b8d`, `extras/AFC_autocal.py`); it
    is not in upstream AFC `main` (`5ea84ea`) or `DEV` (`993fb45`).
- **Writes:**
  - `SET_MAP` before a print,
  - `SET_COLOR`/`SET_MATERIAL`/`SET_WEIGHT`/`SET_SPOOL_ID`/`AFC_SET_SPOOL_TEMP` from its lane editor,
  - `CHANGE_TOOL`/`TOOL_UNLOAD`/`LANE_UNLOAD`,
  - dryer commands (`AFC_ACE_DRY`, `AFC_BAMBU_HEATER_START`).
- It also renumbers the G-code's `T` commands to AFC T numbers.
- It is the most complete Klipper consumer found, and it lives outside upstream Orca.

## 2. Where `lane_data` came from

[V Moonraker PR #1022, AFC `extras/AFC_utils.py#send_lane_data`]

- In 2025-09 jimmyjon711 (AFC) opened Moonraker PR #1022, a `lane_data` component. Its body says it
  was "requested by AJAX3D to aid in a common API for filament changers so that third-parties like
  slicers can fetch data", and that AJAX3D "has been in talks with Softfever to add this fetch
  functionality into OrcaSlicer".
- Arksine (Moonraker) answered: use the Database API instead, and "The documentation could be
  added to the AFC project directly."
- jimmyjon711 replied: "We can do this for now, we were just trying to push something out there
  that was not necessarily tied just to AFC-Klipper-Add-On."
- Only after that, Arksine wrote, tentatively: "Perhaps we can provide some documentation in
  Moonraker for 3rd party namespaces, including specifications for the structure.
  `integrations.md` might be the best place to put it, let me give it some thought." The PR is
  still open.
- The PR's diff includes a draft schema: a "Lane Data" section for Moonraker's
  `docs/external_api/integrations.md` with a field table (`color`, `material`, `bed_temp`,
  `nozzle_temp`, `lane` marked REQUIRED; `lane` is a string described as "Current tool mapping for
  lane/slot. eg. T0/T1/T2/etc."; also `td` and `scan_time`). It was never merged.
- Moonraker `master` documents no `lane_data` namespace (grep of `docs/`).
- OrcaSlicer documents the reader side only: `docs/HLSD/filament_id.md` § "Moonraker
  `lane_data`" lists the fields its agent takes (`lane`, `material`, `filament_id`, `color`,
  `bed_temp`, `nozzle_temp`) and says the schema is not standardized (commit `20388f846d`,
  2026-10-26; current at `adea4e48a6`).
- AFC keeps a commented-out call to `machine/set_lane_data`.

So a draft schema exists (PR #1022's unmerged `integrations.md` table), but **no schema was ever
merged** anywhere, and Arksine's offer to host one was tentative ("Perhaps … let me give it some
thought"). Moonraker's `integrations.md` is the venue he named. This is directly relevant to the naming question in the main report: the AFC
maintainer wanted this surface to be ecosystem-wide, not AFC-branded [I from the quote].

**Writers of `lane_data` today** [V]:
- **AFC `main`.** Key = lane name. Value fields: `color`, `material`, `bed_temp`, `nozzle_temp`,
  `scan_time`, `td`, `lane`, `extruder_index`, `spool_id`, `weight`. `lane` is the lane's
  **mapped tool number** as a string (`lane_index` = `map` without the `T`), and records are written
  only for lanes whose `map` contains `T`. An emptied lane is rewritten with `""` for `color`,
  `material`, `bed_temp`, `nozzle_temp`, `scan_time` and `td`, `spool_id` as `null`, `lane` and
  `extruder_index` kept, and no `weight` (`clear_lane_data`). AFC `DEV` (`993fb45`) keys one
  record per mapped `T(n)` and removes stale `T` keys (`AFC_lane.py#send_lane_data`); it also
  writes `vendor_name`, `name` and `initial_weight`, and `clear_lane_data` blanks them.
- **Happy Hare.** Writes `laneN` records (`push_lane_data`), with a numeric `filament_id`.
- **HelixScreen.** Writes `laneN` or `T<n>` (`docs/specs/filament_slots.md`).
- **Mainsail.** Writes `T<n>` records for Spoolman tool changers (#2510).
- **zmod_afclite.** Re-checks every 15 s (`LANE_DATA_RECHECK`). It writes only where an entry disagrees with Z-Mod, merging `color`/`material` into existing entries found by their `lane` field (including HelixScreen's `T0..T3`), and drops spool-specific keys when the spool changed. It polls every 2 s until the slots settle.
- **spoolman-lane-sync** (broncosis, `1660093`). Keys are bare `"0"`, `"1"`, …. It writes `material`, `color`, `vendor`, `name`, `remaining_weight`, `setting_id`, and `filament_id`, copied from the Spoolman filament extra `orca_filament_id`. Its README tells users to put Bambu `GF*` codes there, which miss in the Moonraker path because only `BBLPrinterAgent` translates them. It claims the namespace with a top-level `source: "spoolman"` key (`#_claim_source`).

## 2a. Happy Hare already does much of this

Yes, Happy Hare already covers a good part of this ground. Source: moggieuk/Happy-Hare `main`
`70aa90fb` (2026-10-19), plus Moonraker PR #962 and Mainsail `develop`.

### 2a.1 What Happy Hare does

All citations are moggieuk/Happy-Hare `main` at commit **`70aa90fb`** (2026-10-19), written as
`path#symbol`.

**A. The gate map: the per-gate state model.**
`extras/mmu/mmu_gate_maps.py#__init__` declares the persisted arrays in `_gate_map_vars`. Each
array has `num_gates` entries and a config default `default_<attr>`.

| Status key | Element type | Default | Values and units | Persisted as (save_variables) |
|---|---|---|---|---|
| `gate_status` | int | −1 | `GATE_UNKNOWN` −1, `GATE_EMPTY` 0, `GATE_AVAILABLE` 1, `GATE_AVAILABLE_FROM_BUFFER` 2 (`mmu_constants.py`) | `mmu_state_gate_status` |
| `gate_filament_name` | str | `""` | Free text. The slicer path fills it from Orca/Prusa `filament_settings_id`, i.e. the **slicer preset name** | `mmu_state_gate_filament_name` |
| `gate_material` | str | `""` | Uppercased by `MMU_GATE_MAP`'s per-gate `GATE=` path; the batch `MAP=` path (Spoolman sync, Fluidd) stores it as given (`commands/mmu_gate_map.py`) | `mmu_state_gate_material` |
| `gate_vendor` | str | `""` | | `mmu_state_gate_vendor` |
| `gate_color` | str | `""` | `""`, a **W3C color name** (`red`), or `rrggbb[aa]` lowercase **without `#`** (`mmu_utils.py#MmuColorUtils.validate_color`) | `mmu_state_gate_color` |
| `gate_color_rgb` | (r,g,b) floats 0-1 | derived | `MmuColorUtils.color_to_rgb_tuple`; not persisted | — |
| `gate_temperature` | int °C | `default_extruder_temp` | Nozzle temp. `MMU_GATE_MAP`'s batch `MAP=` path raises it to at least the default; the per-gate `GATE=` path stores `TEMP` as given | `mmu_state_gate_temperature` |
| `gate_spool_id` | int | −1 | Spoolman spool id, −1 none | `mmu_state_gate_spool_id` |
| `gate_spool_rfid` | str | `""` | One normalized tag UID per gate (`normalize_gate_rfid`). Spoolman aliases are cached separately and not persisted | `mmu_state_gate_spool_rfid` |
| `gate_td` | number\|None | None | HueForge TD (TD-1 measurement) | `mmu_state_gate_td` |
| `gate_td1_color` | str | `""` | TD-1 measured color | `VARS_MMU_GATE_TD1_COLOR` |
| `gate_speed_override` | int % | 100 | Per-gate load speed | `mmu_state_gate_speed_override` |

- **Lifecycle.**
  - `load_persisted_state` restores the arrays from `[save_variables]` through
    `mmu_utils.py#SaveVariableManager` (the `mmu_state_*` keys).
  - `persist_gate_map` writes them all, then:
    - rewrites the `T<n>` macro variables (`update_t_macros`),
    - syncs Spoolman in push or readonly mode,
    - fires `MACRO_EVENT_GATE_MAP_CHANGED`,
    - **pushes `lane_data`** (`_moonraker_push_lane_data`).
  - A gate going EMPTY clears its filament attributes (`set_gate_status` →
    `clear_gate_attributes`), and `gate_filament_changed` drops its measurements.
  - Arrays are replaced rather than mutated (`renew_gate_map`, "to ensure that webhooks sees
    get_status() change after gate map update").
- **Multi-color: none.** One color per gate, and no gradient or segment model (grep of
  `extras/` and `components/`).

**B. Tool-to-gate map, endless spool, slicer tool map.**
- **Tool-to-gate map.** `ttg_map`: list[int], tool → gate, default identity. Persisted as
  `mmu_state_tool_to_gate_map`; set with `MMU_TTG_MAP TOOL= GATE=` or `MAP=`
  (`commands/mmu_ttg_map.py`). `MMU_REMAP_TTG` is an alias macro for it
  (`config/macros/mmu_misc.cfg`). Several tools may share a gate.
- **Endless spool.** `endless_spool_groups`: list[int], gate → group id. Persisted as
  `mmu_state_endless_spool_groups`, plus `mmu_state_enable_endless_spool`. `get_next_endless_spool_gate`
  picks the next non-empty gate in the same group.
- **Slicer tool map.** `slicer_tool_map` is
  `{'tools': {str(t): {color, material, temp, name, in_use}}, 'referenced_tools', 'initial_tool',
  'purge_volumes', 'total_toolchanges', 'skip_automap'}`. `_reset_slicer_tool_map` sets all
  but `skip_automap`, which `restore_automap_option` adds.
  - It is **not persisted**; it is rebuilt every print.
  - It is filled by `MMU_SLICER_TOOL_MAP TOOL= MATERIAL= COLOR= NAME= TEMP= USED=
    [PURGE_VOLUMES=] [NUM_SLICER_TOOLS=] [INITIAL_TOOL=] [TOTAL_TOOLCHANGES=] [AUTOMAP=]
    [SKIP_AUTOMAP=]` (`commands/mmu_slicer_tool_map.py`).
  - The shipped macro `MMU_START_SETUP` calls it once per tool (`config/macros/mmu_software.cfg`).
    It takes its arguments from placeholders that Happy Hare's Moonraker pre-processor fills in
    the slicer's start G-code (D below).
- **Automap.** `mmu_gate_maps.py#automap_gate` maps each slicer tool to a gate with
  `MMU_TTG_MAP`, by one of these strategies:

  | Strategy | How it matches |
  |---|---|
  | `filament_name` | `gate_filament_name`, NFKC-normalized, case-insensitive |
  | `material` | exact match |
  | `color` | `ljust(8,'F')` equality |
  | `closest_color` | material must match first, then nearest color, with quality warnings |
  | `spool_id` | rejected: "not yet supported" |

  With several matches it warns and the last one wins.
- **`T<n>` macro variables.** `update_t_macros` sets `gcode_macro T<n>.color` (hex, no `#`) and
  `.spool_id` on the tool macros "for Mainsail/Fluidd to pick up". The `t_macro_color` mode
  (`slicer`/`gatemap`/`allgates`/`off`) chooses whether the color comes from the slicer map or
  the gate map.

**C. Units (multi-MMU).**
- `extras/mmu_machine.py#MmuMachine` builds one `MmuUnit` per `[mmu_unit <name>]`, and gates are
  numbered contiguously across units.
- Its status (`machine_status`, static after load) has `happy_hare_version`, `num_units`,
  `num_gates`, and `unit_<i>` = `extras/mmu/mmu_unit.py#get_status`:
  - `name`, `display_name`, `vendor`, `version`
  - `num_gates`, **`first_gate`**
  - `selector_type`, `is_homed`, `has_bypass`, `multi_gear`, `filament_buffer`, plus other
    hardware flags
  - optional `environment_sensor(s)`, `filament_heater(s)`, `fan(s)`, `vent_servo(s)`,
    `nfc_reader(s)`, `td1_device(s)`
- Each unit is configured with an `extruder_name` (`mmu_unit.py`), but **it isn't in the
  status**, so a consumer can't tell which extruder a unit feeds.

**D. Moonraker component `components/mmu_server.py`.**
- **(1) Remote methods.** No HTTP endpoints; all are remote methods called from Klipper, listed
  in `MmuServer.__init__`.
  - Spoolman (only when `[spoolman]` exists):
    - `spoolman_refresh`, `spoolman_get_filaments`
    - `spoolman_push_gate_map`, `spoolman_pull_gate_map`
    - `spoolman_clear_spools_for_printer`
    - `spoolman_set_spool_gate`, `spoolman_unset_spool_gate`
    - `spoolman_get_spool_info`, `spoolman_display_spool_location`
    - `spoolman_get_spool_by_uid`, `spoolman_set_spool_uid`
  - Always: `mmu_td1_request`, `moonraker_push_lane_data`, `moonraker_cleanup_lane_data`.
- **(2) Spoolman sync.** The mode is `spoolman_support` = `off` / `readonly` / `push` (the gate
  map is the truth) / `pull` (Spoolman is the truth) (`mmu_constants.py`).
  - **State in Spoolman.** The printer↔gate assignment lives in Spoolman **extra fields** that
    the component creates if missing: `printer_name`, `mmu_gate_map`, and `rfid_tag` (display name "RFID"; `MMU_RFID_FIELD = 'rfid_tag'` l.87, `_add_extra_field` l.267; only the file-header comment says `rfid`) (a
    comma-separated UID list) (header comment, `_add_extra_field`).
  - **Caching.** A cache `spool_location` maps spool → (printer, gate, attrs). Attrs are
    `_get_filament_attr`: `material`, `color` (`color_hex` without `#`), `name`, `temp`
    (`settings_extruder_temp`), `bed_temp`, `vendor`, `filament_id` (**Spoolman's filament
    record id**), `rfids`.
  - **Back to Klipper.** Results return to Klipper as
    `MMU_GATE_MAP MAP="{gate: attrs}" [REPLACE=1] FROM_SPOOLMAN=1 QUIET=1`
    (`_send_gate_map_update`).
  - **Tag scans.** A scan resolves a UID to a spool, or auto-creates vendor → filament → spool
    from tag metadata, using SpoolmanDB densities and Bambu lookups
    (`get_spool_by_uid`, `_create_spool_from_metadata`).
- **(3) `lane_data` writer.** `push_lane_data` queries the `mmu` object and batch-writes
  `lane_data` with `database.insert_batch`.
  - Key: `lane{gate}`, **0-based**.
  - Value: `vendor_name` (`gate_vendor`, else the Spoolman vendor), `name` (`gate_filament_name`,
    else the Spoolman name), `color` (**raw `gate_color`**, so possibly a W3C name), `td`,
    `material`, `bed_temp` (Spoolman only), `nozzle_temp` (`gate_temperature`, default 200),
    `scan_time` (always None), `lane` (str(gate)), `spool_id`, `filament_id` (Spoolman's numeric
    id).
  - Empty gates (status −1 or 0) get the same keys, all None except `lane`.
  - `cleanup_lane_data` reads the whole namespace and deletes records whose `lane` ≥
    `num_gates`. It looks only at the `lane` field, so it also deletes **other writers'**
    records in that range.
  - Nothing else in the component reads `lane_data` (grep).
- **(4) G-code metadata pre-processor.**
  - `setup_placeholder_processor` **replaces Moonraker's `file_manager.METADATA_SCRIPT`** with
    this module. That is a Moonraker monkeypatch with long-standing precedent.
  - It runs for files from `PrusaSlicer`, `SuperSlicer`, `OrcaSlicer` and `BambuStudio`
    (`AUTHORZIED_SLICERS`), then chains to Moonraker's own `metadata` parser.
  - It substitutes these placeholders in non-comment lines:

    | Placeholder | Source in the slicer's file |
    |---|---|
    | `!referenced_tools!` | `T<n>` / `MMU_CHANGE_TOOL` scan |
    | `!total_toolchanges!` | same scan |
    | `!colors!` | Orca `filament_colour`, PS `extruder_colour` |
    | `!temperatures!` | `(nozzle_)temperature` |
    | `!materials!` | `filament_type` |
    | `!purge_volumes!` | `flush_volumes_matrix` × `flush_multiplier`, Orca ≥2.3.2 aware |
    | `!filament_names!` | `filament_settings_id` |

  - Optionally rewrites `T<n>` as `MMU_CHANGE_TOOL TOOL=<n> NEXT_POS="x,y"`, and appends
    `; referenced_tools = …`.
  - `; processed by HappyHare` marks a processed file.

**E. Upstream Moonraker metadata from the same author.** Moonraker PR #962 (moggieuk, merged
2025-02-26) added metadata fields to Moonraker itself, in `file_manager/metadata.py`:
`filament_colors`, `extruder_colors`, `filament_temps`, `referenced_tools` and `mmu_print`. It
left the `filament_name`/`filament_type` parsers single-valued and changed only their
documentation to "name(s)"/"type(s)". Pedro Lamas's later commit `5b92e52e` (2025-03-12,
"metadata: filament_name and filament_type as array") made them multi-valued: with several
values they return a **JSON-encoded string** of the list, otherwise a plain string
(`metadata.py#parse_filament_type`). (`filament_change_count` and `filament_weights` are also in
`metadata.py`, from neither.) The PR says the new fields map "tools to MMU spools when using
Happy Hare". These are generic `server.files.metadata` fields any producer can read.

**F. Slicers.**
- Happy Hare has no slicer plugin. Its slicer story is three pieces:
  - the start-G-code placeholders (D4),
  - the Moonraker file metadata (E),
  - `lane_data` for OrcaSlicer (D3).
- Orca reads `lane_data` first and falls back to `mmu.gate_*` (§1). Orca PR #12307 (merged
  2026-02-15) added the Happy Hare path.
- **Interop bug:** a W3C color name in `gate_color` reaches `lane_data.color` unchanged, and
  Orca's `normalize_color_value` keeps only hex digits, so `red` arrives as invalid and falls
  back to `00000000` [V both sides; not seen in the field].

**G. Consumers of Happy Hare's surface.** Sources: KlipperScreen-Happy-Hare-Edition `master`
`1ad01a1`, Fluidd `develop` `738ff93`, Mainsail `develop` `03a4d7b1`. Fields were found by
grepping access patterns, so a read through a renamed variable may be missing.

- **How each decides Happy Hare is present.**
  - KlipperScreen-HH: an `mmu` or `mmu_machine` config section (`ks_includes/printer.py#reinit`).
    It treats `mmu_machine.happy_hare_version` ≥ 4.0.0 as v4.
  - Fluidd: `printer.mmu != null`. Its print-start tool-map dialog needs `mmu.enabled` and a file
    with `referenced_tools` (`FileSystem.vue`).
  - Mainsail: `printer.mmu`. Its start-print section needs `mmu.enabled && mmu.gate != -2`
    (`StartPrintDialog.vue#existsMmu`).
- **Per-gate fields all three read:** `gate_status`, `gate_color`, `gate_material`,
  `gate_spool_id`. The web UIs also read `gate_filament_name`, `gate_temperature` and
  `gate_speed_override`.
- **Unit fields:** `mmu_machine.unit_N.{name, num_gates, first_gate, has_bypass, …}`.
- **File metadata.**
  - Fluidd reads `filament_colors` (preferred for Orca/Bambu files), `extruder_colors`,
    `filament_type`, `filament_temps`, `filament_name` and `referenced_tools`
    (`mixins/mmu.ts#toolDetails`).
  - Mainsail prefers `extruder_colors` (`helpers.ts#getFileFilamentColors`). Its start-print MMU
    section takes the tools a file uses from `referenced_tools`
    (`StartPrintDialogMmu.vue#summary`); its TTG-map dialog uses the indexes where
    `filament_weights > 0` (`MmuEditTtgMapDialog.vue#fileTools`).
  - KlipperScreen-HH reads no MMU tool metadata (`referenced_tools`, `filament_colors`,
    `extruder_colors`, `filament_temps`, `filament_weights`). Its file browser shows
    `filament_type` and `filament_name` (`panels/gcodes.py`).
- **Commands, and the three disagree on shape.**
  - Gate edits: Fluidd sends every gate in one
    `MMU_GATE_MAP MAP="{g: {status, spool_id, material, color, name, temp, speed_override}}"`.
    Mainsail and KlipperScreen send `MMU_GATE_MAP GATE=n <KEY>=v`: Mainsail one attribute per
    call (several for its Spoolman fill), KlipperScreen-HH several in one call.
  - Tool remap: Mainsail sends `MMU_REMAP_TTG TOOL= GATE=` per tool; the others send
    `MMU_TTG_MAP MAP=a,b,c`.
  - Load: `MMU_CHANGE_TOOL TOOL=n` (or `GATE=n`: "whichever tool is mapped to" that gate; errors if none), or `MMU_SELECT GATE=n` then `MMU_LOAD`. `MMU_LOAD` (`EXTRUDER_ONLY`, `SKIP_PURGE`, `RESTORE`) and
    `MMU_UNLOAD` take no gate.
  - Eject: `MMU_EJECT GATE=n`.
  - Preload: `MMU_PRELOAD GATE=n`.

### 2a.2 Verdict, piece by piece

| Happy Hare piece | Verdict | Why |
|---|---|---|
| `lane_data` record shape (`vendor_name`, `name`, `td`, `spool_id`, null-filled empty lanes, one batch write) | **Adopt** the field names. **Stay compatible** with the 0-based `lane{gate}` keys | Two writers (Happy Hare, AFC) already agree on `td`, `spool_id`, `color`, `material`, `bed_temp`, `nozzle_temp`. Happy Hare adds `vendor_name`/`name`, and AFC `DEV` (`993fb45`) writes them too. Inventing `filament_vendor`/`filament_name` would make a third spelling, so §7 now uses Happy Hare's |
| Happy Hare's numeric `filament_id` | **Stay compatible**, don't adopt | Orca's string `filament_id` (the `OF…` hash) already owns the key on the consumer side. The spec gives the Spoolman number its own field (`spoolman_filament_id`), and consumers MUST read a numeric `filament_id` as Spoolman's |
| Moonraker file metadata (`referenced_tools`, `filament_colors`, `filament_type`, `filament_name`, `filament_temps`) | **Adopt as-is** | Already upstream in Moonraker. It replaces this draft's earlier `; lane_map` header proposal: a producer can map with no slicer-side write at all |
| Printer-side automap (`MMU_SLICER_TOOL_MAP` + `AUTOMAP=` strategies) | **Adapt** | The right model for "slicer says what it needs, printer decides which lane". The strategy names become the spec's `automap` vocabulary (§9). The command itself stays Happy Hare's |
| `MMU_TTG_MAP TOOL= GATE=` vs AFC `SET_MAP LANE= MAP=T` | **Stay compatible with both** | Same intent, two syntaxes, both in use by Mainsail. The spec names the intent (`set_map`) and gives each profile its own syntax (§9) |
| Spoolman extras `printer_name`/`mmu_gate_map`/`rfid_tag` | **Stay compatible**; don't spread further | A third tag-UID convention beside SpoolLink's `card_uids` (paxx, zmod_afclite, KILN_AFCLite; Part C) and Spoolman's native tags (Spoolman ≥0.27, which paxx's open PR #799 would migrate to). The spec carries tags in `lane_data.tag` and leaves Spoolman storage to each producer |
| `gate_*` arrays as a slicer source | **Stay compatible** (bridge table §7a) | Orca already reads them. A producer MAY mirror lanes into them only by publishing a real `mmu` object, which Appendix B advises against |
| Metadata pre-processor and placeholders | Out of scope | It rewrites G-code for Happy Hare's own tool-change path. Producers that need the same data read Moonraker's file metadata |
| `T<n>` macro variables `color` / `spool_id` (`update_t_macros`) | **Stay compatible; recommend** | A third existing channel that Mainsail and Fluidd already read per tool. A producer that knows the tool→lane map SHOULD set them the same way: hex color without `#`, and `spool_id` only when the lane is loaded |
| W3C color names in `gate_color` | **Don't adopt** | The spec requires hex. A Happy Hare bridge or writer MUST convert names before writing `lane_data` or the profile objects (see the interop bug in §2a.1 F) |
| `gate_filament_name` = slicer preset name (`filament_settings_id`) | **Adapt** | In Happy Hare's flow the gate "name" is often an Orca/Prusa **preset name**, while in Spoolman-fed gates it is the Spoolman filament name. The spec keeps `name` as the product name (the mint input) and adds `slicer_preset` for a preset name (§7) |
| `cleanup_lane_data` deletes by `lane` index regardless of writer | **Fix upstream / guard against** | With two writers on one machine it deletes records it doesn't own. Rule S2 (one writer, `writer` field) is the answer; propose to moggieuk that cleanup only deletes `lane{n}` keys it wrote |
| Units: `mmu_machine.unit_<i>` with `first_gate`/`num_gates`; per-unit `extruder_name` not published | **Adopt** the unit model in the bridge; **ask upstream** to publish `extruder_name` in `unit_<i>` | The unit → extruder link is what multi-toolhead slicer grouping needs (Orca #16508) |
| Replacing Moonraker's `METADATA_SCRIPT` | Precedent only | Relevant to the main report's §4: Happy Hare already modifies Moonraker internals from a component, which tells us the plugin route is tolerated by users. It doesn't change the recommendation against injecting printer objects |

## 3. What Orca needs for a good match

From what the matcher and parsers consume [V], plus what the open PRs and the fork add [V PR
bodies]:

| Need | Orca uses it for | Who can supply it today |
|---|---|---|
| **Orca `filament_id`** (string, `OF…`) | Exact preset: the matcher's first key. Without it, everything falls back to "Generic <type>" | spoolman-lane-sync writes the field, but from a user-filled Spoolman extra that its README fills with Bambu `GF*` codes (which miss on this path). No producer writes a minted `OF…` id. It is **computable on the printer**: `"OF" + base62_6(uuid5(c4d3ff49-4c32-5534-a3e3-00894157ab97, "filament_product/<vendor>/<type>/<name>"))` [V filament_id.md, Minting]. It only resolves if the name equals an Orca preset's base name |
| `setting_id` | A specific preset (user presets). Read in the Bambu path | Bambu only |
| vendor, product name | Vendor-aware matching (#13719, CFS scoring, Snapmaker vendor match) | Spoolman, RFID tags, CFS, U1 |
| type + sub-type | Type fallback | Everyone |
| color; colors + gradient flag | Slot color; multi-color spools (`tray_colors`, `cols`/`ctype`) | AFC `multi_color_hexes`. U1 multicolor is proposed in paxx PR #818, an open draft |
| nozzle temp min/max, bed temp | Tray hints; only `nozzle_temp_max` and `bed_temp` reach the tray today | Spoolman, tags |
| remaining weight or percent | AMS display (`remain`) | AFC `weight`, Spoolman |
| diameter, density | Tray display; preset creation in the fork | Spoolman |
| pressure advance (K) | Bambu `k`, the fork's `flow_k` override | Spoolman extra `flow_k`, written by J0eB0l's AFC fork (`AFC_autocal.py`, `BridgeBox` branch). Not upstream AFC. HelixScreen uses a Spoolman extra `pressure_advance` |
| extruder per lane; nozzle diameter per extruder | Multi-toolhead grouping (#16508); nozzle sync | AFC `extruder_index`; Klipper `configfile` `nozzle_diameter` |
| tag identity (uid, format) | Bambu `tag_uid`/`tray_uuid` | U1 `filament_detect` publishes `CARD_UID` and `CARD_TYPE`; Orca reads `filament_detect.info` only for `VENDOR`, `BED_TEMP` and `FIRST_LAYER_TEMP`. Others: see the tracker survey (Part C) |
| Spoolman spool id | Fork: details and flow K | AFC, HelixScreen, Happy Hare |

**An Orca `filament_id` names a product, not a preset [V Orca `adea4e48a6`].**
- Orca's rule: the id "identifies one **filament product** … shared by all of that product's
  per-printer / per-nozzle variants". It is "never … per-preset (per-preset identity is
  `setting_id`)". Profile authors are told "one already in the tree has one — inherit it"
  (`docs/HLSD/filament_id.md`, opening and "The design").
- Children inherit it in code. A system preset with no `filament_id` of its own takes its
  parent's (`src/libslic3r/PresetBundle.cpp#PresetBundle::resolve_vendor_preset`, through
  `filament_id_maps`). A user preset with `inherits` always takes its parent's, even if its
  own file carries one (`src/libslic3r/Preset.cpp#PresetCollection::resolve_user_preset`).
  Example: `Generic ABS @System` (OrcaFilamentLibrary) carries `OFY9muEs`, and Sovol's
  `Generic ABS @Sovol SV08` and `Generic ABS @Sovol SV08 MAX` inherit it with no id of their
  own (`resources/profiles/`).
- The matcher takes the first compatible preset whose id matches. Its filter
  `get_preset_base(f) == &f` (`PresetBundle.cpp`) passes every system preset, per-printer
  system variants included, because `get_preset_base` returns a system preset itself
  (`Preset.cpp#PresetCollection::get_preset_base`, `adea4e48a6`); it also passes a user root
  preset, and excludes only user presets that inherit from another. Orca's own doc describes this more narrowly, as a
  derived preset that "resolves through its root and never matches directly"
  (`filament_id.md`, "Who consumes the id"). Two compatible presets with one id make the pick
  arbitrary: Orca calls that a profile bug, logs it
  (`PresetBundle.cpp#warn_ambiguous_filament_id_match`) and checks for it in its validator
  (`#check_duplicate_filament_subtypes`).

So one id is shared by every variant of a product by design, and writing the id of a user's
tuned preset selects the first compatible system preset carrying that id, not the user's
preset. A producer that needs one particular preset
names it in `slicer_preset` (§7). A user-made root filament carries its own `P*` id
(`filament_id.md`).

## 4. The write direction

| Path | What is pushed | Transport |
|---|---|---|
| Bambu | `ams_mapping` in the print command; `ams_filament_setting` (tray id, setting id, color(s), temps, type) from the tray editor; `extrusion_cali_sel` (K) | MQTT [V `DeviceManager.cpp`, `DevMappingNozzle.cpp`] |
| Creality CFS | `colorMatch {path, list:[{id,type,color,boxId,materialId}]}` and then `multiColorPrint` at print start | Creality WS [V `CrealityPrint.cpp#start_print_commands`] |
| Moonraker agent (upstream) | **Nothing.** `handle_request` handles pause/resume/stop, `home`, temperatures, `gcode_line`, `get_version` and `get_access_code`, but not `ams_filament_setting`. The print starts with `SDCARD_PRINT_FILE` and no mapping [V] | — |
| J0eB0l fork; Mainsail/Fluidd print dialogs | `SET_MAP LANE=<lane> MAP=T<n>` per tool before the print; lane edits as AFC G-code | Moonraker `printer.gcode.script` [V] |
| Python agent plugins | `PrintParams.ams_mapping`/`ams_mapping2`/`nozzle_mapping` are exposed to plugins (`PrinterAgentPluginCapability.cpp`), so a plugin agent could send a mapping however it likes [V] |

**How a Klipper printer should receive it** [I]:
- G-code through `printer.gcode.script` is the only channel every Klipper producer already
  accepts, and the one the UIs and the fork use. A mapping is a batch of `SET_MAP` lines sent
  **before** `SDCARD_PRINT_FILE`, so a failure shows before the print starts.
- An alternative that needs no round trip is a header comment in the uploaded G-code
  (`; lane_map = T0=E2,T1=E0`) that the producer applies at `PRINT_START`. This is more robust
  (the mapping travels with the file and survives a reprint from the printer's own screen), but
  every producer has to parse it.

## 5. Transport fit

- **Slicers poll; they don't subscribe [V].** Each agent fetches on demand over HTTP: a sync
  click, or a connect (#16802). Only the status stream (print progress) uses Moonraker's
  WebSocket.
- **`printer/objects/query` suits live state, not profile data [I].**
  - It returns Klipper objects, which hold presence, the active lane and errors.
  - Profile identity (Orca id, PA, Spoolman links, tag payloads) changes rarely and should
    survive a Klipper restart.
  - In Klipper it needs a Python producer; HelixScreen's app (C++) and plugin (Moonraker)
    can't feed it without the Klipper extra from the main report.
- **The Moonraker database (`lane_data`) is the better slicer surface [V + I].**
  - Orca already reads it with one GET.
  - Moonraker's maintainer recommended it for exactly this.
  - It persists, and it answers while Klippy is down.
  - Any process can write it: a Klipper extra (AFC, zmod_afclite over HTTP), a Moonraker
    component (Happy Hare's, our HelixPrint plugin), or an app (HelixScreen).
  - Its weakness is ownership. AFC deletes the namespace on boot, several writers collide, and
    the only schema is PR #1022's unmerged draft. spoolman-lane-sync already claims the namespace with a top-level `source: "spoolman"` key and refuses to write when another source owns it (`#_claim_source`). Part II answers this with a version field and a per-record `writer` field, and treats a namespace-level `source` key as legacy ownership a producer MUST respect.
- **A HelixPrint endpoint (`/server/helix/…`) is the wrong surface for slicers [I].**
  - Orca would need HelixScreen-specific code, against the direction of `filament_id.md` ("they
    are intended to converge on the same pattern") and against the neutral spec.
  - The plugin's right role is **writer**: it lives in Moonraker, so it can maintain the
    extended `lane_data` records from the app's resolved lanes.
- **The AFC webhook (`printer.afc.status`) is a precedent for an on-demand Klipper endpoint
  [V].** It isn't needed if `lane_data` carries the profile data and `lane_status` carries
  live state.

---

## Part B.II: Proposal, the slicer profile

## 6. Model

The two halves stay separate on purpose:
- **Live state:** the Klipper objects of the Part A (`lane_status`, and the `afc` profile
  for current UIs). Presence, active lane and errors.
- **Slicer profile:** one record per lane in the Moonraker database, namespace `lane_data`,
  extending the record Orca already reads. Identity, matching hints, calibration, tags.

A slicer reads both in two HTTP GETs:
1. `/printer/objects/query?lane_status` (live; optional).
2. `/server/database/item?namespace=lane_data` (profile; required).

**S1.** A producer conforming to this profile MUST write one `lane_data` record per lane it
publishes, and MUST keep every field it wrote in agreement with the lane's current spool.

**S2.** Exactly one writer per machine SHOULD own `lane_data`. A producer MUST set `writer` (§7)
and MUST NOT overwrite a record whose `writer` names another producer unless that producer is
absent from `lane_status`. Records with no `writer` are legacy: a producer MAY replace them.

**S3.** The record key is advisory; consumers MUST use the `lane` field. Keys SHOULD follow
`docs/specs/filament_slots.md` §4 (`T<n>` for tool changers, `laneN` otherwise).

## 7. Record fields

Column key:
- **Req**: required level for a producer of this profile.
- **Orca today**: whether `MoonrakerPrinterAgent` on `main` reads the field.
- **Writers today**: A = AFC `main`, HH = Happy Hare, H = HelixScreen, Z = zmod_afclite,
  S = spoolman-lane-sync.

Group 1: fields Orca reads today (the compatibility core).

| Field | Type | Units | Req | Meaning | Orca today | Writers today |
|---|---|---|---|---|---|---|
| `lane` | string (int) | | MUST | Slot index Orca lays out. MUST be a JSON string (Orca drops a numeric `lane`). SHOULD equal the slicer tool number the lane serves (`T<n>` → `"n"`) when a mapping exists. This formalises AFC `main`'s existing meaning and PR #1022's draft ("Current tool mapping"); Happy Hare uses the gate number | yes (required) | A H HH Z S |
| `material` | string | | MUST | Slicer-matchable type (`PLA`, `PETG-CF`). **Empty means empty lane** in Orca today | yes | A H HH Z S |
| `color` | string | `#RRGGBB` | MUST if loaded | Primary color. MUST be hex: Happy Hare's W3C names (`red`) don't survive Orca's normalizer | yes | A H HH Z S (Mainsail and S write it without `#`) |
| `nozzle_temp` | int | °C | SHOULD | Recommended nozzle temp. MUST be a JSON number. Orca sends it as `nozzle_temp_max` | yes | A H HH (default 200 when unset). Z deletes it on a spool change |
| `bed_temp` | int | °C | SHOULD | MUST be a JSON number | yes | A H HH (from Spoolman). Z deletes it on a spool change |
| `filament_id` | string | | SHOULD | **Orca filament id** (`OF…`), copied from Orca's library or minted as in §3 from `vendor_name`/`filament_type`/`name`. It names a product, not a preset (§3): a producer MUST NOT write an id that more than one product carries. MUST be a string. A numeric value means Spoolman's filament id (Happy Hare's usage); Orca ignores it | yes (string only) | S (HH numeric) |

Group 2: proposed fields Orca doesn't read yet.

| Field | Type | Units | Req | Meaning | Orca today | Writers today |
|---|---|---|---|---|---|---|
| `lss_version` | string | | MUST | `"0.1"`. Consumers ignore unknown majors | no | — |
| `writer` | string | | MUST | Producer name, as in `lane_status.producer.name` | no | — |
| `lane_name` | string | | MUST | The lane's name in `lane_status` / `AFC_lane` (`E2`, `lane4`). It joins the record to live state | no | (A: the key) |
| `loaded` | bool | | MUST | Explicit presence. Consumers that know this field MUST prefer it over inferring from `material` | no | — |
| `extruder_index` | int | | SHOULD | 0-based extruder the lane feeds | no (#16508) | A |
| `tool` | string\|null | | SHOULD | Current slicer tool, `"T3"`, or null when unmapped | no | — |
| `vendor_name` | string | | SHOULD | Manufacturer (`"Polymaker"`, `"Generic"`) as Orca's `filament_vendor` means it. **Happy Hare's key, adopted** | no (#13719) | HH, H (also `vendor`); S writes `vendor` |
| `name` | string | | SHOULD | Product name without color or variant (`"PolyLite PLA"`): the mint input. **Happy Hare's key, adopted** | no | HH, H (also `spool_name`), S |
| `td` | number\|null | | MAY | HueForge transmission distance (TD-1). Same key as Happy Hare and AFC | no | HH A |
| `slicer_preset` | string\|null | | MAY | A slicer preset name this lane was set from (Orca/Prusa `filament_settings_id`, e.g. `"Bambu PLA Basic @BBL X1C"`). It is Happy Hare's `filament_name` automap key | no | (HH puts it in `name`) |
| `filament_type` | string | | SHOULD | Precise type (`"PLA Silk"`). `material` stays the matchable one | no | H (`helix_material`) |
| `colors` | list[string] | `#RRGGBB` | SHOULD for multi-color | Every color in order. `color` = `colors[0]` | no (struct has `tray_colors`) | (AFC status `multi_color_hexes`) |
| `color_mode` | string | | SHOULD with `colors` | `"solid"`, `"segments"`, `"gradient"` (Orca `ctype`) | no | — |
| `nozzle_temp_min`, `nozzle_temp_max` | int | °C | MAY | Range | no | — |
| `diameter_mm` | number | mm | SHOULD | | no | — |
| `density_g_cm3` | number | g/cm³ | MAY | | no | — |
| `remaining_weight_g` | number | g | SHOULD | Negative = unknown | no | H (AFC `weight`) |
| `total_weight_g` | number | g | MAY | Full net weight | no | H |
| `remain_percent` | int | % | MAY | 0-100, −1 unknown | no | — |
| `pressure_advance` | number | s | MAY | Klipper `pressure_advance` for this spool on `extruder_index`'s nozzle | no | (fork: Spoolman `flow_k`) |
| `pressure_advance_source` | string | | with PA | `"spoolman"`, `"calibration"`, `"tag"`, `"user"` | no | — |
| `flow_ratio` | number | | MAY | Extrusion multiplier | no | — |
| `nozzle_diameter_mm` | number | mm | MAY | Of the extruder in `extruder_index`. Saves a `configfile` read | no | — |
| `spool_id` | int\|null | | SHOULD | Spoolman spool id | no | A H HH |
| `spoolman_filament_id` | int\|null | | MAY | Spoolman filament record id. Happy Hare writes this number into `filament_id`, so a consumer MUST read a **numeric** `filament_id` as this field | no | H (`helix_spoolman_filament_id`), HH (as `filament_id`) |
| `spool` | object\|null | | MAY | The spool this lane holds, in any tracker: `{"tracker": string, "id": string}`. `tracker` is a short lowercase name for the tracker (`spoolman`, `filaman`, `simplyprint`, …; Part C); the vocabulary is open, and a consumer that doesn't know a name treats the reference as opaque. `id` is that tracker's spool id as a string. When `tracker` is `spoolman`, `id` MUST equal `spool_id` written as a string. null when the lane references no spool | no | — |
| `tag` | object\|null | | MAY | `{"format": "openprinttag"\|"openspool"\|"bambu"\|"creality"\|"snapmaker"\|"tigertag"\|"uid", "uid": hex string, "data": object}` | no | — |
| `scan_time` | string | ISO-8601 UTC | SHOULD | Last write | no | A H |

**V-S1.** Vendor-specific keys MUST carry a `<vendor>_` prefix, as in the Part A §8.
HelixScreen's existing `helix_*` keys conform.

**V-S2.** An unprefixed key above MUST NOT be written with another meaning.
- For an unknown value, a producer MUST write `null` or omit the field. It MUST NOT write a
  placeholder (`"NONE"`, `0` for an unknown temperature).
- An empty lane SHOULD keep its keys with `null` values, as Happy Hare does for empty gates, so
  a consumer sees the same key set either way. Today's writers break this rule in places: AFC
  `main`'s `clear_lane_data` writes `""` for `color`, `material`, `bed_temp`, `nozzle_temp`,
  `scan_time` and `td` (it sets `spool_id` to `null` and keeps `lane` and `extruder_index`),
  and Happy Hare writes `nozzle_temp` 200 when `gate_temperature` is missing.

**V-S3.** Keys vary widely in the wild:
- Happy Hare: `lane{gate}`, 0-based (`lane0`).
- AFC `main`: the user's lane name (often `lane1`, but `E2` and others are possible).
- AFC `DEV`: `T(n)`, one record per mapped tool.
- Mainsail: `T<n>`.
- HelixScreen: `laneN` (1-based) or `T<n>`.
- spoolman-lane-sync: bare `"0"`, `"1"`, plus a non-record `source` key.

Consumers MUST ignore the key, read `lane` (S3), and skip non-object values.
`docs/specs/filament_slots.md` §4 should be corrected to record Happy Hare's 0-based keys
[V `mmu_server.py#push_lane_data`].

## 7a. Bridge: `lane_status` / `afc` profile ↔ Happy Hare `mmu`

The consumer-side columns come from a grep of access patterns in KlipperScreen-Happy-Hare-Edition `1ad01a1`, Fluidd `738ff93` and Mainsail `03a4d7b1` (`hh_consumer_census.txt`). Reads through renamed variables may be missed.

Happy Hare publishes per-gate **arrays**. The `afc` profile publishes one object per lane. Gate
`g` ↔ the lane whose `lane` index is `g` (Happy Hare gates are 0-based and contiguous).

| Concept | `lane_status` / `afc` profile | Happy Hare `mmu` (and friends) | Notes |
|---|---|---|---|
| Lane list | `lane_status.lanes`, `AFC.lanes` | `range(num_gates)` | Happy Hare has no lane names. Use `str(g)`, or a producer-chosen name with `lane = g` |
| Units | `AFC.units` / `AFC_unit <u>.lanes` | `mmu_machine.num_units`; `mmu_machine.unit_<i>.{name, display_name, first_gate, num_gates}` | Unit `i` covers gates `first_gate … first_gate+num_gates-1` |
| Lane → extruder | `AFC_lane.extruder`, record `extruder_index` | Not published (unit config `extruder_name` only) | Gap on the Happy Hare side |
| Presence | `AFC_lane.load` / `prep`, record `loaded` | `gate_status[g]` ∈ {1,2} (0 empty, −1 unknown) | `prep && load` ≈ `gate_status ≥ 1`. −1 has no AFC equivalent: report `load=false` |
| Material | `AFC_lane.material` | `gate_material[g]` | |
| Vendor | record `vendor_name` | `gate_vendor[g]` | |
| Name | `AFC_lane.filament_name`, record `name` | `gate_filament_name[g]` | |
| Color | `AFC_lane.color` (`#RRGGBB`) | `gate_color[g]`: lowercase `rrggbb`, `rrggbbaa` or a W3C name, no `#` (`validate_color`); `gate_color_rgb[g]` (floats) | Convert names to hex; add or strip `#`; drop alpha |
| Nozzle temp | `AFC_lane.extruder_temp`, record `nozzle_temp` | `gate_temperature[g]` | |
| Spool | `AFC_lane.spool_id` | `gate_spool_id[g]` (−1 none) | −1 ↔ null |
| Tag UID | record `tag.uid` | `gate_spool_rfid[g]`: **one** normalized UID per gate (`normalize_gate_rfid`; commas rejected). The multi-UID list lives in the Spoolman extra `rfid_tag` | |
| TD | record `td` | `gate_td[g]` | |
| Active lane | `AFC.current_load`, `lane_status.current_lane` | `gate` (when `filament` is "Loaded") | |
| Active tool | (none; `AFC_lane.map` of the active lane) | `tool` | |
| Tool → lane map | `AFC_lane.map` (lane → `T<n>`) | `ttg_map[t]` = gate (tool → gate) | Inverse directions. `ttg_map` lets several tools share a gate. AFC `DEV` (`993fb45`) allows several tools on one lane (`map` is a list); AFC `main`'s `SET_MAP` swaps maps between two lanes, one `T` per lane (`AFC_spool.py#cmd_SET_MAP`) |
| Set map | `SET_MAP LANE=<l> MAP=T<n>` | `MMU_TTG_MAP MAP=a,b,c` (whole map; Fluidd, KlipperScreen-HH) or `MMU_REMAP_TTG TOOL=<n> GATE=<g>` (one tool; Mainsail). Happy Hare's own automap emits `MMU_TTG_MAP TOOL= GATE=` | A bridge MUST accept both the whole-map and the per-tool form |
| Load | `CHANGE_TOOL LANE=<l>` | `MMU_SELECT GATE=<g>` then `MMU_LOAD`, or `MMU_CHANGE_TOOL GATE=<g>` (errors if no tool maps to that gate) / `TOOL=<n>` | `MMU_LOAD` takes no gate; params `EXTRUDER_ONLY`, `SKIP_PURGE`, `RESTORE`; it loads the selected gate (`commands/mmu_load.py`, `mmu_change_tool.py`) |
| Unload / eject | `TOOL_UNLOAD` / `LANE_UNLOAD LANE=<l>` | `MMU_UNLOAD` (no gate: `EXTRUDER_ONLY`, `SKIP_TIP`, `RESTORE`; unloads the loaded gate) / `MMU_EJECT GATE=<g>` (`GATE` defaults to current; also `FORCE`, `EXTRUDER_ONLY`, `SKIP_TIP`, `RESTORE`) | `commands/mmu_unload.py`, `mmu_eject.py` |
| Edit metadata | `SET_COLOR` / `SET_MATERIAL` / `SET_SPOOL_ID LANE=` | Per gate: `MMU_GATE_MAP GATE=<g> <KEY>=v`, one attribute per call (Mainsail) or several in one call (Mainsail's Spoolman fill; KlipperScreen-HH `panels/mmu_filaments.py`). Bulk: `MMU_GATE_MAP MAP="{g: {status, spool_id, material, color, name, temp, speed_override}}"` covering every gate (Fluidd) | A bridge MUST accept both shapes. The bulk literal's keys differ from the per-gate parameter names |
| Runout backup | `AFC_lane.runout_lane` | `endless_spool_groups[g]` (group id per gate) | Pairwise vs group model; not losslessly convertible |
| Error | `AFC.error_state` / `message` | `print_state == "pause_locked"`, `reason_for_pause` | |
| What the print needs | (none in the AFC profile) | `slicer_tool_map.tools[n]` = {color, material, temp, name, in_use} | Comes from Moonraker file metadata (§2a.1 E) |
| Tools a file uses (UI side) | (none) | Fluidd: `referenced_tools` (`FileSystem.vue`). Mainsail: `referenced_tools` in its start-print section (`StartPrintDialogMmu.vue#summary`), but the indexes where `filament_weights > 0` in its TTG-map dialog (`MmuEditTtgMapDialog.vue#fileTools`) | Mainsail's TTG-map dialog disagrees with the rest. A producer automapping at print start SHOULD prefer `referenced_tools` and fall back to `filament_weights > 0` |
| Tool colors from file metadata | (none) | Fluidd prefers `filament_colors` for OrcaSlicer/BambuStudio files and `extruder_colors` otherwise (`mixins/mmu.ts#toolDetails`). Mainsail always prefers `extruder_colors` (`helpers.ts#getFileFilamentColors`) | The same file can show different tool colors in each UI |
| Per-tool color / spool for UI tool buttons | (none) | `gcode_macro T<n>.color`, `.spool_id` (`update_t_macros`) | Mainsail and Fluidd read these on any printer |
| Slicer preset name | record `slicer_preset` | `gate_filament_name` (when set by the slicer flow), `slicer_tool_map.tools[n].name` | |

## 8. Read flow (slicer)

1. Optional: GET `/printer/objects/query?lane_status`. If present, read `capabilities`, `lanes`
   and `current_lane`. If absent, the slicer treats the printer as a legacy `lane_data` source.
2. GET `/server/database/item?namespace=lane_data`.
   - Keep records with `lss_version` and `writer`. If any exist, discard records from other
     writers.
   - Otherwise take all records (today's behaviour).
3. For each record with `loaded` (or non-empty `material` on a legacy record), build the tray:
   - `filament_id` → `tray_info_idx`
   - `colors`/`color_mode` → `tray_colors`/`ctype`
   - temperatures, `remain_percent` or `remaining_weight_g`, `diameter_mm`
   - `pressure_advance` → `k`
   - `extruder_index` → the unit's extruder
4. Match:
   1. `filament_id`.
   2. On a miss, `vendor_name` + `name`.
   3. Then `material`.

   This is a simplification. #13719's order is `vendor + full lane name` → `vendor + material` → `lane name` → `filament_id_by_type`.

The step-3 fields below map to existing `ParseAgentFilament` keys (§1). `color_mode`, `bed_temp` and a single `nozzle_temp` have **no** key there; `ctype` is derived from the color count, so solid vs gradient can't be expressed; and `external[]` entries don't parse `k`, `diameter_mm` or `weight_g`:

| Record field | `ParseAgentFilament` key |
|---|---|
| `filament_id` | `preset_id`* |
| `loaded` | `loaded` |
| `material` | `material` |
| `color` | `color` |
| `colors` | `colors` |
| `nozzle_temp_min/max` | `nozzle_temp_min/max` |
| `remain_percent` | `remain_percent` |
| `pressure_advance` | `k` |
| `diameter_mm` | `diameter_mm` |
| `remaining_weight_g` | `weight_g` (open question: `weight_g` sets the same `tray->weight` as Bambu's `tray_weight`; whether Orca treats it as remaining or nominal weight is not established) |
| `extruder_index` | `units[].extruder` |

\* `preset_id` there sets `tray->setting_id`, a per-preset id. A filament-level id would need
`tray_info_idx`. Flag this to Orca rather than overloading the field.

## 9. Write flow (slicer to printer)

All writes are G-code through `printer.gcode.script`. A slicer MUST send only commands listed in
`lane_status.capabilities` (Part A §6.2). A printer without `lane_status` gets no writes.

| Intent | Command | Capability | Status |
|---|---|---|---|
| Map project tool to lane, before print | `SET_MAP LANE=<lane_name> MAP=T<n>`, one per tool, all before `SDCARD_PRINT_FILE` | `set_map` | Sourced (UIs, fork) |
| Assign a filament to a lane from the slicer's tray editor (Bambu `ams_filament_setting` equivalent) | `SET_LANE_FILAMENT LANE=<lane_name> [FILAMENT_ID=] [VENDOR=] [NAME=] [MATERIAL=] [COLOR=] [NOZZLE_TEMP=] [BED_TEMP=]` | `set_filament` | **Proposal.** Producers that lack it accept `SET_COLOR`/`SET_MATERIAL` (sourced) |
| Bind a Spoolman spool | `SET_SPOOL_ID LANE= SPOOL_ID=` | `set_spool_id` | Sourced |
| Mapping without a round trip | None. The producer reads Moonraker's standard file metadata (`referenced_tools`, `filament_colors`, `filament_type`, `filament_name`, `filament_temps`; Moonraker #962) at print start and assigns tools to lanes itself. It advertises the strategy in `lane_status.automap` ∈ {`none`, `filament_name`, `material`, `closest_color`, `color`}, which is **Happy Hare's AUTOMAP vocabulary** less `spool_id` (Happy Hare's `AUTOMAP_OPTIONS` lists it, but `automap_gate` rejects it as "not yet supported"; §2a.1 B). A producer MAY add `filament_id` as a strategy | `automap` | **Adapted from Happy Hare.** This replaces the header-comment idea in an earlier version of this draft |
| Set map in Happy Hare syntax | `MMU_TTG_MAP TOOL=<n> GATE=<g>` | `set_map` on a Happy Hare producer | Sourced. A slicer picks the syntax by profile: `afc` → `SET_MAP`, Happy Hare (`mmu` object present) → `MMU_TTG_MAP` |

**W1.** A producer MUST answer each write by applying it to the record *and* the native system, or
by raising the Part A's §6.3 error. The slicer SHOULD re-read `lane_data` after a write instead
of assuming success.

**W2.** Pressure advance is never written by the slicer. It belongs to the spool (Spoolman) or the
printer's calibration.

## 10. Versioning

- `lss_version` follows the Part A's major.minor rule.
- This profile is `"slicer"`. A producer lists `"slicer"` in `lane_status.compat_profiles` when
  it maintains the records.
- New fields arrive as minor versions. Consumers MUST ignore unknown fields.

## 11. What Orca would need to change

Smallest first. Items 1 and 2 also work for every existing `lane_data` writer.

1. **`fetch_moonraker_filament_data`** reads `loaded`, `colors`/`color_mode` (into
   `tray_colors`/`colors_gradient`, which `AmsTrayData` already has), `nozzle_temp_min`,
   `remain_percent`/`remaining_weight_g`, `diameter_mm`, `pressure_advance` (into `k`),
   `extruder_index`.
2. **Vendor and name matching** from `vendor_name`/`name` (Happy Hare's keys, already in Happy Hare's records) when `filament_id`
   misses: #13719's flow, keyed on the neutral field names.
3. **Writer filtering** by `lss_version`/`writer`, so a stale AFC or Happy Hare record doesn't
   shadow the live producer's.
4. **Write path in `MoonrakerPrinterAgent`:**
   - handle `ams_filament_setting` as `SET_LANE_FILAMENT` (or `SET_COLOR` + `SET_MATERIAL`);
   - send the print's `ams_mapping` as `SET_MAP` lines before `SDCARD_PRINT_FILE`;
   - gate both on `lane_status.capabilities`.

   Today the agent ignores both.
5. **Or move all of this into a Python printer-agent plugin** that emits the `ParseAgentFilament`
   document. That needs a caller for the parser on `main`, which doesn't exist yet.

## Appendix A: sourcing

| Part | Sourced (revision) or proposal |
|---|---|
| §1 table: transports, URLs, fields, matching | **Sourced**: Orca `main` `adea4e48a6`: `MoonrakerPrinterAgent.cpp` (`fetch_filament_info`, `fetch_moonraker_filament_data`, `fetch_hh_filament_info`, `build_ams_payload`), `SnapmakerPrinterAgent.cpp` (incl. the `VENDOR` gate on `filament_detect` temps), `QidiPrinterAgent.cpp#fetch_slot_info`, `CrealityPrintAgent.cpp`, `CrealityPrint.cpp#query_boxes_info`, `DevFilaSystem.cpp`, `docs/HLSD/filament_id.md` |
| `ParseAgentFilament` schema; no caller | **Sourced**: `DevFilaSystem.cpp`, `git grep` on `main`; introduced in `ecddf3d18f` |
| Upstream PR descriptions | **Sourced** from PR bodies (gh), not from their diffs, except #16508's Moonraker hunk (skimmed) |
| J0eB0l fork behaviour | **Sourced**: PR #1 body, `doc/AFC.md`, grep of `AFCProtocol.cpp`/`AFCManager.cpp` at `6d89dcf` |
| §2 lane_data history | **Sourced**: Moonraker PR #1022 body and comments; AFC `AFC_utils.py#send_lane_data`, `AFC_lane.py#send_lane_data`, `#clear_lane_data` at `5ea84ea` and `DEV` `993fb45`; Orca `docs/HLSD/filament_id.md` § "Moonraker `lane_data`" (`20388f846d`) |
| §3 mint algorithm | **Sourced**: filament_id.md "Minting" |
| §3 one id per product; inheritance | **Sourced** at Orca `adea4e48a6`: `docs/HLSD/filament_id.md` (opening, "The design", "Who consumes the id"); `src/libslic3r/PresetBundle.cpp#PresetBundle::resolve_vendor_preset`, `#warn_ambiguous_filament_id_match`, `#check_duplicate_filament_subtypes`, the matcher's `get_preset_base(f) == &f` filter; `src/libslic3r/Preset.cpp#PresetCollection::resolve_user_preset`, `#PresetCollection::get_preset_base`; example profiles `OrcaFilamentLibrary/filament/Generic ABS @System.json`, `Sovol/filament/Generic ABS @Sovol SV08.json`, `Generic ABS @Sovol SV08 MAX.json`. The MUST NOT is **proposal** |
| §3 `flow_k` | **Resolved** by the fact-check: J0eB0l's AFC fork only (`BridgeBox` `26c8b8d`, `extras/AFC_autocal.py`); absent from AFC `main` `5ea84ea` and `DEV` `993fb45` |
| §4 write paths | **Sourced**: `DeviceManager.cpp` (`ams_filament_setting`, `extrusion_cali_sel`), `DevMappingNozzle.cpp`, `CrealityPrint.cpp#start_print_commands`, `MoonrakerPrinterAgent.cpp#handle_request`, `PrinterAgentPluginCapability.cpp`. The G-code header alternative is a **proposal** |
| §5 transport analysis | Facts **sourced** as tagged; conclusions **inference** |
| §6-§10 | **Proposal**. The Group 1 fields in §7, and their Orca and writer columns, are sourced |
| §11 | **Proposal** |
| ACE, Prusa MMU, OpenPrintTag, OpenSpool in Orca | **Sourced absence** (grep of `main`, issue search). Prusa's own slicer and Connect were not examined |
| §2a Happy Hare facts | **Sourced**: Happy-Hare `main` `70aa90fb`: `components/mmu_server.py` (header, remote methods, `push_lane_data`, pre-processor regexes), `extras/mmu/commands/mmu_slicer_tool_map.py`, `mmu_gate_map.py`, `mmu_ttg_map.py`, `mmu_gate_maps.py#get_status`, `mmu_controller.py` status, `mmu_constants.py` (AUTOMAP, Spoolman modes), `mmu_gate_maps.py#restore_automap_option`, `config/macros/mmu_misc.cfg` (`MMU_REMAP_TTG`); Moonraker PR #962 diff + `file_manager/metadata.py` and commit `5b92e52e`; Mainsail `StartPrintDialogMmu.vue`, `MmuEditTtgMapDialog.vue`; Orca PR #12307 (title only) |
| §2a.1 G consumers | **Sourced**: KlipperScreen-Happy-Hare-Edition `1ad01a1` (incl. `panels/mmu_filaments.py`, `panels/gcodes.py`), Fluidd `738ff93`, Mainsail `03a4d7b1` (incl. `MmuEditTtgMapDialog.vue#fileTools`, `StartPrintDialogMmu.vue#summary`), found by grepping access patterns (reads through renamed variables may be missed) |
| §2a.2 verdicts, §7a bridge semantics | **Proposal**. The field and command names on both sides are sourced. Command shapes are as consumers send them, not checked against each Happy Hare command's full parameter set |
| Appendix B | **Proposal** |

## Appendix B: Happy Hare, revisited

The Part A's Appendix B said publishing a fake `mmu` object to get Happy Hare's panel was
"not recommended". That answered a different question from this one.

- **Faking Happy Hare's surface: the verdict stands, and for more reasons now.** An `mmu`
  object makes Mainsail and Fluidd draw a selector panel with servo, encoder, filament position
  and sync state, which a status-only producer would have to fabricate. It also makes every
  consumer treat the printer as Happy Hare and send `MMU_*` commands:
  - Orca's fallback,
  - HelixScreen's discovery, which claims Happy Hare on `mmu`,
  - KlipperScreen,
  - Mainsail's tool-map dialogs (`MmuEditTtgMapDialog.vue`, `MmuEditTtgMapDialogDetails.vue`, opened from `StartPrintDialogMmu.vue`), which send `MMU_REMAP_TTG`.

  The `afc` profile asks far less of a producer.
- **The three Happy Hare consumers don't agree on one command shape**, which makes faking the
  surface harder still. From the census (grep of access patterns; reads through renamed variables
  may be missed):
  - **Gate edits:** Fluidd sends every gate in one `MMU_GATE_MAP MAP=` literal. Mainsail and
    KlipperScreen-HH send `MMU_GATE_MAP GATE=n KEY=v` (Mainsail usually one key per call,
    KlipperScreen-HH several).
  - **Tool remaps:** Mainsail sends `MMU_REMAP_TTG TOOL= GATE=` per tool. Fluidd and
    KlipperScreen-HH send the whole `MMU_TTG_MAP MAP=`.
  - **Tool colors:** Fluidd prefers `filament_colors` for Orca/Bambu files. Mainsail always
    prefers `extruder_colors`.
  - **Tools a file uses:** Fluidd and Mainsail's start-print section read `referenced_tools`.
    Mainsail's TTG-map dialog uses the indexes where `filament_weights > 0`.

  A fake `mmu` producer would have to accept every one of these forms.
- **Happy Hare as precedent and data source: yes, and the spec now leans on it.**
  - Its `lane_data` field names are adopted (`vendor_name`, `name`, `td`, null-filled empty
    lanes).
  - Its Moonraker file metadata (#962) is the slicer-to-printer channel.
  - Its AUTOMAP strategies are the `automap` vocabulary.
  - `MMU_TTG_MAP` is an accepted `set_map` syntax.
  - The §7a bridge lets a consumer read a Happy Hare printer through the spec's concepts with no
    Happy Hare change.
  - What it does *not* adopt is the numeric `filament_id` and the Spoolman `rfid_tag` extra field,
    where Happy Hare collides with Orca and with SpoolLink respectively.
- **Net effect on this draft.** The slicer profile is now mostly "Happy Hare's `lane_data` record,
  plus Orca's string id, plus explicit presence/extruder/version/writer fields". Happy Hare could
  conform by adding `lss_version`, `writer`, `loaded`, `lane_name` and `extruder_index` to
  `push_lane_data`, and moving its numeric id to `spoolman_filament_id`. That is the change to
  propose to moggieuk alongside the Moonraker `integrations.md` write-up.

---

# Part C: Spool trackers and tag formats


Every row was read in source at the revision named, except: the SimplyPrint row, which comes
from its unversioned public web pages (fetched 2026-10-27), and two cells marked "Not checked"
or "Not traced", which are listed under Open questions with everything else not established.

## Trackers

| Tracker | What it tracks | API | Tags | Source |
|---|---|---|---|---|
| **Spoolman** | **Vendor**: `name`, `empty_spool_weight`, `external_id`.<br>**Filament**: `name`, `material`, `density`, `diameter`, `weight` (net, g), `spool_weight`, `settings_extruder_temp`/`_bed_temp`, `color_hex`, `multi_color_hexes` + `multi_color_direction` (`coaxial`\|`longitudinal`), `article_number`, `external_id`.<br>**Spool**: `initial_weight`, `spool_weight`, `used_weight`, `location`, `lot_nr`, `archived`, first/last used.<br>Key/value **extra fields** on each entity | REST `/api/v1` (spool, filament, vendor, field, tag, …) | Since **v0.27.0** (2026-09-25): a `tag` table. `uid` is unique, and a tag targets a spool or a filament; the table also defines a location target (`target_value`), which nothing writes yet. `format` is free text; known values are `openprinttag`, `opentag3d`, `ntag`, `bambu`, `tigertag`, `qidi`, `creality`, `prusa`. Routes: `POST /spool/{id}/tag`, `DELETE /spool/{id}/tag/{uid}`, `GET /spool?tag=<uid>`, `POST /tag/scan` | Donkie/Spoolman `0fb375c` `spoolman/database/models.py#Vendor,Filament,Spool,Tag`, `spoolman/tags.py#KNOWN_FORMATS`, `spoolman/api/v1/spool.py#link_tag` and find `tag` param, `spoolman/api/v1/tag.py`, `spoolman/tags.py` (`TARGET_*`: "TARGET_SPOOL and TARGET_FILAMENT are written today"). Tag table at tag `v0.27.0` (`bed3f14`); first commit `de533bc` 2026-08-13 |
| **FilaMan** (Fire-Devils/filaman-system) | Manufacturer, color, filament (material, diameter, density, multi-color), spool (remaining/initial/empty weight g, location, lot, `rfid_uid`, `rfid_uid_2`), spool weight events | REST `/api/v1`: tag scan, device write-tag, Spoolman import | UID match only; schema comment: "FilaMan never decodes tag contents, it only matches the uid" | `4b8dac48` `backend/app/models/spool.py#Spool`, `backend/app/api/v1/schemas_tag.py#TagScanRequest` |
| **SimplyPrint** filament manager | Spool: id, uid, type, brand, color name/hex, diameter, total and remaining length (mm), qr id, nfc id, printer/extruder, location | Yes: `GetFilament`, `GetSpecific`, `locations/GetList`, `resolve/FindBySearch` (`nfc_id`, `nfc_content`) | The feature page lists Creality CFS, OpenPrintTag, OpenSpool and OpenTag as read/write. On Bambu it contradicts itself (Open question 3) | apidocs.simplyprint.io and simplyprint.io/features/filament-management, fetched 2026-10-27. Unversioned pages |
| **SpoolEase** (yanshay/SpoolEase) | README: "tracks spool inventory, remaining weight, storage locations, and printer slots" | Not checked (Open question 4) | NTAG and MIFARE Classic | `3532f8d9` README.md |
| **Bambu filament ids** (Bambu Studio profiles) | Filament *products* only (`filament_id` such as `GFB00`). No spool inventory found | No public API found | A Bambu tag's block 1 carries the material id | bambulab/BambuStudio `79df7ae3` (profile JSON via gh api). OrcaSlicer `adea4e48` maps Bambu ids to its own `OF*` ids in `resources/printers/bambu_filament_ids.json`, loaded by `BBLPrinterAgent.cpp#bambu_filament_id_map` |
| **Creality material DB** | A material catalog (id such as `01001`, brand, name, type, colors, density, temps). No spool inventory | A Creality cloud endpoint used by Creality Print's `scripts/ParamPackUtil.py`; not a documented public API | A tag carries the 5-char material code (`01001` in DnG's example). DnG-Crafts reads it as a 6-char `filamentId` = 1-char prefix + code; OpenRFID and Bambu-Research-Group read a 4-char supplier id followed by the 5-char code, so DnG's prefix is the supplier id's last character (Open question 2). On the printer: `/mnt/UDISK/creality/userdata/box/material_database.json` | CrealityPrint `59ae8cbb`. DnG-Crafts/K2-RFID `7101bac4` README, `db/k2.json`. suchmememanyskill/OpenRFID `a13bc9e1` `src/tag/creality/processor.py`; Bambu-Research-Group/RFID-Tag-Guide `a26cf83d` `CrealityRfid.md` |
| **Snapmaker U1 filament_detect** | Per channel: vendor, main/sub type codes, up to 5 RGB colors, alpha, temps, `CARD_UID`, `CARD_TYPE`, SKU, weight. No inventory | Moonraker `GET /printer/objects/query?filament_detect`; extended firmware adds `POST /printer/filament_detect/set`, which its RFID patch README calls "intended as an internal integration API" | Stock: Snapmaker's own MIFARE tags. Extended firmware adds NTAG/OpenSpool, and more through OpenRFID | paxx SnapmakerU1-Extended-Firmware `417e6394` `docs/design/filament_detect.md`, `docs/design/rfid.md`, `overlays/firmware-extended/13-patch-rfid/README.md` (the quote) |
| **TigerTag registry** | CC0 id tables (`id_brand`, `id_material`, `id_aspect`, `id_type`, `id_diameter`, `id_measure_unit`, `id_version`, `id_catalog`) | `api.tigertag.io` (README: "free, no key"); the tables also ship offline | Native format | TigerTag-Project/TigerTag-RFID-Guide `a07f1e28` (protocol v2.2, 2026-09-30) |
| **OpenPrintTag database** (Prusa-originated) | Brands, materials, packages (GTIN), containers, as YAML. No spool inventory | No documented API. The YAML is in git, and database.openprinttag.org serves static JSON its web UI fetches (`/api/brands/basic.json`, `/api/search-index.json`, fetched 2026-10-27) | Native format | OpenPrintTag/openprinttag-database `ad505ca2` README.md |
| **OpenSpool** (spuder/OpenSpool) | No inventory: an ESP32 reader that pushes a tag's filament to a Bambu printer | None found | OpenSpool read and write | `c060a06c` README.md |

## Who links lanes to a tracker today

| System | Link stored as | Copies into the lane | Tag ↔ spool | Source |
|---|---|---|---|---|
| **Moonraker** `[spoolman]` | One active spool for the machine, DB `moonraker` / `spoolman.spool_id`. Charges filament from the rise in `toolhead.position[3]` | — | — | Moonraker `b559b9a` `components/spoolman.py` (`DB_NAMESPACE`, `ACTIVE_SPOOL_KEY`, `_handle_status_update`). Endpoints `/server/spoolman/{spool_id,proxy,status}`; remote method `spoolman_set_active_spool`. `SET_ACTIVE_SPOOL` is an **example macro** in `docs/configuration.md`, not built in |
| **AFC** | `AFC_lane.spool_id` (`SET_SPOOL_ID`). Calls Moonraker's `spoolman_set_active_spool` remote method (`#set_active_spool`) | `material`, extruder temp (unless `ignore_spoolman_material_temps`), bed temp (always), `density`, `diameter`, `spool_weight`, `remaining_weight`, `initial_weight`, filament `name`, vendor name, `color_hex` or the first of `multi_color_hexes`. Then writes `lane_data` | — | AFC-Klipper-Add-On `5ea84ea` `extras/AFC_spool.py#set_spoolID`, `#set_active_spool`; `extras/AFC_utils.py#get_spool` (through `server/spoolman/proxy`) |
| **Happy Hare** | `gate_spool_id[g]`, persisted in save_variables. Modes `off`/`readonly`/`push`/`pull` | Gate `material`, `color`, `name`, `vendor`, `temp` from Spoolman (`MMU_GATE_MAP … FROM_SPOOLMAN=1`) | Spoolman extra fields `printer_name`, `mmu_gate_map`, `rfid_tag` (display name "RFID"; comma-separated UIDs; `MMU_RFID_FIELD = 'rfid_tag'` l.87, `_add_extra_field(... field_key=MMU_RFID_FIELD)` l.267. Only the file-header comment says `rfid`); scan resolve and auto-create | Happy-Hare `70aa90fb` `components/mmu_server.py#_get_filament_attr`, `#_send_gate_map_update`, `#get_spool_by_uid`; `extras/mmu/mmu_constants.py#SPOOLMAN_*` |
| **Mainsail** (tool changers) | `SET_GCODE_VARIABLE MACRO=T<n> VARIABLE=spool_id`; `SAVE_VARIABLE t<n>__spool_id` | `lane_data` key `T<n>`: `lane`, `color` (Spoolman `color_hex`, **no `#`**), `material`, `bed_temp`, `nozzle_temp` | — | Mainsail `03a4d7b1` `src/components/dialogs/SpoolmanChangeSpoolDialog.vue#setSpool`, `#setMacroVariable`; from PR #2510 (`0fe8db42`, 2026-06-16) |
| **spoolman-lane-sync** (broncosis) | Reads save_variables `t<n>__spool_id` (Mainsail's convention), else Spoolman `location` = `T<n>` | `material`, `color` (uppercase, no `#`), `vendor`, `name`, `remaining_weight`; `filament_id` and `setting_id` from Spoolman filament extras **`orca_filament_id`**, **`orca_setting_id`** | — | broncosis/spoolman-lane-sync `1660093` `spoolman_lane_sync.py#_spool_to_lane`. It **claims the namespace** with a `lane_data` key `source` = `"spoolman"` and exits if another source holds it (`#_claim_source`) |
| **SpoolLink** (paxx U1) | Per-channel spool ids in Klipper's `print_task_config.filament_spool_id` (`_ptc_spool_ids`). `_sync_active_spool` pushes the current extruder's channel id into Moonraker's single active spool | Not traced for this survey (Open question 5) | Spoolman spool extra **`card_uids`** (`_parse_card_uids`). Remote methods `spoollink_resolve_spool`, `spoollink_unlink_card` | SnapmakerU1-Extended-Firmware `417e6394` `overlays/firmware-extended/38-feature-spoollink/.../components/spoollink.py`. Open PRs #767 (2026-10-06) and #799 (2026-10-19) move to Spoolman native tags on ≥0.27.0; #799 migrates `card_uids` once. Neither is merged |
| **zmod_afclite / KILN_AFCLite** | `save_variables` `zmod_afclite_spool_<lane>` / `kiln_afclite_spool_<lane>` | Spool color/material written into the firmware slot | `card_uids`, the SpoolLink convention | DIEHARDave/zmod_afclite `375c86d` `zmod_afclite.py` (`SPOOL_VARIABLE_PREFIX = "zmod_afclite_spool_"`, `CARD_UIDS_FIELD = "card_uids"`, `SAVE_VARIABLE`) and KILN_AFCLite `67aae0d` `extras/kiln_afclite.py` (`SPOOL_VARIABLE_PREFIX = "kiln_afclite_spool_"`, `CARD_UIDS_FIELD`); both READMEs. KILN writes the slot through `IFS_SET_MATERIAL` (README) |
| **HelixScreen** | `lane_data` `spool_id` (its own records) and Spoolman via `server.spoolman.proxy` | Spoolman filament extra **`pressure_advance`** read as pressure advance | `TagReading::spoolman_id`, set only for an id a tag declares | helixscreen origin/main `11cf6f0f29` `src/api/moonraker_spoolman_api.cpp`, `include/spoolman_types.h#kSpoolmanPaExtraKey`, `include/ams_tag_reading.h#TagReading`, `include/tag_spool_link.h#TagSpoolSighting`; `docs/specs/filament_slots.md` |

Notes:
- **Pressure advance:** HelixScreen uses the Spoolman **filament** extra `pressure_advance`.
  The J0eB0l OrcaSlicer fork (a fork of Snapmaker/OrcaSlicer) uses the Spoolman **spool** extra
  `flow_k`, which only J0eB0l's own AFC fork writes (`BridgeBox` `26c8b8d`: `extras/AFC_autocal.py`
  through `AFC_RFID.py#write_flow_k`, field `SPOOL_EXTRA_FLOW_K`); it is absent from AFC
  `5ea84ea` and `DEV` `993fb45`.
- **Tag UIDs** have three stores in use: Happy Hare's `rfid_tag`, SpoolLink's `card_uids` (also
  written by J0eB0l's AFC fork), and Spoolman's native tag table (≥0.27.0). J0eB0l's AFC fork
  also adds a Spoolman spool extra `tray_uid` for a Bambu reel's identity (`AFC_RFID.py`),
  which is a tag-identity key but not a chip UID.

## Tag formats

Each format is cited from its **git repository at a commit**. The published spec sites (openspool.io, specs.openprinttag.org, opentag3d.info) were fetched on 2026-10-27 and agree with the repositories. Bambu and Creality have no first-party spec, and their rows come from reverse-engineering write-ups and decoders. Snapmaker has no spec document but publishes the reader's source (Snapmaker/u1-klipper, GPL-3.0).

| Format | Chip / encoding | Carries | Signed / encrypted | Specified in |
|---|---|---|---|---|
| **OpenPrintTag** | ISO 15693 NFC-V (designed for ICODE SLIX2). NDEF MIME `application/vnd.openprinttag`, CBOR meta + main + aux | Main (integer keys): uuids, GTIN, material class/type/name, brand, weights and lengths, primary + secondary colors, density, diameter, temps, drying. Aux: consumed weight, storage location | Not signed: "does not define on-tag cryptographic signing". Optional write protection | OpenPrintTag/openprinttag-specification `060e4208` `docs_src/nfc_data_format.md`, `nfc_technical_details.md`, `data/*_fields.yaml` (repo has no version tags) |
| **OpenSpool** | NTAG215/216. One NDEF record, `application/json` | `protocol: "openspool"`, `version`, `type`, `color_hex`, `brand`, `min_temp`, `max_temp`. The U1 extension adds bed temps, subtype, alpha, extra colors, weight, diameter | Neither | spuder/OpenSpool `c060a06c` `docs/rfid.md`. U1 extension: paxx `417e6394` `docs/design/openspool.md` |
| **TigerTag** | NTAG21x, 144-byte big-endian binary at pages 0x04-0x27 | Product, material, aspect, type, diameter, brand (registry ids), up to 3 colors (color 1 RGBA, colors 2 and 3 RGB), measure + unit, measure available, nozzle/bed/dry temps, TD, timestamp / twin id, tag index and count, a 28-byte message | TigerTag+ only: ECDSA-P256; key in the registry. Not encrypted | TigerTag-RFID-Guide `a07f1e28` README §2 (protocol v2.2, 2026-09-30) |
| **Bambu** | MIFARE Classic 1K; per-sector keys derived from the UID with HKDF-SHA256 | Material id (`GF*`) + variant, type, RGBA (+ extra colors), weight, diameter, temps, drying, tray UID, length, date | RSA-2048 signature in sectors 10-15; keyed sectors | Reverse-engineered: Bambu-Research-Group/RFID-Tag-Guide `a26cf83d` `BambuLabRfid.md`. No first-party spec found |
| **Creality CFS** | MIFARE Classic 1K; the sector-1 key is the first 6 bytes of AES-128-ECB(fixed key, UID×4), written as both key A and key B (DnG; Happy Hare calls it key B); blocks 4-6 ASCII, AES-128-ECB-encrypted (DnG) | Date, vendor id, batch, filament id, color (`0RRGGBB`), length code, serial | Encrypted with a fixed key. OpenRFID also detects plaintext payloads and decodes them as they are; which tags carry plaintext was not established. No signature found | Reverse-engineered: DnG-Crafts/K2-RFID `7101bac4` README "Tag Format", `Utils.java#createKey`, `MainActivity.java`; OpenRFID `a13bc9e1` `src/tag/creality/processor.py`; Happy Hare `tag_parser.py#_creality_derive_key_b`. The write-ups disagree on the layout (Open question 2) |
| **Snapmaker U1** | MIFARE Classic 1K; HKDF-derived sector keys | Vendor, main/sub type, up to 5 RGB colors, SKU, diameter, weight, length, temps, drying, date | RSA PKCS#1 v1.5 SHA-256 signature | No spec document. First-party source: Snapmaker/u1-klipper `93cb12faeb` `klippy/extras/filament_protocol.py` (`verify_signature_pkcs1`: `PKCS1v15` + `SHA256`; `M1_PROTO_*_POS` layout; `FILAMENT_PROTO_COLOR_NUMS_MAX = 5`), `klippy/extras/fm175xx_reader.py` (`FM175XX_M1_CARD_HKDF_SALT_KEY_A/B`). Matching decoders: suchmememanyskill/OpenRFID `a13bc9e1` `src/tag/snapmaker/{constants,processor}.py`; paxx `docs/design/filament_detect.md` |
| **OpenTag3D** | NTAG215/216; NDEF MIME `application/opentag3d` (`_data/spec.json` `mime_type`; OpenRFID takes it from `schemas/v2.json` through `src/tag/opentag3d/schema.py`). Happy Hare's parser instead matches **`application/vnd.opentag3d`** (`70aa90fb` `extras/mmu/unit/nfc/tag_parser.py#_OPENTAG3D_MIME`, l.1551). Binary | Material, manufacturer, color name + up to 4 colors, SKU, diameter, temps, density, weights, measured length/weight, TD, data URL | Spec: data "must remain unencrypted" (`spec.md`) | GooborgStudios/OpenTag3D `6c0cfe5e` `_data/spec.json` (v2.004), `spec.md` |

Readers that decode several of these (secondary evidence of the layouts):
- Happy Hare `70aa90fb` `extras/mmu/unit/nfc/tag_parser.py`: OpenPrintTag, OpenSpool,
  TigerTag, Bambu, Creality, OpenTag3D and SimplyPrint URLs.
- suchmememanyskill/OpenRFID `a13bc9e1` `src/tag/*`.

## Settled questions

- **Snapmaker U1.** There is no spec document, but Snapmaker publishes the reader's source
  (Snapmaker/u1-klipper `93cb12faeb`, GPL-3.0; see the table). It confirms the chip, keys,
  fields and signature that the decoders describe.
- **OpenPrintTag.** specs.openprinttag.org serves the repo's `docs_src/*.md`; its
  `nfc_data_format.md`, `nfc_technical_details.md` and `changelog.md` match `060e4208` apart
  from rendered template blocks. The repo has no tags or releases, so a commit is the only
  revision. The database site's JSON is undocumented (trackers table).
- **OpenSpool.** `version` has never gone past `"1.0"`: every commit of `docs/rfid.md` that
  shows a version shows `"1.0"` (the first, `301ff4bd`, has no version key), the firmware rejects anything else (`firmware/bambu.h`), and openspool.io
  documents only 1.0.
- **zmod_afclite and KILN_AFCLite.** Their spool-link storage is confirmed in code (the
  link table above).
- **SimplyPrint lookup.** `resolve/FindBySearch`'s `nfc_id` "matches a tag's serial number, with
  or without colons", so the lookup key is the chip UID (apidocs, fetched 2026-10-27).

## Open questions

1. The OpenTag3D MIME mismatch is a cited fact (see the table), and Happy Hare's
   `_try_opentag3d` returns early when the MIME type differs. Whether another Happy Hare path
   picks up a spec-conformant tag was not tested.
2. Creality CFS tag layout: the write-ups disagree, and this survey doesn't choose between them.
   - Bambu-Research-Group/RFID-Tag-Guide `a26cf83d` `CrealityRfid.md`: batch(3), date YYMDD(5),
     supplier(4), material(5), color(6), with no mention of encryption.
   - DnG-Crafts/K2-RFID `7101bac4` README "Tag Format" (followed by Happy Hare's
     `_try_creality_tag`): date MDDYY(5), vendor(4), batch(2), filamentId(6), color(7), with AES
     encryption.
   - OpenRFID `a13bc9e1` `src/tag/creality/processor.py` uses Bambu-Research-Group's order and
     decrypts AES-128-ECB, falling back to plaintext. So the order does not split along
     encryption.
   - Both orders put the same 5 characters at `[12:17]` (the material code). DnG and OpenRFID
     differ only on the first 12 characters; Bambu-Research-Group's table also differs after
     them (color, length 6, at offset 17).
   - Which real tags carry a plaintext payload, as OpenRFID's fallback allows, was not
     established.
3. SimplyPrint: the spool's own `nfcId` field is documented only as "NFC ID (null if empty)".
   On Bambu its feature page contradicts itself: the FAQ says it "reads and writes Bambu Lab"
   tags, the body that it "reads Bambu Lab AMS tags".
4. Inventory APIs: SpoolEase's was not checked. For Bambu and TigerTag Studio no per-spool
   inventory endpoint was found; TigerTag's README points to a Firebase integration
   (`TigerTag-Project/TigerTag_Firebase_Integration`) that was not read.
5. SpoolLink: what it copies into the lane was not traced.
