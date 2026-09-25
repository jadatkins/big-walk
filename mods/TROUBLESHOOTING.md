# Big Walk macOS modding: troubleshooting

For installation and downloading mods, start with [README.md](README.md). For the implementation history and rebuild commands, see [MACOS_MODDING_FIXES.md](MACOS_MODDING_FIXES.md).

Paths below are relative to the folder containing `Big Walk.app`, unless they begin with `~/`. Investigate while the game is closed before replacing files. Test game launches through Steam.

## 1. Steam does not launch the game

Check **Properties → General → Launch Options**:

```text
"/FULL/PATH/TO/Big Walk/run_bepinex.sh" %command%
```

Use your own absolute path, with quotes. Steam does not expand `~`. If the launcher is not executable:

```sh
chmod +x "/FULL/PATH/TO/Big Walk/run_bepinex.sh"
```

Steam's logs are under `~/Library/Application Support/Steam/logs/`:

| Log | What to look for |
|---|---|
| `console_log.txt` | Actual launch command, failed spawn, process added/removed |
| `gameprocess_log.txt` | Game PID and exit code |

An exit code of `-1` was observed in the earlier native failures; it does not by itself identify the cause.

## 2. BepInEx is not loading

Read `BepInEx/native-launch.log`. The launcher captures native stdout/stderr and enables `DYLD_PRINT_LIBRARIES=1`. Look for:

```text
libdoorstop.dylib
.../Frameworks/GameAssembly.dylib
dotnet/libcoreclr.dylib
dotnet/libclrjit.dylib
```

- Missing Doorstop: check the launch command, extraction, and macOS loading error.
- Doorstop present but CoreCLR absent: check architecture selection and that the supplied patched loader/runtime files are present.
- The supplied launcher must use `archpreference="x86_64"`; bundled native runtime components are x86_64.
- The launcher passes DYLD variables through `arch -e`. Replacing that with ordinary exported DYLD variables can break injection because macOS strips them before the platform binary executes.
- The tested setup worked with Gatekeeper assessments enabled. Do not globally disable Gatekeeper as a routine setup step; inspect the actual reported failure instead.

Some later native output may appear in Unity's `Player.log` after Unity redirects output.

## 3. Paths or symlinks were lost during extraction

From the game root, inspect:

```sh
ls -l Data GameAssembly.dylib
```

Expected relative link targets:

```text
Data -> Big Walk.app/Contents/Resources/Data
GameAssembly.dylib -> Big Walk.app/Contents/Frameworks/GameAssembly.dylib
```

If a link is missing, recreate it from the game root:

```sh
ln -s "Big Walk.app/Contents/Resources/Data" Data
ln -s "Big Walk.app/Contents/Frameworks/GameAssembly.dylib" GameAssembly.dylib
```

Run only the command for a missing link. If an archive tool created a regular file or directory in its place, inspect it and move the external copy aside first. Do not overwrite the real files inside `Big Walk.app`.

## 4. BepInEx starts, but a mod does not work

Read `BepInEx/LogOutput.log`. The normal progression is:

1. Preloader and runtime information.
2. Chainloader initialization and runtime-invoke hook.
3. Interop assemblies preloaded.
4. Plugin discovery and `Loading [...]` messages.
5. `Chainloader startup complete`.
6. Mod-specific runtime output.

`Chainloader initialized` does not mean plugins have loaded. Even chainloader completion and a first component Update did not establish stability during the original diagnosis. Verify the actual mod in gameplay.

Check that:

- The mod's DLL is under `BepInEx/plugins/` and does not end in `.disabled`.
- Its dependencies are installed and required companion files are beside it.
- It is an IL2CPP-compatible Big Walk mod, not a BepInEx 5 Mono mod or a Windows-only native plugin.
- There is only one active copy of each plugin.
- Mod Settings Menu still has `BigWalkLocalizationAPI.dll` installed and `ModSettingsMenu.Localization.json` beside its own DLL.

Only loaded mods with configuration entries appear in Mod Settings. A mod can use its own key/menu instead. Consult its instructions and inspect its generated config under `BepInEx/config/`.

## 5. The game quits before the menu

First read the managed and native logs before they are overwritten. Disable newly added mods individually and repeat the Steam launch. Keep required dependencies enabled for the mods that remain.

The original pre-menu failure was traced to incorrect macOS class-injection hook targets/argument layouts. The supplied runtime corrects these. Its diagnostic log includes:

```text
GenericMethod::GetMethod (macOS struct reference)
```

For the tested build, subtract the logged native image base from the hook addresses to get:

| Hook | Expected RVA |
|---|---|
| Generic method | `0x933170` |
| Metadata type lookup | `0x974130` |
| Field default value | `0x9736F0` |

These are diagnostic reference offsets, not values to patch manually. The runtime selects them using unique masked signatures. A missing/ambiguous-signature error can mean the game updated; do not bypass the check or substitute these offsets into another build.

### Zero-plugin isolation

Temporarily disable **all plugin DLLs**, including Mod Settings Menu and its dependency, by appending `.disabled`. Keep `UnityLogListening = false` in `BepInEx/config/BepInEx.cfg` for this test: Unity log forwarding can itself exercise class injection even with no mods.

Folder renames inside `plugins/` do not disable DLL discovery. Move folders outside that tree if you prefer not to rename DLLs.

### What is BigWalkInjectionProbe?

`BepInEx/plugins/BigWalkInjectionProbe/BigWalkInjectionProbe.dll.disabled` is our diagnostic plugin, not a gameplay mod. It registers/adds an otherwise empty Unity component and logs its first Update. It helped distinguish the shared class-injection failure from individual mod behavior.

It is useful after runtime changes or when investigating a similar crash. With the game closed, disable the regular plugins and rename the probe to `BigWalkInjectionProbe.dll` for a controlled test. Look for:

```text
Diagnostic MonoBehaviour added
Diagnostic MonoBehaviour reached its first Update
```

Then check that the menu and gameplay remain usable. Restore the `.dll.disabled` suffix after testing. With the old, uncorrected hooks, the probe reached its first Update before crashing later, so the messages alone are not sufficient.

## 6. Interop generation fails or the game has updated

`BepInEx/interop/` is required compatibility data, **not** a disposable cache. Its `assembly-hash.txt` and generated assemblies belong to the tested game build.

The current configuration has `UpdateInteropAssemblies = false`. It avoids a Cpp2IL failure when chained-fixup processing tries to write into the universal Mach-O reader's read-only slice stream:

```text
System.NotSupportedException: Stream does not support writing.
```

Generation originally succeeded using a temporary thin x86_64 copy at the external root `GameAssembly.dylib` path. The root symlink was restored for normal gameplay. Regenerating after a game update requires repeating that procedure against the new binary and metadata; see the [technical report](MACOS_MODDING_FIXES.md#3-external-paths-and-interop-cache-generation).

Editing the hash alone does not make stale interop assemblies valid. The native hook signatures may also need updating.

`BepInEx/cache/` is different: it contains disposable plugin-discovery data, which BepInEx can recreate. This directory is excluded by the packaging command in the README.

## 7. Log locations and cleanup

| Location | Contents |
|---|---|
| `BepInEx/LogOutput.log` | Managed loader and plugin diagnostics; rewritten each launch |
| `BepInEx/native-launch.log` | Native stdout/stderr and dyld diagnostics; rewritten each launch |
| `~/Library/Logs/House House/Big Walk/Player.log` | Most recent Unity player log |
| Same directory, `Player-prev.log` | Previous Unity launch |
| Same directory, `bigwalk-N.log` | Game's own timestamped logs |
| `Big Walk.app/Contents/MacOS/preloader_*.log` | Early preloader exception, only when generated |
| `~/Library/Logs/DiagnosticReports/` | macOS crash reports, if generated |

The Unity/Steam logs outside the game folder are not included when zipping this folder. Keep useful logs before the next launch overwrites them. The absence of a new macOS crash report does not establish a clean exit.

To reset game-folder logs, quit the game and run this **in zsh from the game root**, if the `trash` utility is installed:

```zsh
logs=(BepInEx/LogOutput.log(N)
      BepInEx/native-launch.log(N)
      "Big Walk.app/Contents/MacOS/"preloader_*.log(N))
if (( ${#logs} )); then trash "${logs[@]}"; fi
```

Alternatively, move the named files to Trash in Finder. `(N)` prevents zsh from aborting when no files match. The launcher recreates its logs. There is no reason to delete Steam-wide logs to clean this package.

For system-log investigation, use `/usr/bin/log` explicitly if your shell defines another command/function named `log`.

## 8. Rollback and retained backups

With the game closed, disable the problem mod first. Leave the working core libraries installed unless specifically testing a runtime rollback.

| Backup under `BepInEx/core/` | Meaning |
|---|---|
| `libdobby.dylib.orig-20230818` | Original Dobby, before Rosetta fixes |
| `Il2CppInterop.Runtime.dll.orig-ci829` | Original Runtime, before macOS module discovery |
| `Il2CppInterop.Runtime.dll.orig-macos-ci829` | Earlier discovery-patched Runtime, before latest-master port and corrected class-injection hooks |

These backups are **not** equivalent to the final mod-enabled setup. The previously verified zero-plugin baseline uses `.orig-macos-ci829` with patched Dobby, all plugins disabled, and Unity log forwarding off. Restoring the fully original files reintroduces the earlier problems. See the technical report for details.

## 9. Known working-launch fallbacks

The verified modded launch still logged a Class::Init substitute and a Harmony backend fallback for `SettingsMenu::.ctor()`. Those messages alone were not blockers in the tested setup. A newly introduced error or changed game build still needs investigation.

Native Dobby and dyld debug output remains enabled to help diagnose this local build. Re-enabling Unity log forwarding or changing the supplied core/runtime versions has not been verified as part of this clean package.
