[MAC_BUILD_NOTES.md](https://github.com/user-attachments/files/34224582/MAC_BUILD_NOTES.md)
# Snes9x (emunwa fork) - macOS build notes

## Build and install

```
pkill -x Snes9x
cd ~/Documents/Games/snes9x-emunwa
xcodebuild -project macosx/snes9x.xcodeproj -scheme snes9x -configuration Release MACOSX_DEPLOYMENT_TARGET=12.0 CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO build
cp -R "$(xcodebuild -project macosx/snes9x.xcodeproj -scheme snes9x -configuration Release -showBuildSettings | awk -F' = ' '/ BUILT_PRODUCTS_DIR/{print $2}')/Snes9x.app" /Applications/
open /Applications/Snes9x.app
```

The first time a ROM is opened from Documents, macOS asks for access to the Documents folder. Click Allow.

## Using the emulator

- **File > Open ROM Image...** (Cmd+O): opens the file dialog. A **Show** popup at the bottom filters the list:
  - **SNES ROM Files** (default): .sfc, .smc, .swc, .fig, .bs, and archives .zip, .gz, .7z. Other files are greyed out.
  - **All Files**.
  - The dialog starts in the ROM folder set in Settings > Files.
- **File > Open Recent**: recently opened ROMs. Opened ROMs are added automatically.
- **Double-click a .sfc file** in Finder (or Open With): loads it in Snes9x.
- **Emulation > Software Reset / Hardware Reset**: enabled whenever a game is loaded.
- **Settings...**: opens the Settings window, including while a game is running.
- **Emulation > Enable Network Access**: the emunwa server (for trackers / SNI). On by default. Only this Mac can connect.
- **Emulation > Act as Server on Network**: off by default. When checked, other computers on your network can connect too. Turning it on or off restarts the server, which drops open connections.
- Red close button and Cmd+Q quit the app, including while a game is running.

## Settings added

- **Settings > Display**
  - **Video Mode:** Blocky, Smooth, Super Eagle, 2xSaI, Super 2xSaI, EPX, HQ2x, HQ3x, HQ4x, and the NTSC modes (Composite, S-Video, RGB, Monochrome, and NTSC+TV versions with scanlines). Changes apply while a game runs.
  - **Video Timing:** Auto (NTSC/PAL), PAL, or NTSC. Applies the next time a game loads.
  - **Frame Skipping:** a menu with Automatic (default) or a fixed 0 to 10 frames. A saved value above 10 appears as its own entry.
  - **Show Frame Rate:** shows the frame rate in the game window.
  - **Stretch Image:** on (default) fills the window. Off keeps the game's shape with black bars.
  - **VSync:** on (default) waits for the display's refresh before drawing each frame. Off draws as soon as a frame is ready.
  - **Bilinear Filtering:** smooths the picture when scaled, for every video mode. Smooth is always smooth.
  - **Integer Scaling:** with Stretch Image off, uses the largest whole-number size that fits. Ignored when Stretch Image is on.
  - **Full Screen:** toggles full screen for the game window.
  - **Full Screen on ROM Open:** enters full screen when a game loads.
- **Settings > Sound** (applies live unless noted)
  - **Volume:** applied to the sound as it plays.
  - **Playback Rate:** 8000 to 48000 Hz. Restarts the audio output when changed during a game.
  - **Buffer Length:** 16 to 176 ms (affects latency). Restarts the audio output when changed during a game.
  - **Synchronize with sound core** and **Dynamic Rate Control**: applied during play.
- **Settings > Files**
  - **Save files in:** Application Support (default), Next to ROM, or a Custom Folder you choose.
  - **ROM folder:** where File > Open ROM Image starts. Remembered between launches.
- Saved choices are loaded at launch.

Emulate Fullscreen was dropped; the Full Screen setting covers it. Nothing else from the Display list is outstanding.

## What changed from upstream

Changed files (in `macosx/` unless noted):

- `mac-os.mm`
  - Emunwa server starts at launch (`Settings.NWAccess = true`) and stops at quit.
  - Adds the Enable Network Access and Act as Server on Network items to the Emulation menu.
  - Removed click-to-pause: clicking the game window no longer pauses it.
  - `loadROM` no longer hangs waiting for an emulation thread that isn't running.
  - Adds `S9xSetPause` / `S9xClearPause`, used by the emunwa code.
  - `S9xEngine` gets setters and getters for the sound settings, and `chooseROMFile` (which shows the Open panel).
  - Changing the playback rate or buffer length while a game runs restarts the audio output.
  - Includes `mac-cart.h` and `mac-audio.h`.
- `mac-audio.mm`: volume is applied to the samples in the audio callback. The output unit's hardware gain was ignored on the Mac.
- `mac-snes9x.mm`: `SNES9X_OpenCart` takes the ROM path from the URL the user chose (it was reading it from a nil URL, so the load silently failed).
- `mac-render.mm`: software video filters (Super Eagle, 2xSaI, Super 2xSaI, EPX, HQ2x/3x/4x), the NTSC modes, Stretch Image and Integer Scaling (fitting the picture in the view), VSync (the layer's display sync), and the Bilinear Filtering flag.
- `shaders.metal`: nearest sampling for every mode except Smooth, which is linear, and linear for any mode with Bilinear Filtering on.
- `filter/blit.cpp` and `filter/blit.h`: `S9xBlitNTSCFrame`, which picks the NTSC blitter for the resolution and whether scanlines are on.
- `mac-cart.mm`: the Open panel gets the **Show** filter popup and starts in the ROM folder. It also shows its options disclosure by default, so the panel is visible.
- `Snes9x/AppDelegate.m`
  - Quits when the last window closes (red button), and Quit stays enabled during play.
  - Software and Hardware Reset are enabled whenever a game is loaded.
  - File > Open uses the engine's Open panel. Full Screen is applied when a ROM opens, if enabled.
  - Applies the saved Files, Video Timing, and Sound settings at launch.
- `Snes9x/S9xPreferences/`: new Display and Sound panels, the Files panel, and the constants used by them.
- `snes9x.xcodeproj/project.pbxproj`
  - Builds `snes9x-nwaccess.cpp` with `-std=gnu++17`.
  - Removed the AGL framework reference (nothing in the source uses it).
- `emu-nwaccess/snes9x-nwaccess.h` and `snes9x-nwaccess.cpp`
  - Uses pthreads on macOS for the server thread.
  - Adds missing `return` statements and the `<string>` include.
  - Declares `NWA_AllowNetwork`.
- `emu-nwaccess/generic_poll_server.c`
  - Connections from other computers are closed unless `NWA_AllowNetwork` is set.
- `gtk/CMakeLists.txt`: patch to skip X11 on Apple. Not needed for the macOS app; it was an abandoned attempt.

## Known limits

- The Act as Server on Network setting resets to off each time the app starts.
- The server still listens on the network port, but connections from other computers are closed right away unless Act as Server on Network is on.
- Pause-when-inactive is not enabled (removed at your request).
- The NTSC modes use the bundled snes_ntsc filter. They are not checked on hires games or the freeze screen yet.
- The Video Mode filters are CPU filters, so they cost more processing than Blocky.
