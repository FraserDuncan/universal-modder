# The Binding of Isaac: Rebirth (Afterbirth+, Repentance, Repentance+)

**Identify.** `isaac-ng.exe` (32-bit native), `resources/packed/*.a`, a `mods/` folder in the install (the
exact path is in `savedatapath.txt`) and `tools/` with `ResourceExtractor`, `IsaacAnimationEditor`,
`IsaacItemPoolEditor`, `RoomEditor`, `ModUploader` and `LuaDocs` (an offline Doxygen dump of the API).
`options.ini` and `log.txt` are in `Documents\My Games\Binding of Isaac <newest DLC>\` (`Repentance+` today);
unlock data is in Steam `userdata\<id>\250900\remote\`. No anti-cheat. Repentance+ (November 2024) added
online co-op: mods are single-player only there, and nothing here is for the online side.

**Route: a Lua mod on the vanilla API, no decompiling.** Custom entities with Lua AI, items and pools,
familiars, HUD rendering, input, save data and console commands are all covered. Reach for REPENTOGON only
when you need what vanilla hides (XML data at runtime, ImGui, extra callbacks, unlocks for modded content).
- Reference: IsaacDocs, `https://wofsauge.github.io/IsaacDocs/rep/` (classes, 74 callbacks, every content XML,
  the console, launch options, troubleshooting). Tutorials: Isaac Blueprints, `https://isaacblueprints.com/`
  (crash course, current for Repentance+). Community: the Modding of Isaac Discord and r/themoddingofisaac.
- Worked example in the knowledge base: `games/the-binding-of-isaac-rebirth/` (Terraria bosses, items and a
  HUD inside Isaac, with 21 gotchas and a file bridge for testing).

## Mod anatomy
- `mods/<folder>/` with `metadata.xml` (`<name>`, `<directory>` = the folder name, `<id>` filled in by the
  uploader, `<description>`, `<version>`, `<visibility>`); launching with an empty folder generates one. Mods
  load alphabetically by folder name. Workshop subscriptions install as `<name>_<workshop id>`.
- `main.lua`: `local mod = RegisterMod("Name", 1)`, then `mod:AddCallback(ModCallbacks.MC_X, fn, filter)`
  where the optional filter is an entity type, variant, item or card id. `mod:AddPriorityCallback` orders
  mods on the same callback. `main.lua` runs even when mods are disabled in the menu.
- `content/*.xml` adds data: `entities2.xml`, `items.xml`, `itempools.xml`, `sounds.xml`, `music.xml`,
  `players.xml`, `challenges.xml`, `costumes2.xml`, `bossportraits.xml`, `rooms/*.stb` and about twenty
  more (IsaacDocs' XML index lists them all). `content/bosspools.xml` has no effect in a mod: custom bosses
  can't join vanilla boss pools without StageAPI, or without replacing a vanilla boss at spawn.
- `resources/` mirrors the vanilla tree and overrides any file by path (PNG, anm2, fonts, music).
  Repentance kept its own files in `resources-dlc3/`; Repentance+ merged them into `resources/`. A mod's
  `resources-dlc3/` is an overlay checked before `resources/` (Repentance-only overrides for a mod that
  also supports Afterbirth+); on Repentance+ `resources/` alone is enough.
- `require("x")` caches and resolves across every loaded mod, so two mods with `lib.lua` collide: keep files
  under a folder named like the mod. `include("x")` never caches and only sees your own mod; it's what
  `luamod` reloads.
- Persistence: `mod:SaveData(str)`, `LoadData()`, `HasData()`, `RemoveData()` write `data/<directory>/
  save1-3.dat` per save slot, the only writable files without `--luadebug`, and only during a run. Save in
  `MC_PRE_GAME_EXIT`, load once in `MC_POST_PLAYER_INIT`, `pcall` the JSON decode (`require("json")`), and
  don't put Vector, Color or RNG userdata in it. `entity:GetData()` is a plain table that dies with the
  entity. Any enabled mod blocks achievements until Mom has been beaten with mods off.
- Vanilla assets to read or copy sizes from: run `tools/ResourceExtractor/ResourceExtractor.exe` from its
  own folder with no arguments; it unpacks every archive to `<game>/extracted_resources/` (the first
  argument is the *input* path; an output folder there gives "No archive files found"). `filelist.txt` has
  the names. Never ship those files: a UI override is your own PNG at the vanilla sheet's size.

## Versions
- Afterbirth+ → Repentance changed the API in places (IsaacDocs marks them); there is no separate
  Afterbirth+ edition of the docs.
- Repentance+ (v1.9.7.4, November 2024; v1.9.7.17 in 2026) is the current build: it broke many Workshop
  mods, renumbered the `ButtonAction` enum (hard-coded ints break), added `Entity:KillWithSource`,
  `ItemPool:GetCollectible(..., backupPool)`, new fonts, and merged `resources-dlc3`. Isaac Blueprints keeps
  a "Repentance+ changes" page.
- REPENTOGON (`https://repentogon.com/`, GPL-2.0) is a native script extender for Repentance+: a launcher
  kept outside the game folder drops `dsound.dll`, `libzhl.dll` and `zhlREPENTOGON.dll` into the install
  and runs the game (Steam launch option `"<path>\REPENTOGONLauncher.exe" --isaac=%command%`). Lua 5.4,
  800+ extra functions, ImGui console and menus. A snippet that calls `Sprite:HasAnimation` or `XMLData`
  assumes it; vanilla Repentance+ lacks them. Installing it changes game files, so ask first.

## Facts to know
- Entities are addressed as `type.variant.subtype` everywhere (console, XML, `Isaac.Spawn`). Custom NPCs
  take an id vanilla doesn't use (vanilla types run to 413, then 801-970; the cap is 4095; 3 = familiar,
  1000 = effect) and are renumbered when mods overlap, so resolve them at runtime with
  `Isaac.GetEntityTypeByName` / `GetEntityVariantByName`. `boss="1"` in `entities2.xml` marks a boss; in
  practice that is enough for room clear, the boss drop and the trapdoor to work for a custom boss.
- `Isaac.Spawn(type, variant, subtype, pos, vel, spawner)` and `Game():Spawn(type, variant, pos, vel,
  spawner, subtype, seed)` take different argument orders; `Isaac.Spawn` can roll seed 0 and crash, so
  prefer `Game():Spawn` with a seed from `GetDropRNG()` or similar. `RNG():SetSeed(seed, 35)` needs a
  non-zero seed.
- Rates: `MC_POST_UPDATE` and `MC_POST_PEFFECT_UPDATE` 30/s and paused with the game; `MC_POST_RENDER`
  and `MC_POST_PLAYER_UPDATE` 60/s. `Isaac.GetFrameCount` counts in menus, `Game():GetFrameCount` doesn't.
  `Isaac.GetPlayer()` is nil in the main menu.
- `MC_PRE_ENTITY_SPAWN(type, variant, subtype, pos, vel, spawner, seed)` returning `{type, variant,
  subtype, seed}` replaces anything before init, room-placed bosses included; the docs say the last
  callback to return a value wins (observed on 1.9.7.17: once one callback returned a replacement, a later
  observe-only callback didn't run). Returning type 0, a non-table, or swapping across entity classes
  (pickup to effect) crashes. `MC_PRE_GET_COLLECTIBLE` must return a collectible id or nil.
- Boss rooms: a mod's `content/rooms/00.special rooms.stb` with rooms whose subtype is the boss id gets
  picked; the same rooms in `01.basement.stb` were never used (observed; the console's `goto x.boss.<n>`
  shows the stage files can hold boss rooms, so the rule is about selection, not loading). Write STB from
  Basement Renovator, or from Python using its STB1 layout.
- anm2 is XML (ElementTree is enough); layers in ascending id order, every layer needs a
  `<LayerAnimation>`, paths are case-sensitive in practice. `<Spritesheet Path>` is relative to the anm2's
  own folder: vanilla `gfx/ui/main menu/titlemenu.anm2` names `TitleMenu.png` beside itself. IsaacDocs
  says "starting from gfx/", which only agrees for anm2 files that sit in `gfx/` directly.
- Invisible entity with `Sprite:IsLoaded()` true: wrong sheet path, or a PNG the game won't decode (pngjs
  output, odd bit depth; a black or red square is the bit-depth case). Re-save PNGs with Pillow as 32-bit.
- A UI sheet override of the wrong size (Repentance+ moved and resized several: the title logo is its own
  `logo.png`, the completion-mark sheets are 384x384) hangs the game at file select with no log line.
  Extract the vanilla sheet and keep its size; bisect hangs with `EnableMods=0`.
- `MC_EXECUTE_CMD(cmd, args)` is the documented way to add console commands (it skips built-ins and
  crashes on any non-nil return). On Repentance+ 1.9.7.17 it did not fire in one session for any name,
  stock launch, handler registered; test it before relying on it, and fall back to `lua <code>` or a
  command file.
- Input: `Input.IsButtonTriggered` is consumed by the first call per update and missed automated short
  presses in a 30 fps callback; edge-detect `Input.IsButtonPressed` in `MC_POST_RENDER` when it matters.
- `Isaac.DebugString` truncates at about 10 KB; `log.txt` is wiped each launch and shows nothing on a
  crash, so print-bisect. Fully restart after enabling or disabling a mod.

## Tools
- Basement Renovator (`github.com/Basement-Renovator/basement-renovator`, Python): rooms and `.stb`, with a
  Repentance+ compatibility mode; launch it with your mod enabled or its entities are missing. The install's
  `tools/RoomEditor` is the older official editor.
- `tools/IsaacAnimationEditor` edits anm2 (its FPS and attribute panels are preview-only); Anm2ed
  (`github.com/ShweetsStuff/anm2ed`) is the community alternative. `tools/IsaacItemPoolEditor` for pools.
- Mod Config Menu Pure (Workshop 2681875787, `github.com/Zamiell/isaac-mod-config-menu`): the standard
  in-game options UI, detected through the global `ModConfigMenu`.
- `tools/ModUploader/ModUploader.exe`: point it at `metadata.xml`, set description (BBCode), tags, visibility
  and a thumbnail under 1 MB; the first upload writes `<id>`, every update needs a higher `<version>`.
  IsaacScript ships a GitHub Action for the same upload.

## Console and iteration
`EnableDebugConsole=1` in `options.ini`; the key depends on the layout (grave on US; IsaacDocs has the
table; VK 0xDF, the key left of 1, worked on a UK layout). `luamod <folder>` re-runs the mod's Lua and
`clearcache` reloads anm2 (observed), and both close the console: reopen it before typing more or the text
becomes WASD movement. XML and PNG changes need a restart. Useful: `lua <code>`, `restart [character]`,
`stage 1-13[a-d]`, `goto s.boss.<variant>` / `d.<n>` / `x.<type>.<variant>`, `spawn t.v.s`, `giveitem c15`,
`reseed`, `debug 3` (god mode; the debug flags toggle), `testbosspool`. `Isaac.ExecuteCommand` runs the same
from Lua. `--luadebug` unlocks `io`, `os`, `debug` and unsandboxed `require` for any mod, which IsaacDocs
warns can compromise the machine; prefer a stock launch.

## Automation
Launch through Steam (`um win launch --steam 250900`); on one machine, starting `isaac-ng.exe --luadebug`
directly made Steam hold a "launch with arguments?" dialog that wedged every later launch until
`steam.exe -shutdown`. Without `io`, a complete file bridge is still possible: `include()` re-reads its file
from disk every call, so a mod that includes a command file every few render frames and logs the result
through `Isaac.DebugString` lets an agent run Lua in the live game and read answers from `log.txt`. Windowed
mode (`Fullscreen=0`) for `um win shot`. The main menu defaults to Continue when a run exists; spawn test
bosses in rooms without pits or Isaac falls in and respawns at the door.
