# Big Walk modding on macOS

A preconfigured BepInEx IL2CPP setup for the macOS Steam version of **Big Walk**. It includes the local Doorstop, Dobby, and Il2CppInterop fixes that made mods work under Rosetta.

## Compatibility

- Tested on an **Apple M1**, running the game's **x86_64** version through Rosetta.
- Tested game build: **1.5.1 2608271544**, Unity **6000.3.17f1**.
- Includes BepInEx **6.0.0-be.755** and its x86_64 **.NET 6.0.7** runtime.
- No separate .NET installation, SDK, or compilation is needed to use this folder.
- Other Mac models and game versions have not been independently verified. The included interop cache and native-hook signatures are specific to the tested game build; a game update may require new fixes or a regenerated cache.

## What's included

Only the shared settings tools are enabled by default:

| Plugin | Version | Purpose |
|---|---|---|
| Mod Settings Menu | 1.1.2 | Adds **Mod Settings** to the main and pause menus |
| BigWalkLocalizationAPI | 1.1.0 | Required localization dependency for Mod Settings Menu |

Third Person, Big Visuals, and BigSpitty were verified during development but are **not included in this clean setup**. Install the mods you want using the instructions below.

A disabled `BigWalkInjectionProbe.dll.disabled` is retained for troubleshooting. It does not load or change gameplay in this state.

## Setup

### 1. Locate your Steam installation

Install Big Walk through Steam and launch it normally once. Quit the game before copying mod files.

In Steam, right-click **Big Walk → Manage → Browse Local Files**. The folder containing `Big Walk.app` is the **game root**. Its default location is:

```text
~/Library/Application Support/Steam/steamapps/common/Big Walk
```

On Apple Silicon, Rosetta must be available. If macOS prompts to install it when launching an Intel application, complete that installation.

### 2. Put the modding files beside the app

Extract the ZIP first. Copy the modding files and directories from its `Big Walk` folder into your game's root. Keep your Steam-installed `Big Walk.app`; the included modding setup does not require modifying the bundle.

If the ZIP contains an app bundle as well, check that your Steam game matches the tested build above before using this setup. The presence of a bundled app is not a substitute for installing and launching the game through Steam.

The resulting layout should be:

```text
Big Walk/
├── Big Walk.app/
├── BepInEx/
│   ├── config/BepInEx.cfg
│   ├── core/
│   ├── interop/
│   ├── plugins/
│   │   ├── ModSettingsMenu/
│   │   ├── BigWalkLocalizationAPI/
│   │   └── BigWalkInjectionProbe/    # disabled diagnostic DLL
│   └── ...
├── dotnet/
├── Data -> Big Walk.app/Contents/Resources/Data
├── GameAssembly.dylib -> Big Walk.app/Contents/Frameworks/GameAssembly.dylib
├── libdoorstop.dylib
├── run_bepinex.sh
├── .doorstop_version
├── README.md
├── TROUBLESHOOTING.md
└── MACOS_MODDING_FIXES.md
```

**Merge existing folders rather than replacing them** if you already have mods or configuration you want to keep. Preserve the supplied patched files in `BepInEx/core/`; installing a stock BepInEx package over them can undo the fixes.

`Data` and `GameAssembly.dylib` are **relative symbolic links**. They should still point inside the neighboring `Big Walk.app` after extraction. See [Troubleshooting](TROUBLESHOOTING.md) if the archive tool did not preserve them.

### 3. Set the Steam launch option

Open **Big Walk → Properties → General → Launch Options** and enter:

```text
"/FULL/PATH/TO/Big Walk/run_bepinex.sh" %command%
```

Replace `/FULL/PATH/TO/Big Walk` with your own absolute game-root path. Keep the quotation marks and `%command%`. **Do not use `~` or another person's username.** Steam does not expand `~` here.

For a default Steam library, this looks like:

```text
"/Users/YOUR_USERNAME/Library/Application Support/Steam/steamapps/common/Big Walk/run_bepinex.sh" %command%
```

If extraction removed the script's executable permission, run this in Terminal using your actual path:

```sh
chmod +x "/FULL/PATH/TO/Big Walk/run_bepinex.sh"
```

### 4. Launch through Steam

Start Big Walk using Steam's **Play** button. Look for **Mod Settings** in the main menu or pause menu. It lists loaded mods that expose configuration entries; available options depend on the mods installed.

The supplied launcher selects x86_64 and loads the bundled runtime. Launching the app directly does not use that launcher.

## Downloading and installing mods

1. Quit Big Walk.
2. Browse the [Big Walk community on Thunderstore](https://thunderstore.io/c/big-walk/), or use the mod author's published download page.
3. Choose a mod for **Big Walk / BepInEx IL2CPP** and read its requirements. This setup is not for BepInEx 5 Mono plugins. Mods with Windows-only native dependencies may need a Mac-specific build.
4. Use **Manual Download** and extract the ZIP somewhere temporary.
5. Check `manifest.json` or the mod page for **dependencies**. Download any additional mod dependencies too. The base BepInEx IL2CPP requirement is already supplied here; do not replace the patched runtime just to satisfy that entry.
6. Install the plugin files according to the archive layout below.
7. Launch through Steam and test the mod's actual controls or menu. Add one mod at a time when possible so a failure is easy to identify.

### Archive with loose plugin files

For an archive containing `ExampleMod.dll` and companion files at its top level, create a folder under `BepInEx/plugins/` and put them there:

```text
BepInEx/plugins/ExampleMod/
├── ExampleMod.dll
├── ExampleMod.Localization.json    # if supplied
└── other assets                    # sounds, bundles, etc., if supplied
```

Keep the mod's required assets and relative layout. A DLL alone is not always enough. Archive README/manifest files can stay inside that mod's folder rather than overwriting this README.

### Archive containing `BepInEx/plugins/`

Merge the **contents of the archive's `BepInEx/plugins/`** into your actual `BepInEx/plugins/`, or into a named plugin subfolder while preserving the plugin's companion-file layout. Do not create another nested `BepInEx` directory inside `plugins/`.

If a mod explicitly supplies patchers or config files, follow its instructions for those destinations. Ordinary mod installation should not replace the launcher, bundled runtime, or patched core libraries.

### Configure a mod

- Use **Mod Settings** when the mod exposes compatible settings.
- Some mods also provide their own panel or hotkey; consult their README.
- Configuration is normally generated under `BepInEx/config/` on first load. Quit the game before editing files directly; the game may otherwise overwrite your edits on exit.
- A mod's displayed name and config filename can differ. For example, the tested Big Visuals package creates `Big_Visuals.cfg` and defaults to **F11** for its own panel.

### Disable, remove, or update a mod

- **Disable:** rename its plugin DLL from `ExampleMod.dll` to `ExampleMod.dll.disabled`, or move its folder outside `BepInEx/plugins/`.
- Renaming a folder while leaving it inside `plugins/` does **not** disable its DLLs: BepInEx searches subdirectories.
- **Remove:** move the plugin files and, if desired, its specific configuration to Trash. Do not remove a dependency still needed by another mod.
- **Update:** quit the game, replace that mod's files, and avoid leaving two active copies of the same plugin. Keep settings unless the author requires a reset.

## Important files and settings

Keep `BepInEx/interop/`: it is the generated compatibility data needed by this setup, not a disposable log/cache directory.

The working configuration deliberately has:

```ini
[IL2CPP]
UpdateInteropAssemblies = false

[Logging]
UnityLogListening = false
```

Automatic interop generation is disabled because the shipped generator cannot process this game's universal Mach-O correctly. Unity log forwarding was disabled during diagnosis and remains off in the verified setup. See the troubleshooting guide before changing either setting.

The launcher and BepInEx create fresh logs on each run. Debug logging remains enabled for this patched build. Files ending in `.orig-*` under `BepInEx/core/` are rollback backups, not additional active runtime assemblies.

## Further documentation

- [Troubleshooting](TROUBLESHOOTING.md): launch failures, log locations, isolation tests, and rollback.
- [Technical fix history](MACOS_MODDING_FIXES.md): native/managed changes, source revisions, tests, rebuild commands, and binary fingerprints. This is a historical development report and includes the original developer's local paths and previously tested mods.
