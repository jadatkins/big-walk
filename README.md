# Big Walk

## Save Games

On a Mac, it goes in
- `~/Library/Application Support/House House/Big Walk/user_data/save_games`

On Windows, it goes in
- `%LOCALAPPDATA%Low\House House\Big Walk\user_data\save_games`
- aka `%USERPROFILE%\AppData\LocalLow\House House\Big Walk\user_data\save_games`

## Mods on Mac OS

To use mods on a Mac, you need to copy the contents of the `mods` folder into
your `/Library/Application Support/Steam/steamapps/common/Big Walk` folder,
which should already contain `Big Walk.app`. See the [README](mods/README.md).

Note that `Data` and `GameAssembly.dylib` should be symlinks, pointing at things
inside `Big Walk.app`.
