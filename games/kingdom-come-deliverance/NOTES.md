# Kingdom Come: Deliverance — working notes

## Game identity (phase 0)
| Field | Value |
|---|---|
| Store | Steam |
| OS tested | Windows (x64) |
| Game version | 1.9.8 (`wh_sys_version` in system.cfg); exe FileVersion 1.9.7.0 |
| Build | `404-504czj4`, bin `Steam_ver_01_09/Bin64_MasterRetail ver_01_09_598307_726`, includes 2026 "Gen9" updates per whdlversions.txt |
| PE timestamps | KingdomCome.exe 2026-04-01 08:18:09 UTC; WHGame.dll 2026-04-01 08:32:21 UTC |
| SHA-256 KingdomCome.exe | cde5c7d03e76abef91b5dfbcd4e2ccf070ab4811a0fc652b429dacf5559bca73 |
| SHA-256 WHGame.dll | cf9f6dc384edcf35c20647a912745ddb8adb5ba65953e329c24e89dd9c4381aa |
| Engine | CryEngine (Warhorse fork), monolithic: engine modules statically linked into WHGame.dll (47.9 MB) |

## Observations (ran)
- WHGame.dll exports only `CreateGameStartup` and `wh::game::C_Game::CreateInstance`.
- WHGame.dll imports WS2_32/WSOCK32/IPHLPAPI and steam_api64.
- Strings show CryEngine's network module compiled in: `CEngineModule_CryNetwork`, `CGameServerNub`, `CGameClientNub`, `INetChannel`, `CScriptRMI`, 29 `net_*` and 39 `sv_*` cvars, `sv_port`, `sv_bind`, `cl_serveraddr`, `connect`/`disconnect` commands, `IsMultiplayer`.
- No `lua_pcall`/`luaL_loadbuffer` symbol strings (Lua statically linked, names stripped).

## Derived (not tested)
- Like other CryEngine games, single player probably runs as a local server + client, so the engine's own netcode may be dormant rather than absent. This is the same starting point the Crysis co-op mods used (campaign as network map, host-authoritative SP logic).
- Warhorse's own systems (quests, dialogue, RPG stats, souls/inventory) are unlikely to be network-aware; those will need custom sync.
- 2018 MafiaHub PoC (KCDMP) targets a 2018 build; offsets will not match this 2026 build, and its license forbids forks. Not used.

## Unknown
- Whether `connect`/server commands work from the dev console (needs `-devmode` test on a test copy).
- Whether DLL injection works under Proton.

## Sources
- https://github.com/MafiaHub/kcdmp_archived (PoC: movement, animation, horses only; no-fork license)
- https://github.com/benjaminfoo/KCD-Coding-Guide (Lua console, .pak mods)
- https://gist.github.com/CrumblyLiquid/a7d9c6d9077ae362b2b7deaad8259409 (KCD1 modding resource list)
- games/crysis/README.md in this repo (CryEngine campaign co-op design)

## Console test 1 (2026-10-25, test copy, user.cfg con_restricted=0)
Observations (ran):
- `status` at main menu: "Server Status ... gamerules: SinglePlayer, players: 1/1"; Connection Status shows player "Dude" at `<local>:16`. Single player runs as an in-process server + local client.
- cvars: `sv_port = 64087`, `sv_bind = 0.0.0.0`, `sv_maxplayers = 32`, `sv_gamerules = SinglePlayer`, `cl_serveraddr = localhost`.
- Startup log: "Initializing CryNetwork done", "[Network Version]: RELEASE PURE CLIENT", "Remote console listening on: 4600".
- `map rataje s` (load level as network server): loads `data\config\multiplayer.cfg`, then "[net] Failed creating network nub at 0.0.0.0:64087" -> "Failed initializing game"; game hangs on the loading screen and has to be killed.
Derived (not yet tested):
- Either the server nub is compiled out (PURE CLIENT), or the port was busy. Test 2: sv_port=64100.
