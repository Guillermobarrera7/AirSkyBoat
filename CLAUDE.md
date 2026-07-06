# AirSkyBoat Server "Moe" — Guillermo's solo local FFXI server

Single-player local server. Priorities: stability while playing, log-driven debugging.

## Layout

- `Z:\AirSkyBoat\AirSkyBoat` — this repo (server source + prebuilt exes in root)
- `Z:\AirSkyBoat\SquareEnix` — FFXI client (April 2024 client, version `30240426_0`)
- `Z:\AirSkyBoat\Windower` — Windower 4 launcher
- `Z:\AirSkyBoat\xiloader` — private-server bootloader
- `Z:\AirSkyBoat\README\scripts.md` — GM command cheat sheet
- `Z:\AirSkyBoat\backup-2026-07-05\` — pre-update backup: DB dumps, old May-2024 exes + PDBs, settings

## Version state

- Working branch is **`moe-fixes`** — all local fixes are committed here and pushed to the fork.
- `base` stays pristine at `44f323f158f5`, the **final** commit of the archived upstream `base` branch
  (upstream https://github.com/AirSkyBoat/AirSkyBoat archived 2025-02-24; no further updates will ever exist).
  `git diff base` shows every local fix ever made.
- Updated from `f37ebf70c6` (May 2024) on 2026-07-05. Rollback = old commit + `backup-2026-07-05` exes + DB dump restore.
- `origin` = Guillermo's fork (https://github.com/Guillermobarrera7/AirSkyBoat.git); `upstream` = archived original.
- Do NOT look for upstream updates; the project is frozen. Fixes are ours to make locally.
- GitHub auth: `gh` CLI is installed and logged in as Guillermobarrera7; it is git's credential helper. Push after each committed fix.

## Start / stop

- `Z:\AirSkyBoat\start-server.ps1` — checks MariaDB, launches xi_connect → xi_search → xi_world → xi_map
- `Z:\AirSkyBoat\stop-server.ps1` — stops all four (map first)
- Player connects via Windower/xiloader to 127.0.0.1.

## Database

- MariaDB 10.6 (Windows service `MariaDB`), db `xidb`, root @ 127.0.0.1:3306, password in `settings/network.lua`.
- `tools/dbtool.py` (run from `tools\`): `backup`, `update` (diffs sql/ since `db_ver` in `tools/config.yaml`, runs `tools/migrations/`, never touches the 35 protected player tables — accounts, char_*, auction_house, etc.).
- `auto_backup: 0` in `tools/config.yaml` → dbtool does NOT back up automatically. Always `python dbtool.py backup` before schema changes.
- Backups land in `sql\backups\`.

## Settings (important)

- Two layers: `settings/default/*.lua` (tracked, never edit) then top-level `settings/*.lua` (gitignored user overrides, key-by-key wins).
- **Edit only the top-level files.** The top-level files are full copies from May 2024 — when a default key changes meaning upstream, the stale copy silently overrides it (this bit us with `DIG_FATIGUE`).
- Current intent: **1x everything, era-authentic** (all EXP/drop/gil multipliers 1.0).
- `settings/login.lua`: `CLIENT_VER` must stay `30240426_0` to match the installed client (`VER_LOCK = 2` rejects clients older than this value). dbtool `update` may bump it — check after every dbtool run.

## Logs & debugging

- Logs: `log\map-server.log`, `connect-server.log`, `search-server.log`, `world-server.log` (append-only, can grow large).
- Quick error scan: `Select-String -Path Z:\AirSkyBoat\AirSkyBoat\log\*.log -Pattern "error","crash","critical"`
- Crashes write a symbolized text report + minidump to `dmp\` (report includes full call stacks — read the .log there first, no debugger needed).
- Known crash history (old May-2024 build, pre-update):
  - Ctrl+C shutdown crash in `ConsoleService::stop` (console_service.cpp:226) — benign, shutdown-only.
  - LuaJIT fault during `luautils::OnZoneTick`/`GetCacheEntryFromFilename` (luautils.cpp:799) after 14 min uptime, followed by a double-fault in cleanup (`CItemContainer::Clear`). Watch for recurrence on the new build.
- Lua script errors appear in map-server.log with `[lua]` tag; script fixes need no rebuild (scripts are loaded/cached at runtime; `!reloadglobal`/zone re-entry or restart picks them up).

## Building (C++ changes only)

- Use the **VS-bundled** CMake — the PATH `cmake` is devkitPro/MSYS2 and cannot build this:
  ```
  & "C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe" -S . -B build_update -G "Visual Studio 17 2022" -A x64 -DCMAKE_BUILD_TYPE=Release
  & "...\cmake.exe" --build build_update --config Release -j
  ```
- Exes output directly into the repo root (gitignored). Stop servers before building or linking fails.

## GM commands (in-game, GM char)

`!godmode 1`, `!givegil <n>`, `!giveexp <n>`, `!togglegm`, `!speed <n>`, `!zone <zone>` — see `Z:\AirSkyBoat\README\scripts.md`.

## Enabled modules (`modules/init.txt`)

- `custom/commands/`
- `custom/lua/test_npcs_in_gm_home.lua`
