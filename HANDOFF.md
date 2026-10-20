# ProMenu – LIVE STATUS + hand-off (read this first)

> **RULE FOR EVERY AI/SESSION THAT TOUCHES THIS REPO:** after every change, (1) bump the version in `ui.h` (About line) and `main.cpp` (loaded log line),
> (2) update the "LIVE STATUS" and "CHANGE LOG" sections below, (3) deliver an **overlay zip with ONLY the changed files**, laid out exactly as in the repo
> (e.g. `jni/ui.h`, `jni/main.cpp`, `HANDOFF.md`) so the user extracts it into the repo root and overwrites. Always include this file in the zip.

## LIVE STATUS (updated 2026-10-17, v0.9.5)
- **Goal:** a professional touch-first cheat/tool menu (ProMenu, AML plugin) for GTA SA Android 2.10 64-bit with the same functions as the ARM CheatMenu v1.6 (9 pages: Player, Ped, Vehicle, Weapon, Teleport, Scene, Visual, Game, Menu + the 100 cheats), in our own cleaner layout, Thai/English.
- **Working style the user wants:** Thai replies, short, answer first; do NOT test or do extra work "just in case" (the user tests on his phone himself); one staged step at a time; be honest about what is untested; never over-build.
- **How files are delivered:** overlay zip, changed files only, same paths as the repo, user uploads/extracts into the repo root, GitHub Actions builds `libProMenu64.so` (artifact), user copies it to `Android/data/com.rockstargames.gtasa/mods/` and removes the old `libCheatMenu64.so`.
- **How build errors are read:** the workflow prints the first ndk-build error lines as `::error::` annotations; the run page is readable without login (web fetch `https://github.com/<user>/<repo>/actions/runs/<id>`); full logs need login. Ask the user for the run link if a build fails.
- **Current version v0.9.5 = v0.9.4 + "Fix vehicle" button** (Vehicle tab, top). Written without a compile check on request – if the build fails, read the annotation and fix the small patch in `ui.h` (`DoFixVehicle`, resolver lines `g_fix*`, the button in `PageVehicle`).
- **Waiting for the user's test results of:**
  1. v0.9.4 drop-weapon fix (SAUtils outputs now 8 bytes; Weapon tab shows a readout line pair; "test keep weapon" switch available). Expected: pickup appears ~3 units in front, weapon leaves the hand.
  2. v0.9.5 Fix vehicle (works for car/monster truck/trailer → `CAutomobile::Fix`, quad, heli, plane, bike; refuses boat/train/bmx with a message).
- **Open problems:** (a) game crashed once after "Remove all weapons" with no weapons (`reference/crash_enter_car.txt`, cause unknown; mitigated in v0.9.1: no game call when no weapons) – need `previous_session.txt` if it repeats; (b) a stray pickup exists at (2038, 0, 11) in the user's save from the buggy drop (harmless).
- **Next planned steps (in this order unless the user says otherwise):** Flip vehicle + vehicle options/seat switching · Ped page (big head, thin body, density) · Player extras (infinite energy, super sprint, drunk, CJ skin) · Scene/cutscenes · remaining Visual/Game items · Overlay location name.
- **Tools lost in a sandbox reset (recreate only if needed):** the Python AArch64 decoder and the PC unit-test harness (see section 9). Without them, every new feature that needs the ARM binary's internals is slower; the verified facts below cover everything done so far.

## CHANGE LOG
- **v0.9.5** – Vehicle tab: "Fix vehicle" button (verified symbols `CAutomobile/CBike/CQuadBike/CHeli/CPlane::Fix`, vehicle type at +0x734). Hand-off file moved into the repo root and turned into a live status file.
- **v0.9.4** – Drop weapon root cause found from the user's screenshot (SAUtils writes script outputs as 64-bit values; 4-byte output variables clobbered neighbours: spot y became 0, model became 0, the pickup was created 1600 units away). Fixed with 8-byte `ScriptOut` variables.
- **v0.9.3** – Weapon tab readout (handle, model, positions) and "test: keep weapon" switch to diagnose the drop.
- **v0.9.2** – Drop weapon re-done exactly like the ARM menu: 4 SAUtils script commands (04C4 spot in front, 0781 weapon model, 032B pickup with ammo, 0555 remove weapon); weapon removed only if the pickup handle is valid.
- **v0.9.1** – "Remove all weapons" reads the weapon slots first and never calls the game when empty; "Drop current weapon" and "Remove current weapon" added; breadcrumbs for cheats and vehicle spawns.
- **v0.9** – Weapon tab (39 weapons, ammo choice) and Visual tab (9 game flags), on top of the user's v0.8 (his SafeMem/diag/teleport hardening).
- **v0.8 (by the user)** – teleport crash fixed: `safemem.h`, `diag.h` breadcrumbs, vtable teleport lookup, polled waypoint.
- **v0.7** – Teleport tab; unsafe `FindPlayerCoors` replaced by reading entity memory. **v0.6** – Game page + Overlay. **v0.5** – Thai. **v0.4** – 100 cheats, Player, Vehicle. **v0.3** – correct touch event types. **v0.1–0.2** – floating button, panel, first cheats.

## Extra facts learned for script commands (SAUtils)
- ARM's binary contains ~39 `SCRIPT_COMMAND` blobs (u16 opcode + 16-byte param string at even offsets). Scan recipe: regex for `[opcode][ifsvbpu]{1,15}\0…` padded to 18 bytes. Certain ones: `04C4 "ifffvvv"` offset from char, `0781 "iv"` weapon model, `032B "iiifffv"` pickup with ammo, `0555 "ii"` remove weapon, `0605`/`0A1A "issfbbbbi"` task play anim, `0687 "i"` clear char tasks. Others were not mapped to features.
- Remember: outputs (`v`) = 8-byte variables; floats go in as doubles; get the ped handle with `CPools::GetPedRef`.

---
*สรุปภาษาไทย:* โปรเจกต์ปลั๊กอินเมนูสัมผัส (ปุ่มวงกลมลอย) สำหรับ GTA SA Android 2.10 64 บิต ผ่าน AML โฟลเดอร์ `ProMenu/` คือซอร์สล่าสุด (v0.9.4) ไฟล์นี้รวมสิ่งที่ยืนยันแล้ว สิ่งที่ยังไม่ทำ และบทเรียนทั้งหมด ผู้ใช้พูดไทย อยากได้คำตอบไทยสั้น ๆ บนมือถือ

## 0. What this package is
- `ProMenu/` – complete, latest source of the plugin (ImGui overlay mod, arm64) with the GitHub Actions build.
- `reference/` – crash log analysis, the drop-weapon bug readout, original cheat order (JSON).
- **Not included (lost in a sandbox reset):** the PC-side unit tests and the Python AArch64 disassembler used for reverse engineering.
  Section 9 explains how both were done so they can be recreated. Treat all code as **not yet verified in-game for v0.9.4**.

## 1. The user and how to work with them
- Thai speaker. Reply in Thai, short, answer-first, readable on a phone (a few sentences / one screenful). No preamble.
- Develops on a phone: uploads zips to GitHub (web), builds with Actions, tests on a Xiaomi (HyperOS, Android 16, build string
  `Xiaomi/goya_global/goya:16/...`) running GTA SA Android 2.10 64-bit with other AML mods (SAUtils, JPatch, GlowingPickups, RubbishPort).
- Wants **overlay zips** that extract into the repo root: only changed files, same paths (`jni/ui.h`, `jni/main.cpp`, `README.md`).
- Dislikes: rushing/over-building ("don't burn the limit"), repeated guessing, unnecessary questions, unrequested extras.
  Likes: staged work, honest "untested" labels, being shown a layout/plan before big changes. He tests quickly and reports short results
  (or screenshots/crash logs). When he says "continue" he means continue the current plan.
- He edits code himself. His repo copy is the source of truth: **diff against what he uploads before changing anything**
  (he added `safemem.h` and `diag.h` and hardened teleport in v0.8).
- Assistant-side limits seen: no NDK, no network from bash, cannot run the game. GitHub Actions run pages are readable via web fetch
  (the workflow turns ndk-build errors into `::error::` annotations); full logs need a login. Sandbox files can vanish between turns – keep sources in outputs.

## 2. Repository layout (`ProMenu/`)
- `.github/workflows/android.yml` – Actions workflow (name "Build ProMenu (arm64)"): runs `setup.sh`, `ndk-build`, uploads artifact `libProMenu64`; on failure prints error lines as annotations.
- `setup.sh` – clones ImGui **v1.91.9** and AML, copies `aml/mod` to `jni/include/mod`, appends a hidden-visibility `IMGUI_API` define to `imconfig.h`, writes `jni/aml_inc.mk`.
- `jni/Android.mk`, `Application.mk` – arm64-v8a only, `c++_static`, C++17, `-Wl,-Bsymbolic`; compiles AML's `mod/logger.cpp` (or `logger_fallback.cpp`).
- `jni/main.cpp` – AML entry, symbol resolution, 4 hooks, per-frame function, lazy SAUtils acquisition, font loading (Latin + Thai merge).
- `jni/ui.h` – everything UI: data tables, game access, action queue, teleport/weapon/drop engines, touch handling, widgets, all pages, config.
- `jni/rw_backend.h` – ImGui renderer on RenderWare. `jni/safemem.h` – fault-proof memory reads. `jni/diag.h` – breadcrumb log.
- `jni/cheats100.inc`, `cheats100_th.inc`, `cheat_groups(.inc/_th.inc)`, `vehicles.inc` – data (cheat index order must not change). `cheat_table.inc` is an obsolete leftover.
- Runtime files (phone): `Android/data/com.rockstargames.gtasa/files/ProMenu/` → `config.txt`, `spots.txt`, `last_action.txt`, `previous_session.txt` (breadcrumbs of the previous run), optional `font.ttf`, `font_th.ttf`.
- Install: put `libProMenu64.so` in `Android/data/com.rockstargames.gtasa/mods/`; remove the old `libCheatMenu64.so` (same hooks, they conflict).

## 3. Platform facts
- Game lib `libGTASA.so` (arm64). AML **1.4.1** (RusJJ/AndroidModLoader). Mod macros: `MYMOD`, `NEEDGAME(com.rockstargames.gtasa)`, `ON_MOD_PRELOAD/LOAD`, `DECL_HOOKv`, `HOOKSYM`; `aml->GetLibHandle`, `aml->GetSym`; `GetInterface("name")` (inline in `mod/interface.h`) returns another mod's interface object.
- Link needs AML's `mod/logger.cpp` compiled in, otherwise undefined `Logger::*` symbols. `-Wl,-Bsymbolic` keeps the private ImGui copy from clashing with other mods.
- IAML is a C++ vtable interface: an older loader than the headers can mismatch.

## 4. Architecture (as implemented)
**Hooks (all verified in the ARM CheatMenu binary):**
- `_Z14AND_TouchEventiiii(type, finger, x, y)`: **type 1 = UP, 2 = DOWN, 3 = MOVE** (getting this wrong caused the early "can't tap" bugs). Pixel space = `RsGlobal` size.
- `_ZN6CDebug22DebugDisplayTextBufferEv`: once per frame after the HUD → call original, then draw ImGui.
- `_ZN5CGame18ShutdownRenderWareEv`: destroy the font texture before GL goes away (recreated lazily).
- `_ZN6CClock6UpdateEv`: game tick. **All game-modifying actions run here** through a small queue (`Enqueue(kind,i,f,g)` → `RunQueued`). The UI/touch side never calls the game.
- Frame guards: `gGameState` (7 = front-end → hide, 9 = playing), `gMobileMenu` int at +0x24 ≠ 0 → pause menu, `RsGlobal` width +8 / height +12.

**Rendering:** 28-byte vertex `{x,y,z,rhw,ABGR,u,v}`, `z=*CSprite2d::NearScreenZ`, `rhw=*CSprite2d::RecipNearClip`; `RwIm2DRenderIndexedPrimitive(3,…)` exported as `…P14RwOpenGLVertexiPti`; render states in this order: ZTEST(6)=0, ZWRITE(8)=0, VERTEXALPHA(12)=1, SRC(10)=5, DEST(11)=6, FOG(14)=0, CULL(20)=1, BORDER(13)=0, ALPHAFUNC(29)=5, ALPHAREF(30)=2, FILTER(9)=2, ADDRESS(2)=3, then TEXTURERASTER(1)=raster. Font atlas → `RwImage` → `RwRaster` (RwImage stride +0x10, pixels +0x18); raster is the `ImTextureID`. Clip with `CWidget::SetScissor(CRect{left,bottom,right,top})`, reset with zeros afterwards.

**Touch UI:** ImGui is used for drawing only. Taps/drags/scroll are processed from raw events: one menu finger at a time (others pass to the game), slop ≈ 48 px, tappable rectangles registered each frame under a mutex, tap fires on UP when DOWN and UP hit the same rectangle, list scrolling from deltas with fling, 5 s watchdog for a lost UP. The floating button stays where dropped (position saved), lock/fade options.

**Safety layers (user's v0.8):** `SafeMem::Read/Get` (kernel-copy based, never faults) for every raw game-memory read; `Diag::Crumb(...)` writes breadcrumbs to `last_action.txt` (rotated to `previous_session.txt` on next launch) – use it before every risky call; teleport finds the vehicle's own `Teleport` through the vtable and calls it with both argument conventions; `GetWaypoint` is polled ~4 Hz and understands both radar-table layouts.

**Language:** `Tr("English key")` table `kTr` (EN→TH), per-cheat Thai labels (`cheats100_th.inc`). Thai glyphs come from a merged font (user `font_th.ttf`, else a `*Thai*` file in `/system/fonts`). ImGui does no text shaping (stacked marks may look off). Every `kTr` pair must keep identical printf conversions.

**Config:** `config.txt` line: `fx fy alpha scale fade lock lang overlayFlags overlayPos overlayColor` (older shorter lines still load).

## 5. Verified game facts (64-bit v2.10)
| Topic | Facts |
|---|---|
| CPed | health `+0x6AC`, max health `+0x6B0`, armour `+0x6B4` (float); weapon array `+0x730`, 13 slots × `0x20` (type int `+0`, total ammo `+0xC`), active slot int8 `+0x8DC`; position: matrix ptr `+0x18` → pos at `matrix+0x30`, else placement `+0x8` |
| CPlayerInfo | money int `+0xF0` via `CPlayerPed::GetPlayerInfoForThisPlayerPed()` |
| Vehicle | type int `+0x734` (9 = bike), class int `+0x738` (3/4 = heli/plane) |
| CWeaponInfo | `GetWeaponInfo(type, 1)`; model ids at `+0xC`, `+0x10` |
| Radar | `CRadar::ms_RadarTrace`, 250 × `0x30`; sprite byte `+0x28` (**41 = waypoint**), x,y,z floats `+0x8/+0xC/+0x10` (the original dereferences a pointer; v0.8 also accepts an inline array) |
| Cheats | `CCheat::m_aCheatFunctions` (`_ZN6CCheat17m_aCheatFunctionsE`, `void(*)()`, null = plain flag) and `m_aCheatsActive` (`…15m_aCheatsActiveE`, bytes). Press = call function, else flip flag. 100 cheats, index order = `cheats100.inc` (WEAPON1 … TAXINITRO) |
| Clock/time | `CClock::CurrentDay` u8 (assumed 1 = Sunday), `ms_nGameClockHours/Minutes` u8, `ms_nMillisecondsPerGameMinute` u32 (1000 = 24 min/day), `CClock::SetGameClock(h,m,day)`, `CTimer::ms_fTimeScale` |
| Weather / camera | `CWeather::ForceWeather(short)`, `ReleaseWeather()`; `TopDownCamera::m_bEnable` (bool), `m_nZoom` (int 20–60) |
| Visual flags (single bytes) | `CPostEffects::m_bInfraredVision/m_bNightVision/m_bDarknessFilter/m_bFog`, `CSpecialFX::bVideoCam/bLiftCam`, `CTheScripts::bDisplayHud`, `CTheScripts::bPlayerIsOffTheMap` (grey radar), `CHud::bScriptDontDisplayRadar` (**inverted**: it means "hide") |
| Teleport recipe | `CStreaming::LoadSceneCollision/LoadScene(const CVector*)`, `LoadAllRequestedModels(false)`, `CWorld::FindGroundZForCoord(x,y)+1`, `CWaterLevel::GetWaterLevelNoWaves(x,y,0,&lvl,0,0)` (stand on water unless "underwater"), `CBike/CAutomobile::Teleport` + `PlaceOnRoadProperly`, or `CPed::Teleport`; aircraft keep their altitude |
| Streaming | `CStreaming::RequestModel(model, 0x10)` (priority) + `LoadAllRequestedModels(false)` before `CPed::GiveWeapon(type, ammo, true)` and `SetCurrentWeapon(type)` |
| ABI | `CVector` is non-trivial in this ABI: passed **by pointer** (`CPed::Teleport(ped, CVector*, uchar)`) and returned through a hidden pointer (do **not** call `FindPlayerCoors`-style functions with a trivially-copyable struct; read memory instead) |

**SAUtils script interface (powerful, used for Drop Weapon):** `GetInterface("SAUtils")` → object; `SAUtils::ScriptCommand(const SCRIPT_COMMAND*, ...)` (symbol `_ZN7SAUtils13ScriptCommandEPK14SCRIPT_COMMANDz`, also vtable slot 39 / offset 0x138). `SCRIPT_COMMAND` = `{uint16 opcode; char params[16];}` (18 bytes) with parameter letters `i` int, `f` float (pass **double**), `v` output pointer. Internally it fills a buffer and runs the game's `ProcessOneCommand` on its own script thread (`gst`), so the call works from any hook.
**Critical:** every `v` output is written with a **64-bit store** (value in the low 32 bits, zero in the high 32 bits). Output variables must be 8 bytes (`unsigned long long`), or the neighbour gets wiped. This was the real cause of "dropped weapon not on the ground" (see `reference/drop_weapon_bug_screenshot_readout.txt`).
Drop Weapon = `04C4 "ifffvvv"` (offset from char: ped ref, 0, 3.0, 0 → x,y,z), `0781 "iv"` (weapon type → model), `032B "iiifffv"` (model, pickup type 3, ammo, x,y,z → handle), `0555 "ii"` (ped ref, weapon type). `CPools::GetPedRef(ped)` gives the ped handle. The weapon is removed only if a pickup handle came back.
SAUtils exports (same library) also include helpers such as `CreateObject`, `SetPosition`, `LoadModelId` (see its dynsym) – useful for other mods, unverified.

## 6. Features in v0.9.4 (tabs)
Player (wanted 0–6, health, armour, money steppers) · Vehicle (88 vehicles by category via `CCheat::VehicleCheat(model)`) · Weapon (39 weapons × ammo 100/500/1000/9999, drop held weapon, remove held, remove all with double-tap, test switch "keep weapon", on-screen readout) · Teleport (waypoint, X/Y steppers, 10 places, 6 saved spots, underwater switch) · Visual (9 flag switches) · Game (sync to phone clock, game speed, hour/minute, day length, day of week, weather, top-down camera) · Cheats (100 cheats in 12 groups, ON/OFF badge where the flag is real, "Cheat NAME Activated/Deactivated" toast, danger group needs double tap) · Menu (English/Thai, overlay FPS/coordinates/playtime with position/colour/RGB/no background, floating-button settings, About/diagnostics: touch counters, symbol count, game state, radar layout).

## 7. Not done yet (see also LIVE STATUS above)
Scene (cutscenes: ARM's `CutsceneMgr::Play` calls `CCutsceneMgr::LoadCutsceneData`, `Update`, `StartCutscene` plus ped/camera handling; name list not extracted) · Vehicle tools (flip, colour, current-vehicle options, seat switching, spawn as driver/in air, don't fall off bike, flip-no-burn, density) · Ped (big head, thin body, density multiplier, spawn ped, ped tasks) · Player extras (infinite energy, super sprint, drunk, disable input, change to CJ, skins; god mode exists as cheat INVINCIBILITY) · Weapon infinite ammo · Visual: keep fog on low graphic, invisible water · Game: freeze mission timer, lock camera zoom, solid water, gravity · Overlay: location name, ped tasks.
**Open bug:** crash when pressing "Remove all weapons" with no weapons – see `reference/crash_enter_car.txt`; unresolved. First step next time: get `previous_session.txt`.
**Unverified in game:** everything added after v0.7 except what the user reported (v0.8 teleport fix by the user; drop-weapon behaviour observed through v0.9.3).

## 8. Bug history / lessons (do not repeat)
1. Touch type codes were swapped (1/2/3) → nothing tappable. Always trust the original binary, not memory.
2. Missing `mod/logger.cpp` in the build → link error.
3. Calling `FindPlayerCoors` with a by-value struct return → ABI mismatch (hidden-pointer return). Read position from memory.
4. Reading the radar table blindly crashed the render hook → use `SafeMem` for any raw game pointer.
5. ImGui's mouse click logic is unreliable for touch → own tap/scroll logic.
6. Writing output pointers for SAUtils script commands: 8-byte variables only (see above).
7. Never remove a weapon (or any resource) before confirming the replacement exists (pickup handle valid).
8. Do not assume `GetSym` names from the PC game exist on Android; confirm in the ARM binary or fail gracefully (disable the control, say why).
9. Keep each answer/package small; the user prefers one verified step at a time.

## 9. Reverse-engineering method (tools were lost; recreate if needed)
Inputs the user has: `GTA_SA_AML_PLUGIN_2_10.zip` (ARM CheatMenu v1.6 = `libCheatMenu64.so`, SAUtils, configs, Arial font) – no source exists. Method: `readelf -Ws --dyn-syms`, `strings`, resolve PLT stubs through `R_AARCH64_JUMP_SLOT/GLOB_DAT` relocations, recover pointer tables (e.g. the 100 cheat names at data `0x23d190`) via `R_AARCH64_RELATIVE` relocations, then decode `XxxPage::Draw` functions with a small hand-written AArch64 decoder (adrp/add string resolution, GOT symbol names, `bl` targets). Tables of single-char `SCRIPT_COMMAND` params were read straight from rodata (18 bytes each). Host-side verification: compile `main.cpp` with stand-in `imgui.h`/AML headers (`g++ -fsyntax-only`) and unit-test the pure logic (touch, queue, teleport, weapons, drop with a fake script engine that stores outputs with 64-bit writes, config round trips, translation completeness and printf-format equality) against fake game memory under ASan/UBSan.

## 10. References and licensing
- AML source (`AndroidModLoader-main.zip`, has `template_of_mod`), **aml-psdk** (GTA SA structures for 2.10 64-bit; **GPL v3** – reusing its code makes the mod GPL), JPatch (example AML mod), Dear ImGui (MIT).
- No ARM CheatMenu code is copied; it was analysed to learn the game's behaviour and symbols. Do not redistribute ARM's files.

## 11. Ideas for other mods (e.g. a Minecraft-style system) – not investigated
The map is static data (IPL/IDE/COL). Block building needs runtime objects (pools, model info, collision, streaming) and custom models/textures. Starting points: SAUtils script commands (create object / set position / load model), then check which `CObject`, `CPools`, `CModelInfo`, `CStreaming`, `CWorld::Add/Remove`, `CColModel` symbols libGTASA exports (use the same `GetSym` + `SafeMem` + breadcrumb discipline). Prototype one placed object first; expect streaming and performance limits. Build the UI on the existing touch/overlay layer.
