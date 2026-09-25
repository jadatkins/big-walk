# Big Walk on macOS: BepInEx fixes and working mod setup

**Date:** 2026-09-25  
**Outcome:** playable through Steam, with Third Person, Mod Settings Menu, and Big Visuals confirmed working by the user. BigSpitty also worked and was then removed because it was only a test mod.

This report records the setup changes, native and managed code fixes, diagnostic evidence, and the final installation. Earlier entries in `README.md` describe intermediate failures; the outcome here supersedes their pending-test status.

**Packaging note:** this report describes the development and verification snapshot.
The folder was subsequently cleaned for sharing: Third Person and Big Visuals
and their configs were removed, leaving Mod Settings Menu and its localization
dependency enabled. The diagnostic probe remains disabled. The old upstream
`changelog.txt` and launch logs were removed. The current user-facing instructions
are in [README.md](README.md), with debugging guidance in
[TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## 1. Environment and final state

| Component | Tested version / state |
|---|---|
| Hardware | Apple M1 |
| Execution | macOS, x86_64 under Rosetta; BepInEx reports kernel `26.7.0` |
| Game | Big Walk, Steam AppID `1478500`; game log reports `1.5.1 2608271544` |
| Engine | Unity `6000.3.17f1`, IL2CPP metadata version 39 |
| BepInEx | `6.0.0-be.755`, commit `3fab71a1914132a1ce3a545caf3192da603f2258` |
| Game's managed runtime | Bundled x86_64 .NET `6.0.7` in `dotnet/` |
| Build SDK | Installed arm64 .NET SDK `8.0.424`; managed projects target .NET 6 |
| Doorstop | CI build at `d8973b2`, including the chained-fixups correction |
| Dobby | Local x86_64 build from `9d27180` plus uncommitted fixes; `DOBBY_DEBUG=ON` |
| Il2CppInterop source | Upstream master `81a6f78`, macOS port commit `3763627`, plus uncommitted native-hook corrections |
| Runtime assembly identity | Kept at `1.5.1.0` to match the installed BepInEx stack |

Game root:

```text
/Users/alexander/Library/Application Support/Steam/steamapps/common/Big Walk
```

All installed fixes are external to **`Big Walk.app`**, which was left intact. Game launches and gameplay verification were performed through Steam.

### Installed and verified mods

| Mod | Version | Installed location | Controls / result |
|---|---|---|---|
| Third Person / Big Person View | 1.4.0 | `BepInEx/plugins/ThirdPerson.dll` | `]` cycles views; `;` opens settings; camera operation confirmed |
| Mod Settings Menu | 1.1.2 | `BepInEx/plugins/ModSettingsMenu/` | Mod Settings in main/pause menu; user confirmed working |
| BigWalkLocalizationAPI | 1.1.0 | `BepInEx/plugins/BigWalkLocalizationAPI/` | Required dependency of Mod Settings Menu |
| Big Visuals | Package 1.0.1; plugin reports 1.0.0 | `BepInEx/plugins/BigVisuals/` | Graphics controls confirmed; current saved menu key is **P**, changed from default F11 |

Big Visuals actually generated **`BepInEx/config/Big_Visuals.cfg`**, not the longer filename advertised in its archive README. Third Person uses `BepInEx/config/deck.bigwalk.thirdperson.cfg`.

BigSpitty 1.0.0 successfully loaded and spat with **X** during gameplay. Its installed folder, including DLL and three WAV files, was subsequently moved to Trash at the user's request. The minimal diagnostic plugin remains disabled as `BepInEx/plugins/BigWalkInjectionProbe/BigWalkInjectionProbe.dll.disabled`.

## 2. Launcher, architecture, and Doorstop

### Problem

Big Walk is a universal macOS application, but the installed CoreCLR and Dobby libraries are x86_64. The launcher also has to preserve the DYLD injection variables across macOS platform binaries. Simply exporting those variables before invoking `arch` was insufficient because they were stripped.

The older Doorstop 4.5.0 release also could not correctly hook this Unity player's chained-fixup layout.

### Changes

Configured the external `run_bepinex.sh`:

```sh
executable_name="Big Walk.app"
archpreference="x86_64"
target_assembly="BepInEx/core/BepInEx.Unity.IL2CPP.dll"
coreclr_path="dotnet/libcoreclr"
corlib_dir="dotnet"
```

Installed the Doorstop CI dylib at commit `d8973b2`, incorporating chained-fixups fix `d22adc8` / UnityDoorstop PR #117.

The macOS execution path sets `ARCHPREFERENCE` and passes DYLD variables using `arch -e`:

```sh
export ARCHPREFERENCE="${archpreference}"
exec arch \
    -e DYLD_LIBRARY_PATH="${dyld_library_path}" \
    -e DYLD_INSERT_LIBRARIES="${dyld_insert_libraries}" \
    -e DYLD_PRINT_LIBRARIES=1 \
    "$executable_path" "$@" > "${BASEDIR}/BepInEx/native-launch.log" 2>&1
```

Steam launch option:

```text
"/Users/alexander/Library/Application Support/Steam/steamapps/common/Big Walk/run_bepinex.sh" %command%
```

An absolute path is required here; Steam does not expand `~` as a shell would. `sh -n run_bepinex.sh` passed. Gatekeeper assessments were enabled during successful loader startup; globally disabling Gatekeeper was not required.

## 3. External paths and interop-cache generation

### Problem

BepInEx's path assumptions did not directly match this app bundle. Separately, Cpp2IL failed while processing the universal Mach-O GameAssembly: its slice stream was read-only, but chained-fixup processing attempted to write to it.

The resulting error was:

```text
System.NotSupportedException: Stream does not support writing.
```

### Changes

Created these external symlinks:

```text
Data -> Big Walk.app/Contents/Resources/Data
GameAssembly.dylib -> Big Walk.app/Contents/Frameworks/GameAssembly.dylib
```

For **interop generation only**, temporarily replaced the root GameAssembly symlink with a thin x86_64 copy extracted from the bundled universal binary:

```sh
lipo "Big Walk.app/Contents/Frameworks/GameAssembly.dylib" \
    -thin x86_64 -output GameAssembly.dylib
```

That command describes the generation step, not a command to run over the current runtime symlink. Remove or move only the external symlink first, with the game closed, so the bundle cannot be overwritten.

Generation produced 157 managed assemblies in `BepInEx/interop/`; subsequent launches report preloading 154 interop assemblies. After generation, restored the root symlink to the bundled universal library for normal gameplay.

Set `[IL2CPP] UpdateInteropAssemblies = false` to preserve the generated cache while the universal-reader problem remains. `assembly-hash.txt` was adjusted from the thin binary's hash (`195e1a25eed1914cde2124d09fc8068c`) to the universal binary's hash (`2c03ec2e6a747e4916eb860aa07c8b35`) after restoration.

**After a game update:** regenerate the cache from the updated binary and metadata using the thin-copy procedure, then restore the runtime symlink. Changing the hash alone does not make an old cache compatible with a new game build.

## 4. Dobby: patches that actually execute under Rosetta

Repository: `~/Code/BepInEx/Dobby`  
Base: `9d271803881f467aabfc24593c7dcaeb710fe861`

### 4.1 Executable-memory writer

**Symptom:** the original library reported successful hooking and changed the bytes in memory, but Rosetta continued executing the old function. The simple test target still returned `1` instead of the hook's `99`.

Isolated experiments distinguished the paths:

- Remapping patched code alone: stale execution remained.
- Remap followed by read-only to executable protection transitions: the hook executed.
- Direct copy-on-write write followed by restored executable protection: the hook executed.

Implemented the direct-write path for x86_64 in `source/UserMode/ExecMemory/code-patch-tool-darwin.cc`:

1. Validate the address, buffer, size, and range arithmetic.
2. Query every covered page with `mach_vm_region` and save its original protections.
3. Make the pages writable using `VM_PROT_READ | VM_PROT_WRITE | VM_PROT_COPY`.
4. Write the replacement bytes in place.
5. Restore each page's original protections and clear the instruction cache.
6. Return failures; if making a later page writable fails, attempt to restore already-modified page protections.

The original non-x86_64 writer remains in the other compile-time branch.

### 4.2 Near-memory allocator

**Symptom:** allocation could skip a valid one-page gap, then overwrite an occupied page. During diagnosis, the relocated trampoline overwrote the nearby jump-pointer slot.

Changes:

- `source/MemoryAllocator/NearMemoryArena.cc`: corrected gap calculations, last-region selection, and range/space checks. Region ends were already aligned; advancing another page could skip the available gap.
- `source/UserMode/UnifiedInterface/platform-posix.cc`: on macOS, fixed-address reservations now use `vm_allocate(..., VM_FLAGS_FIXED)` followed by `vm_protect`, instead of an overwriting `mmap(..., MAP_FIXED)`. Occupied ranges cause allocation failure rather than replacing existing mappings.

### 4.3 Error propagation

Previously, important callers ignored code-patching failure. Changed:

- `source/InterceptRouting/InterceptRouting.h` and `.cpp`: `Active()` and `Commit()` return success/failure; mark an entry committed only after a successful patch.
- `source/InterceptRouting/Routing/FunctionInlineReplace/FunctionInlineReplaceExport.cc`: `DobbyCommit` rejects a missing hook entry and propagates commit failure.
- `source/dobby.cpp`: `DobbyDestroy` reports restore failure instead of removing the hook entry and reporting success.

### Verification

Standalone x86_64 tests passed for normal and page-boundary targets:

```text
Before hook:          1
After commit:        99
Original trampoline:  1
After destroy:        1
```

Null-address, duplicate-hook, and never-hooked destruction cases were also tested. In-game runtime-invoke callbacks then began executing, exposing the next managed blocker.

These tests did not deliberately force OS protection-restoration failures or establish concurrent patching safety. Other Dobby `CodePatch` callers were not comprehensively audited. This is a tested local fix, not a claim that every Dobby failure path was repaired.

## 5. Il2CppInterop: locating the actual macOS native image

Repository: `~/Code/BepInEx/Il2CppInterop`

### Problem

Once the runtime-invoke hook worked, class injection failed during `InjectorHelpers` initialization:

```text
System.TypeInitializationException: ... InjectorHelpers ...
System.InvalidOperationException: Sequence contains no matching element
```

The originally installed revision, `6d9007c18cc8440830379c5e1d5714085e7ec577` (`1.5.1-ci.829`), searched `Process.Modules` for Windows/Linux names and omitted `GameAssembly.dylib`.

Adding that filename alone was insufficient. A standalone x86_64 probe using the game's actual .NET 6.0.7 reported only the host executable in `Process.Modules`, with base address and size both zero.

### Changes

Added `Il2CppInterop.Runtime/NativeModule.cs`:

1. Obtain `il2cpp_init` from the native handle used for IL2CPP calls.
2. Use `dladdr` to identify the image containing that export.
3. Match the image in dyld and obtain its actual ASLR slide.
4. Parse the loaded 64-bit Mach-O header and `LC_SEGMENT_64` commands with bounds checks.
5. Expose only readable, executable segment ranges for signature scanning, excluding `__PAGEZERO`, data-only segments, and inter-segment gaps.

Updated `Injection/InjectorHelpers.cs` to initialize the native handle before discovering the image, and log the image name, base, and scan-range count.

Updated `MemoryUtils.cs` to scan the supplied ranges. The inner scan stops at `blockSize - mask.Length`, avoiding reads past the end when testing the last candidate position.

The probe passed on a fixture and the real GameAssembly, including end-of-range matches, undersized blocks, and a protected-page boundary test.

## 6. Moving the patch onto current master

The first module-discovery patch was deliberately built on the installed revision to minimize simultaneous changes. It was then preserved and ported:

| Branch / revision | Purpose |
|---|---|
| `preserve/bigwalk-macos-ci829`, commit `605aeaa` | Checkpoint of the original three-file macOS patch on `6d9007c` |
| Upstream `origin/master` at `81a6f78` | Verified remote master at the time of the port |
| `bigwalk/macos-latest`, commit `3763627` | Adapted cherry-pick of the macOS patch onto that master |

Resolved conflicts in `InjectorHelpers.cs` and `MemoryUtils.cs`. Retained upstream's Windows `VirtualQuery` scanning, which skips unreadable/guard regions, and its scan-boundary fix. Used our dyld/Mach-O discovery and executable-segment ranges on macOS. Kept upstream's case-insensitive Windows module-name matching.

Upgrading the source **did not by itself stop the pre-menu exits**. The next section describes the decisive hook corrections, made on this latest-master branch.

### Runtime and dependency compatibility

Master's default assembly identity was `1.5.3.0`; the installed BepInEx/Il2CppInterop stack used `1.5.1.0`. Built the newer runtime source with the existing identity:

```text
Version=1.5.1-ci.829
AssemblyVersion=1.5.1.0
FileVersion=1.5.1.0
```

These are compatibility labels, not an indication that the deployed source is still the old revision. Added the new dependency `TerraFX.Interop.Windows.dll` version `10.0.22621.2`, using its net6.0 build, to `BepInEx/core/`.

Only the Runtime DLL and that additional dependency were deployed for the managed upgrade. The installed Common, Generator, and HarmonySupport assemblies were retained. No different .NET SDK or game runtime was required: the arm64 .NET 8 SDK built net6.0 assemblies, and the bundled x86_64 .NET 6.0.7 executed them under Rosetta.

Big Visuals references Runtime `1.5.3.0`, while this build retains `1.5.1.0`. BepInEx's resolver matches loaded assemblies by simple name; the final launch successfully loaded the mod and applied its settings. This establishes compatibility for this tested combination, not arbitrary assembly-version substitutions.

## 7. Il2CppInterop: correcting macOS Unity 6000.3 injection hooks

### How the failure was isolated

Both Third Person and BigSpitty loaded and logged startup success, but the game exited before showing the menu. Removing all plugins initially did not help: Unity log forwarding also triggered class injection for `Il2CppToMonoDelegateReference`.

With zero plugins and `[Logging] UnityLogListening = false`, the game was playable. BigSpitty alone with forwarding disabled still caused the exit.

An otherwise empty diagnostic MonoBehaviour reproduced the problem even on the latest-master port. Its log showed successful registration, component creation, chainloader completion, **and its first Unity `Update`** before the process exited. Thus, initial registration and the first callback were not sufficient evidence of stable injection.

Steam recorded exit code `-1`; there was no new useful macOS crash backtrace identifying a single faulting hook. Native disassembly nevertheless demonstrated incompatible target selections and parameter layouts. Applying the following corrections together changed the diagnostic and real-mod tests from pre-menu exits to working gameplay.

### Corrected targets and native calling conventions

Offsets below are RVAs relative to this game's loaded GameAssembly image, not absolute addresses and not hardcoded discovery offsets.

| Hook | Incorrect interpretation | Corrected interpretation |
|---|---|---|
| `MetadataCache::GetTypeInfoFromTypeDefinitionIndex` | RVA `0x974050`, a helper taking an image pointer and an integer, hooked as `(int index)` | RVA `0x974130`, the index-to-class resolver taking `(int index)` |
| Field default value | RVA `0x951250`, a four-argument value-copy helper, hooked as `(FieldInfo*, Il2CppType**)` | RVA `0x9736F0`, the metadata field-default resolver taking `(FieldInfo*, Il2CppType**)` |
| `GenericMethod::GetMethod` | RVA `0x933170`, assumed to take three separate pointers | Same target, correctly hooked as `(Il2CppGenericMethod*)`, a pointer to a three-pointer structure |

Supporting observations:

- The incorrect metadata target dereferences `RDI` as a pointer and uses `ESI` as another input. The replacement compares `EDI` against `-1` and indexes a class-pointer cache with the integer index.
- The field-static-value path calls the Class default-value thunk at RVA `0x9909E0`, which tail-jumps to `0x9736F0`. That resolver saves the output-type pointer from `RSI` and writes through it. The old positional traversal instead selected the later value-copy operation.
- Both the generic virtual-method path and the three-pointer overload construct a three-pointer structure on the stack and pass its address in `RDI` to `0x933170`. Treating that address as a direct `MethodInfo*` misinterprets the input.
- Positional cross-reference traversal is compiler-sensitive: the existing scanner can continue after a tail jump into adjacent code, and a Windows/Linux call-order assumption does not reliably identify these macOS functions.

### Implementation

Added:

- `Il2CppInterop.Runtime/Injection/MacUnity6HookTargets.cs`
- `Il2CppInterop.Runtime/Injection/Hooks/GenericMethod_GetMethod_MacUnity6_Hook.cs`

Modified:

- `Il2CppInterop.Runtime/Injection/InjectorHelpers.cs`
- `Il2CppInterop.Runtime/Injection/Hooks/MetadataCache_GetTypeInfoFromTypeDefinitionIndex_Hook.cs`
- `Il2CppInterop.Runtime/Injection/Hooks/Class_GetFieldDefaultValue_Hook.cs`

The new discovery path applies only to **macOS + x86_64 + Unity major 6000, minor 3**. It searches executable segments for masked instruction signatures, masking relocation-dependent values. Each signature must match exactly once; absent or ambiguous matches raise an error instead of falling back to the old positional heuristic. All three targets are resolved before this injection setup applies them.

The generic-method hook uses a one-argument Cdecl delegate, reads `methodDefinition` and `context.method_inst` from the structure, and preserves the existing injected-generic-method lookup/inflation behavior. Ordinary methods forward to the original trampoline. Other platforms and Unity versions retain their prior selection paths.

### Verification

Standalone checks under the bundled x86_64 .NET 6.0.7:

- Uniquely resolved all three functions to the expected RVAs in the real GameAssembly.
- Accepted a unique fixture pattern and rejected missing/ambiguous ones.
- Called the real metadata resolver with index `-1`; it returned null before accessing initialized game metadata.
- Applied Dobby to that resolver; verified managed callback dispatch, original-trampoline forwarding, destruction, and the restored native call.

Experimental native runs used **two-second timeouts** to contain possible Rosetta hangs. Builds used longer timeouts.

The three corrections were tested together; the evidence does not isolate which individual mismatch caused every earlier exit. The final gameplay tests validate the corrected combination.

## 8. End-to-end verification sequence

| Stage | Result |
|---|---|
| Patched Dobby, original managed module discovery | Native runtime hook executed; `InjectorHelpers` initialization failed |
| Fixed module discovery, Third Person | Plugin loaded; game exited before menu |
| Fixed module discovery, BigSpitty | Plugin loaded; game exited before menu |
| No plugins, Unity log forwarding enabled | Injection still ran; game exited |
| No plugins, forwarding disabled | Playable baseline |
| BigSpitty, forwarding disabled, old hook selection | Exited before menu |
| Latest-master port, minimal injected component, old hook selection | First Update ran; exited before menu |
| Corrected hooks, minimal injected component | User confirmed playable gameplay |
| Corrected hooks, BigSpitty | User confirmed gameplay and working spitting |
| Corrected hooks, BigSpitty + Third Person | User confirmed camera mod worked too |
| Corrected hooks, Third Person + Mod Settings Menu + localization API + Big Visuals | User confirmed everything working; BigSpitty removed |

The final log reports four plugins loaded, Mod Settings Menu initialized, Third Person changing between Behind/Front/FirstPerson, and Big Visuals applying camera and render settings after the local player becomes available. It also shows subsequent live settings changes.

Remaining logged fallbacks include the Class::Init substitute and a Harmony patch-backend fallback for `SettingsMenu::.ctor()`. They were present in the user-confirmed working launch and were not separately fixed. Invalid Big Visuals menu-key warnings occurred while the setting was being edited; the final saved binding is `P`.

## 9. Diagnostics and retained configuration

Current deliberate settings:

```ini
[IL2CPP]
UpdateInteropAssemblies = false

[Logging]
UnityLogListening = false

[Logging.Disk]
LogLevels = All
```

`UnityLogListening = false` was an isolation measure retained in the working configuration. Re-enabling Unity log forwarding has not been separately verified after the final fixes. Unity still writes its own logs.

`DOBBY_DEBUG=ON` and `DYLD_PRINT_LIBRARIES=1` remain enabled as requested.

| Log | Purpose |
|---|---|
| `BepInEx/LogOutput.log` | Managed startup, selected hook addresses, plugin loading, mod output |
| `BepInEx/native-launch.log` | Launcher-captured stdout/stderr and dyld diagnostics; overwritten per launch |
| `~/Library/Logs/House House/Big Walk/Player.log` | Unity player output |
| `~/Library/Logs/House House/Big Walk/Player-prev.log` | Previous Unity launch |
| Same directory, `bigwalk-N.log` | Game's own log catcher |
| `~/Library/Application Support/Steam/logs/gameprocess_log.txt` | Launch PIDs and exit codes |
| Same Steam directory, `console_log.txt` | Launch command and process tracking |
| `Big Walk.app/Contents/MacOS/preloader_*.log` | Early preloader exceptions, when generated |
| `~/Library/Logs/DiagnosticReports/` | macOS crash reports, when generated |

We used `/usr/bin/log` for system-log queries; plain `log` was shadowed by a zsh function. Game-specific logs were moved to Trash between selected tests. In zsh, optional log globs need `(N)` and an empty-array guard to avoid an unmatched glob aborting cleanup.

Neither `Chainloader initialized`, `Chainloader startup complete`, nor a first Update alone proves that a mod works. Actual menu/gameplay and mod-specific controls were the acceptance checks.

## 10. Rebuilding the working components

These commands rebuild the **current local patched source trees**. Fresh checkouts at the committed revisions alone do not contain all final fixes.

### Dobby

Run in `~/Code/BepInEx/Dobby`:

```sh
cmake -S . -B build-rosetta-x86_64 \
    -DCMAKE_SYSTEM_NAME=Darwin \
    -DCMAKE_SYSTEM_PROCESSOR=x86_64 \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_OSX_ARCHITECTURES=x86_64 \
    -DDOBBY_GENERATE_SHARED=ON \
    -DDOBBY_DEBUG=ON
cmake --build build-rosetta-x86_64 --parallel 4 --target dobby
```

Output: `build-rosetta-x86_64/libdobby.dylib`.

### Il2CppInterop runtime

Run in `~/Code/BepInEx/Il2CppInterop` on `bigwalk/macos-latest`, retaining the local hook changes:

```sh
dotnet build Il2CppInterop.Runtime/Il2CppInterop.Runtime.csproj \
    -c Release \
    -p:Version=1.5.1-ci.829 \
    -p:AssemblyVersion=1.5.1.0 \
    -p:FileVersion=1.5.1.0 \
    -p:GeneratePackageOnBuild=false \
    --nologo -consoleLoggerParameters:ErrorsOnly
```

Output: `bin/Il2CppInterop.Runtime/net6.0/Il2CppInterop.Runtime.dll`.

TerraFX dependency source used:

```text
~/.nuget/packages/terrafx.interop.windows/10.0.22621.2/lib/net6.0/TerraFX.Interop.Windows.dll
```

With the game closed, deployment destinations are respectively:

```text
BepInEx/core/libdobby.dylib
BepInEx/core/Il2CppInterop.Runtime.dll
BepInEx/core/TerraFX.Interop.Windows.dll
```

### Source preservation status

- The first macOS module-discovery fix and its master port are committed as `605aeaa` and `3763627`.
- All seven Dobby source-file changes described above remain **uncommitted**.
- The final Il2CppInterop hook corrections remain **uncommitted**, including the two new source files. Saving only `git diff` would omit those untracked files.
- No fixes were pushed upstream during this work.

Temporary diagnostic sources and executables are under:

```text
/var/folders/5y/1mkpj0g11hz15y_f558f7nr40000gn/T/opencode/
  rosetta_patch_test.c
  dobby_api_test.c
  module_probe/                 # x86_64 host, discovery/scan/hook checks, disassembly
  assembly_refs/                # assembly identity/reference inspection
  bigwalk_injection_probe/      # minimal diagnostic plugin
```

These are temporary working artifacts, not a permanent test suite. `module_probe` includes a native CoreCLR host to exercise the game's x86_64 runtime rather than the machine's arm64 runtime.

## 11. Backups and rollback

All paths below are relative to the game root. Perform replacements with the game closed.

| Backup | Contents |
|---|---|
| `BepInEx/core/libdobby.dylib.orig-20230818` | Original installed Dobby, `Dobby-20230818-888d971` |
| `BepInEx/core/Il2CppInterop.Runtime.dll.orig-ci829` | Original installed Runtime, before macOS discovery fixes |
| `BepInEx/core/Il2CppInterop.Runtime.dll.orig-macos-ci829` | Earlier macOS-discovery-patched Runtime, before the master port and final hook corrections |

To restore the **previously verified zero-plugin baseline**:

1. Disable all currently enabled plugin DLLs (rename to `.dll.disabled`).
2. Copy `Il2CppInterop.Runtime.dll.orig-macos-ci829` over `Il2CppInterop.Runtime.dll`.
3. Keep the patched Dobby and `UnityLogListening = false`.
4. TerraFX can be moved to Trash when reverting the latest-master runtime.

Copying `.orig-ci829` back instead is a full managed-runtime rollback and reintroduces the module-discovery problem. Restoring the original Dobby likewise removes the Rosetta fixes. These original backups are not the working mod-enabled setup.

For an ordinary mod rollback, rename the relevant DLL to `.dll.disabled` or move its folder outside `BepInEx/plugins/`, keeping the now-working runtime fixes. Renaming a folder inside the plugins tree does not stop recursive DLL discovery. Mod Settings Menu requires BigWalkLocalizationAPI, and its localization JSON must remain beside its DLL.

## 12. Installed binary fingerprints

SHA-256 values recorded from the working installation on 2026-09-25:

```text
5f31b9fca678536ed1636206f47b77431ac5b972ff92a0a2badf84bc065f9562  libdoorstop.dylib
fbec010a0da9b1ca87765cfe1b3ef6f7b0e75b8a85ce639b94294ea68b2abff6  BepInEx/core/libdobby.dylib
81cb02271e4e5d17b797e6f95fed156a4e7324c6538db87f3dd46229b41a9a44  BepInEx/core/Il2CppInterop.Runtime.dll
6e3dd6e4cdbbc4a9439ead4f6a49a25911379680c6342562132c3e6b6192969b  BepInEx/core/TerraFX.Interop.Windows.dll
```

The code signatures and interop cache were validated against this specific game build. After an update, both may need revisiting even if the broad Unity version still matches.
