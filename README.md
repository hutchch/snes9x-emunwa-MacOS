[MAC_BUILD_NOTES.md](https://github.com/user-attachments/files/34218754/MAC_BUILD_NOTES.md)
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

- **File > Open ROM Image...** (Cmd+O): opens the file dialog with the ROM type, interleave, video and header options.
- **File > Open Recent**: recently opened ROMs. Opened ROMs are added automatically.
- **Double-click a .sfc file** in Finder (or Open With): loads it in Snes9x.
- **Emulation > Software Reset / Hardware Reset**: enabled whenever a game is loaded.
- **Settings...**: the normal Settings window (video, emulation, controllers).
- **Emulation > Enable Network Access**: the emunwa server (for trackers / SNI). On by default. Only this Mac can connect.
- **Emulation > Act as Server on Network**: off by default. When checked, other computers on your network can connect too. Turning it on or off restarts the server, which drops open connections.
- Red close button quits the app.

## What changed from upstream

Changed files (in `macosx/` unless noted):

- `mac-os.mm`
  - Emunwa server starts at launch (`Settings.NWAccess = true`) and stops at quit.
  - Adds the Enable Network Access and Act as Server on Network items to the Emulation menu.
  - Removed click-to-pause: clicking the game window no longer pauses it.
  - `loadROM` no longer hangs waiting for an emulation thread that isn't running.
  - Adds `S9xSetPause` / `S9xClearPause`, used by the emunwa code.
  - Includes `mac-cart.h`.
- `mac-snes9x.mm`: `SNES9X_OpenCart` takes the ROM path from the URL the user chose (it was reading it from a nil URL, so the load silently failed).
- `mac-cart.mm`: attaches the ROM-type options panel to the Open dialog (it was built but never shown).
- `Snes9x/AppDelegate.m`
  - Quits when the last window closes (red button).
  - Software and Hardware Reset are enabled whenever a game is loaded.
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
- Leftover files in the repo that can be deleted: `*.orig`, `*.orig2` backups, and `.emunwa_mac_patch.py`.
