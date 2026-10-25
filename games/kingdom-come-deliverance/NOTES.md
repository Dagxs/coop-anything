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

## Console test 2 (2026-10-25)
- `sv_port` in user.cfg is overridden at startup (still 64087 in console); setting it in the console at the main menu works.
- Test 1 repeated at 64087: same "Failed creating network nub at 0.0.0.0:64087".
- `sv_port 64100` then `map rataje s`: NO nub error this time. multiplayer.cfg loads, CBreakReplicator initializes, then the local client does
  "connection requested to: <local>:64100", retries 110 times over 30 s, "Connection to <local>:64100 timed out" -> "Failed initializing game".
Derived:
- The 64087 failure was most likely the port being in use, not hosting being compiled out.
- At 64100 the server side gets further but never answers its own client's handshake. Possible causes: server-side channel handling stripped (PURE CLIENT), or the server nub silently not listening. Next check: is UDP 64100 actually bound (netstat on the host).
- Host check while stuck after test 2: `netstat -ano -p udp` shows `UDP 0.0.0.0:64100 *:*` owned by the game process. The server socket IS open; the server just never answers the connection handshake.
- String search of WHGame.dll for connection-request/reject handling was inconclusive (mostly libcurl/TLS strings). Next: static analysis of the CryNetwork connect path in Ghidra.

## Server-side handshake strings (static, ran)
WHGame.dll still contains CNetNub server-side handshake strings: "processing KeyExchange0 with illegal sized packet", "Too short connection packet", "processing KeyExchange1 with illegal sized packet", "Protocol version mismatch", "Build version mismatch", "Client token id mismatch", "Pending connection not found". Server join handling does not look compiled out.

## Where we left off (2026-10-25)
- Test copy on the owner's PC in a separate folder; user.cfg sets con_restricted=0. Real install untouched.
- Hosting test recipe: main menu, console: `sv_port 64100`, then `map rataje s`. Server binds UDP 64100, host's own client connects to `<local>:64100` and times out; game hangs (kill via Task Manager).
- Hypothesis (unverified): the `<local>` loopback path is only wired for normal single player (client at `<local>:16`); a remote client over real UDP may behave differently.
- Next steps:
  1. Ghidra analysis of WHGame.dll (Ghidra 12.2 + Temurin JDK 25; first pass was stopped before finishing). Find xrefs to "Retrying connection to %s (%d)", "connection requested to: %s", "processing KeyExchange0..." and follow how `<local>` addresses are routed vs UDP.
  2. Try a real second client over UDP (second PC, or bypass the single-instance check on one PC) connecting to the host's 64100.

## Session 2 (2026-10-25 evening): the listen-server join block
Tooling: own PE helpers (pefile + capstone + numpy) instead of Ghidra; Ghidra 12.2 headless ran out of heap (6 GB) on the 48 MB WHGame.dll.
RVAs below are relative to WHGame.dll image base 0x180000000, build SHA-256 cf9f6dc3...81aa.

Observations (static, ran):
- CNetNub connection-packet functions (found via their log strings): conn-setup checks (size / "Protocol version mismatch" / "Client token id mismatch" / "Build version mismatch") at 0x2102670; KeyExchange0 at 0x2101d48; KeyExchange1 at 0x2101fac; disconnect-packet handler at 0x2101818; client connect "connection requested to: %s" at 0x20fa250; client retry loop ("Retrying connection", "Resending KeyExchange1") at 0x2103688.
- Packet dispatcher 0x393900 (+ cold part 0xc32d3a) switches on packet type. For type 5 (connection setup) at 0xc32d99 and type 0x8A at 0xc32e58 it tests three globals:
  0x29d18d0 = gEnv bServer, 0x29d18d1 = gEnv bMultiplayer, 0x29d1959 = gEnv bClient (all three written together in CCryAction::StartGameContext around 0x183cf96..0x183cfc1 from the game-context flags; the "multiplayer"/"singleplayer" config choice happens there too).
- If bServer && bMultiplayer && bClient (a listen server), a connection-setup packet is passed to the DISCONNECT handler with length 1 ("Corrupted disconnect packet") instead of the server conn-setup handler, and type 0x8A goes to KeyExchange0 instead of KeyExchange1. A dedicated server (bClient=0) or pure client takes the normal path.
- `map` console command options (CCryAction::MapCmd ~0x1887640): "dedicated", "record", "server", "nonblocking".

Experiment (ran): proxy dinput8.dll in the test copy's Bin\Win64 (source kept in local tooling) patches 0xc32d99 -> `EB 16` and 0xc32e58 -> `EB 2D` in memory after verifying original bytes; loader log confirmed both patches applied. Result of `sv_port 64100; map rataje s`: unchanged, host's own client still times out after 30 s retrying `<local>:64100`.
Derived:
- The listen-server block is real but is not the only obstacle for the host's own `<local>` client; its setup packet may never reach the dispatcher, or fails later.
Next:
- Try `map rataje s dedicated` (no local client) and check `status`; then connect a second client over real UDP (single-instance check in KingdomCome.exe blocks a second copy on one PC: test from a second PC or bypass that check in the test copy).

## Dedicated server test (ran)
- With the loader patches active: `sv_port 64100; map rataje s dedicated` -> log reaches "Level system is loading \"rataje\"" and the process exits with no further log lines, no dump, no error dialog. Dedicated mode does not survive level load in this client build (cause unknown; patches not ruled out, but the dispatcher is not involved before level load).

## Status after session 2
Waking CryEngine's built-in netcode has hit three separate barriers so far: (1) listen-server join block in the packet dispatcher (found, patchable), (2) the host's own `<local>` client still never completes its handshake with the block removed, (3) `dedicated` map mode exits during level load. Each could be investigated further, but the path is uncertain.
Alternative (the KCDMP / most-mods route): leave both games in normal single player and add our own sync layer from an injected DLL (positions/animations/horses first, then combat, then story), with our own transport (Steam relay). Loader/proxy DLL from this session is a usable injection point.
