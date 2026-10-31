# Saved games that survive a moved memory window

Work in progress, kept in a stash. This describes the problem, what was
measured and what the fix would be.

## The problem

A user reported on the Switch: a checkpoint loaded within the run it was
saved in, but after quitting to HOS and starting the game again, Continue
restarted the level.

A saved game (`savegame.bin`, a core) is the game state's memory, written as
it is. It holds pointers into itself and into the map's tag data, so
`game_state_header_valid` (`source/saved games/game_state.c`) refuses one
whose `layout_address` is not this run's game state address. That refusal
was silent, so the level just started over.

The game state lies inside the Xbox memory window, at
`PORT_WINDOW_ADDRESS(0x81A00000)` (window base + 0x1A00000). The window does
not always land in the same place:

- The Switch has two address space layouts. In the 39-bit one (the LDN test
  runs) the window maps at 0x80000000. In the 36-bit one (the 20:02-20:12 runs
  on 2026-10-07) svcMapMemory maps nothing at or above 0x80000000
  (`0xdc01`). The fallback, 0x20000000, lies under the first memory pool
  (0x1ca0c000 in that run), so the window went to the first free range the
  search found: 0x60000000, 256 MB.
- Android puts the window wherever the Java runtime leaves room. The Lenovo
  uses 0x80000000.

A fix that pins the window to one address per layout was written and then
rolled back: it cannot help a save that moves between the two layouts, or
between devices. Translating the save instead lets one load at any window.

`find_gap` in `port/switch/host/host_memory.c` has a bug found along the way:
it rounds its starting point down to the alignment, so the search retries
0x80000000 128 times (about 1.6 s of start-up in the 36-bit layout). Not
fixed here.

## What is in this stash

- `source/saved games/game_state.c`: the "different address" refusal is now
  logged, with both addresses and both image shifts.
  `game_state_port_base_address()` gives this run's game state address
  (declared in `game_state.h`).
- `debug.window_base` (Android host, `config.toml`'s `[debug]`): the window
  goes at this address if it will fit there (`host_memory_window_wanted`,
  tried first in `place_window`, `port/android/host/host_memory.c`). The guest
  warns that it does not know the setting; only the host reads it.
- `debug.core_dump_tick` (`port/linux/src/port_config.c`, read with
  `config_integer` in `source/main/main.c`'s `main_port_core_dump`): a core of
  the first level played, written at that game tick to
  `d:\core\game_state_<address>.bin`.
- `tools/game_state_compare.py`: compares two images (cores or saved games)
  taken with the window at different bases, and says which words are
  pointers, with `--fields` to count them by the field they are in.

## The experiment

On the Lenovo (`com.halo.decomp.vk`), two runs: `window_base = 0xC0000000`,
then `0xA0000000`, with `core_dump_tick = 1800`. Each run: Campaign, New Game,
"Halo" (a30), Normal (keys: ENTER, DPAD_DOWN, ENTER, ENTER, ENTER), the core
at tick 1800, then Save and Quit (ESCAPE, DPAD_DOWN x3, ENTER). The profile's
`savegame.bin` was pulled after each run.

    tools/game_state_compare.py A/core.bin 0xc0000000 B/core.bin 0xa0000000 --fields

A word is a pointer if it lies in each run's window at the same offset from
that run's base. Results:

| | core (tick 1800) | savegame.bin |
|---|---|---|
| pointers | 2721 | 2509 |
| into the game state | 2709 | 2501 |
| into the tag data | 12 | 8 |
| equal values inside one window ("false positives") | 12291 | 10497 |
| into the game's image (equal in both) | 12527 | 8233 |

The false positives are mostly floats (0xBF800000 is -1.0): the 0xA0000000
and 0xC0000000 windows overlap common negative floats. **Scanning for
anything that looks like an address cannot work.** At 0x80000000 there would
be fewer such words, but never none.

## Where the pointers are

Every pointer found is in a known field. None are in the object data inside
the objects pool.

| Allocation | Field | Points into |
|---|---|---|
| every data array | header `+0x34` (`data`) | its elements |
| `objects` memory pool | header `+0x24` base, `+0x30` first, `+0x34` last block | itself |
| `objects` pool blocks | header `+0x8` reference, `+0xC` next, `+0x10` previous | the `object` array, the pool |
| `object` data array | element `+0x8` | its block in the objects pool |
| `decal vertex cache` lruv cache | header `+0x3C` (`blocks`) | itself |
| `hs thread` data array | `+0x10` (`stack`), then the frames in `stack_data` | its own stack: each `hs_stack_frame`'s `previous` and `result` |
| `recorded animations` data array | `+0x10` (`event_stream`) | the map's tag data |
| `structure detail objects` (no signature, +0x990058) | `cells[i][j].z_reference_vector` (`+0x14`, stride 0x18) | the map's tag data |
| same | `layers[n].cells` (+0x5100), `view_data.layers` (+0x5200) | itself |
| `particle` data array | `+0x2C` (`location.bonus`) | junk: padding with leftover address bits, only in the live core |
| game state header | `+0x12C` (`layout_address`) | the port's own field |

`structure detail objects` was named from `d:\gamestate.txt`, which the game
writes at start-up (`game_state_allocation_record`): every allocation's name,
type and size, in order. Adding the sizes gives each allocation's offset.

Tag data is loaded at the same offset into the window for the same map, so
the window's shift fixes those pointers as well. The image does not move on
the Switch; Android's moves by `PORT_IMAGE_SHIFT`, already in the header.

## The fix this leads to

Saved games written with addresses relative to a standard base, and fixed up
on load:

- On save, each field above is written as an offset from the window's base
  (or in the window's linked form, 0x80000000-based: `PORT_WINDOW_REBASE`).
- On load, this run's base is added back, then `game_state_image_accept`'s
  checks run as now. `game_state_header_valid` no longer needs the same
  address.
- Each field is found the way `game_state_image_accept` already walks the data
  arrays, pools and caches; the script stacks by following each frame's
  `previous`.

Saves would then load at any window, in either Switch layout, and between
devices of the same build and map.

## Still to do before trusting the list

- **Coverage.** This is one moment in one level. Fields that are empty here
  (vehicles, cutscenes, other AI states, split-screen's second player) could
  hold pointers too. Capture more pairs (a vehicle, a cutscene, split-screen,
  another level), and search the game state structs' sources for pointer
  members.
- `tools/game_state_compare.py` could read `gamestate.txt` to name the
  allocations that have no signature.
- The Lenovo's `config.toml` still has `window_base = 0xA0000000` and
  `core_dump_tick = 1800`; its profile's save was written at window
  0xA0000000.
