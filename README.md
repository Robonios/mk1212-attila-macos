# MK1212 on the Feral macOS port: investigation record

A record of what it took to run the Medieval Kingdoms 1212 AD mod on Feral Interactive's macOS port of Total War: ATTILA, and of why ten building slots have no clean fix on that port.

**This is not a launcher.** The launcher is [ausxen's MK1212 macOS Launcher](https://github.com/ausxen/MK1212_macOS) — start there if you want to play the mod. This repository holds findings, a patch of local changes against that launcher, and the static analysis behind them.

This is independent work. It is not affiliated with, endorsed by, or supported by Feral Interactive, SEGA, Creative Assembly, or the MK1212 team.

### Contents

| Path | What it is |
|---|---|
| `patches/mk1212-mac-fixes.patch` | Local changes against ausxen's launcher at commit `c897ffe` |
| `analysis/` | The disassembly excerpts the ten-slot findings rest on |

---

Status as of 28 September 2026. This record covers the work from 13 to 28 September 2026 to run the Medieval Kingdoms 1212 AD mod (MK1212) on Feral Interactive's macOS port of Total War: ATTILA.

## Summary

- **MK1212 runs.** All 13 packs load through ausxen's launcher with local fixes, and the campaign scripts run: the Holy Roman Empire, population, papal favour, crusades, decisions, and the rest of the mechanics registry.
- **Ten-slot settlements work but degrade input, and the cause is now known.** A runtime patch gives ten building slots. After roughly 5 to 10 minutes of play, clicks stop taking effect, and later the camera zooms on its own. The trigger is modifying one of the game's signed code pages in memory, not the slot value or ten slots as such: a control library that loads into the game and rewrites a page but only at instructions that never execute reproduces the degradation, while an otherwise identical library that writes nothing runs clean for 27 minutes. Both routes that change the value without writing a code page were unavailable: rewriting the page is what causes the fault, and a hardware-breakpoint approach was out of scope. So the port has no accepted ten-slot fix, and six slots is the only stable configuration.
- **Buffer-state release is broken for 47 regions.** The spear units for six Mediterranean factions don't exist in the game data. This is an MK1212 bug on every platform.
- **Some papal features are absent by design.** MK1212 disables the College of Cardinals, papal missions, and the papal election panel in its own source.

## Environment

| Item | Value |
|---|---|
| Mac | MacBook Pro 13-inch, M1 (`MacBookPro17,1`), 8 GB unified memory |
| macOS | 26.6.2 (`25G83`) |
| Game | Total War: ATTILA, Feral port 1.6.1, build `480285.103778`, released 2 July 2026 |
| Game executable | SHA-256 `13f5d523…`, Mach-O UUID `392D6F66-9183-329E-8A98-8C90FC828326` |
| Python | 3.9.6 at `/usr/bin/python3`, from the Xcode Command Line Tools |
| Launcher | ausxen's MK1212 macOS Launcher, local clone at `~/mk1212-review`, based on commit `c897ffe` |

### Mod packs

| Pack file | Workshop ID | Content |
|---|---|---|
| `1-1212scripts.pack` | 1934544571 | Campaign scripts |
| `Custom cities beta.pack` | 3010246623 | Settlements, replacing the old siege map replacer |
| `1212compbuild_v2.pack` | 1429109380 | Base Pack, Campaign Alpha |
| `1212models1_v2.pack` | 1429140619 | Models 1 |
| `1212models2.pack` to `1212models9.pack` | 1371434091, 1371420895, 1371491650, 1592154821, 1934591700, 2221976170, 2660365008, 3003589041 | Models 2 to 9 |
| `1212music.pack` | 1582067661 | Music |

## Current configuration

### Launching the game

```
cd ~/mk1212-review
python3 src/mk1212_mac_tool.py launch
```

Keep the Terminal window open for the whole session. The launcher supervises the game and removes its temporary files when the game exits.

### Local changes to the launcher

The fork differs from upstream in the ways listed here. The file `~/mk1212-mac-fixes.patch` records them; refresh it after any change.

1. **Lowercase shim names.** The launcher named converted packs with the original capitalisation. Feral compares filenames in lower case, so it rejected `Custom cities beta` as missing. The fork lowercases every generated name.
2. **Isolation accepts a real `VFS/Local` directory.** Upstream only accepts a symlink there. The fork also accepts a real directory, which the isolation fix needs.
3. **Running-game check** also matches the name of a patched app copy. This is left over from the on-disk patch experiment and does no harm.
4. **Four local Lua repairs.** The launcher applies six in all; the other two are upstream's and are listed below for completeness.
   - `mk1212_slots.lua`: upstream's 0.7.0 replacement, which removes the Windows helper dialog and its executable dropper. Upstream `c897ffe` already repairs this file, and the fork replaces that repair with the newer one.
   - `frontend_disclaimer.lua`: the second executable dropper neutralised. Upstream `c897ffe` doesn't touch this file.
   - `mechanics_buffer_states.lua` and `mk1212_common.lua`: diagnostic tracing for buffer-state release, written to the mod's debug log with the prefixes `BUFFER:` and `TRANSFER:`.

   The two remaining repairs are **not** local changes: `frontend_scripted.lua` gets two interface geometry repairs that are already present in upstream `c897ffe`, so they don't appear in `~/mk1212-mac-fixes.patch`.
5. **Lua validation before install.** Every repaired file must compile under a real Lua 5.1 runtime (the `lupa` package), pass a check for line breaks inside quoted strings, and parse under `luaparser`. Any failure stops the install.
6. **Runtime ten-slot patch.** A small library patches the game in memory at launch. The launcher refuses to treat a session as valid unless the patch log reads `status=patched-6-to-10 kern_return=0`.

### Isolation

The launcher runs the game with a private copy of Feral's preferences, in `~/Library/Application Support/MK1212 Mac Launcher/runtime-home`. Inside it, `…/VFS/Local` is a real directory holding an empty `mods/` directory at mode 555. This stops Feral from linking the original Workshop packs into the session, so only the launcher's 18 converted packs load.

A full rebuild deletes this directory. After any rebuild, re-apply it.

## Problems solved

### Feral's mod manager doesn't control what the game opens

**Symptom:** load order changes in the launcher had no effect, and packs appeared to load in an unexpected order.

**Cause:** at launch, Feral's launcher symlinks every subscribed Workshop pack into `VFS/Local/mods`, and the game opens every one of them, whatever its ticked state. It opens them in alphabetical order of filename. The launcher also saves its settings only when it quits, so changes made immediately before pressing **Play** don't reach that session.

**Resolution:** superseded by ausxen's launcher, which builds its own set of packs and controls the order itself.

### Mod scripts never ran

**Symptom:** the mod's campaign mechanics were missing entirely.

**Cause:** Feral's port refuses any file under `data/` that isn't listed in `TotalWarAttilaData/feral/en/manifest.txt`. Lua inside mod-type packs never executes, and the scripts' relative search paths only work when the game starts in `TotalWarAttilaData`.

**Resolution:** ausxen's launcher converts each pack into a movie-type pack, extracts the 155 Lua files as loose files, adds them to the manifest for the length of the session, and starts the game in the right directory.

### The original packs loaded alongside the converted ones

**Symptom:** 31 packs open in each session: the 18 converted packs plus the 13 originals.

**Cause:** the launcher's private preferences mark every pack as disabled, but Feral links the originals into the shared `VFS/Local` regardless.

**Resolution:** the read-only `mods/` directory described in [Isolation](#isolation). Sessions then open only the 18 converted packs.

### A Workshop update locks the launcher

**Symptom:** after Steam updated `1212models2.pack` on 19 September 2026, the launcher refused to launch, verify, or uninstall.

**Cause:** verification compares each pack with the version the launcher was built from, and uninstall runs verification first.

**Resolution:** delete the launcher's state directory and cache by hand, reinstall, and re-apply the isolation directory. See [After a Workshop update](#after-a-workshop-update).

### Two executable droppers

MK1212 ships a Windows helper, `MK1212_10slots.exe`, as a hexadecimal byte array in `lua_scripts/slots_binaries.lua`. Two scripts can write it to disk and run it: `mk1212_slots.lua`, when you click **OK** on the Increase Slots prompt, and `frontend_disclaimer.lua`, without any prompt when `MK1212_config.txt` contains `disclaimerAccepted = 1`. The launcher neutralised only the first. The fork neutralises both. On macOS the executable can't run, but the file would still be written.

### Features that look missing but aren't broken

- **College of Cardinals, papal missions, and the papal election panel** are commented out in MK1212's own scripts.
- **Population** has no panel. It appears through the **Province Effects** icons at the bottom left of the screen.
- **The HRE panel** didn't appear in an older save and works in new campaigns. Loaded saves don't pick up mechanics that initialise at campaign start.

### A syntax error silently disabled the mod

**Symptom:** from 27 to 28 September 2026 the campaign played as near-vanilla ATTILA, with no error anywhere.

**Cause:** a diagnostic added to `mk1212_slots.lua` contained line breaks inside quoted strings, a syntax error in Lua 5.1. The game loads that file with an unprotected `require`, so the failure stopped every script loaded after it.

**Resolution:** fixed, and the three-layer Lua validation added so that no future repair can install with a syntax error. A real Lua 5.1 compile rejects the broken file with the same error the game hit.

## Ten-slot settlements

### How the limit works

- MK1212's settlement panel layout defines ten building slots, and the base game's `units_panel` layout also has ten.
- The game's campaign model already supports more than six slots. The AI builds eight or more in its own settlements, and buildings constructed in slots 7 to 10 keep working after the patch is removed. They only disappear from the interface.
- The six-slot limit exists only in the game executable, as two instructions that store the value 6 into an object at offset `+0x74`. The pattern `strb w8,[x0,#0x70]; mov w8,#6; str w8,[x0,#0x74]` occurs exactly twice in the 100 MB binary, at file offsets `0x03457244` (site 1) and `0x034577F4` (site 2), both near the interface selection code.
- On Windows, MK1212's helper scans the running game for one byte pattern and writes 10 at the first match only.

### Attempt 1: patching the executable file

The launcher's `native-patch` command copies the app, changes both instructions to `mov w8,#10`, and re-signs the copy. The copy crashed at startup every time, before Steam initialised.

Four controlled runs isolated the cause:

| Copy | Instructions | Name | Signature | Result |
|---|---|---|---|---|
| Original app | unpatched | original | Feral Developer ID | runs |
| Copy | patched | changed | ad-hoc, 10 permissions | crash |
| Copy | patched | changed | ad-hoc, 9 permissions | crash |
| Copy | **unpatched** | changed | ad-hoc | crash |
| Copy | unpatched | **original** | ad-hoc | crash |

The app refuses to start under any signature except Feral's own. The patch itself was never at fault.

### Attempt 2: patching in memory

The launcher's upstream version 0.7.0 patches the running game instead. A small library loads into Feral's original, signed executable through `DYLD_INSERT_LIBRARIES` and rewrites the instruction in memory before the game starts. Feral's own permissions allow this: the executable carries `allow-dyld-environment-variables`, `disable-library-validation`, and `allow-unsigned-executable-memory`. The file on disk and its signature never change.

This works. Settlements show ten slots, and buildings in slots 7 to 10 build and function.

### The input problem

With the patch applied, input degrades during play:

1. Clicks show their animation but don't take effect, at first for a few seconds at a time.
2. The degradation worsens.
3. The camera starts zooming in and out on its own.
4. The game has to be force quit, or crashes.

Onset came about 5 minutes into a large England campaign and about 10 minutes into a new Castile campaign.

### What's established

The tests below narrow the cause step by step. Read together with the two findings that follow the table, they place the trigger on the act of modifying a signed code page.

| Test | Result |
|---|---|
| Same save, injection on and off (26 September 2026) | On: degrades. Off: stays responsive |
| New campaign, patch on (28 September 2026) | Degrades |
| Site 1 only, ten slots | Degrades |
| Site 2 only, six slots | Degrades |
| Both sites | Ten slots, degrades |
| Patch applied mid-session instead of at launch | Degrades |
| Upstream's replacement slots script instead of the original | Degrades |
| Library loaded, guards run, nothing written (28 September 2026) | **Six slots, ran clean for 27 minutes** |
| Memory at about 9 minutes | 4.1 GB patched, 4.2 GB unpatched |
| Thread activity | Identical patched and unpatched; no thread spinning |
| Main thread | Waiting in the same nested event loop in both |

Two further findings settle what the value has to do with it: nothing.

- **Site 2 is dead code.** Static analysis of the function-start table found that site 2 sits in a 20-byte function nothing calls, branches to, or takes the address of. It's an out-of-line copy of a method that was also inlined at site 1. Writing to site 2 changes no running behaviour, so the site-2-only session ran with six slots yet still degraded. The write, at inert bytes, was enough on its own.
- **The inject-only control ran clean.** A library injected exactly like the patch, running the same guards but never changing memory protection and never writing, played for 27 minutes with no degradation. Removing the memory write is the one change that removed the fault.

The value stored, and ten slots as such, are therefore not the cause. Modifying a signed code page in memory is, even when the modification is inert. Feral's executable is code-signed, and rewriting one of its pages appears to move the process into a state macOS treats differently. The exact mechanism isn't identified. This also explains why the degradation has no measurable cost in memory, CPU, or thread activity, and why the Windows helper is unaffected: Windows doesn't police code pages the same way.

### What `+0x74` actually controls

Static analysis identified the patched object as `EMPIRECAMPAIGN::CampaignSettlementCallback`, a polymorphic class. Its `+0x74` field is the number of `building_slot_N` widgets the capital's settlement panel fills in and refreshes. The constructor defaults it to 4; site 1 raises it to 6 for provincial capitals, matching vanilla ATTILA's 4-slot minor settlements and 6-slot capitals. The field is read only as a loop bound over interface widgets. It never sizes memory, indexes an array, or reaches an allocator, which is consistent with the patch never appearing near any crash and with the degradation having no memory cost.

### Routes to a fix, and why they're closed

- **Rewrite the code page** (the current patch): works, but is the cause of the degradation.
- **Hardware breakpoint:** set an execution breakpoint at site 1 and change the register value on each hit, writing no game memory. This needs a library that drives the game's own threads through the debug registers and an exception port, which is a debugger in all but name and was ruled out of scope.
- **Rewrite the executable file:** rejected by code signing, established by four controlled runs (see [Attempt 1](#attempt-1-patching-the-executable-file)).

No route changes the slot count without either modifying a code page or attaching a debugger, so the port has no accepted ten-slot fix.

### Why the interface couldn't be measured directly

An earlier theory was that something in the interface accumulates while the value is 10. Three attempts to count interface components from the mod's Lua each crashed the game within seconds of their first run, through the Lua interpreter into engine interface code (fault offsets `+0x214f348` twice, then `+0x1b9c864`). Each walk also saw only about 21 components, far fewer than a live interface holds. Walking the interface from Lua isn't safe in this engine. The inject-only result later made this line moot: the cause is the code-page write, not anything the interface does with the larger value.

## Buffer states

**Symptom:** the release button appears for some regions but nothing happens when clicked, and some regions never offer a release.

**Findings:**

- **Missing units.** Releasing a faction spawns four of its spear units. For 11 of the 93 factions in the release table, that unit doesn't exist in the game data, so the army silently fails to spawn and the release never completes. This affects 47 of the 186 regions. The six in the Mediterranean:

  | Unit key | Factions affected |
  |---|---|
  | `mk_alm_t1_mushud` | Almohads, Granada |
  | `mk_ara_t1_spearmen` | Aragon |
  | `mk_byz_t1_militia_spearmen` | Nicaea, Epirus, Trebizond, Achaea, Thessalonica |
  | `mk_haf_t1_mashshain` | Hafsids |
  | `mk_mar_t1_moroccan_mushud` | Marinids |
  | `mk_por_t1_spearmen` | Portugal |

- **Wrong region on transfer.** The script hands over the region on a timer, and reads whichever region is selected when the timer fires. Selecting a different settlement in between transfers the wrong one.
- **No release for the Principality of Antioch.** The faction has release assets, but no region maps to it. Antioch releases the Ayyubids, and only while they're dead.

**Mediterranean releases that work:** Sicily, Castile, Croatia, Genoa, Hungary, Pisa, Venice, Bologna, Dauphiné, Milan, Navarre, the Papacy, Provence, Savoy, Toulouse, Verona, Bulgaria, Serbia, the Ayyubids, the Seljuks, the Zengids, and Armenia, each only while that faction is dead.

## Saves

- **`Kingdom of England Trade.save` and `auto_save.save`:** written while the mod's scripts weren't running, between 27 and 28 September 2026. The HRE state regressed from reform 1 to reform 0, so the mod's saved state was lost. Don't play on these.
- **`Kingdom of England Battle.save`:** written 26 September 2026, while the scripts worked. The last good England save, though it also carries buildings from ten-slot sessions.

## Open questions and next steps

The ten-slot cause is now understood (see [What's established](#whats-established)), and the two routes that might change the value without writing a code page are both closed on this setup: the hardware-breakpoint approach is out of scope, and rewriting the file is rejected by code signing. What remains is a question for someone with more macOS code-signing knowledge, and the reports.

1. **The code-signing question for the author.** Is there an entitlement or signing arrangement under which an injected library can modify a signed code page without moving the process into the state that degrades input? If not, ten slots has no clean fix on this port, and six slots stands as the stable configuration.
2. **Report the findings** to the launcher's author and the MK1212 team.

## Findings to report

### To the launcher's author (ausxen)

- Shim filenames keep capital letters, and Feral rejects them. Lowercasing fixes it.
- Feral links the original Workshop packs into the shared `VFS/Local`, so they load alongside the converted packs. A read-only `mods/` in the private home prevents it. The launcher should create that directory itself, since a rebuild deletes it.
- A Workshop update blocks uninstall, so the README's recovery advice doesn't work.
- Version 0.6.1 left the second executable dropper in `frontend_disclaimer.lua` unpatched. Version 0.7.0 replaces that file.
- Version 0.6.1's `bin/` scripts point at a path that doesn't exist.
- The on-disk patch fails purely on signature, established by four controlled runs.
- **Version 0.7.0's input degradation is caused by the in-memory code-page write itself rather than by ten slots.** An inject-only control that runs every guard but writes nothing played clean for 27 minutes, while a write to dead code (site 2, confirmed unreferenced) degraded input the same way the real patch does. The `+0x74` value is only a widget-loop bound and never sizes memory. The likely reading is that rewriting a signed code page moves the process into a state macOS handles differently; the exact mechanism is open. If there's a code-signing or entitlement angle that lets the page be modified cleanly, that would be the fix; otherwise the remaining route is a hardware breakpoint that writes no game memory.

### To the MK1212 team

- Six Mediterranean spear-unit keys, and five elsewhere, don't exist, which breaks buffer-state release in 47 regions.
- The buffer-state timer reads the global selected region when it fires, so a changed selection transfers the wrong region.
- The Principality of Antioch has release assets but no region.
- The HRE debug log writes to a hard-coded Windows path, which becomes a single oddly named file on macOS.
- Several files are written into the game's data directory and never cleaned up.

## Operating notes

### After a Workshop update

If a launch fails with `Workshop sources changed; rebuild required`, have Claude Code:

1. Confirm the game isn't running and the live tree is clean: no converted packs in `data/`, and `manifest.txt` matching `~/mk1212-manifest-original.txt`.
2. Delete `~/Library/Application Support/MK1212 Mac Launcher` and `TotalWarAttilaData/.mk1212-cache`.
3. Confirm the local changes are still in `src/mk1212_mac_tool.py`, or reapply them from `~/mk1212-mac-fixes.patch`.
4. Run `python3 src/mk1212_mac_tool.py install`.
5. Re-apply the isolation directory.

### Memory

The Mac has 8 GB of memory, and a long session reaches a footprint of about 4.2 GB. Some crashes late in long sessions came from failed memory allocations. These are separate from the input degradation, which occurs at the same memory use as unpatched play.

## Reference

| Item | Value |
|---|---|
| Patch site 1 | File offset `0x03457244`, inside the function at `0x103456db8` that builds the settlement panel |
| Patch site 2 | File offset `0x034577F4`, a 20-byte function at `0x1034577EC` that nothing references (dead code) |
| Patched object | `EMPIRECAMPAIGN::CampaignSettlementCallback`; `+0x74` is the capital's building-slot widget count, default 4, raised to 6 for capitals |
| Bytes before the patch | `08 c0 01 39  c8 00 80 52  08 74 00 b9` |
| Patched instruction | `48 01 80 52` (`mov w8,#10`) |
| Patch library in use | Load-time, site 1 only, SHA-256 `368739c2…` |
| Original manifest | SHA-256 `c65f9103…`, backed up at `~/mk1212-manifest-original.txt` |
| Crash on startup, on-disk patch | `+0x682a8` |
| Crashes from interface walking | `+0x214f348`, `+0x1b9c864` |
