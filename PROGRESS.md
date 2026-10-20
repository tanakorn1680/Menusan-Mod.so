# ProMenu — Vehicle tab expansion — progress notes (0.10)

Read this first. Written for the next AI session picking this up, so it does not have to
rediscover any of this from scratch.

## Build status: fixed after one failed CI run

The first version of this patch (0.10 as originally sent) **failed to compile** — GitHub
Actions run #24, `ndk-build`, 5 errors, all ordering bugs: `g_spawnAsDriver` / `g_spawnInAir`
used inside the `aVehicle` case of the action-queue switch before they were declared further
down the file, and `Section()` called from the new `PageVehicle()` before `Section()`'s own
definition (which originally sat just above `PageGame()`, later in the file). `ui.h` is a
single translation unit built top-to-bottom with no header/forward-declarations for its own
free functions, so every helper has to be defined before its first caller — this file's own
existing code already relies on that (e.g. `Header()` is defined before `PageVehicle` uses it);
the new code just didn't follow it in two places.

**This has been fixed** in this version: `g_spawnAsDriver` / `g_spawnInAir` / `g_spawnModelId` /
`g_densityCache` moved up next to the other new `g_pVehAsDriver` / `g_pVehInAir` globals (right
after the comment block explaining why `SpawnVehicle(std::string)` isn't called directly), and
`Section()` moved up to right after `RowWidth()`, before `PageVehicle()`. Nothing about *what*
the code does changed, only *where* in the file it's defined. If a future session needs to
add more free functions to this file: **define them before their first caller**, not just
"near the related code" — grep for the function/variable name's first match and confirm it's
the definition, not a call, same check this session ran to catch the above.

No other issues were found in the rest of the build log (`0_build.txt`,
`build/1_Set up job.txt` etc. — the other steps, checkout/header-fetch, all passed; only the
ndk-build step failed). This fix has not been run through CI again yet — next push's Actions
run should confirm it actually builds clean.

## What this session was asked to do

Port a specific list of Vehicle-tab features from the original ARM CheatMenu (binary plugin,
screenshot showed "Cheat Menu v1.6") into ProMenu's own Vehicle tab:

- Fix vehicle, Flip vehicle (easy–medium)
- No burn on flipped, Stay on bike, Perfect handling (medium)
- 8 current-vehicle toggles: Always Skidmarks, Engine State, No Particles, Comedy Control,
  HS Targetted, Invisible Car, Lights State, Lock Door (medium)
- Vehicle Density Multiplier (medium)
- Seats: swap seats of a nearby vehicle (medium)
- Spawn as driver / in air + pick model by numeric ID (medium)
- Enter Near Vehicle

Three files were supplied as reference, not as part of the repo:
`Menusan-Mod_so-main.zip` (this repo itself), `GTA_SA_AML_PLUGIN_2_10.zip` (the original AML
plugin pack, including `mods/libCheatMenu64.so` — the actual v1.6 binary the feature list and
screenshot came from), and `gta-reversed-android-main.zip` (a GitHub repo that looked like a
reversed GTA:SA source tree).

## Key finding: how the reference material actually panned out

**`gta-reversed-android-main.zip` is not usable.** It is a skeleton/scaffold repo — every
actual reversed game-class file (`Vehicle.h`, `Vehicle.cpp`, `Ped.cpp`, etc.) is 0 bytes. Only
~13 infrastructure files (hook plumbing, logging, `reGTA.cpp`) have real content. Confirm with:
`find . -name "*.cpp" -o -name "*.h" | xargs -I{} sh -c 'test -s "{}" && echo {}'` — only a
handful come back. Do not spend time re-exploring it; there is nothing there.

**`GTA_SA_AML_PLUGIN_2_10.zip` → `mods/libCheatMenu64.so` is the useful one**, but not in the
obvious way. The .so is stripped (no debug symbols, no local symbol names visible via normal
disassembly), BUT its **`.dynsym` dynamic symbol table is intact** — 6400+ entries, almost all
of them real mangled C++ names for `libGTASA.so` classes (`CVehicle`, `CCheat`, `CCarCtrl`,
`VehiclePage`, etc.), the exact same kind of name this project's own `ResolveGame()` already
looks up. This is because the plugin itself calls into the game via `dlsym`-style symbol
lookups (same technique `main.cpp`/`ui.h` use here), so those names had to survive stripping.

How to re-extract them if needed again:
```
readelf --wide --dyn-syms mods/libCheatMenu64.so | awk '$7 != "UND" {print $NF}'
```
(`$7 != "UND"` filters out imported libc functions, keeping only the plugin's own
references into the game — these are almost all what you want.) Cross-reference against
`strings mods/libCheatMenu64.so | grep -i "<ui label text>"` to confirm a given UI string is in
this exact binary before trusting a nearby symbol is the one driving it.

## What got implemented (confirmed, safe — ship this)

All in `jni/ui.h` (and one log string in `jni/main.cpp`). Search for `"Vehicle tab additions
(0.10)"` to find every change quickly.

1. **Fix vehicle** — resolves `_ZN11VehiclePage17FixCurrentVehicleEP8CVehicle`
   (`VehiclePage::FixCurrentVehicle(CVehicle*)`), called on the player's current vehicle via the
   existing `g_findPlayerVehicle` symbol already in this project. Verified: this is literally
   the function the original menu's own "Fix vehicle" button calls.
2. **Flip vehicle** — resolves `_ZN4Util7FlipCarEP8CVehicle` (`Util::FlipCar(CVehicle*)`), same
   calling pattern.
3. **Vehicle Density Multiplier** — resolves `_ZN8CCarCtrl20CarDensityMultiplierE`
   (`CCarCtrl::CarDensityMultiplier`, a `float` global, same as how `g_pTimeScale` /
   `CTimer::ms_fTimeScale` is already handled elsewhere in this file). Read 4×/sec like
   `RefreshVis()` does for the Visual tab, written directly like `aFlag`/`aTopZoom` already do.
   UI: 0x/1x/2x/4x preset chips, matching the existing chip-row convention.
4. **Spawn as driver / in air, with numeric model-ID entry** — resolves
   `_ZN11VehiclePage12ms_bAsDriverE` and `_ZN11VehiclePage9ms_bInAirE` (two bools). These are
   set immediately before calling the *already-resolved* `g_vehicleCheat`
   (`CCheat::VehicleCheat(model)`) — the same spawn entry point the existing 88-car grid already
   uses successfully. A new on-screen digit keypad (not `ImGui::InputText`, see below) lets the
   player type a model ID (400–611, the real SA vehicle ID range) and spawn it directly, with
   the two toggles applied.

   **Deliberately NOT used:** `VehiclePage::SpawnVehicle(std::string)` exists as a named symbol
   too, but it takes a libc++ `std::string` *by value*. Calling that correctly requires knowing
   the exact SSO (small-string-optimization) layout the game's STL was built with — get it
   wrong and you corrupt the stack instead of getting a clean failure. Reusing the
   already-proven `g_vehicleCheat(int)` call sidesteps that risk entirely. If a future session
   wants `SpawnVehicle` itself (e.g. because it turns out to do something `VehicleCheat` alone
   doesn't), that ABI needs to be nailed down first — don't just wire it up and hope.

5. New `PageVehicle()` layout: Fix/Flip row → Vehicle Density Multiplier section → Spawn Vehicle
   section (As driver / In air toggles + model-ID keypad) → Browse Vehicles section (the
   original 88-car grid, unchanged, just renamed/re-sectioned so it's not the first thing in the
   tab anymore).
6. All new UI strings have Thai translations added to the existing table.
7. `g_expanded[]` index map, since this file reuses one shared array across tabs with no
   per-tab namespacing: **20–24** = Browse Vehicles sections (one per `vehicles.inc` section,
   currently 5), **40–42** = Visual tab, **44–51** = Weapon tab (dynamic, one per weapon
   section — currently 8, so don't go past 51 without recounting
   `jni/weapons.inc`-equivalent data first), **52–54** = new Vehicle sections added this
   session. **If you add more sections anywhere, grep `g_expanded\[` across the whole file
   first** — there is no central registry, it's just convention, and it already bit this
   session once (first draft picked 42–44 and collided with both Visual and Weapon before this
   was caught).
8. Button IDs: existing spawn grid is `2000 + i` (i up to ~88, i.e. 2000–2087ish). New vehicle
   buttons use `2900+` specifically to stay clear of that. If you add more Vehicle-tab buttons,
   keep using the 29xx block (or check `grep -oP '2[0-9]{3}'` first — this session also hit one
   self-inflicted collision here, between a digit-pad key and the Spawn button, before fixing
   it).

## What did NOT get implemented — and why, honestly

These remain **todo**. Do not guess at them — wrong offsets here don't misbehave, they crash
the game (see this project's own README history: the 0.7/0.8 fixes were exactly this class of
bug, from `GetWaypoint` trusting an offset that wasn't quite right for every build).

- **No burn on flipped** — the *toggle* for this exists as a real symbol,
  `_ZN11VehiclePage17ms_bVehFlipNoBurnE` (a bool), and there's also
  `_ZN11VehiclePage20ms_pCfgVehFlipNoBurnE` (presumably the saved-config pointer for it). These
  were found but not wired up this session — should be a quick follow-up, same pattern as
  `g_pVehAsDriver`/`g_pVehInAir` above (resolve the symbol, add a `ToggleRow` near Flip vehicle,
  read it before/write it alongside the flip call). **This one is low-risk, do it first.**
- **Stay on bike** — likewise has a real symbol, `_ZN11VehiclePage16ms_bDontFallBikeE` (bool).
  Same situation as "No burn on flipped": found, not wired up, should be easy to add the same
  way. **Also low-risk, do it alongside the above.**
- **Perfect handling** — no symbol found under any obvious name (`PerfectHandling`,
  `HandlingPerfect`, etc. all came back empty from the dynsym table). The UI label string
  "Perfect handling" *is* present in `libCheatMenu64.so`'s strings, so the feature is real, but
  whatever flag or function drives it isn't a separate named symbol — likely inlined logic
  inside `VehiclePage::Draw()` (which is itself a real, large, but entirely unnamed/stripped
  function). Needs disassembly to find.
- **The 8 per-vehicle toggles** (Always Skidmarks, Engine State, No Particles, Comedy Control,
  HS Targetted, Invisible Car, Lights State, Lock Door) — all 8 label strings were confirmed
  verbatim in the binary (`strings mods/libCheatMenu64.so | grep -i "skidmark\|comedy\|..."`),
  so this is definitely the right reference, but none of them resolved to named symbols. These
  are almost certainly raw per-instance bool fields on `CVehicle` (bitflags written directly at
  some byte offset within the object, the same category of thing as this project's own
  `kOffHealth = 0x6AC` / the `0x734`/`0x738` vehicle-class-id offsets already used in
  `DoTeleport`), not functions or globals — so they will never show up in a symbol table no
  matter how much more searching is done there. The only way to find their offsets is to either
  (a) disassemble `VehiclePage::Draw()` in the stripped .so (needs an AArch64-capable
  disassembler/decompiler — this session only had x86-64 binutils available, so `objdump`
  couldn't produce real disassembly here; something like Ghidra/IDA with an AArch64 processor
  module would be the right tool), or (b) find a public, verified offset table for this exact
  game build (2.10, arm64) from the SA modding community and then **test every single one of
  them in-game before shipping**, exactly as this project's own README describes doing for
  every other offset it uses. Don't skip the in-game verification step even with a "known good"
  public offset — this project's history is full of offsets that looked right and weren't.
  `CVehicle::GetVehicleLightsStatusEv` exists as a *function* and might be a cleaner read-only
  path for showing Lights State's current value even before the write-side offset is known, if
  that's useful.
- **Seats (swap seats of a nearby vehicle)** — no symbol found.
  `CCarCtrl::SetUpDriverAndPassengersForVehicle(CVehicle*, int, int, bool, bool, int)` exists
  and is plausibly related (it's how the game assigns peds to seats when a vehicle spawns), but
  that is not the same thing as moving the *player* between seats of a vehicle they're already
  in, and calling it with guessed arguments is exactly the kind of risk this file avoids
  elsewhere. Needs more research — possibly also inlined into `VehiclePage::Draw()`, same as the
  8 toggles above.
- **Enter Near Vehicle** — label string confirmed in the binary, no symbol found. The
  `_Z32IsVehiclePointerValid_NotInWorldP8CVehicle` symbol exists and might be part of how the
  original menu scans for a nearby vehicle, but that's a guess, not a confirmed read. Needs
  disassembly or further research.

## Why a digit keypad instead of `ImGui::InputText` for the model-ID entry

This project's touch handling is entirely custom — see the comment block at the top of
`main.cpp`: *"ImGui is used for DRAWING ONLY. Tap, drag and scroll are handled here from the
raw touch events... so nothing depends on ImGui's mouse click state machine."* `io.MouseDrawCursor
= false` and `ConfigFlags |= ImGuiConfigFlags_NoMouse` are set in `InitImGui()`. `ImGui::InputText`
depends on ImGui's own keyboard/IO handling, which this project does not drive — there is no
existing code path that would deliver Android soft-keyboard input into ImGui's IO struct. Rather
than add a whole new, never-tested input bridge for one feature, the model-ID field is a
12-button digit pad (`Btn()` calls, same widget everything else uses), which stays consistent
with how every other free-form value in this menu already works (compare the money +/- buttons
in `PagePlayer()`).

## Build / install

Unchanged from before: push to the repo → GitHub Actions builds → `libProMenu64.so` goes in
`Android/data/com.rockstargames.gtasa/mods/`. Nothing in this session's changes touches the
build config (`jni/Android.mk` / `jni/Application.mk` untouched, no new source files, no new
`.inc` files, no new dependencies).

## Suggested next steps, in order of risk (lowest first)

1. Wire up **No burn on flipped** and **Stay on bike** — real symbols already found and named
   above, same pattern as `g_pVehAsDriver`/`g_pVehInAir`, should be ~15 minutes of work.
2. Get an AArch64-capable disassembler on `mods/libCheatMenu64.so` and look at
   `VehiclePage::Draw()` specifically (its symbol is real:
   `_ZN11VehiclePage4DrawEv`, even though its *contents* aren't — the function's start address
   is known from `.dynsym`, which is enough to point a disassembler at it) to find the offsets
   for the 8 toggles, Seats, Enter Near Vehicle, and Perfect handling.
3. Only after (2) is verified in-game: wire those up following the exact same
   SafeMem-read / raw-pointer-write pattern already used throughout this file (see `g_visPtr` /
   `RefreshVis()` / the `aFlag` queue case for the closest precedent — a per-item array of raw
   pointers, refreshed on a timer, written directly from the queue).
