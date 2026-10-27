# Lane Status Surface

A draft open spec for publishing multi-material filament lane state from Klipper printers, so
that web UIs, touchscreens and slicers can all see what is loaded in each lane.

Today each filament system exposes its lanes in its own shape: AFC and Happy Hare each have their
own Klipper objects, and the Snapmaker U1 and the AD5X IFS have others again. Firmware mods work around this by
impersonating AFC so Mainsail and Fluidd will draw a panel, and slicers get almost nothing. This
spec writes down one surface that existing UIs already understand, plus the filament data a
slicer needs to pick the right profile.

- **[SPEC.md](SPEC.md)** is the short contract. Start here.
- **[REFERENCE.md](REFERENCE.md)** holds the detail: the full `afc` and `lane_data` field tables, how every existing shim
  maps each command, a bridge to Happy Hare's `mmu` object, how OrcaSlicer reads printers today,
  and sources for every claim.

## Status

Draft 0.1, for discussion. "Lane Status Surface" is a working name. Nothing here is endorsed
by the AFC, Happy Hare, Moonraker, Mainsail, Fluidd or OrcaSlicer projects yet, and the point of
publishing it is to get their input before anything is built.

## Contributing

Open an issue or a pull request. Corrections to anything in REFERENCE.md marked as read from
source are especially welcome, since those claims are pinned to specific revisions and will drift.

## License

The text of this specification is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
