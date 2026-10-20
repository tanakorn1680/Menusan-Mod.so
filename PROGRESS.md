# ProMenu — progress notes / handoff

Read this first if you're picking this up cold. It's written so another AI or dev can
understand the situation immediately without re-reading the whole history.

## What this project is
GTA SA Android (arm64, AML plugin) touch cheat menu, "ProMenu", ~0.9.3. See `README.md`
for the full feature list and version history (in Thai). Code lives under `jni/`:
- `jni/main.cpp` — hooks (touch, draw, shutdown, game tick), symbol resolution entry point
- `jni/ui.h` — everything else: the UI, the action queue, all feature logic (~1870 lines)
- `jni/safemem.h` — fault-proof reads of game memory (pipe-write trick; **read only, no write**)
- `jni/diag.h` — breadcrumb logging to `last_action.txt` / `previous_session.txt` on-device
- `jni/vehicles.inc`, `jni/cheats100*.inc`, `jni/cheat_groups*.inc` — data tables
- `jni/cheat_table.inc` — **stale, from an old version, README says it can be deleted**

The house style, consistently followed everywhere in `ui.h`: every pointer/offset the menu
pokes is either (a) a real exported symbol resolved by name via `aml->GetSym` (see `Res()` /
`ResolveGame()` in `ui.h`), confirmed to exist in the original ARM CheatMenu's own import
table, or (b) a struct offset that was read out of that original menu's disassembly. Nothing
is guessed. Comments say exactly where each offset/symbol came from. **Keep doing this.**

## What was asked this round
Thai request, paraphrased: add the Vehicle-tab functions that don't exist yet. The request
listed, with difficulty tags the user gave:
1. Fix vehicle, Flip vehicle (ง่าย-กลาง / easy-medium)
2. No burn on flipped, Stay on bike, Perfect handling (กลาง / medium)
3. 8 "current vehicle" option toggles: Always Skidmarks, Engine State, No Particles,
   Comedy Control, HS Targetted, Invisible Car, Lights State, Lock Door (กลาง / medium)
4. Vehicle Density Multiplier (กลาง / medium)
5. Seats: swap seats in nearby vehicles (กลาง / medium)
6. Spawn "As driver" / "In Air" + pick by numeric model ID (กลาง / medium)
7. Enter Near Vehicle

A screenshot of the *original* ARM CheatMenu (not this project) was pasted alongside, showing
these as existing buttons there (Vehicle / Visua / Weapon / Menu tabs, "copcarvg", "Driver",
"Passenger 1/2/3", "Seats: 4", "Fix vehicle", "Flip vehicle", "Cheat Menu v1.6"). That's
reference for what the *end result* should look like, not something in this repo.

## What was actually done this round
**Only item 1's second half: "Flip vehicle".** "Fix current vehicle" already existed.

- `jni/ui.h`: new `DoFlipVehicle()` next to the existing `DoFixVehicle()`. New "Flip vehicle"
  button, same row as "Fix current vehicle" (now a half-width pair instead of one full-width
  button) in `PageVehicle()`. Thai strings added.
- How it works: re-levels the vehicle's RenderWare matrix (reads via `SafeMem::Read`, same as
  everywhere else; **writes directly**, no safe-write primitive exists in this codebase — see
  "Known risk" below). Keeps the compass heading (the direction the car points), zeroes roll
  and pitch, nudges Z up by 0.5 so it doesn't clip into the ground on landing. Standard
  "flip fix" approach (re-orthogonalize Right/Up/At around world-up), nothing fancier.

### ⚠️ Known risk — read before shipping
`DoFixVehicle()` and every other struct-poking function in this file uses offsets the author
says were **confirmed against the original menu's own binary** (import table, or disassembly).
`DoFlipVehicle()` is different: it assumes the *standard* RenderWare `CMatrix` layout
(Right/Up/At/Pos, each 3 floats + 1 pad, 16 bytes per axis, 0x40 total) sitting at the matrix
pointer that `EntityPos()` already reads (header `+0x18` → matrix `+0x30` = position — **that
part is confirmed**, it's reused from working code). The axis layout at `+0x00/+0x10/+0x20`
(Right/Up/At) is the well-known SA/RenderWare convention used throughout the public re3/reversed
codebases, but it has **not** been checked against this specific game build's actual memory the
way the rest of this file's offsets have. It reads the matrix back and sanity-checks it
(finite floats, non-degenerate heading vector) before writing, and the write itself can't
crash any *worse* than the existing `aHealth`/`aArmour`/`aMoney` direct writes in `RunQueued`
already can — but "re-levels correctly" is unverified in-game. **Test this one specifically**
before trusting it, and if it's wrong, the fix is almost certainly just the matrix axis order,
not the overall approach.

## What was NOT done, and why
Everything else in the request needs information nobody has provided yet: real offsets or
symbol names for things like per-vehicle flag bytes (skidmarks always-on, engine-state,
particle suppression, "comedy control", "HS targeted", invisibility, light state, door lock),
a handling-table pointer/multiplier, the traffic density global, and the ped pool / vehicle
pool walking needed for "nearest vehicle" and "swap to seat N". None of these exist anywhere
in this codebase today (confirmed: `grep -rn` for flip/skid/burn/handling/density/seat across
`jni/` before this change found nothing). Inventing offsets for these would silently corrupt
game memory or crash on a guess — exactly what this project's whole `SafeMem`/`diag.h` design
is built to avoid, and the opposite of how the rest of this file is written. So: not attempted.

Breakdown against the request list:
- **Fix vehicle** — already existed, untouched.
- **Flip vehicle** — done this round (see above, test it).
- **No burn on flipped** — not started. Likely needs a per-vehicle "engine on fire" flag/offset
  (possibly related to the "Engine State" toggle below — may be the same underlying field).
- **Stay on bike** — not started. Likely a ped/bike "fall off" flag or a hook into whatever
  function throws the rider off; needs the symbol for that function or flag identified first.
- **Perfect handling** — not started. GTA SA handling comes from `CHandlingDataMgr` /
  `tHandlingData` per-vehicle-class entries loaded from `handling.cfg`; "perfect" would mean
  either overwriting the current vehicle's handling struct fields (traction/grip/suspension) or
  swapping its handling ID. Needs the handling table's base symbol and the struct layout (this
  is a well-documented struct in the wider SA modding community, but — per this project's own
  rule — needs confirming against this specific binary, not assumed).
- **8 option toggles** (Always Skidmarks, Engine State, No Particles, Comedy Control,
  HS Targetted, Invisible Car, Lights State, Lock Door) — not started. These read as per-vehicle
  flag bits (likely packed in `CVehicle`'s flag DWORDs/bitfields, similar in spirit to the
  existing `kOffHealth`/`kOffArmour` pattern but for a different struct). Needs each bit's byte
  offset and bit position confirmed, same bar as every other offset in this file. "Current
  vehicle" implies these only make sense when `g_findPlayerVehicle` returns non-null, same
  guard as Fix/Flip.
- **Vehicle Density Multiplier** — not started. Likely `CCarCtrl`/traffic density global(s)
  (there are usually several: parked, ambient, etc.) — needs the symbol name(s).
- **Seats: swap nearby vehicle seats** — not started. Needs (a) a way to enumerate nearby
  vehicles (pool walk — `CPools` is already used once, for `GetPedRef`, so the pool pattern
  exists and could extend to vehicles if the vehicle pool symbol is found), and (b) the
  "move ped into seat N of vehicle" call (script command, like the weapon-drop SAUtils calls,
  or a direct `CPed`/`CTaskSimpleCarSetPedOut`-style entry — the screenshot's "Driver" /
  "Passenger 1/2/3" buttons suggest a small finite seat list rather than true N-seat generality).
- **Spawn as driver / in air + pick by model ID** — partially has infrastructure: `g_vehicleCheat`
  (`CCheat::VehicleCheat(model)`) already exists and is how the current spawn-by-tapping-icon
  flow works (`aVehicle` in the action queue). "As driver" vs "in air" and a free-text/numeric
  model ID field are new UI + possibly a different underlying call if `VehicleCheat` always
  spawns the same way — needs checking what `VehicleCheat` actually does positionally before
  assuming it can be reused as-is.
- **Enter Near Vehicle** — not started. Same pool-walk dependency as Seats above (find nearest
  vehicle within some radius), plus a "put player ped into driver seat of vehicle X" call.

## Round 2: on-device offset scanner ("Debug Scan")
The user doesn't have access to a computer-based disassembler/trace setup, only a rooted
*virtual/cloud* Android device (so GameGuardian works fine there, but juggling two apps —
GameGuardian plus the game — side by side felt like too much friction for 8+ separate scan
sessions). Request: build the "snapshot, do the thing, snapshot again, diff" workflow **into
the menu itself**, read-only, so it can be done without a second app.

Added `Scan` namespace in `jni/ui.h` (right after `DoFlipVehicle`) + a collapsible
"Debug Scan" section at the bottom of the Menu tab (`PageMenu`, id base 5000,
`g_expanded[50]`):
- **Snapshot A** — reads `0x800` bytes from the start of the current vehicle struct
  (`g_findPlayerVehicle(-1, true)`) via `SafeMem::Read`, into memory only.
- **Snapshot B (diff vs A)** — reads the same region again, diffs byte-by-byte against
  Snapshot A, and appends a dated block (`+0xOFFSET  old -> new`) to
  `Android/data/com.rockstargames.gtasa/files/ProMenu/scan_result.txt`. Several scans in one
  sitting append to the same file instead of overwriting, so toggles can be done back-to-back.
- **Read-only, by design.** It never writes to game memory, so unlike every feature-adding
  change in this file, a scan session itself cannot crash the game or corrupt state no matter
  what offset it stumbles on — the worst case is a useless diff, not a segfault. This is also
  why it was fine to build without first having confirmed offsets: there's nothing to get
  wrong in a way that matters.
- Scan region is `0x800` bytes from the vehicle struct base, which is a guess at "probably big
  enough to cover whatever small flag bytes live near the already-confirmed `0x734`/`0x738`
  fields" — not a confirmed struct size. If a toggle's offset turns out to be further out,
  `kRegionSize` in `Scan::` is the one constant to raise.

### How the user should use this (already explained to them in chat, repeating here for whoever
### picks this up in case they come back mid-process)
Per toggle (skidmarks, engine state, no particles, comedy control, HS targeted, invisible car,
lights state, lock door — and in principle door-lock-like toggles for the other missing
features too): get in a vehicle → Menu → Debug Scan → Snapshot A (feature off) → toggle the
feature on by whatever in-game means they have available → Snapshot B. Read back the appended
block in `scan_result.txt`. If more than one byte changed, they were told to re-run the same
A/B pair with only that one feature flipped a second time and keep whichever offset shows up
in both diffs (poor-man's "changed AND changed again" filter, same idea as GameGuardian's
repeated-search narrowing).

**Still waiting on:** the actual `scan_result.txt` contents back from the user. Nothing in the
8-toggle group (or No burn on flipped / Perfect handling / Density multiplier, which need
their own separate before/after actions — flipping a car, editing handling, spawning lots of
traffic) can be wired up to a real offset until that comes back. Seats / Enter Near Vehicle /
spawn-as-driver-or-in-air still aren't reachable by this tool at all (pool walking and calling
a function, not a static flag byte — see the "not done" breakdown above), so they still need
a different approach entirely, not just a bigger scan region.

## Suggested next step for whoever continues this
The blocking dependency for most of the remaining list is the same: a disassembly/offset-dump
session against the actual game binary (the way `DoFixVehicle`, the Teleport vtable lookup, and
every `Res(lookup, ...)` symbol in `ResolveGame()` were originally derived). Concretely, still
needed, roughly in order of how many remaining items they unblock:
1. Vehicle pool symbol + iteration (unblocks Seats, Enter Near Vehicle)
2. `CVehicle` flag byte(s)/bit layout for the 8 toggles (unblocks that whole group, and maybe
   "no burn on flipped" if it's the same flags block)
3. Handling table base + per-class struct layout (unblocks Perfect handling)
4. Traffic/car density global(s) (unblocks Density Multiplier)
5. Whatever function throws a rider off a bike (unblocks Stay on bike)
6. What `VehicleCheat(model)` actually does re: placement, to know if "as driver"/"in air" is a
   flag to that call or a different call entirely (unblocks the spawn-mode item)

Until then, please don't guess offsets for these just to check the box — a wrong offset here
doesn't just misbehave, it segfaults the whole game (these are raw writes, not `SafeMem::Read`,
which only guards reads). That's consistent with how carefully the rest of this file is written.
