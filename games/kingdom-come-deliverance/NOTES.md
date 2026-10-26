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

## Route change: own sync layer (2026-10-25)
Owner chose "do what works": both players run normal single player; an injected DLL syncs state (KCDMP-style).

### Script bridge (ran, works)
- Injection: proxy `dinput8.dll` in the test copy's Bin\Win64 (WHGame.dll imports DINPUT8!DirectInput8Create).
- `gEnv->pConsole` is the static pointer at RVA 0x29d1768 (seen as `mov rcx,[rip+..]; call [vt+0xb8]` = GetCVar).
- CXConsole::ExecuteString(cmd, bSilent, bDefer) at RVA 0xec6920 = console vtable[35] (verified at runtime). Deferred commands run on the main thread next frame.
- ICVar vtable: +0x10 GetIVal, +0x20 GetFVal, +0x28 GetString, +0x48 Set(const char*).
- "#lua" console commands are executed only if ISystem::IsDevMode() (check at 0xec7026: `call [rax+0x360]; test al,al; je` ). Loader nops that `je` (0xec702e `74 5A` -> `90 90`) instead of enabling dev mode. con_restricted is set to 0 via ICVar.
- Data back from the game: Lua `System.SetCVar('cl_serveraddr', ...)`, DLL reads it with GetString.
- Result: `[coop] hello from the kcdcoop loader` in kcd.log; loader log shows live `P x y z rotZ` (e.g. `P 733.88 3421.90 63.91 -1.965`), updating as the player moves.
### Puppet (in progress)
- v2.2: Lua `CoopSetRemote(x,y,z,rz)` spawns a BasicEntity "coop_puppet" with `objects/characters/humans/skeleton/male.cdf` and moves it; `CoopFakeFollow()` solo test keeps it 2 m beside the player.
- Puppet result (ran, owner-confirmed with screenshot): after New Game in Skalitz, a T-posed male character (BasicEntity + male.cdf) stands next to the player and follows as the player moves (fake-follow test, +2 m on world X). kcd.log: `[coop] puppet spawned id=... character=true`. The puppet spawned once in the menu scene and persisted into the new game (same level).
Next: real transport between two games (needs a second PC, or a second instance on one PC past the exe's single-instance check), then animation on the puppet.

### Two copies on one PC + transport (v3, testing)
- KingdomCome.exe single-instance check is a named mutex `WhKcdGameMutex` (CreateMutexA). Test copy gets a second launcher `KingdomCome2.exe` with the string changed to `WhKcdGameMutx2` (one occurrence, file offset 0x65b0); original exe untouched.
- Loader v3: role A = KingdomCome.exe, role B = KingdomCome2.exe. Plain UDP, A binds 64200 -> 127.0.0.1:64201, B the reverse (override with kcdcoop_A.ini / kcdcoop_B.ini: `bind=`, `peer=ip:port`). Each sends `KCDCOOP1 P x y z rotZ` at ~10 Hz and calls Lua `CoopSetRemote` with the newest received state. Fake follow now only with a `kcdcoop_fake` file.
- Result (ran 2026-10-25 20:53, owner screenshot of both windows + loader logs): two copies on one PC (A = KingdomCome.exe, B = KingdomCome2.exe) both loaded the same save; each shows a T-posed puppet driven by the other copy. Loader stats after ~2 min: A sent=1438 recv=1058, B sent=1112 recv=1289, bad=0; received positions track the other side's movement.
QA level for this milestone: manual (owner-observed, logs), not automated yet.

## Next steps
1. Puppet animation: replay locomotion (idle/walk/run) from speed, then the player's actual animation state.
2. Smoothing: interpolate between 10 Hz updates instead of snapping.
3. Friends over the internet: Steam relay (playbook/networking-steam-spacewar.md) instead of direct UDP.
4. Automated QA: bot that walks player A along a path and asserts B's puppet follows within tolerance (playbook/automated-qa.md).
5. Then combat/NPCs (host authority) and story sync.

## Session 2b: smoothing + animation (ran, owner-confirmed)
- Loader v3.1: ~30 Hz loop, exponential smoothing toward the newest remote state (snap if >5 m), remote ground speed estimated from packet deltas; command file `kcdcoop_cmd_<A|B>.lua` in the game root is run as one Lua chunk and deleted (live scripting without restarts). Commands queued while a level is loading are lost: send them once in the world.
- Owner: "it's very smooth now".
- `System.ScanDirectory` works inside paks. Male locomotion lives in Mannequin blendspaces under Animations/humans/male/blendspace/locomotion. Puppet (male.cdf) accepts as StartAnimation names (GetAnimationLength > 0): `1d_relaxed_idle_bigturns_nw` (2.07 s), `2d_relaxed_walk_strafe_spd_fwd` (2.61 s), `2d_relaxed_run_strafe_spd_fwd` (1.21 s), plus `_ext` and `3d_relaxed_*_turn_strafe` variants.
- Lua CoopAnimate(e, speed): idle / walk / run by remote speed with hysteresis (walk >0.7, run >3.5, back to walk <2.9, idle <0.3). Observed speeds: walk 1.7-2.7 m/s, run 3.3-4.0 m/s. Owner screenshot: both puppets idle-posed; log shows idle->walk->run->idle switching.
- Puppet looks like the base male model (bald, plain shirt). Henry's appearance/equipment is a later step.

## Code
Loader source, Lua scripts and analysis helpers: https://github.com/Dagxs/kcd-coop (MIT, no game files).

## Session 2c: stand-in looks like Henry (ran, owner screenshots)
- Player script API found by dumping metatables: `player.actor:MakeLookAsActor(id)`, `:MakeLookAsSoul(guid)`, `EquipClothingPreset`, `EquipInventoryItem`; `player.soul:OverrideHead(guid)`, `:OverrideHair(guid)`, `GetId()`; `player.inventory:GetInventoryTable()`; `player.human:PlayAnim`, `SetAnimMotionParam`.
- A BasicEntity has no `.actor`, so the stand-in is now `System.SpawnEntity({class='NPC', ...})` (gets actor/soul/human).
- Henry's soul in libs/tables/rpg/soul.xml (Tables.pak) is `dude` 43144483-f3bb-fab8-9ceb-f77e3020598a, head `henry` 4b8b7289-41e9-2785-a0b9-1651465445a5, hair `hair_henry` 4bc51034-3ad5-0050-f5c1-ba11046ed4ba. (Tables.pak entries use backslash names; Python zipfile needs manual extraction.)
- MakeLookAsActor(player.id) alone copies clothes but leaves the default bald head; MakeLookAsSoul with the WUID userdata from soul:GetId() did nothing visible. MakeLookAsSoul(guid string) + OverrideHead + OverrideHair + MakeLookAsActor gives Henry's face, hair and shirt.
- `AI.SetBehaviorTreeEvaluationEnabled(id, false)` switches the NPC's brain off (it wandered before).
- Owner screenshot: both copies show a Henry stand-in.
- Loader v3.2 (kcd-coop main): lua/coop_core.lua embedded into the DLL at build time; after a full restart both copies show the Henry NPC stand-in immediately and it idles/walks/runs (owner screenshot + "he runs properly"). Remaining look gap: stand-in wears the local player's clothes (shirt, braies, barefoot here), not the remote player's gear.

## Gear sync research (in progress, 2026-10-25)
Observations (ran):
- `player.inventory:GetInventoryTable()` -> item WUIDs; `ItemManager.GetItem(wuid)` -> {class (guid string), health, amount, entity, id}. `ItemManager.GetItemName(class)` gives names (e.g. shirt_002_green, pants_005, boots_002, hood_004_down_ptacek). No equipped flag: the list is identical before and after equipping boots + shirt in the inventory UI. `entity` stays 0.
- Game Lua scripts (Scripts.pak, 495 files) only ever call EquipInventoryItem / EquipClothingPreset / UnequipInventoryItem; no equipped query exists in script.
- `player.actor.__this` / `player.human.__this` = entity id (0x7777), not a pointer; `player.inventory.__this` and `player.__this` are native pointers.
- Lua binding stubs: EquipInventoryItem -> `mov rax,[rcx]; jmp [rax+0x298]`, Unequip -> +0x2A0 (handlers 0xb1ab78 / 0xb1ab84, registered in 0x10b7800). The C_Player primary vtable (0x26c3e30) slots there are trivial getters, so `rcx` is some other object resolved by the binding template (0x10b056c); not traced yet.
- `MakeLookAsActor` copies visuals only: the target's inventory stays empty.
- Relevant RTTI classes: wh::entitymodule::C_ActorEquipmentHandler, C_EquipmentManager, I_EquipmentManagerListener, C_EquippableItemRuntimeData, C_ClothingAttachmentManagerBase, C_ClothingSystem. tools/rtti.py in kcd-coop lists classes and vtables (incl. secondary subobjects).
Next: trace the binding template 0x10b056c to the object that owns equip (likely C_ActorEquipmentHandler), find its equipped-items container or register an I_EquipmentManagerListener, then send worn item classes and dress the stand-in with inventory:CreateItem + actor:EquipInventoryItem.

## Gear sync working (ran 2026-10-25 21:54, owner screenshot + logs)
- Script handler for actor:EquipInventoryItem is C_ScriptBindActor vtable (subobject +0) slot +0x298 = 0x10c52cc (the 0xb1ab78 stub is a virtual-member thunk on the binder, not the actor). It resolves the actor, takes `[actor+0x650]->vt[+0x268]()` = equipment handler, and calls the internal equip routine 0x44d6c8(handler, itemWuid). Unequip: 0x10e9a7c -> 0x44c7bc.
- Clothing (item type 0x100f) is kept in an MSVC std::unordered_set<WUID> at handler+0x10 (list head ptr at +0x18, size at +0x20, nodes {next, prev, wuid}); equip inserts, unequip erases (0x450ef4).
- Loader v3.4 inline-hooks 0x44d6c8 (17-byte prologue verified, relocatable) and finds the player's handler by having Lua equip a keyring as a probe and matching the WUID. Worn WUIDs (hex) -> Lua matches `tostring(wuid)` from the inventory table -> item class GUIDs -> `KCDCOOP1 G <classes>` every 2 s (filtered to GUID characters on receipt).
- Receiver: reset stand-in to Henry (MakeLookAsSoul + head/hair overrides), `inventory:CreateItem(class, 1, 1)` (returns nothing: new item found by diffing GetInventoryTable), `actor:EquipInventoryItem(id)`.
- Result: stand-in wears the other player's green shirt, red hood, trousers, boots; live change (shirt removed) re-dressed within ~2 s, both directions.
- Lesson: device file copies of a same-size DLL were silently skipped once; verify SHA-256 of deployed files after every copy.

## Weapons (ran 2026-10-25 22:01, loader v3.5 / core v4)
- Weapon slots: 12 WUIDs at equipment handler +0x50 (counter at +0xB0); empty slot value = global at RVA 0x300F8D8. Included in the gear list -> stand-in gets the same weapon items equipped.
- Drawn flag: `player.human:IsWeaponDrawn()` appended to the position packet (`P x y z rot drawn`); remote calls `e.human:DrawWeapon()` / `HolsterWeapon()` on the stand-in (brain off still works) and skips locomotion anims while drawn.
- Result: prologue training sword (class e5fc1a89-9bb1-44a9-a524-c6834a5e2e76) appeared in A's gear list; B logged repeated `remote weapon drawn/holstered ok=true` as A drew and sheathed; owner screenshot shows the stand-in holding a sword. Combat moves are not mirrored yet.

## Next steps (updated)
1. Mirror attacks/blocks (combat animations), then shared enemies with host authority.
2. Steam relay for internet play.
3. Automated QA bot (playbook/automated-qa.md); current QA level is manual.

## Attack mirroring research (2026-10-25 22:20, not working yet)
Observations (ran):
- `player.actor:GetCurrentAnimationState()` returns a coarse state string; while fighting it cycles `CombatAttack`, `CombatIdle`, `CombatMovement`, and `MotionMovement` when out of combat. Blocking did not show as its own state in a 20 s capture.
- Playable (GetAnimationLength > 0) loose animations matching attack/block/parry on the male skeleton are only block blendspaces (`1d-free-block_*_shld`, `1d-combat_lg_block_nw`, `1d-combat_rg_block_nw`, `freeblock/*`), `aimposes_free_attacks` (an aim pose) and cutscene .caf files. Directional attacks are not loose files (inside .dba databases / Mannequin fragments).
- Game scripts never play combat fragments; `human:PlayAnim` is unused in Scripts.pak.
Derived:
- Mirroring attacks needs the native combat module (RTTI wh::combatmodule::C_CombatPlayer etc.): find the routine that starts an attack for an actor and call it on the stand-in (with its brain off), or drive the stand-in's Mannequin action controller.
Lua jobs: kcd-coop lua/watch_anim_state.lua, lua/find_attack_anims.lua.

## Combat mirroring v1 (ran 2026-10-26, loader v3.6 / core v5)
- Animations/Mannequin/ADB/*.adb are in Animations-part1.pak (351 MB, stageable). kcd_male_database.adb has 612 fragment IDs; CombatAttack fragments map tags like `rightGuard+eZ3+aZ0+slash+attack_heavy` -> `1d-combat_rg_z0_natksw_lngsw`; CombatIdle -> `combat_{lg|rg}_z{0-5}_idle01_lngsw`; CombatBlockSlash -> `1d-combat_{rg|lg}_z{0-4}_nblksw_lngsw`; CombatPreBlock -> `combat_*_nblksw01_lngsw_over`.
- Position packet now carries combat state from `player.actor:GetCurrentAnimationState()` (CombatIdle=1, CombatAttack=2, CombatMovement=3). Remote with weapon drawn: loops the combat stance; on each new CombatAttack plays an attack (alternating guard, random zone 0-4) for 0.9 s.
- Free swings with no opponent stay MotionIdle/MotionMovement, so only real fights are mirrored.
- Result: owner saw stance and swings on the stand-in in a random encounter; "a little finicky".
- Test helper: `player.inventory:CreateItem('e5fc1a89-9bb1-44a9-a524-c6834a5e2e76', 1, 1)` gives the training sword (`player:CheatGiveSword()` errors in this build).
- Lesson: the device file transfer reuses stale content when the same staged file name is committed again; always commit from a fresh file name and verify.
Next: real attack direction/zone and blocks from the combat module (C_CombatActorActionAttack etc.), smoother transitions; then shared enemies.
