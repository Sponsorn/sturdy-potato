# NaowhQOL Code Review

**Version:** v0.1.0 BETA
**Interface:** 120000 (Retail TWW), 110207 (Classic)
**Review Date:** 2026-02-08
**Verified Against:** Blizzard WoW UI Source v12.0.0.65655 (TWW)

---

## Overview

NaowhQOL is a WoW quality-of-life addon with ~50 files covering 20+ features: combat timer, combat alerts, crosshair overlay, dragonriding vigor bar, buff/consumable monitoring, GCD tracker, stealth/stance reminders, range check, focus cast bar, emote detection, combat logger, auto keystone, faster loot, easy item destroy, loot warning suppression, queue skip, durability warnings, auto repair, and UI clutter hiding. The addon uses a custom settings UI (no AceGUI/AceConfig) with a sidebar-driven panel system.

The codebase is functional and ambitious but has several categories of issues ranging from crash-causing bugs to performance inefficiencies and dead code.

---

## CRITICAL / HIGH Severity

### 3. FocusCastBar Uses TWW "Secret Value" APIs
**File:** `Modules/FocusCastBarDisplay.lua` lines 108-131, 278-282

```lua
local cooldownDuration = C_Spell.GetSpellCooldownDuration(spellId)
local isReady = cooldownDuration:IsZero()
barTexture:SetVertexColorFromBoolean(isReady, readyColor, cdColor)
castBarFrame:SetAlphaFromBoolean(currentNotInterruptible, 0, 1)
local duration = UnitCastingDuration("focus")
progressBar:SetTimerDuration(duration, ...)
```
These are Blizzard's "opaque value" APIs designed to prevent cooldown/casting info leaking in PvP. They are documented in Blizzard's `APIDocumentationGenerated` source (e.g. `SpellDocumentation.lua`, `SimpleRegionAPIDocumentation.lua`, `UnitDocumentation.lua`) with `SecretArguments` and `SecretReturns` markers. They are official Mainline-only APIs and will not exist on Classic clients. The TOC declares `## Interface: 110207` (Classic support), where these APIs will produce hard errors.

---

### 4. TOC Claims Classic Support But Modules Use Retail-Only APIs
**File:** `NaowhQOL.toc` line 1

The TOC specifies `## Interface: 120000, 110207` -- both Retail TWW and Classic. However, many Retail-only APIs are used without compatibility checks:
- `C_UnitAuras.GetAuraDataByIndex` (BuffMonitor, ConsumableChecker, DisplayUtils)
- `C_Spell.GetSpellCharges`, `C_Spell.GetSpellCooldown` (Dragonriding, GcdTracker)
- `C_PlayerInfo.GetGlidingInfo` (Dragonriding)
- `C_ChallengeMode.*` (AutoKeystone, CombatLogger)
- `C_AddOns.GetNumAddOns` (Profiler)
- `GetSpecialization`, `GetSpecializationInfo` (SpecUtil, StealthReminder)
- `ColorPickerFrame:SetupColorPickerAndShow` (Widgets -- introduced in 10.2.5)

Loading on the Classic client (110207) would produce immediate Lua errors in multiple modules. None of these APIs have fallback checks or `pcall` guards.

---

## MEDIUM Severity

### 5. `MaxSpellQueueWindow` Is Not a Valid CVar
**File:** `Core.lua:555`

```lua
SetCVar("MaxSpellQueueWindow", 150)
```
This CVar does not exist in Blizzard's source. `SpellQueueWindow` (set on line 553) is the correct one. This call silently fails or throws an error depending on the client version.

---

### 7. Buff Tracker Reset Popup Nils the Wrong Key
**File:** `Data/StaticPopups.lua:74`

```lua
NaowhQOL.buffTracker = nil  -- should be NaowhQOL.buffMonitor
```
The buff tracker was migrated to `buffMonitor` (see Core.lua lines 166-172). Clicking the reset popup nils a key that may no longer exist and does NOT reset the current buff monitor settings. The user thinks they reset their settings, but nothing happens.

---

### 8. Deprecated `UIDropDownMenuTemplate` -- Taint Risk
**File:** `Data/Widgets.lua` lines 327-361

The entire legacy dropdown system (`UIDropDownMenu_SetWidth`, `UIDropDownMenu_Initialize`, `UIDropDownMenu_AddButton`, etc.) was deprecated in WoW 10.0. It still works via a compatibility layer (`Blizzard_SharedXML/Mainline/UIDropDownMenu.lua`), but is known to cause taint when interacted with during or shortly after combat, leading to "action blocked" errors. The modern replacement is `MenuUtil` (used in 10+ Blizzard UI files).

---

### 9. `SetUserPlaced(true)` Conflicts With Addon Position Saving
**File:** `Data/Widgets.lua:1207`

`SetUserPlaced(true)` tells WoW to save/restore frame positions via `layout-local.txt`. This competes with the addon's own position saving in `OnDragStop`. On login, WoW restores the frame to its `layout-local.txt` position, then the addon re-applies from SavedVariables. This causes visible frame "jumping" and the wrong position can win.

---

### 10. O(n^2) `ClearFrame` Implementation
**File:** `Data/Widgets.lua:286`

```lua
for i = 1, frame:GetNumChildren() do
    local child = select(i, frame:GetChildren())
```
Each `select(i, frame:GetChildren())` rebuilds the full vararg and scans to position i. For frames with many children, this is measurably slow. Should use `{frame:GetChildren()}` + `ipairs`.

---

### 11. O(n) LRU Cache Scan on Every Hit
**File:** `Data/DisplayUtils.lua` lines 17-24

```lua
for i, id in ipairs(cacheOrder) do
    if id == spellId then
        table.remove(cacheOrder, i)  -- O(n) shift
```
On every cache hit, up to 100 entries are linearly scanned and shifted. For a hot path during aura display updates, this adds up. Should use a hash-map indexed approach.

---

### 12. CombatLogger Potential Nil Dereference
**File:** `Modules/CombatLoggerDisplay.lua:151`

```lua
C_Timer.After(0.5, function()
    OnZoneChanged(ns.ZoneUtil.GetCurrentZone())
end)
```
If `GetCurrentZone()` returns nil, `OnZoneChanged` tries to access `zoneData.instanceType` which throws "attempt to index a nil value". The 0.5s delay mitigates but doesn't guarantee zone data is ready.

---

### 13. DisableLootWarnings / HideUIClutter -- No Runtime Toggle
**Files:** `Modules/DisableLootWarnings.lua`, `Modules/HideUIClutter.lua`

Multiple features use `hooksecurefunc` or `UnregisterAllEvents` which are irreversible. Settings changes require a `/reload` to take effect:
- Alert suppression (`hooksecurefunc(AlertFrame, "AddAlertFrame", ...)`) -- permanent
- TalkingHead suppression (`hooksecurefunc(TalkingHeadFrame, "Show", ...)`) -- permanent
- Zone text suppression (`frame:SetScript("OnShow", frame.Hide)`) -- permanent
- Loot warning events registered once at login, never unregistered

---

### 14. GcdTracker Spellcast Events Not Unit-Filtered (Performance Only)
**File:** `Modules/GcdTrackerDisplay.lua` lines 484-489

```lua
eventFrame:RegisterEvent("UNIT_SPELLCAST_START")
eventFrame:RegisterEvent("UNIT_SPELLCAST_SUCCEEDED")
-- etc.
```
These fire for ALL units. In a 20-player raid, these fire many times per second. The handler correctly filters `unit ~= "player"`, so **this is functionally safe** -- but the dispatch overhead remains. Should use `RegisterUnitEvent("UNIT_SPELLCAST_START", "player")` for efficiency.

---

### 16. Dragonriding -- Magic Numbers for Skyriding Detection
**File:** `Modules/Dragonriding.lua` lines 131, 134

```lua
if GetBonusBarIndex() == 11 and GetBonusBarOffset() == 5 then
if UnitPowerBarID("player") == 650 then
```
These undocumented internal values can change between patches without notice. If they change, the entire Dragonriding module silently stops working.

---

### 17. StealthReminder -- Hardcoded Shapeshift Form Indices
**File:** `Modules/StealthReminderDisplay.lua` lines 113-127

```lua
DRUID = { [1] = 4, [2] = 2, [3] = 1 },
WARRIOR = { [1] = 2, [2] = 2, [3] = 1 },
```
Shapeshift form indices are determined by `GetShapeshiftFormInfo` order, which has changed in past patches. Hero Talents in TWW could further alter these. Wrong indices produce false "CHECK STANCE" warnings.

---

### 18. CVar Case Mismatch [UNVERIFIED]
**File:** `Config/Optimizations.lua`

```lua
SetCVar("sunshafts", "0")  -- line 569, lowercase 's'
-- but elsewhere:
["sunShafts"] = "2",        -- line 748, capital 'S'
```
CVar names can be case-sensitive. If the CVar is actually `sunShafts`, the lowercase `SetCVar` silently fails.

**Note:** Neither `sunshafts` nor `sunShafts` appears in the Blizzard UI source (CVars are defined in the engine, not UI code). This claim cannot be verified from the UI source alone.

---

### 19. LowEndOptimization Sets CVars Not Tracked by SaveCurrentSettings
**File:** `Config/Optimizations.lua`

`SaveCurrentSettings()` (line 254) iterates over `OPTIMAL_FPS_CVARS` to save current values. But the low-end optimization path sets additional CVars NOT in that table:
- `graphicsTextureQuality` (line 507)
- `graphicsTextureFiltering` (line 550)
- `lightingQuality` (line 575)
- `maxFPSLoading` (line 582)

If a user applies Low-End mode then clicks Revert, these CVars are NOT restored.

---

### 20. Profiler AVG and MAX Columns Show Identical Data
**File:** `Config/Profiler.lua` lines 287, 292

```lua
rows[i].avg:SetText(format("%.3f", data[i].c))
rows[i].peak:SetText(format("%.2f", data[i].c))
```
Both columns display `data[i].c` from `GetAddOnCPUUsage(i)`. The headers say "CPU AVG" and "CPU MAX" but they show the same value with different decimal precision.

---

### 22. Emote Detection Content Height Immediately Overwritten
**File:** `Config/EmoteDetection.lua` lines 393 vs 400

`BuildAutoEmoteList()` dynamically calculates content height (line 393), then immediately after, line 400 overwrites it to a static 200px. The initial render always uses 200px regardless of actual content count.

---

### 23. `GroupUtil` Passes Mutable `cachedClasses` to Callbacks
**File:** `Data/GroupUtil.lua:40`

```lua
pcall(fn, cachedClasses)
```
Callbacks receive a direct reference to the internal cache. A misbehaving callback could mutate this table, corrupting state for all other consumers.

---

### 24. ZoneUtil Retry Fires Callbacks With Potentially Invalid Data
**File:** `Data/ZoneUtil.lua:114`

When retries are exhausted and zone info is still unresolved (difficulty 0 inside an instance), `NotifyCallbacks()` fires with incorrect data. Consumers may misidentify dungeon difficulty.

---

### 25. AutoKeystone Missing Nil Check on C_Item.GetItemInfo
**File:** `Modules/AutoKeystone.lua:15`

`C_Item.GetItemInfo` returns nil when item data hasn't been cached from the server. No retry mechanism exists (should use `GET_ITEM_INFO_RECEIVED`). The keystone is silently never slotted on cold login.

---

## LOW Severity

### 26. Permanently Running OnUpdate Handlers When Disabled
**Files:** `RangeCheckDisplay.lua:126`, `CombatTimerDisplay.lua:119`, `FocusCastBarDisplay.lua:437`, `GcdTrackerDisplay.lua:308`

All have OnUpdate scripts that run every frame even when the feature is disabled. Each returns early on the disabled check, but the per-frame callback overhead (function call + accumulator + comparison) exists across 4 modules permanently.

---

### 27. Permanently Running C_Timer Tickers When Disabled
**Files:**
- `DurabilityDisplay.lua:72` -- 3s ticker, runs even when `durabilityWarning` is off
- `ConsumableCheckerDisplay.lua:471` -- 1s ticker, never cancelled by `DisableConsumableChecker`
- `BuffMonitorDisplay.lua:452` -- 1s ticker, never cancelled by `DisableBuffMonitor`

---

### 30. ConsumableChecker `SetAttribute` With String Instead of Number
**File:** `Modules/ConsumableCheckerDisplay.lua:389`

```lua
slot.click:SetAttribute("target-slot", tostring(m.targetSlot))
```
The `target-slot` attribute expects a number. String coercion may cause weapon enchant items to fail to target the correct slot.

---

### 31. Main Frame Not Clamped to Screen
**File:** `UI/MainFrame.lua` lines 7-15

`SetClampedToScreen(true)` is never called. The 950x650 settings window can be dragged entirely off-screen with no way to retrieve it (except `/reload` which resets to center).

---

### 32. Dual/Inconsistent Color Definitions Across Files
Colors are defined independently with different values in `Core.lua`, `Data/Widgets.lua`, `UI/Sidebar.lua`, and `UI/MainFrame.lua`. The addon's "orange" is `ffa900` in one place, `{255/255, 169/255, 0}` in another, and `{1.00, 0.66, 0.00}` in a third -- these are mathematically different colors.

---

### 33. BarStyleList and FontList Built at Parse Time
**Files:** `Data/BarStyleList.lua:78-79`, `Data/FontList.lua:115-116`

`Rebuild()` and `HookLSM()` run before `ADDON_LOADED`. LibSharedMedia may not be fully populated yet. The initial list only contains built-in entries; LSM-registered entries from other addons are missing until the first registration callback.

---

### 34. `SetCVar`/`GetCVar` Local Cache Asymmetry
**File:** `Config/Optimizations.lua` lines 241-242

```lua
local SetCVar = SetCVar
local GetCVar = GetCVar
```
These are declared AFTER `SaveCurrentSettings` (defined at line 254), which captures the global versions. Functions after line 242 use the locals. If any addon hooks `SetCVar` globally, the save function sees the hook but the apply functions do not.

---

## DEAD CODE / ORPHANED FILES

### Files Not in TOC (Never Loaded)

| File | Lines | Notes |
|------|-------|-------|
| `Modules/BuffTrackerDisplay.lua` | ~600 | Superseded by BuffMonitorDisplay |
| `Modules/PoisonReminderDisplay.lua` | ~200 | Orphaned module |
| `Modules/RaidAlerts.lua` | ~300 | Orphaned module |
| `Config/BuffTracker.lua` | 248 | Calls nonexistent `W:CreateFontDropdown` |
| `Config/PoisonReminder.lua` | 188 | Uses deprecated templates |
| `Config/RaidAlerts.lua` | 401 | References nonexistent display module |
| `Config/test.lua` | 0 | Empty file |

Total: ~1,937 lines of dead code that should be removed.

### Other Dead Code
- `Config/Optimizations.lua:24` -- Unused `original` variable for AlertFrame hook
- `Config/FocusCastBar.lua:221,236` -- Unused return values `soundCB` and `ttsCB`
- `Core/Init.lua` -- Entire file is a nil check that can never trigger (WoW addon `ns` is never nil)
- `Core/PerfMonitor.lua` -- Both functions are no-op stubs (monitoring stripped for release)

---

## Architecture / Design Observations

### No Runtime Enable/Disable Pattern
Most modules create frames and register events at file load time, regardless of whether the feature is enabled. There is no consistent pattern for dynamically enabling/disabling features -- some use early returns in handlers (good), some use permanent hooks (irreversible), and none properly start/stop their OnUpdate or C_Timer tickers.

### No Error Boundaries
When one module errors, it can prevent subsequent modules from loading (if the error is at file parse time). There are no `pcall` wrappers around module initialization. A single bug in any module can take down the entire addon.

### Mixed Initialization Patterns
- Some defaults use `ApplyDefaults()` with a table (combatTimer, combatAlert)
- Some use 40+ individual `if x == nil` checks (dragonriding, buffMonitor)
- Some use `for k,v in pairs(defaults)` inline (gcdTracker)

This inconsistency makes it easy to miss new default values.

### Custom Settings UI
The addon builds its own settings UI from scratch rather than using AceConfig/AceGUI or Blizzard's Settings API. This means every UI element (checkboxes, sliders, dropdowns, color pickers, scroll frames) is manually constructed. The `Widgets.lua` file at ~1100 lines essentially reimplements a UI toolkit, and it uses deprecated Blizzard templates (`UIDropDownMenuTemplate`, `InterfaceOptionsCheckButtonTemplate`). Note: both templates still function via Blizzard's `DeprecatedTemplates.xml` compatibility layer on Mainline, but should be migrated to modern equivalents (`MenuUtil`, `SettingsCheckboxTemplate`).

### SavedVariables Structure
All settings live in a single global `NaowhQOL` table with flat keys per module. There is no per-character or per-profile support. The addon stores SavedVariables at the account level, meaning all characters share the same settings (including position data for UI elements).

---

## Summary

| Severity | Count | Notes |
|----------|-------|-------|
| Critical / High | 2 | #3/#4 (Classic API incompatibility) |
| Medium | 14 | Verified issues |
| Low | 7 | Minor issues |
| Dead Code / Orphaned Files | 7 files (~1,937 lines) | |

### Top Priority Fixes
1. **Fix buff tracker reset popup** to nil `NaowhQOL.buffMonitor` instead of `NaowhQOL.buffTracker`
2. **Remove `MaxSpellQueueWindow` CVar call** -- invalid CVar name
3. **Add `RegisterUnitEvent` filtering** for GCD tracker spellcast events -- performance in group content
4. **Replace deprecated `UIDropDownMenuTemplate`** with `MenuUtil` -- taint risk in combat
5. **Remove 7 orphaned files** -- nearly 2,000 lines of dead code
6. **Add Classic API guards or remove Classic interface version** -- Retail-only APIs will error on Classic
7. **Migrate deprecated templates** (`InterfaceOptionsCheckButtonTemplate`, `UIDropDownMenuTemplate`) -- functional but deprecated

---

## Fixed Since Review #1

The following bugs were identified in Review #1 and have been fixed in the current codebase.

---

### ~~#1. `C.WARNING` Does Not Exist~~ -- FIXED
**File:** `Config/Optimizations.lua`

**Original bug:** `C.WARNING` was used in 8 combat-lockdown guards but was never defined in `ns.COLORS`, causing `attempt to concatenate a nil value` errors.

**Status:** `C.WARNING` no longer appears anywhere in the codebase. The combat-lockdown guards have been fixed.

---

### ~~#2. MouseCursor `LoadSettings` Clobbers Boolean `false` Back to `true`~~ -- FIXED
**File:** `Config/MouseCursor.lua`

**Original bug:** The pattern `(db.enabled ~= nil) and db.enabled or true` silently discarded saved `false` values, resetting 4 boolean settings (`enabled`, `showOutOfCombat`, `gcdEnabled`, `castSwipeEnabled`) to `true` on every login.

**Status:** Now uses the correct pattern `db.enabled == nil and true or db.enabled` which properly preserves boolean `false`.

---

### ~~#21. MouseCursor `gcdReadyRing` Not Cleaned Up on Zone Change~~ -- FIXED
**File:** `Config/MouseCursor.lua:1349`

**Original bug:** During cleanup, `frames.gcdReadyRing` was not nil'd alongside the other frame references, causing a texture leak.

**Status:** Line 1349 now includes `frames.gcdReadyRing = nil` in the cleanup block.

---

### ~~#28. `SafeSetCVar` Defined But Never Called~~ -- FIXED
**File:** `Config/Optimizations.lua`

**Original bug:** A `SafeSetCVar` wrapper function was defined but never called anywhere in the codebase.

**Status:** The dead function has been removed.

---

## Found in Review #1, Corrected in Review #2

The following items from the original review were found to be incorrect, overstated, or not bugs when verified against the Blizzard WoW UI source v12.0.0.65655 (TWW).

---

### ~~#2. `InterfaceOptionsCheckButtonTemplate` Breaks Config Panels~~ -- INCORRECT
**Files:** `Config/Crosshair.lua:119`, `Config/ImportExport.lua:106`

**Original claim:** This template was removed in WoW 10.0 (Dragonflight) and crashes with `Couldn't find inherited node`.

**Correction:** This template was NOT removed. It still exists in `Blizzard_FrameXML/DeprecatedTemplates.xml` (line 34) and is loaded on Mainline via `Blizzard_FrameXML_Mainline.toc` (lines 25-26). It inherits from `OptionsBaseCheckButtonTemplate`. The template is deprecated but fully functional -- no crash occurs. Migrating to a modern template is good practice but not urgent.

---

### ~~#6. Sidebar PLAYER_LOGIN Calls InitOptOptions Without `self`~~ -- OVERSTATED
**File:** `UI/Sidebar.lua:224`

**Original claim:** `SwitchTab(OptBtn, ns.InitOptOptions)` passes the function without binding `self`, causing a nil self crash.

**Correction:** While the mechanism is real (`self` is nil when called via `initFunction()` at line 97), `ns:InitOptOptions()` (defined at `Config/Optimizations.lua:802`) uses `ns.MainFrame.Content` directly via the `ns` upvalue and never references `self`. **No crash occurs.** This is a code smell (inconsistent calling convention) but not a functional bug.

---

### ~~#15. Dragonriding -- Hiding Protected Frames Causes Taint~~ -- OVERSTATED
**File:** `Modules/Dragonriding.lua` lines 294-308

**Original claim:** `BuffIconCooldownViewer` is a "Blizzard protected cooldown viewer frame" and calling `Hide()` on it during combat generates taint.

**Correction:** `BuffIconCooldownViewer` is defined in `Blizzard_CooldownViewer/CooldownViewer.xml` (line 315) and does NOT inherit from `SecureFrameTemplate`. It inherits from `CooldownViewerTemplate` and `UIParentBottomManagedFrameTemplate`. It is an EditMode-managed frame, not a protected/secure frame. Taint risk through EditMode's managed frame system is possible but the original characterization as "protected frames" is inaccurate.

---

### ~~#29. FasterLoot References Undefined Config Key~~ -- NOT A BUG
**File:** `Modules/FasterLoot.lua:27`

**Original claim:** `lootThrottle` is never defined and references an undefined config key.

**Correction:** The code `(ns.DB.config.lootThrottle) or 0.2` safely falls back to 0.2 when the key doesn't exist. This is defensive programming, not a bug. The only issue is inconsistency with the codebase pattern of declaring defaults explicitly in `DefaultConfig`.

---

### ~~#35. PlaceSlider Defined Twice~~ -- INCORRECT
**File:** `Config/MouseCursor.lua`

**Original claim:** Two identical `PlaceSlider` functions exist at lines 7-12 and 976-981, with the second shadowing the first.

**Correction:** `PlaceSlider` is defined only ONCE in `MouseCursor.lua` at lines 976-981. Lines 7-12 contain asset path constants and local variable declarations, not a function definition. The function IS duplicated as separate `local` functions across 10 different config files (BuffTracker, BuffMonitor, ConsumableChecker, CombatTimer, Dragonriding, Crosshair, FocusCastBar, GcdTracker, MouseCursor, Optimizations) -- a DRY violation worth addressing, but not a shadowing bug within a single file.
