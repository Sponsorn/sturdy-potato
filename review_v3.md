# NaowhQOL Code Review v3

**Version:** v0.1.0 BETA
**Interface:** 120000 (Retail TWW 12.0), 110207 (Retail TWW 11.0.7)
**Review Date:** 2026-02-09
**Scope:** Full codebase review (~50 files, 20+ features)

---

## Overview

NaowhQOL is a WoW quality-of-life addon covering: combat timer, combat alerts, crosshair overlay, dragonriding vigor bar, buff monitor, consumable checker, GCD tracker, stealth/stance reminders, range check, focus cast bar, emote detection, combat logger, mouse cursor ring, auto keystone, faster loot, easy item destroy, loot warning suppression, queue skip, durability warnings, auto repair, UI clutter hiding, talent reminder, raid alerts, buff tracker, poison reminder, slash commands, and a custom settings import/export system.

The addon uses a fully custom settings UI (no AceGUI/AceConfig) with a sidebar-driven panel system. The codebase was previously refactored from ~4MB to ~2MB. This review identifies remaining issues across all severity levels.

**Note:** Review v2 listed several items as "Fixed Since Review #1". Those fixes exist in a *different codebase* and have NOT been applied to this repository. All previously-reported bugs that were marked fixed should be assumed to still exist here. Where applicable, they are re-listed below with current line references.

---

## CRITICAL Severity

### 1. SetCVar("MaxSpellQueueWindow", 150) -- Overrides User Setting Every Login
**File:** `Core.lua` line 731

```lua
SetCVar("MaxSpellQueueWindow", 150)
```

This forcibly resets a player's spell queue window to 150ms on every login. This is a WoW-wide CVar that persists across all characters and addons. Players who customize this value (many top-end PvP and PvE players use 200-400ms) will have their setting silently overridden with no opt-in, no config toggle, and no way to know this addon is responsible. This is the single most impactful bug -- it actively harms gameplay for any player who has tuned this CVar.

### 2. UNIT_SPELLCAST Events Without Unit Filtering -- O(n) Per-Cast Overhead
**File:** `Modules/GcdTrackerDisplay.lua` lines 635-639

```lua
events:RegisterEvent("UNIT_SPELLCAST_START")
events:RegisterEvent("UNIT_SPELLCAST_STOP")
events:RegisterEvent("UNIT_SPELLCAST_SUCCEEDED")
events:RegisterEvent("UNIT_SPELLCAST_CHANNEL_START")
events:RegisterEvent("UNIT_SPELLCAST_CHANNEL_STOP")
```

These should use `RegisterUnitEvent(event, "player")` to only receive events for the player's unit. Without filtering, these fire for EVERY unit casting in the game world. In a 20-man raid with everyone casting, this generates hundreds of extra event dispatches per second, each requiring the handler to check `if unit ~= "player" then return end`. Compare with `Config/MouseCursor.lua` lines 1314-1321 which correctly uses `RegisterUnitEvent`.

**Also in:** `Modules/EmoteDetection.lua` -- `UNIT_SPELLCAST_START` and `UNIT_SPELLCAST_CHANNEL_START` are not unit-filtered (fires for all units).

### 3. ClearFrame Uses O(n^2) select(i, ...) Pattern
**File:** `Data/Widgets.lua` lines 286-295

```lua
function ns.Widgets.ClearFrame(frame)
    local numChildren = select("#", frame:GetChildren())
    for i = 1, numChildren do
        local child = select(i, frame:GetChildren())
```

`frame:GetChildren()` is called on EVERY iteration, rebuilding the return-value list each time. With `select(i, ...)`, each call scans i elements, yielding O(n^2) total work. For frames with many children (settings panels), this is a significant performance hit. Should capture children once: `local children = {frame:GetChildren()}`.

### 4. Dead Code Files On Disk (Not in TOC, but Still Present)
**Files not loaded via TOC but still present in the project:**
- `Modules/BuffTrackerDisplay.lua` (624 lines) -- Fully functional module with its own DB, events, OnUpdate ticker. It initializes itself on PLAYER_LOGIN regardless of any enable toggle. References `ns.AuraWhitelist` which doesn't exist elsewhere. Has its own `Config/BuffTracker.lua` (247 lines) config panel.
- `Modules/PoisonReminderDisplay.lua` (194 lines) -- Rogue poison tracker. Starts a `C_Timer.NewTicker(15, ...)` on PLAYER_LOGIN for every rogue, running forever. Has no config panel or sidebar entry.
- `Modules/RaidAlerts.lua` (322 lines) -- Feast/cauldron/warlock spell alerts with `Config/RaidAlerts.lua` (420 lines) config panel. Registers `UNIT_SPELLCAST_SUCCEEDED` globally (not unit-filtered) for ALL units.
- `Modules/DragonRidingDisplay.lua` (437 lines) -- **Duplicate** of `Modules/Dragonriding.lua` using the OLD `ns.DB.config` pattern (prefixed keys like `dr_enabled`, `dr_barWidth`) instead of the new `NaowhQOL.dragonriding` pattern. Both modules register the same events, both create frames on the same screen position.

These 6 files (totaling ~2,244 lines) are not listed in the TOC so they are not loaded at runtime, but they add project clutter and confusion. They should be deleted or moved to an archive branch. If any were accidentally re-added to the TOC in the future, they would register events, start timers, and create duplicate frames with no way to disable them.

---

## HIGH Severity

### 5. Multiple Inconsistent Color Definition Systems
At least 5 separate color tables exist:

| Location | Table | Orange Value |
|---|---|---|
| `Core.lua:22` | `COLORS = { ORANGE = "ffa900" }` | hex string |
| `Data/Widgets.lua:15` | `ns.COLORS = { ORANGE = "ffa900" }` | hex string |
| `UI/Sidebar.lua:5` | `COLORS = { NAOWH_ORANGE = { r=1.00, g=0.66, b=0.00 } }` | r/g/b table |
| `UI/MainFrame.lua:12` | `NaowhOrange = { r=255/255, g=169/255, b=0/255 }` | r/g/b table |
| `Modules/CombatLoggerDisplay.lua:6` | `COLORS = { ORANGE = "ffa900" }` | hex string (local) |
| `Modules/TalentReminder.lua` | `local COLORS = { ... }` | hex string (local) |

The hex value "ffa900" in RGB is (255, 169, 0), and `g=0.66` rounds to (255, 168, 0) -- these are *different oranges*. Each module defines its own, making consistent branding impossible and creating maintenance burden. All should reference `ns.COLORS` from `Data/Widgets.lua`.

### 6. Massive Code Duplication Across Modules
Several code blocks are copied verbatim across multiple files:

**FRAME_BACKDROP table** (identical Tooltip-style backdrop):
- `Modules/BuffMonitorDisplay.lua`
- `Modules/ConsumableCheckerDisplay.lua`
- `Modules/StealthReminderDisplay.lua`
- `Modules/RaidAlerts.lua`
- `Modules/EmoteDetection.lua`

**SetFrameUnlocked function** (unlock mode with backdrop):
- `Modules/BuffMonitorDisplay.lua`
- `Modules/ConsumableCheckerDisplay.lua`
- `Modules/StealthReminderDisplay.lua`
- `Modules/EmoteDetection.lua`

**MakeSlot / icon creation helper**:
- `Modules/BuffMonitorDisplay.lua`
- `Modules/ConsumableCheckerDisplay.lua` (local MakeSlot duplicates DisplayUtils.MakeSlot)
- `Data/DisplayUtils.lua` (the shared version that should be used)

**PlaceSlider helper** (identical 4-line function):
- `Config/GcdTracker.lua`
- `Config/BuffTracker.lua`
- At least 4-5 other Config files

**GetSpellName / GetSpellIcon helpers** (legacy API fallback):
- `Config/GcdTracker.lua`
- `Config/MouseCursor.lua`
- `Modules/RaidAlerts.lua`
- `Modules/GcdTrackerDisplay.lua`

Each duplicated block is ~10-30 lines. With 5+ copies of each, this represents hundreds of lines of pure duplication that should be extracted to shared utilities.

### 7. Config Panels Use `ChatConfigCheckButtonTemplate` (Deprecated)
**Files:** Every single Config file uses this template for checkboxes.

`ChatConfigCheckButtonTemplate` is a deprecated Blizzard template from the chat config UI. The standard replacement is `UICheckButtonTemplate` (still valid and actively used in 12.0 Blizzard source). For the Settings panel specifically, Blizzard uses the `Settings.CreateCheckbox()` API, but for custom addon UI like NaowhQOL's settings panels, `UICheckButtonTemplate` is the correct choice.

### 8. `UIDropDownMenuTemplate` Still Used in BuffMonitor
**File:** `Config/BuffMonitor.lua` line 240

```lua
local classDrop = CreateFrame("Frame", "NaowhBMClassDrop", editor, "UIDropDownMenuTemplate")
```

`UIDropDownMenuTemplate` is deprecated in 12.0 (moved to deprecated/legacy compatibility areas in Blizzard source). The modern replacement is the `MenuUtil` description-based menu system. BuffMonitor is the only file still using this template. Note: `UICheckButtonTemplate` used in `Data/Widgets.lua:395` for collapsible sections is still valid and NOT deprecated.

### 9. InitializeDB Inconsistent Defaults Pattern
**File:** `Core.lua` lines 90-460

The `InitializeDB` function uses two completely different patterns for setting defaults:

**Pattern A** (ApplyDefaults -- used for 4 modules):
```lua
ns:ApplyDefaults(NaowhQOL.combatTimer, combatTimerDefaults)
```

**Pattern B** (40+ individual nil checks -- used for everything else):
```lua
if db.enabled == nil then db.enabled = true end
if db.unlock == nil then db.unlock = false end
-- ... 40 more lines per module
```

Pattern B is error-prone (easy to miss a field), verbose, and inconsistent with Pattern A. All modules should use `ApplyDefaults`.

### 10. Sidebar Missing Entries for Active Modules
Modules that are loaded, active, and functional but have NO sidebar entry:
- **AuctionHouseFilter** -- Sets AH expansion filter on load, no way to disable
- **FasterLoot** -- Only configurable from `Config/Modules.lua` misc toggles, but it's a module with its own file
- **BuffTrackerDisplay** -- Has its own config file (`Config/BuffTracker.lua`) and sidebar references in Locales, but **no sidebar entry** in `UI/Sidebar.lua`
- **RaidAlerts** -- Has its own config file (`Config/RaidAlerts.lua`) and sidebar references in Locales, but **no sidebar entry** in `UI/Sidebar.lua`
- **PoisonReminderDisplay** -- No config, no sidebar, no toggle

Conversely, locale strings exist for:
- `SIDEBAR_TAB_BUFF_TRACKER` -- defined in enUS.lua but not used in Sidebar.lua
- `SIDEBAR_TAB_RAID_ALERTS` -- defined in enUS.lua but not used in Sidebar.lua

### 11. FasterLoot References Undefined Config Key
**File:** `Modules/FasterLoot.lua`

References `ns.DB.config.lootThrottle` which is never defined in any defaults table. The `ns.DB.config` pattern isn't even used by this module (it accesses `NaowhQOL.misc.fasterLoot` for the enable toggle). If `lootThrottle` is meant to control the delay, it has no default, no slider, and likely always evaluates to nil.

---

## MEDIUM Severity

### 12. OnUpdate Handlers Run When Disabled

Multiple modules keep their OnUpdate handlers running even when the feature is toggled off:

| Module | File:Line | Cost |
|---|---|---|
| GCD Tracker | `GcdTrackerDisplay.lua` tick handler | Runs at 40fps even when disabled |
| Focus Cast Bar | `FocusCastBarDisplay.lua:543` | OnUpdate every frame when bar hidden |
| Range Check | `RangeCheckDisplay.lua` | 0.1s tick even when disabled |
| Combat Timer | `CombatTimerDisplay.lua` | Every-frame throttled to 1s |
| DragonRidingDisplay (dead) | `DragonRidingDisplay.lua:211` | 30fps OnUpdate |
| PoisonReminder (dead) | `PoisonReminderDisplay.lua` | 15s ticker runs forever for rogues |
| BuffTracker (dead) | `BuffTrackerDisplay.lua` | 0.3s OnUpdate |

These should nil-out their OnUpdate scripts or stop their tickers when the feature is disabled.

### 13. C_Timer.NewTicker Never Cancelled

Several modules start tickers that are never stopped:

- `Modules/BuffMonitorDisplay.lua:489` -- `C_Timer.NewTicker(1)` for raid buff scanning. `DisableBuffMonitor()` does NOT cancel this ticker.
- `Modules/ConsumableCheckerDisplay.lua:482` -- `C_Timer.NewTicker(1)` for consumable scanning. Same issue.
- `Modules/DurabilityDisplay.lua` -- `C_Timer.NewTicker(3)` starts at login, properly stops in combat, but never stops when feature is disabled.
- `Modules/PoisonReminderDisplay.lua:180` -- `C_Timer.NewTicker(15)` starts unconditionally for all rogues, runs forever.

### 14. Dragonriding -- Magic Numbers for Skyriding Detection
**File:** `Modules/Dragonriding.lua` lines 131, 134

```lua
if GetBonusBarIndex() == 11 and GetBonusBarOffset() == 5 then
if UnitPowerBarID("player") == 650 then
```

These undocumented internal values can change between patches. The 12.0 API provides `C_PlayerInfo.GetGlidingInfo()` which is the modern, documented way to detect skyriding state. Additionally, `Modules/Dragonriding.lua` references `BCDM_PowerBar` and `BCDM_SecondaryPowerBar` which are undocumented Blizzard internal frames.

**Also in dead code:** `Modules/DragonRidingDisplay.lua` has the same magic numbers.

### 15. StealthReminder -- Hardcoded Shapeshift Form Indices
**File:** `Modules/StealthReminderDisplay.lua` lines 128-142

```lua
local STEALTH_FORM_INDEX = {
    ROGUE = 1,
    DRUID = { stealth = 3, prowl = 4, catStealth = 7 },
}
local STANCE_FORM_INDEX = {
    WARRIOR = { 1, 2, 3 },
    PALADIN = { 1, 2, 3, 4 },
    ...
}
```

Shapeshift form indices change when talents modify the form list. Any talent that adds/removes a form will silently break stealth and stance detection. Should use `GetShapeshiftFormInfo()` to look up by spell ID instead of relying on ordinal indices.

### 16. ConsumableChecker Uses Deprecated GetItemInfo
**File:** `Modules/ConsumableCheckerDisplay.lua`

Uses `GetItemInfo(itemID)` which is deprecated in Retail 12.0. The modern API is `C_Item.GetItemInfo(itemID)`. Same issue in `Modules/AutoKeystone.lua` line 15.

### 17. RaidAlerts UNIT_SPELLCAST_SUCCEEDED Not Unit-Filtered
**File:** `Modules/RaidAlerts.lua` line 248

```lua
loader:RegisterEvent("UNIT_SPELLCAST_SUCCEEDED")
```

This fires for every spell cast by every unit in the game. Should use `RegisterUnitEvent` to filter to party/raid members only, or at minimum check unit membership before doing spell lookups.

### 18. Widgets.lua Picker Components Are Near-Identical (550+ Lines of Duplication)
**File:** `Data/Widgets.lua` lines 1132-2019

`CreateSoundPicker`, `CreateFontPicker`, `CreateBarStylePicker`, and `CreateTTSVoicePicker` are all ~150-200 lines each with identical structure:
1. Create selection button with backdrop
2. Create dropdown panel with overlay
3. Create search box with placeholder
4. Create list area with scroll
5. Create row buttons (BuildRows)
6. Filter/Refresh logic
7. Toggle dropdown

The only differences are: data source, row rendering (text vs font preview vs texture swatch), and preview button. A generic `CreateSearchablePicker(opts)` factory could replace all four, eliminating ~550 lines.

### 19. FontList.lua and BarStyleList.lua Are Structurally Identical
**Files:** `Data/FontList.lua`, `Data/BarStyleList.lua`

These two files share identical structure: dirty flag, Rebuild() with dedup, MarkDirty(), GetName(), HookLSM(). They differ only in the media type string ("font" vs "statusbar") and the built-in entries. A generic `CreateMediaList(mediaType, builtInEntries)` factory could replace both files.

### 20. ZoneUtil Fires Callbacks with Potentially Invalid Data
**File:** `Data/ZoneUtil.lua` line ~125

When retry logic exhausts its attempts (zone data unavailable), `NotifyCallbacks` is called with a snapshot containing `instanceID = 0` and `zoneName = ""`. Callbacks like `CombatLoggerDisplay.OnZoneChanged` use these values to construct cache keys (`instanceID .. ":" .. difficulty`), which could create invalid entries like `"0:0"`.

### 21. MouseCursor.lua Is 1526 Lines -- Config and Display in One File
**File:** `Config/MouseCursor.lua`

Despite being in the `Config/` directory, this file contains the ENTIRE mouse cursor ring implementation: frame creation, OnUpdate handlers, GCD tracking, cast animation, trail system, event handling, AND the config panel. At 1526 lines, it's the largest file in the project. Every other feature separates display (`Modules/`) from config (`Config/`). This should be split into `Modules/MouseCursorDisplay.lua` and `Config/MouseCursor.lua`.

### 22. Multiple Config Files Duplicate Default Tables
**File:** `Config/BuffTracker.lua` lines 21-41

The BuffTracker config file contains a full copy of the defaults table that's ALSO defined in `Modules/BuffTrackerDisplay.lua` lines 87-111 AND backfilled in `Core.lua`. Three copies of the same defaults exist.

### 23. DisableLootWarnings / HideUIClutter -- No Runtime Toggle
**Files:** `Modules/DisableLootWarnings.lua`, `Modules/HideUIClutter.lua`

Features use `hooksecurefunc` or `UnregisterAllEvents` which are irreversible:
- Alert suppression (`hooksecurefunc(AlertFrame, ...)`) -- permanent
- TalkingHead suppression (`hooksecurefunc(TalkingHeadFrame, ...)`) -- permanent
- Zone text suppression (`frame:SetScript("OnShow", frame.Hide)`) -- permanent
- Loot warning events registered once at login, never unregistered

Changing these settings requires `/reload` but the UI gives no indication of this.

### 24. FocusCastBar Uses Undocumented TWW "Secret Value" APIs
**File:** `Modules/FocusCastBarDisplay.lua`

Uses several APIs that return "secret values" in protected combat contexts:
- `UnitCastingDuration` / `UnitChannelDuration`
- `SetVertexColorFromBoolean` / `SetAlphaFromBoolean`
- `SetTimerDuration`

These APIs work but are not documented in the WoW API reference and may change without deprecation notice. The file has pcall guards for some but not all of these.

### 25. Locales Have Strings for Non-Existent/Dead Features
**File:** `Locales/enUS.lua`

Contains locale strings for features that either don't have sidebar entries or are dead code:
- `RAIDALERTS_*` -- ~30 strings for RaidAlerts (no sidebar entry)
- `BUFFTRACKER_*` -- ~25 strings for BuffTracker (no sidebar entry)
- `POISON_*` -- strings for PoisonReminder (dead code, no config)
- `SIDEBAR_TAB_BUFF_TRACKER` -- defined but never referenced in Sidebar.lua
- `SIDEBAR_TAB_RAID_ALERTS` -- defined but never referenced in Sidebar.lua

### 26. Core/Init.lua Contains Unreachable Code
**File:** `Core/Init.lua` line 3

```lua
if not ns then return end
```

WoW's addon loading system always provides the namespace table as the second argument. This check can never be false. It's not harmful but contributes to code that suggests uncertainty about the runtime.

Also in `Data/SettingsIO.lua` line 2: same pattern.

---

## LOW Severity

### 27. Core/PerfMonitor.lua Is Entirely No-Op
**File:** `Core/PerfMonitor.lua` (13 lines)

Both `PerfMonitor:Wrap` and `PerfMonitor:Report` are stubs that return the input or do nothing. Every module wraps its OnUpdate handler through `PerfMonitor:Wrap` which adds an extra function call layer (closure overhead) for zero benefit. Either implement the profiler or remove the wrapper calls.

### 28. SettingsIO Custom Serializer Instead of Standard Library
**File:** `Data/SettingsIO.lua` (528 lines)

Implements a complete Lua table serializer, deserializer, and base64 encoder/decoder from scratch. While the security motivation (avoiding `loadstring`) is valid, this is a lot of surface area for bugs. Libraries like LibSerialize and LibDeflate are battle-tested alternatives used by WeakAuras and other major addons.

### 29. SlashCommands.lua RunSlashCommand Uses ChatEdit Box Hack
**File:** `Modules/SlashCommands.lua` lines 178-198

```lua
local editBox = ChatFrame1EditBox or (DEFAULT_CHAT_FRAME and DEFAULT_CHAT_FRAME.editBox)
editBox:SetText(command)
ChatEdit_SendText(editBox)
editBox:SetText(original)
```

This manipulates the chat edit box to execute slash commands. It's a well-known hack but is fragile: it can interfere with user typing, may fire OnTextChanged handlers, and doesn't work if the chat frame doesn't exist. The modern approach is to look up the handler directly from `SlashCmdList`.

### 30. Crosshair Creates Textures at File Scope
**File:** `Modules/CrosshairDisplay.lua` lines 99-132

Eight textures (arms + shadows + dot + circle + their shadows) are created immediately at file load time, before PLAYER_LOGIN, before any config is loaded, and before the user has even opted into the crosshair feature. These textures persist in memory even if the crosshair is never enabled. Most other modules defer frame creation to PLAYER_LOGIN.

### 31. CombatAlertDisplay Redundant DB Access
**File:** `Modules/CombatAlertDisplay.lua` lines 50-57

```lua
local db = NaowhQOL.combatAlert
...
local db2 = NaowhQOL.combatAlert
local currentText = alertText:GetText() or (db2 and db2.enterText or "++ Combat")
```

`db` and `db2` reference the exact same table. The second access is redundant.

### 32. AuctionHouseFilter Accesses Internal Blizzard State
**File:** `Modules/AuctionHouseFilter.lua`

Accesses `AuctionHouseFrame.SearchBar.FilterButton.filters` which is internal Blizzard frame state, not a public API. This can break without notice if Blizzard restructures the AH UI.

### 33. TalentReminder May Use Internal API
**File:** `Modules/TalentReminder.lua`

Uses `ClassTalentHelper.SwitchToLoadoutByIndex` which appears to be from `Blizzard_ClassTalentUI` internals rather than a public C_ClassTalents API.

### 34. Config Panel RelayoutSections Pattern Duplicated
Every Config file contains a nearly identical `RelayoutSections` function that:
1. Iterates through section wrappers
2. Sets anchor points relative to previous sections
3. Calculates total height
4. Updates scroll content height

This pattern is repeated in: CombatAlert, CombatTimer, GcdTracker, FocusCastBar, Crosshair, MouseCursor, BuffMonitor, ConsumableChecker, StealthReminder, EmoteDetection, RangeCheck, Modules, Dragonriding, SlashCommands, ImportExport, Optimizations, RaidAlerts, BuffTracker (~18 files). It should be extracted to a `Widgets:LayoutSections(sections, scrollContent, baseHeight)` helper.

### 35. Old DragonRidingDisplay.lua Duplicates Active Module
**Files:** `Modules/Dragonriding.lua` (615 lines, active) and `Modules/DragonRidingDisplay.lua` (437 lines, dead code on disk)

The old `DragonRidingDisplay.lua` is not in the TOC so it doesn't currently run. However, it's a near-complete duplicate of the active `Dragonriding.lua`, using the old `ns.DB.config.dr_*` key pattern vs the new `NaowhQOL.dragonriding.*` pattern. Both files register the same events, use the same magic numbers, and create frames at the same anchor position. If the old file were accidentally re-added to the TOC, both modules would conflict. The old file should be deleted.

### 36. BuffTrackerDisplay Has Its Own Full InitializeDB
**File:** `Modules/BuffTrackerDisplay.lua` lines 84-120

This module creates its own `NaowhQOL.buffTracker` table in its own `InitializeDB` function, completely separate from the initialization in `Core.lua`. This means defaults can drift between the two locations, and Core.lua's `RestoreModuleDefaults` may not match what BuffTracker actually expects.

### 37. StaticPopupDialogs Redefined Per Click
**File:** `Data/Widgets.lua` line 2056

`CreateRestoreDefaultsButton` defines `StaticPopupDialogs["NAOWHQOL_RESTORE_DEFAULTS"]` inside the `OnClick` handler, meaning the dialog definition is recreated every time the button is clicked. The dialog should be defined once.

### 38. GroupUtil Fires Redundant Callbacks
**File:** `Data/GroupUtil.lua` line 86

`ForceRefresh()` always fires callbacks even if the class set hasn't changed (bypasses the equality check in `Refresh()`). If consumers call `ForceRefresh` frequently, this can cause unnecessary UI updates.

### 39. CombatLoggerDisplay Hardcodes Difficulty ID
**File:** `Modules/CombatLoggerDisplay.lua` line 88

```lua
elseif zoneData.instanceType == "party" and zoneData.difficulty == 8 then
```

Difficulty ID 8 is Mythic Keystone. This magic number should be a named constant, and other relevant difficulties (Challenge Mode, Mythic dungeon) may also need to be included.

### 40. Config Files Create Frames Inside CachedPanel Closures
Every Config file creates UI elements inside a `W:CachedPanel(cache, key, parent, function(f) ... end)` closure. While the caching prevents re-creation, the pattern means all UI construction happens lazily on first panel visit. If a user never visits a panel, those frames are never created (good), but the first visit to complex panels (GcdTracker with its spell blocklist, MouseCursor with its trail/GCD subsections) can cause a visible hitch.

### 49. FocusCastBar Creates Color Objects Every Tick (GC Pressure)
**File:** `Modules/FocusCastBarDisplay.lua` lines 142-143

The `UpdateBarColor` function is called every tick (~30fps) while the cast bar is visible. Each call creates two new `CreateColor(r, g, b)` objects, which generates garbage for the Lua GC. These color objects should be cached and reused. Over a long fight with frequent focus casts, this accumulates significant throwaway allocations.

### 50. SlashCommands Cleanup Doesn't Actually Unregister Commands
**File:** `Modules/SlashCommands.lua` line 383

The `Cleanup` function calls `wipe(registeredCommands)` but does NOT remove the corresponding `_G["SLASH_*"]` entries or `SlashCmdList` handlers. The slash commands remain functional even after "cleanup", making the function misleading. It should iterate `registeredCommands` and call the existing `Unregister` logic on each before wiping.

### 51. hasSecretInterruptible Dead Variable
**File:** `Modules/FocusCastBarDisplay.lua` line 330

The variable `hasSecretInterruptible` is set in the event handler but never read anywhere in the code. It appears to be a development leftover.

### 52. Font Path Hardcoded in Every Display Module
**Files:** `CombatTimerDisplay.lua`, `CombatAlertDisplay.lua`, `BuffMonitorDisplay.lua`, `ConsumableCheckerDisplay.lua`, `StealthReminderDisplay.lua`, `RangeCheckDisplay.lua`, `EmoteDetection.lua`, `DurabilityDisplay.lua`

The string `"Interface\\AddOns\\NaowhQOL\\Assets\\Fonts\\Naowh.ttf"` (or `"Assets\\Fonts\\Naowh.ttf"`) is hardcoded in each module as the default font path. This should be a single constant in `ns` (e.g., `ns.DEFAULT_FONT`). Same applies to the default sound ID `8959` which appears in 5+ modules.

---

### 41. Global Function Leaks in Config/SlashCommands.lua
**File:** `Config/SlashCommands.lua`

Seven functions are defined without `local`, polluting the global namespace (`_G`):
- `CreateAddDialog`, `ShowAddDialog`, `StartFrameTest`, `CreateTestDialog`, `TestNextFrame`, `RecordTestResult`, `PrintTestResults`

Global leaks can cause taint issues (Blizzard's secure frame system) and name collisions with other addons. Additionally, `StartFrameTest`/`CreateTestDialog`/`TestNextFrame`/`RecordTestResult`/`PrintTestResults` appear to be developer-only debug tooling that should not ship in a release build.

### 42. Config Files Missing Localization (Hardcoded English Strings)
**Files:** `Config/CombatLogger.lua`, `Config/TalentReminder.lua`, `Config/ImportExport.lua`

These three config files do NOT use `local L = ns.L` and instead have hardcoded English strings throughout:
- CombatLogger: "Enable Combat Logger", "SAVED INSTANCES", "Status:", "NOT LOGGING", etc.
- TalentReminder: "Enable Talent Reminder", "SAVED LOADOUTS", "Clear All Loadouts", etc.
- ImportExport: "Export Settings", "Import", "Load", "Import Selected", etc.

All other config files use localization. These three are inconsistent.

### 43. Config Rebuilds Create New Frames Without Pooling (Frame Leak)
**Files:** `Config/CombatLogger.lua` (`rebuildInstances`), `Config/TalentReminder.lua` (`rebuildLoadouts`)

Both files recreate all child frames every time the list is rebuilt. Old frames are hidden but never destroyed or reused, leading to frame accumulation over time. Other config files (BuffMonitor, ConsumableChecker) use frame pools correctly.

### 44. Optimizations Creates 26+ Independent OnUpdate Timer Frames
**File:** `Config/Optimizations.lua`

Each CVar row creates its own OnUpdate-driven timer frame for auto-refresh. With 26+ individual CVars, this creates 26+ independent OnUpdate handlers, each checking elapsed time every frame. A single shared ticker would be far more efficient.

Also: the `OptEngine` table declares `lastFPSCheck` and `performanceData` fields that are never used. The `initFrame` event handler at line 1297-1306 registers for `ADDON_LOADED` and `PLAYER_ENTERING_WORLD` but does nothing meaningful in either handler (just unregisters itself).

---

## Architecture & Modularity Observations

### Module Loading Strategy
The TOC loads ALL modules unconditionally. Dead code (BuffTracker, PoisonReminder, RaidAlerts, old DragonRiding) registers events, creates frames, and starts timers even though users can't interact with them. A proper module registry pattern (like Ace3's `OnEnable`/`OnDisable`) would let modules opt out of initialization entirely.

### Settings Architecture Split
The codebase has TWO settings storage patterns:
1. **Old pattern** (`ns.DB.config.dr_enabled`): Used by `DragonRidingDisplay.lua` (dead code)
2. **New pattern** (`NaowhQOL.dragonriding.enabled`): Used by `Dragonriding.lua` (active code)

The migration is incomplete -- some dead code files still use the old pattern. Core.lua has migration logic but it only runs once per account.

### Sidebar vs Loaded Modules Mismatch
**Sidebar has entries for (19):** Combat Timer, Combat Alert, Combat Logger, GCD Tracker, Mouse Ring, Crosshair, Focus Cast Bar, Dragonriding, Buff Monitor, Consumable Checker, Stealth Reminder, Range Check, Talent Reminder, Emote Detection, Optimizations, Misc (Modules), Profiles, Slash Commands

**Loaded but NO sidebar entry (5):** AuctionHouseFilter, BuffTrackerDisplay, PoisonReminderDisplay, RaidAlerts, DragonRidingDisplay (old)

**Has config file but NO sidebar entry (2):** Config/BuffTracker.lua, Config/RaidAlerts.lua

### File Size Distribution
Top files by line count:
1. `Config/MouseCursor.lua` -- 1526 lines (config + full display implementation)
2. `Data/Widgets.lua` -- 2086 lines (custom UI toolkit)
3. `Config/GcdTracker.lua` -- 457 lines
4. `Modules/GcdTrackerDisplay.lua` -- 718 lines
5. `Modules/FocusCastBarDisplay.lua` -- 662 lines
6. `Modules/BuffTrackerDisplay.lua` -- 624 lines (dead code)
7. `Modules/Dragonriding.lua` -- 615 lines
8. `Locales/enUS.lua` -- 674 lines
9. `Data/SettingsIO.lua` -- 528 lines
10. `Modules/ConsumableCheckerDisplay.lua` -- 556 lines

`Config/MouseCursor.lua` + `Data/Widgets.lua` alone account for ~3600 lines (roughly 30% of the codebase).

---

## Summary Table

| # | Severity | Issue | File(s) |
|---|---|---|---|
| 1 | CRITICAL | SetCVar overrides spell queue window | Core.lua:731 |
| 2 | CRITICAL | UNIT_SPELLCAST not unit-filtered | GcdTrackerDisplay.lua:635, EmoteDetection.lua |
| 3 | CRITICAL | ClearFrame O(n^2) select pattern | Widgets.lua:286 |
| 4 | CRITICAL | Dead code files on disk (not in TOC, project clutter) | BuffTracker, PoisonReminder, RaidAlerts, DragonRidingDisplay |
| 5 | HIGH | 5+ inconsistent color definition systems | Core, Widgets, Sidebar, MainFrame, CombatLogger, TalentReminder |
| 6 | HIGH | Massive code duplication (FRAME_BACKDROP, SetFrameUnlocked, MakeSlot, PlaceSlider, GetSpellName) | ~15 files |
| 7 | HIGH | Deprecated ChatConfigCheckButtonTemplate (replace with UICheckButtonTemplate) | All Config files |
| 8 | HIGH | Deprecated UIDropDownMenuTemplate in BuffMonitor (replace with MenuUtil) | Config/BuffMonitor.lua:240 |
| 9 | HIGH | Inconsistent InitializeDB defaults patterns | Core.lua |
| 10 | HIGH | Sidebar missing entries for active modules | Sidebar.lua vs BuffTracker, RaidAlerts, AuctionHouseFilter, PoisonReminder |
| 11 | HIGH | FasterLoot references undefined config key | FasterLoot.lua |
| 12 | MEDIUM | OnUpdate handlers run when disabled | 7+ modules |
| 13 | MEDIUM | C_Timer.NewTicker never cancelled | BuffMonitor, ConsumableChecker, Durability, PoisonReminder |
| 14 | MEDIUM | Dragonriding magic numbers | Dragonriding.lua:131,134 |
| 15 | MEDIUM | Hardcoded shapeshift form indices | StealthReminderDisplay.lua:128 |
| 16 | MEDIUM | Deprecated GetItemInfo API | ConsumableCheckerDisplay.lua, AutoKeystone.lua |
| 17 | MEDIUM | RaidAlerts UNIT_SPELLCAST not filtered | RaidAlerts.lua:248 |
| 18 | MEDIUM | 550+ lines of near-identical picker code | Widgets.lua:1132-2019 |
| 19 | MEDIUM | FontList/BarStyleList structural duplication | FontList.lua, BarStyleList.lua |
| 20 | MEDIUM | ZoneUtil fires callbacks with invalid data | ZoneUtil.lua:~125 |
| 21 | MEDIUM | MouseCursor config+display in one 1526-line file | Config/MouseCursor.lua |
| 22 | MEDIUM | Defaults tables duplicated 3x (Core, Module, Config) | BuffTracker, others |
| 23 | MEDIUM | No runtime toggle for irreversible hooks | DisableLootWarnings, HideUIClutter |
| 24 | MEDIUM | Undocumented "secret value" APIs | FocusCastBarDisplay.lua |
| 25 | MEDIUM | Locale strings for dead/hidden features | enUS.lua |
| 26 | MEDIUM | Unreachable `if not ns` guard | Init.lua, SettingsIO.lua |
| 27 | LOW | PerfMonitor is entirely no-op with overhead | PerfMonitor.lua |
| 28 | LOW | Custom serializer instead of battle-tested library | SettingsIO.lua |
| 29 | LOW | SlashCommands chat edit box hack | SlashCommands.lua:178 |
| 30 | LOW | Crosshair textures created at file scope | CrosshairDisplay.lua:99 |
| 31 | LOW | Redundant DB access | CombatAlertDisplay.lua:50 |
| 32 | LOW | Internal Blizzard frame state access | AuctionHouseFilter.lua |
| 33 | LOW | Potentially internal API usage | TalentReminder.lua |
| 34 | LOW | RelayoutSections duplicated ~18 times | All Config files |
| 35 | LOW | Old DragonRidingDisplay.lua duplicates active module (dead code) | Dragonriding.lua + DragonRidingDisplay.lua |
| 36 | LOW | BuffTracker has own InitializeDB separate from Core | BuffTrackerDisplay.lua:84 |
| 37 | LOW | StaticPopupDialog redefined per click | Widgets.lua:2056 |
| 38 | LOW | GroupUtil ForceRefresh bypasses change detection | GroupUtil.lua:86 |
| 39 | LOW | Hardcoded difficulty ID 8 for M+ | CombatLoggerDisplay.lua:88 |
| 40 | LOW | Config frames created inside closures (first-visit hitch) | All Config files |
| 41 | HIGH | Global function leaks pollute _G namespace | Config/SlashCommands.lua |
| 42 | MEDIUM | 3 config files have hardcoded English strings (no localization) | Config/CombatLogger, TalentReminder, ImportExport |
| 43 | MEDIUM | Config rebuilds create new frames without pooling (frame leak) | Config/CombatLogger.lua, Config/TalentReminder.lua |
| 44 | MEDIUM | Optimizations creates 26+ independent OnUpdate timer frames | Config/Optimizations.lua |
| 45 | LOW | Unused localized variables (ConsoleExec, GetFramerate, GetNetStats) | Config/Optimizations.lua:244-246 |
| 46 | LOW | Redundant double GetItemInfoInstant call on same item | Config/ConsumableChecker.lua:398 |
| 47 | LOW | StyleTypeBtn function defined twice (shadowed) | Config/SlashCommands.lua:372,555 |
| 48 | LOW | Variable shadowing (offsetXSlider, offsetYSlider) | Config/Dragonriding.lua:138/144 vs 327/333 |
| 49 | MEDIUM | FocusCastBar creates 2 CreateColor objects every tick (~30fps GC pressure) | FocusCastBarDisplay.lua:142-143 |
| 50 | LOW | SlashCommands Cleanup() doesn't unregister commands from _G/SlashCmdList | SlashCommands.lua:383 |
| 51 | LOW | hasSecretInterruptible is set but never read (dead variable) | FocusCastBarDisplay.lua:330 |
| 52 | LOW | Font path hardcoded in every module instead of centralized constant | CombatTimerDisplay, CombatAlertDisplay, BuffMonitorDisplay, + 5 more |

---

## Recommended Priority Order

1. **Remove dead code files** from TOC (BuffTracker, PoisonReminder, RaidAlerts, old DragonRiding) -- eliminates ~1500 lines, several running tickers, and unfiltered event handlers
2. **Fix SetCVar override** -- remove or add opt-in toggle
3. **Add unit filtering** to UNIT_SPELLCAST events in GcdTracker, EmoteDetection, RaidAlerts
4. **Fix ClearFrame** O(n^2) pattern
5. **Consolidate color definitions** into single ns.COLORS
6. **Extract duplicated code** (FRAME_BACKDROP, SetFrameUnlocked, MakeSlot, PlaceSlider, GetSpellName, RelayoutSections) into shared utilities
7. **Stop OnUpdate/tickers** when features are disabled
8. **Update deprecated templates** (ChatConfigCheckButtonTemplate -> UICheckButtonTemplate, UIDropDownMenuTemplate -> MenuUtil)
9. **Unify InitializeDB** to use ApplyDefaults for all modules
10. **Add missing sidebar entries** or remove orphaned modules/locale strings
