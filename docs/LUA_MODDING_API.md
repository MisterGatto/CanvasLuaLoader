# CanvasLuaLoader Modding API Reference (for Canvas)

> [!IMPORTANT]
> **Host vs. Mod Architecture**:
> - **Canvas** is the standard mod loader and host environment. Standard Canvas does **not** include a Lua runtime or GameGuardian APIs natively.
> - **CanvasLuaLoader** (`liblualoader.so`) is the companion plugin mod loaded into Canvas. The dual Lua runtime (**Lua 5.4 & Lua 5.2**), GameGuardian (`gg.*`) compatibility layer, in-process memory scanner, and `canvas.*` native helpers are implemented entirely by **CanvasLuaLoader**.

**CanvasLuaLoader** executes `.lua` and `.luac` scripts directly **in-process** inside *Sky: Children of the Light* on Android ARM64 when running as a mod within Canvas.

This provides two powerful APIs:
1. **`gg.*` (GameGuardian Compatibility Layer)**: Run standard and obfuscated (preferably not heavy) GameGuardian scripts natively without root, external apps, or virtual spaces.
2. **`canvas.*` (CanvasLuaLoader Native Extensions)**: High-speed IDA pattern scanning, direct assembly patching, and game version inspection.

---

## Table of Contents
1. [CanvasLuaLoader Mod Interface](#1-canvaslualoader-mod-interface)
2. [Script Metadata & Header Tags](#2-script-metadata--header-tags)
3. [GameGuardian (`gg.*`) API & Menu Loop](#3-gameguardian-gg-api)
   - [Memory Search & Refine](#1-memory-search--refine)
   - [Reading & Writing Values](#2-reading--writing-values)
   - [Memory Regions](#3-memory-regions)
   - [In-Game ImGui UI Dialogs](#4-in-game-imgui-ui-dialogs)
   - [Interactive Menu Reopening Loop (`gg.isVisible`)](#interactive-menu-reopening-loop-ggisvisible)
   - [Utility & Process Functions](#5-utility--process-functions)
   - [Constants Reference](#6-constants-reference)
4. [CanvasLuaLoader Native (`canvas.*`) API](#4-canvaslualoader-native-canvas-api-optional-power-user-tools)
5. [Examples & Script Templates](#5-examples--script-templates)
   - [Example 1: Value Search, Refine & Edit (Candles/Currency)](#example-1-value-search-refine--edit-candlescurrency)
   - [Example 2: Pattern Scan & Assembly Patch (No-Clip/Godmode)](#example-2-pattern-scan--assembly-patch-no-clipgodmode)
   - [Example 3: Interactive Mod Menu Loop](#example-3-interactive-mod-menu-ui)

---

## 1. CanvasLuaLoader Mod Interface

### Action Toolbar
* **`Import`**: Opens the native Android system file picker dialog (`SAF`). You can select any `.lua` or `.luac` file from your device storage, cloud storage, or messaging downloads. The selected script is automatically imported, registered, and displayed in the loader.
* **`Browse`**: Transforms the current window in-place into an in-game file and directory browser without creating separate popup windows:
  * **Default Location**: Opens directly in `/sdcard/Download`.
  * **`Downloads`**: Quick-jump button that resets the browser to `/sdcard/Download` in one tap.
  * **`.. [Parent Directory]`**: Navigates up to superior folders (allowing navigation through `/sdcard`, internal app storage, and all the way to root `/`).
  * **Directory & Script List**: Folders `[DIR]` and scripts `[LUA]` with live metadata preview.
  * **`Load Mod` & `Load & Run`**: Loads the script into the loader or launches it immediately.
* **`Settings`**: Opens the settings configuration panel:
  * **Allow Multiple Concurrent Scripts**: *(Default: OFF)*. By default, running a new script automatically stops any currently running script so only one mod is active at a time. Enable this toggle to run multiple scripts simultaneously.
  * **Floating In-Game Menu Button**: *(Default: ON)*. Displays a draggable floating button on screen while a script is running to quickly reopen its menu at any time (GameGuardian style).

### Active Script Banner & Menu Reopening
* When any script is active, an **`[ACTIVE]`** banner appears at the top of CanvasLuaLoader:
  * **`Open Script Menu`**: Reopens the script's interactive dialog/menu immediately.
  * **`Stop Script`**: Safely terminates the script worker thread.
* **Floating In-Game Button**: Tapping the floating button while playing wakes up the script and brings its menu back to the screen.

### Recent Mod Quick-Action
* **`Recent:`**: Pinned at the top of the script list, displays the last script used on the device.
* **`Run` / `Stop` / `Menu`**: Quick one-tap execution, termination, or menu trigger.

### Script List & Controls
* **Search Filter (`Search scripts...`)**: Real-time filtering of scripts by name.
* **Script Cards**: Displays script title, extracted metadata (`v1.0.0 by Author`), path, with `Run`, `Stop`, and `Menu` controls.

### Manual Path Entry
* Collapsible header at the bottom (`Manual Path`) allowing manual entry of custom paths.

---

## 2. Script Metadata & Header Tags

CanvasLuaLoader automatically reads the first 100 lines of any `.lua` or `.luac` script upon discovery or loading to extract the **Creator / Author** and **Version**.

### Supported Formats

You can specify creator and version using comment headers or Lua variable declarations:

#### Creator / Author Formats
```lua
-- Author: MrGatto
-- @author MrGatto
-- Creator: MrGatto
-- Created by: MrGatto
-- By: MrGatto
```
or in code:
```lua
author = "MrGatto"
AUTHOR = "MrGatto"
```

#### Version Formats
```lua
-- Version: 1.2.0
-- @version 1.2.0
-- ver: 1.2.0
-- v1.2.0
```
or in code:
```lua
version = "1.2.0"
VERSION = "1.2.0"
```

### Complete Header Template

```lua
-- =========================================================
-- Author: MrGatto
-- Version: 1.0.0
-- Description: Custom mod script for Canvas
-- =========================================================

gg.toast("My Mod v1.0.0 Loaded!", true)
```

When loaded in LuaLoader, the script will automatically be presented as:
```
MyScript
v1.0.0 by MrGatto
/sdcard/Download/MyScript.lua
```

---

## 3. GameGuardian (`gg.*`) API

The **LuaLoader** mod (`liblualoader.so`) implements the standard GameGuardian scripting API for in-process execution inside Canvas.

For full details on the GameGuardian scripting engine, refer to the [Official GameGuardian Lua API Reference](https://gameguardian.net/help/classgg.html).

> [!NOTE]
> **LuaLoader** natively supports all standard search, refine, memory editing, and in-game UI dialog functions (`choice`, `multiChoice`, `prompt`, `alert`, `toast`). Out-of-process root features (such as saved list bookmarks and external speedhack hooks) are omitted in favor of native in-process execution.

#### `gg.searchNumber(text, [type], [encrypted], [sign], [memoryFrom], [memoryTo], [limit]) -> boolean`
Searches for numbers in the configured memory regions.
* `text` *(string)*: Number or range to search for (e.g. `"100"` or `"100~200"`).
* `type` *(int, optional)*: `gg.TYPE_AUTO`, `gg.TYPE_DWORD`, `gg.TYPE_FLOAT`, etc. Default: `gg.TYPE_AUTO`.
* `encrypted` *(bool, optional)*: Ignored in-process. Pass `false`.
* `sign` *(int, optional)*: Comparison operator (`gg.SIGN_EQUAL`, `gg.SIGN_NOT_EQUAL`, etc.). Default: `gg.SIGN_EQUAL`.
* `memoryFrom` *(int, optional)*: Start address bounds (0 for all).
* `memoryTo` *(int, optional)*: End address bounds (0 for all).
* `limit` *(int, optional)*: Max results to collect. Default: `10000`.

```lua
-- Search for DWORD value 500
gg.searchNumber("500", gg.TYPE_DWORD)

-- Search for Float between 1.0 and 2.5
gg.searchNumber("1.0~2.5", gg.TYPE_FLOAT)
```

#### `gg.refineNumber(text, [type], [encrypted], [sign]) -> boolean`
Filters existing search results.

```lua
-- Refine previously found results to 501
gg.refineNumber("501", gg.TYPE_DWORD)
```

#### `gg.clearResults()`
Clears all current search results.

#### `gg.getResultsCount() -> int`
Returns the total number of matched results.

---

### 2. Reading & Writing Values

#### `gg.getResults(count, [offset]) -> table`
Retrieves a list of results from the last search.

Returns an array of result tables:
```lua
local results = gg.getResults(5)
for i, item in ipairs(results) do
    print("Address: " .. string.format("0x%X", item.address))
    print("Value: " .. item.value)
    print("Type: " .. item.flags)
    print("Region: " .. item.name)
end
```

#### `gg.getValues(table) -> table`
Refreshes the live memory values for an array of address tables:

```lua
local items = {
    { address = 0x71BC670, flags = gg.TYPE_DWORD },
    { address = 0x71BC7F0, flags = gg.TYPE_FLOAT }
}
items = gg.getValues(items)
print("Live value: " .. items[1].value)
```

#### `gg.setValues(table) -> boolean`
Writes new values to memory addresses:

```lua
local items = {
    { address = 0x71BC670, flags = gg.TYPE_DWORD, value = "9999" },
    { address = 0x71BC7F0, flags = gg.TYPE_FLOAT, value = "25.0" }
}
gg.setValues(items)
```

#### `gg.editAll(value, type) -> int`
Replaces the values of **all** current search results with a new value. Returns the number of modified addresses.

```lua
-- Change all search results to 99999
gg.editAll("99999", gg.TYPE_DWORD)
```

---

### 3. Memory Regions

#### `gg.setRanges(mask)`
Sets which memory regions to search. Combine region flags using bitwise OR (`|`).

```lua
-- Scan Anonymous Heap and C_ALLOC (standard game variables)
gg.setRanges(gg.REGION_ANONYMOUS | gg.REGION_C_ALLOC)
```

#### `gg.getRanges() -> int`
Returns the current active region bitmask.

#### `gg.getRangesList([filter]) -> table`
Returns a list of all mapped memory ranges matching an optional filter string.

```lua
local ranges = gg.getRangesList("libBootloader")
for _, r in ipairs(ranges) do
    print(string.format("0x%X - 0x%X : %s", r.start, r['end'], r.name))
end
```

---

### 4. In-Game ImGui UI Dialogs

**LuaLoader** maps GameGuardian UI dialogs to smooth in-game **ImGui modal windows**. The Lua worker thread automatically pauses while the dialog is open and resumes immediately when the user interacts.

#### `gg.alert(message, [positive], [negative], [neutral], [options]) -> int`
Shows an alert dialog. Returns `1` for positive, `2` for negative, `3` for neutral.
* `options` *(number or table, optional)*: Custom width number (e.g. `400`) or size table `{ width = 400, height = 200 }`.

```lua
local choice = gg.alert("Do you want to enable Super Speed?", "Yes", "No", "Cancel")
if choice == 1 then
    gg.toast("Speed enabled!")
end
```

#### `gg.toast(message, [isFast])`
Displays a non-blocking in-game floating notification.

```lua
gg.toast("Mod Loaded Successfully!", true)
```

#### `gg.choice(items, [defaultChoice], [title], [options]) -> int or nil`
Shows a single-select interactive list menu. Returns the 1-based index selected, or `nil` if cancelled.
* `options` *(number or table, optional)*: Custom width number (e.g. `450`) or size table `{ width = 450, height = 300 }`.

```lua
-- Standard usage (auto-sizes to fit items)
local menu = gg.choice({
    "[1] Infinite Energy",
    "[2] Super Jump",
    "[3] Teleport to Home",
    "[4] Exit"
}, 1, "LuaLoader Menu")

if menu == 1 then
    -- toggle energy
elseif menu == 2 then
    -- toggle jump
end

-- Custom width (e.g. 480px wide)
local wideMenu = gg.choice(items, 1, "My Wide Menu", 480)
-- OR with explicit width and height
local sizedMenu = gg.choice(items, 1, "My Custom Menu", { width = 500, height = 350 })
```

#### `gg.multiChoice(items, [defaultSelections], [title], [options]) -> table or nil`
Shows a multi-select checkbox menu. Returns a table of `[index] = boolean`.
* `options` *(number or table, optional)*: Custom width number (e.g. `450`) or `{ width = 450, height = 300 }`.

```lua
local options = gg.multiChoice({
    "No-Clip",
    "God Mode",
    "Freeze Time"
}, { [1] = true }, "Toggle Features", 450)

if options and options[1] then
    print("No-Clip enabled")
end
```

#### `gg.prompt(prompts, [defaults], [types], [options]) -> table or nil`
Shows text input fields.
* `options` *(number or table, optional)*: Custom width number (e.g. `450`) or `{ width = 450, height = 300 }`.

```lua
local inputs = gg.prompt({
    speed = "Enter speed multiplier:",
    name = "Enter player nickname:"
}, {
    speed = "2.5",
    name = "SkyKid"
}, {
    speed = "number",
    name = "text"
}, 450)

if inputs then
    print("Speed: " .. inputs.speed)
    print("Name: " .. inputs.name)
### Interactive Menu Reopening Loop (`gg.isVisible`)

CanvasLuaLoader fully supports the standard GameGuardian infinite menu loop pattern. In this workflow:
1. The user taps an option in the menu.
2. The dialog automatically closes while the script executes background memory logic or enters a waiting state.
3. When the user wants the menu again, tapping the **`Open Script Menu`** button in CanvasLuaLoader or the **in-game floating button** triggers `gg.isVisible() -> true`.
4. The script detects the trigger and immediately displays the updated menu with new states or sub-options!

```lua
-- Standard GameGuardian Persistent Menu Pattern
local menuState = 1

function MAIN()
    local choice = gg.choice({
        "[1] Toggle Super Speed (Current: " .. (speedEnabled and "ON" or "OFF") .. ")",
        "[2] Patch Infinite Fly",
        "[3] Exit Script"
    }, 1, "Script Menu")

    if choice == 1 then
        speedEnabled = not speedEnabled
        -- Apply memory edit in background
        gg.toast("Speed is now " .. (speedEnabled and "ON" or "OFF"))
    elseif choice == 2 then
        -- Apply patch
        gg.toast("Patched fly successfully!")
    elseif choice == 3 then
        os.exit()
    end
end

-- Persistent event loop
while true do
    if gg.isVisible(true) then
        gg.setVisible(false)
        MAIN()
    end
    gg.sleep(100) -- Low CPU usage while waiting for user interaction
end
```

---

### 5. Utility & Process Functions

* `gg.sleep(ms)`: Suspends script execution for `ms` milliseconds.
* `gg.getTargetInfo() -> table`: Returns `{ packageName = "com.tgc.sky.android", processId = 1234 }`.
* `gg.getTargetPackage() -> string`: Returns the game package name.
* `gg.requireVersion(ver)`: Returns `true`.
* `gg.allocatePage(size) -> int`: Allocates a new readable/writable/executable memory page via `mmap`.

---

### 6. Constants Reference

#### Data Types
* `gg.TYPE_AUTO` (127)
* `gg.TYPE_BYTE` (1)
* `gg.TYPE_WORD` (2)
* `gg.TYPE_DWORD` (4)
* `gg.TYPE_FLOAT` (16)
* `gg.TYPE_QWORD` (32)
* `gg.TYPE_DOUBLE` (64)
* `gg.TYPE_XOR` (8)

#### Region Flags
* `gg.REGION_ALL` (`0xFFFFFFFF`)
* `gg.REGION_ANONYMOUS` (`0x1`)
* `gg.REGION_C_ALLOC` (`0x8`)
* `gg.REGION_C_DATA` (`0x10`)
* `gg.REGION_C_BSS` (`0x20`)
* `gg.REGION_C_HEAP` (`0x40`)
* `gg.REGION_JAVA_HEAP` (`0x80`)
* `gg.REGION_JAVA` (`0x100`)
* `gg.REGION_STACK` (`0x200`)
* `gg.REGION_CODE_APP` (`0x2`)
* `gg.REGION_CODE_SYS` (`0x4`)

#### Comparison Flags
* `gg.SIGN_EQUAL` (`0x1`)
* `gg.SIGN_NOT_EQUAL` (`0x2`)
* `gg.SIGN_LESS_OR_EQUAL` (`0x4`)
* `gg.SIGN_GREATER_OR_EQUAL` (`0x8`)
* `gg.SIGN_LESS` (`0x10`)
* `gg.SIGN_GREATER` (`0x20`)

---

## 4. LuaLoader Native (`canvas.*`) API (Optional Power-User Tools)

> [!NOTE]
> **`canvas.*` is purely optional.** Standard GameGuardian scripts do not require `canvas.*` and will run 100% identically using standard `gg.*` functions alone.
> 
> The `canvas.*` namespace is provided by the **LuaLoader mod** as a high-performance C++ shortcut toolbox for developers authoring mods running inside Canvas, exposing direct IDA signature scanning and native memory utilities.

#### `canvas.findPattern(ida_pattern, [lib_name]) -> int`
Scans for an IDA-style hex signature in `libBootloader.so` and returns the first match address (or `0`).
* `ida_pattern` *(string)*: Hex pattern with `?` or `??` wildcards.
* `lib_name` *(string, optional)*: Target library (default: `"libBootloader.so"`).

```lua
local addr = canvas.findPattern("FF 83 01 D1 F4 4F 02 A9 ? ? ? ?")
if addr ~= 0 then
    print(string.format("Found function at: 0x%X", addr))
end
```

#### `canvas.findPatternAll(ida_pattern, [lib_name]) -> table`
Finds all occurrences of a signature. Returns an array of integers.

#### `canvas.patch(address, hex_bytes) -> boolean`
Directly patches machine code/bytes at `address` with safety memory unlocking (`mprotect`).

```lua
-- Patch address with ARM64 NOP (1F 20 03 D5)
canvas.patch(0x71A0B2C, "1F 20 03 D5")

-- Patch function to return true (MOV W0, #1 ; RET)
canvas.patch(0x71A0B2C, "20 00 80 52 C0 03 5F D6")
```

#### `canvas.getLibBase([lib_name]) -> int`
Returns the virtual memory base address of `libBootloader.so` (or specified `.so`).

```lua
local base = canvas.getLibBase()
local targetFunc = base + 0x1A4C20
```

#### `canvas.getGameVersion() -> int`
Returns the Sky client build/version integer (e.g. `264120`).

#### `canvas.setDialogSize(width, [height])`
Sets global default minimum dimensions (in pixels) for all in-game `gg.*` dialogs and menus in this script. Pass `0` for height to keep vertical auto-resizing.

```lua
-- Make all subsequent menus at least 480px wide
canvas.setDialogSize(480, 0)
```

#### `canvas.log(message)`
Prints a debug log to Android `logcat` under the `CanvasLua` tag.

---

## 5. Examples & Script Templates

### Example 1: Value Search, Refine & Edit (Candles/Currency)

```lua
-- Author: Modder
-- Version: 1.0.0

gg.toast("Searching for Energy...", true)
gg.clearResults()
gg.setRanges(gg.REGION_C_ALLOC | gg.REGION_ANONYMOUS)

-- Search for initial value (e.g. 5 wings / energy)
gg.searchNumber("5.0", gg.TYPE_FLOAT)

if gg.getResultsCount() > 0 then
    -- Wait 3 seconds for user to spend energy in-game
    gg.sleep(3000)
    
    -- Refine to new value (e.g. 4 wings)
    gg.refineNumber("4.0", gg.TYPE_FLOAT)
    
    -- Edit all remaining matches to 999.0
    local edited = gg.editAll("999.0", gg.TYPE_FLOAT)
    gg.alert("Modified " .. edited .. " energy addresses!", "Awesome")
end
```

---

### Example 2: Pattern Scan & Assembly Patch (No-Clip/Godmode)

```lua
-- Author: Modder
-- Version: 1.2.0

local base = canvas.getLibBase("libBootloader.so")
canvas.log(string.format("libBootloader base: 0x%X", base))

-- Scan for collision check signature
local pattern = "E0 03 1F 32 ? ? ? ? F4 03 00 AA"
local addr = canvas.findPattern(pattern)

if addr ~= 0 then
    -- Patch with NOPs (1F 20 03 D5)
    canvas.patch(addr, "1F 20 03 D5")
    gg.toast("No-Clip enabled!")
else
    gg.alert("Failed to find pattern!", "OK")
end
```

---

### Example 3: Interactive Mod Menu UI

```lua
-- Author: Modder
-- Version: 2.0.0

while true do
    local menu = gg.choice({
        "[1] Set Custom Walk Speed",
        "[2] Patch Infinite Fly",
        "[3] Show Game Info",
        "[4] Exit Menu"
    }, 1, "Lua Mod Menu")

    if menu == nil or menu == 4 then
        gg.toast("Exited Menu")
        break
    elseif menu == 1 then
        local p = gg.prompt({ speed = "Enter speed:" }, { speed = "5.0" }, { speed = "number" })
        if p and p.speed then
            gg.toast("Speed set to " .. p.speed)
        end
    elseif menu == 2 then
        local addr = canvas.findPattern("20 00 80 52 C0 03 5F D6")
        if addr ~= 0 then
            canvas.patch(addr, "1F 20 03 D5")
            gg.toast("Patched successfully!")
        end
    elseif menu == 3 then
        local ver = canvas.getGameVersion()
        gg.alert("Sky Version: " .. ver, "Close")
    end

    gg.sleep(200)
end
```
