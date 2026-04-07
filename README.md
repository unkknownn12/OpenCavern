# OpenCavern
> The First Spelunker World/みんなでスペランカーZ server revival

---

## What is this?

Minna de Spelunker Z (PCSG00581) was a free-to-play cooperative platformer for PS Vita. It was fun. It had servers. Then it didn't. Square Enix pulled the plug, the game became unplayable, and everyone moved on.

Everyone except me.

OpenCavern is an open-source effort to reverse engineer the game's server communication and build a lightweight replacement server so people can actually play again — via the [Vita3K](https://vita3k.org/) emulator, no hardware required.

This repo contains:
- Documentation of the reverse engineering process
- The patch tooling to make the game talk to a local server instead of dead PSN infrastructure
- Eventually: the server itself

**No game files are included or will ever be included. You need to legally obtain the game yourself.**

---

## Current Status

 **Work in progress — title screen loads, main menu blocked**

The game boots, renders, plays audio at a stable 30 FPS in Vita3K. I've successfully bypassed multiple dead PSN service checks through IL patching of `Assembly-CSharp.dll`. The remaining blocker is deep in the native `UnityPrxForPS4.suprx` PSN initialization layer below the C# code, in ARM native territory. Ghidra time.

### What works
- Game boots and loads title screen fully
- Audio, graphics, controller input all functional
- All C# PSN service checks bypassed

### What's blocked
- `sceNpBasicInit` is unimplemented in Vita3K — the native PSN auth thread fires after the network check dialog and then hangs for ~24 seconds before the asset streaming loop restarts
- The game never transitions from title screen to main menu
- Root cause is in `UnityPrxForPS4.suprx`, currently being analyzed in Ghidra

---

## Technical Deep Dive

### Stack
- **Game**: PCSG00581 v1.15 (JP), Unity 4.37, Mono scripting
- **Emulator**: Vita3K v0.2.1 (build 3946)
- **Patching**: dnSpy IL editing of `Assembly-CSharp.dll`
- **Native RE**: Ghidra (ARM Vita SUPRX)

### Patches Applied

All patches are applied to `Assembly-CSharp.dll` via dnSpy **Edit Method Body** (IL mode only — using Edit Method C# caused Mono vtable crashes).

**Save with all MD token preservation options checked** or the game will crash on load.

| Class | Method | Patch | Reason |
|-------|--------|-------|--------|
| `clsNpLibraryDll` | `IsBusy()` | → `return false` | Blocked main game loop |
| `clsNpLibraryDll` | `IsBusyWebApi()` | → `return false` | Blocked web API flow |
| `clsNpLibraryDll` | `IsBusyMainMenu()` | → `return false` | Blocked main menu input |
| `clsNpLibraryDll` | `IsBusyMatching()` | → `return false` | Blocked matchmaking flow |
| `clsNpLibraryDll` | `IsBusyCommerce()` | → `return false` | Blocked commerce flow |
| `clsNpManagerObject` | `CheckClosedService()` | → `return false` | Game detected service as closed and killed all input |
| `clsSequenceManager` | `set_EnableWaitLoading` | always stores `false` | Prevented WAIT_LOADING state from ever engaging |
| `clsSequenceManager` | `ChangeSequende()` | `bEnableWaitLoading` arg → `ldc.i4.0` | Same — belt and suspenders |

### dnSpy Patching Method

1. Open `Assembly-CSharp.dll` from `ux0:/app/PCSG00581/Media/Managed/`
2. Find the method in the left panel tree
3. Right click → **Edit Method Body** (NOT Edit Method C#)
4. Modify opcodes directly in IL view
5. File → Save Module → MD Writer Options tab:
   - Check **ALL** boxes under "Preserve All MD Tokens"
   - Check `#Strings`, `#US`, `#Blob` under "Preserve Heap Offsets"
6. Copy patched DLL back to the game directory

### Key Findings

**`clsLoginFlow.Update()`** gates all game state transitions on `clsNpLibraryDll.IsBusy()`  this was the first major hang, the game was just sitting there waiting for PSN to respond forever.

**`clsNpManagerObject.CheckClosedService()`** called `clsNpLibraryDll.IsClosedService()` which returned true (because the servers ARE closed), then triggered `OnErrorGotoError(eErrorEndOfService)` — this killed all input processing in `clsUICtrlMainMenu.Update()` via an early return.

**`clsSequenceManager` WAIT_LOADING state** — the sequence manager has a state machine that enters `WAIT_LOADING` and stays there until `m_bEnableWaitLoading` goes false. This flag was getting set via `ChangeSequende()` and never cleared because the clearing code lives inside PSN callback handlers that never fire on a dead server.

**The remaining hang** — after the network check dialog completes (`sceNetCheckDialogGetStatus → SCE_COMMON_DIALOG_STATUS_FINISHED`), the game opens `resources.assets.resS` in a streaming loop every ~200ms. After ~24 seconds the loop restarts. This timing matches an internal timeout in the native PSN auth flow inside `UnityPrxForPS4.suprx`. `sceNpBasicInit` is listed as unimplemented in Vita3K's log, and the auth thread appears to block waiting for a callback that never arrives.

### Next Steps

- [ ] Analyze `UnityPrxForPS4.suprx` in Ghidra — find the PSN init/auth callback chain
- [ ] Identify what signal unblocks the auth thread and patch/stub it
- [ ] Once past title screen: capture actual server API calls with a proxy
- [ ] Build minimal Python stub server matching the API shape
- [ ] Test offline single-player mode end to end
- [ ] Eventually: proper multiplayer server

---

## Legal

This project contains **no game files, no copyrighted assets, and no proprietary code**. It is a clean-room reverse engineering effort producing only patch tooling and an independent server implementation.

The game belongs to Tozai Inc. and Square Enix. Go tell them to officially re-release it — I'd happily shut this down for an official server.

Legally obtain the game before using any of this. I won't tell you how. :)

---

## Contributing

If you know Vita ARM reversing, Unity 4 internals, or PSN protocol analysis, pull requests and issues are very welcome. This is a small project and extra eyes are genuinely useful.
(cough cough)... (me only)...
---

## Prior Art

None. As far as I can tell, nobody has attempted this before. Every wall I hit is undocumented. Every patch in this README is original work produced from scratch.

That's kind of the point.

I've spent too long on this.

I will beat that damn alligator 
---

*OpenCavern is not affiliated with Tozai, Square Enix, or Sony Interactive Entertainment.*
