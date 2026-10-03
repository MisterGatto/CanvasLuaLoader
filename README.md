# LuaLoader: Lua & GameGuardian Mod for Canvas

> [!NOTE]
> **Project Notice**:
> LuaLoader is a dedicated **Lua & GameGuardian companion mod** maintained alongside standard [Canvas](https://github.com/skyprotocol/Canvas-Open-Source).
> * **LuaLoader Mod**: A standalone plugin mod providing the GameGuardian compatibility layer, memory scanner, and Lua scripting engine for Canvas.
> * For LuaLoader-specific issues, updates, and script help, please use this repository's issues page.

**LuaLoader** is am in-process **Lua scripting engine** and **GameGuardian (`gg.*`) API compatibility layer** built as a native mod for **Canvas Mod Loader** (targeting *Sky: Children of the Light* on Android ARM64).

It allows you to run standard and obfuscated (preferably not heavy) GameGuardian scripts, custom Lua scripts, and memory automation directly in-game with zero external tools or root access.

---

## Key Features

* **Complete GameGuardian (`gg.*`) Emulation**:
  * Memory searching & editing: `gg.searchNumber`, `gg.refineNumber`, `gg.getResults`, `gg.loadResults`, `gg.setValues`, `gg.editAll`.
  * Pointer scanning: `gg.searchPointer` live over process memory.
  * Memory regions: `gg.getRangesList`, `gg.getRanges`, `gg.setRanges`, `gg.getValuesRange`, `gg.allocatePage`.
  * In-game UI popups: `gg.choice`, `gg.multiChoice`, `gg.prompt`, `gg.alert`, `gg.toast`.
* **In-Process Memory Scanner**:
  * Fast chunk-based scanning for `DWORD`, `Float`, `Double`, `Byte`, `Word`, `QWord`, `XOR`, and ranges (`100~200`).
* **In-Game ImGui Bridge**:
  * Scripts run in non-blocking background threads.
  * Interactive ImGui dialogs render directly on screen over the game with cancel/close support.
* **Legacy & Obfuscation Compatibility**:
  * Embedded Lua 5.1/5.2 polyfills (`setfenv`, `getfenv`, `loadstring`, `unpack`, `table.getn`, `bit`, `bit32`).
* **Live Network & Offline JSON**:
  * Embedded pure Lua JSON engine (`json.encode`, `json.decode`).
  * Live HTTP client via `gg.makeRequest(url)`.
* **Native Canvas Extensions**:
  * Pattern scanning: `canvas.findPattern(ida_pattern)`.
  * Memory patching: `canvas.patch(address, hex_bytes)`.
  * Game introspection: `canvas.getLibBase()`, `canvas.getGameVersion()`.

## Installation

1. Download the compiled `liblualoader.so` from Releases.
2. Open **Canvas** on your device.
3. Tap **Import Mod** and select `liblualoader.so`.
4. Launch Sky

---

## Running Lua Scripts

Place your `.lua` scripts anywhere on your device (for example in your device's `Download/` folder: `/sdcard/Download/`).

When in-game:
1. Open the Canvas Menu and tap **Lua Loader**.
2. **Import**: Tap `Import` to pick any `.lua` or `.luac` file directly from Android's system file manager.
3. **Browse**: Tap `Browse` to navigate folders in-place (defaults to `/sdcard/Download/`, includes quick `Downloads` shortcut, `.. [Parent Directory]` to go up, and live metadata preview).
4. Tap `Load Mod` or `Load & Run`.
5. Run, Stop, or open script dialog Menus directly from the script list.
6. The last run script is always accessible in the pinned `Recent:` quick-action bar.

### Script Metadata (Creator & Version)

LuaLoader automatically reads the first 100 lines of any script to display the creator and version:

```lua
-- Author: YourName
-- Version: 1.0.0
```

---

## Example Script

```lua
-- Sample GameGuardian script running natively in Canvas
local choice = gg.choice({
    "1. Search & Edit Energy",
    "2. Find IDA Function Pattern",
    "3. Show Toast",
    "4. Exit"
}, 1, "Sky Canvas Lua Menu")

if choice == 1 then
    gg.clearResults()
    gg.setRanges(gg.REGION_ANONYMOUS | gg.REGION_C_ALLOC)
    gg.searchNumber("100", gg.TYPE_DWORD)
    local results = gg.getResults(10)
    gg.toast("Found " .. #results .. " results")
    gg.editAll("9999", gg.TYPE_DWORD)
    gg.toast("Values modified to 9999!")
elseif choice == 2 then
    local addr = canvas.findPattern("00 00 A0 E3 1E FF 2F E1")
    if addr ~= 0 then
        gg.alert("Found pattern at 0x" .. string.format("%X", addr))
    else
        gg.toast("Pattern not found")
    end
elseif choice == 3 then
    gg.toast("Hello from LuaLoader in Canvas!")
end
```

---

## API Reference

For the complete documentation of all GameGuardian and Canvas Lua APIs, see the [Lua Modding API Reference](docs/LUA_MODDING_API.md).

---

