# GTA IV: The Complete Edition on macOS (Apple Silicon)

How I got the **Steam version of GTA IV: The Complete Edition** running on a Mac with
[Sikarugir](https://github.com/Sikarugir-App/Sikarugir) (the successor to Wineskin/Kegworks),
including the Rockstar Games Launcher, **Fusion Fix** and **TJGM's Drag n' Drop Archive**.

| | |
|---|---|
| Mac | MacBook Air M4, 16 GB (fanless) |
| macOS | 27.0.1 |
| Wrapper | Sikarugir, template 1.0.27 |
| Wine engine | `WS12WineSikarugir11.0_1` (Wine 11.0) |
| Renderer | D9VK (DXVK for DX9, running on MoltenVK) |
| Game | GTA IV: The Complete Edition (Steam), Rockstar Games Launcher, Fusion Fix 5.1.2 |
| Result | Plays at ~27–30 fps on Low with a 30 fps cap, CPU around 60 °C |

> No game files or mod files are included here. You need to own GTA IV on Steam. Mods are linked to their original sources.

---

## 1. Install Sikarugir

```sh
brew install --cask Sikarugir-App/sikarugir/sikarugir
```

This installs **Sikarugir Creator** in `/Applications`.

## 2. Create the wrapper

1. Open Sikarugir Creator and download the engine **`WS12WineSikarugir11.0_1`**, along with the latest wrapper template (I used 1.0.27).
2. Create a new blank wrapper named **`Steam`**. It ends up in `~/Applications/Sikarugir/Steam.app`.

> **Don't use the CrossOver-based engine (`WS12WineCX24.0.7_7`).** On macOS 27, Steam fails with
> *"Unexpected Transport Error 0x3008"*, because Wine's network-info driver (nsiproxy) can't read macOS 27's TCP table.
> The Wine 11 engine works.

<details>
<summary>Doing it from the terminal instead (what I actually did)</summary>

The `Sikarugir create-prefix` CLI failed for me with `WineAppInitializationError`, so I built the wrapper manually:

```sh
S="$HOME/Library/Application Support/Sikarugir"
W="$HOME/Applications/Sikarugir/Steam.app"

T=$(mktemp -d)
# 1. copy the wrapper template
tar -xJf "$S/Template/Template-1.0.27.tar.xz" -C "$T"
mkdir -p "$(dirname "$W")" && cp -R "$T/Template-1.0.27.app" "$W"
# 2. put the engine in place
tar -xJf "$S/Engines/WS12WineSikarugir11.0_1.tar.xz" -C "$T"
mkdir -p "$W/Contents/SharedSupport/wine"
cp -R "$T/wswine.bundle/" "$W/Contents/SharedSupport/wine/"
# 3. build the prefix
"$W/Contents/MacOS/wineskinlauncher" WSS-wineprefixcreate
```

To run any .exe inside the wrapper, set `Program Name and Path` and `Program Flags` in
`Steam.app/Contents/Info.plist`, then run `Steam.app/Contents/MacOS/wineskinlauncher`.

</details>

## 3. Install Steam

1. Download `SteamSetup.exe` from steampowered.com.
2. Open the wrapper's configuration tool (`Steam.app/Contents/Configure.app`), choose **Install Software**, and pick `SteamSetup.exe`.
3. Set the program to launch as `C:\Program Files (x86)\Steam\steam.exe`.
4. **Turn Debug Mode off.** With it on, one session wrote a 137 MB log and the game hung on the loading screen.

Wrapper `Info.plist` values I ended up with:

| Key | Value |
|---|---|
| `Program Name and Path` | `/Program Files (x86)/Steam/steam.exe` |
| `Debug Mode` | `0` |
| `D9VK` | `1` (the default renderer for DX9) |
| `WINEMSYNC` / `WINEESYNC` | `1` |
| `Symlinks In User Folder` | `0` (otherwise Windows installers drop shortcut files onto your real Mac Desktop) |

> If **Install** in Steam does nothing (a "Steam Service Error" appears in the logs), the Steam Client Service is missing.
> Run `SteamSetup.exe /S` through the wrapper again to fix it.

## 4. Install the game

1. Sign in to Steam and install **Grand Theft Auto IV: The Complete Edition** (app id `12210`).
2. In Steam, open GTA IV → **Properties** → **General** and **turn off the Steam Overlay**. The overlay injection helped cause the hang on the legal screen.
3. Press **Play** once. The **Rockstar Games Launcher** installs itself. Sign in with your Rockstar account.

## 5. Fix VRAM detection (512 MB → 4 GB)

Under Wine the game only detects 512 MB of video memory, which locks the graphics settings. Create
`GTAIV/commandline.txt` in the game folder:

```
-availablevidmem 4096
```

The game folder is
`Steam.app/Contents/SharedSupport/prefix/drive_c/Program Files (x86)/Steam/steamapps/common/Grand Theft Auto IV/GTAIV/`.
A copy is in [`config/commandline.txt`](config/commandline.txt).

## 6. Fusion Fix and TJGM's Drag n' Drop Archive

1. Download [TJGM's Drag n' Drop Archive](https://tjgm.dev/resources/gtaiv/TJGM%27s-Drag-n%27-Drop-Archive/). It includes [Fusion Fix](https://github.com/ThirteenAG/GTAIV.EFLC.FusionFix).
2. Copy its contents into the `GTAIV` folder: `update/`, `plugins/`, `dinput8.dll`, `d3d9.dll` and `vulkan.dll`.
3. Tell Wine to load the **ASI loader** (`dinput8.dll`) from the game folder. Either:
   - open the wrapper's **Winecfg** → **Applications** → add `GTAIV.exe` → **Libraries** → add `dinput8` → **native, builtin**, or
   - import [`config/wine-overrides.reg`](config/wine-overrides.reg) with the wrapper's **Regedit**. It sets the same override and also disables `winemenubuilder` so no shortcuts get created.

To uninstall, delete the files you copied in step 2.

## 7. In-game settings (important)

Set these in the game's menus. Fusion Fix saves them to `GTAIV/plugins/GTAIV.EFLC.FusionFix.cfg`, and
[`config/fusionfix-settings.txt`](config/fusionfix-settings.txt) lists the relevant keys.

- **Borderless windowed: ON.** Exclusive fullscreen at a different resolution (for example 1920x1200) makes Wine's
  macOS driver deadlock on the display-mode change, which gives a black screen and a hard hang. This was the one real crash I had.
- **FPS limit: 30.** It keeps a fanless MacBook Air much cooler and steadier than running uncapped.
- **FPS counter: ON**, if you want to watch performance.
- **Graphics: Low.**

> Fusion Fix's menu shows the renderer as **"Vulkan"** even if you never chose it. That's expected: the wrapper's
> D9VK *is* DXVK (Vulkan) running on MoltenVK, so Fusion Fix detects it and saves `GraphicsAPI = 1`. It's harmless.

## 8. Playing

- Start **Steam.app**, then press Play on GTA IV.
- **Don't Cmd+Tab during loading screens** (the legal screen, mission loads). The game can get stuck. When it's paused, it's safe.
- To leave the game without quitting: press **Esc**, then **Cmd+Tab**. The game shows up as "wine" in the app switcher.
- To quit everything: quit the game, then the Rockstar Games Launcher, then Steam.

## Performance and heat (M4 Air, all settings Low)

| | |
|---|---|
| FPS | ~27–30 (capped at 30) |
| GTAIV.exe | ~230–270 % CPU, 4.3–4.7 GB RAM |
| SocialClubHelper | ~20–35 % CPU (needed; the game quits without the Rockstar launcher) |
| CPU die temperature | ~58–63 °C while driving around |

About 30 processes run while you play: Wine's services, Steam and its browser helpers, and the Rockstar Launcher with its
Social Club browser helpers. That's normal, because Wine runs each Windows program as its own Mac process. Closing the
Steam window (Steam keeps running) saves a few percent of CPU.

## Troubleshooting

| Problem | Cause / fix |
|---|---|
| Steam: *Unexpected Transport Error 0x3008* | CrossOver 24 engine on macOS 27. Use `WS12WineSikarugir11.0_1`. |
| Steam **Install** button does nothing | Steam Client Service missing. Re-run `SteamSetup.exe /S` in the wrapper. |
| Game stuck on the legal/loading screen | Turn off Debug Mode and the Steam Overlay, and don't Cmd+Tab while it loads. |
| Steam still says the game is "running" after a force-quit | Quit Steam properly (Steam → Exit, or run `steam.exe -shutdown`), not with `kill -9`. |
| Graphics settings locked, game detects 512 MB VRAM | `-availablevidmem 4096` in `commandline.txt`. |
| Black screen and hang after changing the resolution | Exclusive fullscreen mode change. Turn on Borderless Windowed. |
| Shortcut `.lnk`/`.url` files appear on your Mac Desktop | Set `Symlinks In User Folder` to `0` and disable `winemenubuilder.exe`. |
| macOS warns that the app needs an update (Rosetta) | macOS's warning that Rosetta is going away. Wine runs the Windows x86 code through Rosetta 2, so this is expected for now. |

## Credits

- [Sikarugir](https://github.com/Sikarugir-App/Sikarugir): the wrapper and Wine engines
- [Fusion Fix](https://github.com/ThirteenAG/GTAIV.EFLC.FusionFix) by ThirteenAG
- [TJGM's Drag n' Drop Archive](https://tjgm.dev/resources/gtaiv/TJGM%27s-Drag-n%27-Drop-Archive/)
- [AppleGamingWiki: Grand Theft Auto IV](https://www.applegamingwiki.com/wiki/Grand_Theft_Auto_IV)
