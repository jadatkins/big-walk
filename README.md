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

## Upstream Projects and Copyright Licences

I had to patch about four different repositories to get this working. When I have
a minute, I'll fork them, push the changes and see whether anything is useful
enough to create a pull request. At the moment this was mostly vibe-coded, and I
haven't figured out the licence details yet. See https://github.com/BepInEx/BepInEx
and all the dependencies linked from there.
