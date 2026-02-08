# NaowhQOL Code Review

**Version:** v0.1.0 BETA
**Interface:** 120000 (Retail TWW 12.0), 110207 (Retail TWW 11.0.7)
**Review Date:** 2026-02-08
**Verified Against:** Blizzard WoW UI Source v12.0.0.65655 (TWW)

---

## Overview

NaowhQOL is a WoW quality-of-life addon with ~50 files covering 20+ features: combat timer, combat alerts, crosshair overlay, dragonriding vigor bar, buff/consumable monitoring, GCD tracker, stealth/stance reminders, range check, focus cast bar, emote detection, combat logger, auto keystone, faster loot, easy item destroy, loot warning suppression, queue skip, durability warnings, auto repair, and UI clutter hiding. The addon uses a custom settings UI (no AceGUI/AceConfig) with a sidebar-driven panel system.

The codebase is functional and ambitious but has several categories of issues ranging from crash-causing bugs to performance inefficiencies and dead code.

---

## MEDIUM Severity

### 8. Deprecated `UIDropDownMenuTemplate` -- Taint Risk
**File:** `Data/Widgets.lua` lines 327-361

The entire legacy dropdown system (`UIDropDownMenu_SetWidth`, `UIDropDownMenu_Initialize`, `UIDropDownMenu_AddButton`, etc.) was deprecated in WoW 10.0. It still works via a compatibility layer (`Blizzard_SharedXML/Mainline/UIDropDownMenu.lua`), but is known to cause taint when interacted with during or shortly after combat, leading to "action blocked" errors. The modern replacement is `MenuUtil` (used in 10+ Blizzard UI files).

---

### 13. DisableLootWarnings / HideUIClutter -- No Runtime Toggle
**Files:** `Modules/DisableLootWarnings.lua`, `Modules/HideUIClutter.lua`

Multiple features use `hooksecurefunc` or `UnregisterAllEvents` which are irreversible. Settings changes require a `/reload` to take effect:
- Alert suppression (`hooksecurefunc(AlertFrame, "AddAlertFrame", ...)`) -- permanent
- TalkingHead suppression (`hooksecurefunc(TalkingHeadFrame, "Show", ...)`) -- permanent
- Zone text suppression (`frame:SetScript("OnShow", frame.Hide)`) -- permanent
- Loot warning events registered once at login, never unregistered

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

### 24. ZoneUtil Retry Fires Callbacks With Potentially Invalid Data
**File:** `Data/ZoneUtil.lua:114`

When retries are exhausted and zone info is still unresolved (difficulty 0 inside an instance), `NotifyCallbacks()` fires with incorrect data. Consumers may misidentify dungeon difficulty.

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

### 32. Dual/Inconsistent Color Definitions Across Files
Colors are defined independently with different values in `Core.lua`, `Data/Widgets.lua`, `UI/Sidebar.lua`, and `UI/MainFrame.lua`. The addon's "orange" is `ffa900` in one place, `{255/255, 169/255, 0}` in another, and `{1.00, 0.66, 0.00}` in a third -- these are mathematically different colors.

---

### 33. BarStyleList and FontList Built at Parse Time
**Files:** `Data/BarStyleList.lua:78-79`, `Data/FontList.lua:115-116`

`Rebuild()` and `HookLSM()` run before `ADDON_LOADED`. LibSharedMedia may not be fully populated yet. The initial list only contains built-in entries; LSM-registered entries from other addons are missing until the first registration callback.

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
| Medium | 5 | #8, #13, #16, #17, #24 |
| Low | 4 | #26, #27, #32, #33 |
| Dead Code / Orphaned Files | 7 files (~1,937 lines) | |

### Top Priority Fixes
1. **Replace deprecated `UIDropDownMenuTemplate`** with `MenuUtil` -- taint risk in combat
2. **Remove 7 orphaned files** -- nearly 2,000 lines of dead code
3. **Migrate deprecated templates** (`InterfaceOptionsCheckButtonTemplate`, `UIDropDownMenuTemplate`) -- functional but deprecated

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

### ~~#5. `MaxSpellQueueWindow` Is Not a Valid CVar~~ -- FIXED
**File:** `Core.lua`

**Original bug:** `SetCVar("MaxSpellQueueWindow", 150)` -- this CVar does not exist. `SpellQueueWindow` is the correct one.

**Status:** The invalid `MaxSpellQueueWindow` line has been removed.

---

### ~~#7. Buff Tracker Reset Popup Nils the Wrong Key~~ -- FIXED
**File:** `Data/StaticPopups.lua`

**Original bug:** `NaowhQOL.buffTracker = nil` should be `NaowhQOL.buffMonitor` after the buff tracker migration.

**Status:** Changed to `NaowhQOL.buffMonitor = nil`.

---

### ~~#9. `SetUserPlaced(true)` Conflicts With Addon Position Saving~~ -- FIXED
**File:** `Data/Widgets.lua`

**Original bug:** `SetUserPlaced(true)` competes with the addon's own position saving in `OnDragStop`, causing frame jumping on login.

**Status:** `SetUserPlaced(true)` removed from `MakeDraggable`.

---

### ~~#10. O(n^2) `ClearFrame` Implementation~~ -- FIXED
**File:** `Data/Widgets.lua`

**Original bug:** `select(i, frame:GetChildren())` in a loop rebuilds the full vararg each iteration.

**Status:** Now uses `{frame:GetChildren()}` + `ipairs` for O(n) performance.

---

### ~~#11. O(n) LRU Cache Scan on Every Hit~~ -- FIXED
**File:** `Data/DisplayUtils.lua`

**Original bug:** Linear scan of up to 100 entries on every cache hit.

**Status:** Added `cachePos` hash map for O(1) position lookup.

---

### ~~#12. CombatLogger Potential Nil Dereference~~ -- FIXED
**File:** `Modules/CombatLoggerDisplay.lua`

**Original bug:** `OnZoneChanged(ns.ZoneUtil.GetCurrentZone())` could pass nil, crashing on `zoneData.instanceType`.

**Status:** Added nil guard: only calls `OnZoneChanged` if `GetCurrentZone()` returns non-nil.

---

### ~~#14. GcdTracker Spellcast Events Not Unit-Filtered~~ -- FIXED
**File:** `Modules/GcdTrackerDisplay.lua`

**Original bug:** 6 `UNIT_SPELLCAST_*` events fired for all units; handler filtered to player but dispatch overhead remained.

**Status:** Changed to `RegisterUnitEvent("UNIT_SPELLCAST_*", "player")` for all 6 events.

---

### ~~#18. CVar Case Mismatch~~ -- FIXED
**File:** `Config/Optimizations.lua`

**Original bug:** `SetCVar("sunshafts", "0")` lowercase vs `["sunShafts"]` elsewhere.

**Status:** Changed to `SetCVar("sunShafts", "0")` for consistency.

---

### ~~#19. LowEndOptimization Sets CVars Not Tracked by SaveCurrentSettings~~ -- FIXED
**File:** `Config/Optimizations.lua`

**Original bug:** 4 CVars set by low-end mode were not in `OPTIMAL_FPS_CVARS`, so revert didn't restore them.

**Status:** Added `graphicsTextureQuality`, `graphicsTextureFiltering`, `lightingQuality`, `maxFPSLoading` to `OPTIMAL_FPS_CVARS`.

---

### ~~#20. Profiler AVG and MAX Columns Show Identical Data~~ -- FIXED
**File:** `Config/Profiler.lua`

**Original bug:** Both columns displayed `data[i].c` from `GetAddOnCPUUsage(i)`.

**Status:** Added `peakCPU` tracking table. AVG column shows current CPU, MAX column shows peak CPU since last reset.

---

### ~~#21. MouseCursor `gcdReadyRing` Not Cleaned Up on Zone Change~~ -- FIXED
**File:** `Config/MouseCursor.lua:1349`

**Original bug:** During cleanup, `frames.gcdReadyRing` was not nil'd alongside the other frame references, causing a texture leak.

**Status:** Line 1349 now includes `frames.gcdReadyRing = nil` in the cleanup block.

---

### ~~#22. Emote Detection Content Height Immediately Overwritten~~ -- FIXED
**File:** `Config/EmoteDetection.lua`

**Original bug:** `BuildAutoEmoteList()` dynamically calculates content height, then line 400 immediately overwrites it to static 200px.

**Status:** Removed the static `SetHeight(200)` and `RecalcHeight()` overwrite.

---

### ~~#23. `GroupUtil` Passes Mutable `cachedClasses` to Callbacks~~ -- FIXED
**File:** `Data/GroupUtil.lua`

**Original bug:** Callbacks received a direct reference to the internal cache table; mutations corrupted shared state.

**Status:** Callbacks now receive a shallow copy of `cachedClasses`.

---

### ~~#25. AutoKeystone Missing Nil Check on C_Item.GetItemInfo~~ -- FIXED
**File:** `Modules/AutoKeystone.lua`

**Original bug:** `C_Item.GetItemInfo` returns nil when item data isn't cached. No retry mechanism existed.

**Status:** Added `GET_ITEM_INFO_RECEIVED` retry. When item data is nil, a one-shot event listener retries `TrySlotKeystone`.

---

### ~~#28. `SafeSetCVar` Defined But Never Called~~ -- FIXED
**File:** `Config/Optimizations.lua`

**Original bug:** A `SafeSetCVar` wrapper function was defined but never called anywhere in the codebase.

**Status:** The dead function has been removed.

---

### ~~#30. ConsumableChecker `SetAttribute` With String Instead of Number~~ -- FIXED
**File:** `Modules/ConsumableCheckerDisplay.lua`

**Original bug:** `tostring(m.targetSlot)` passed a string to `target-slot` attribute which expects a number.

**Status:** Removed `tostring()` wrapper, now passes `m.targetSlot` directly as a number.

---

### ~~#31. Main Frame Not Clamped to Screen~~ -- FIXED
**File:** `UI/MainFrame.lua`

**Original bug:** The 950x650 settings window could be dragged entirely off-screen.

**Status:** Added `SetClampedToScreen(true)`.

---

### ~~#34. `SetCVar`/`GetCVar` Local Cache Asymmetry~~ -- FIXED
**File:** `Config/Optimizations.lua`

**Original bug:** `local SetCVar = SetCVar` / `local GetCVar = GetCVar` were declared after `SaveCurrentSettings`, causing asymmetric behavior.

**Status:** Moved `SetCVar`/`GetCVar` locals to the top of the file, before any function definitions.

---

## Found in Review #1, Corrected in Review #2

The following items from the original review were found to be incorrect, overstated, or not bugs when verified against the Blizzard WoW UI source v12.0.0.65655 (TWW).

---

### ~~#2. `InterfaceOptionsCheckButtonTemplate` Breaks Config Panels~~ -- INCORRECT
**Files:** `Config/Crosshair.lua:119`, `Config/ImportExport.lua:106`

**Original claim:** This template was removed in WoW 10.0 (Dragonflight) and crashes with `Couldn't find inherited node`.

**Correction:** This template was NOT removed. It still exists in `Blizzard_FrameXML/DeprecatedTemplates.xml` (line 34) and is loaded on Mainline via `Blizzard_FrameXML_Mainline.toc` (lines 25-26). It inherits from `OptionsBaseCheckButtonTemplate`. The template is deprecated but fully functional -- no crash occurs. Migrating to a modern template is good practice but not urgent.

---

### ~~#3. FocusCastBar Uses TWW "Secret Value" APIs (claimed Classic incompatible)~~ -- INCORRECT
**File:** `Modules/FocusCastBarDisplay.lua`

**Original claim:** These "opaque value" APIs (`GetSpellCooldownDuration`, `SetVertexColorFromBoolean`, `UnitCastingDuration`, etc.) would produce hard errors on Classic because the TOC declares `## Interface: 110207` as Classic support.

**Correction:** `110207` is Retail TWW 11.0.2 (previous expansion patch), NOT Classic. Both `120000` and `110207` are Retail interface versions. All these APIs exist on both versions. There is no Classic incompatibility.

---

### ~~#4. TOC Claims Classic Support But Modules Use Retail-Only APIs~~ -- INCORRECT
**File:** `NaowhQOL.toc`

**Original claim:** The TOC specifies both Retail TWW and Classic (`110207`), and many Retail-only APIs would produce errors on the Classic client.

**Correction:** `110207` is Retail TWW 11.0.2, NOT Classic. The TOC declares two Retail interface versions for backwards compatibility. All listed APIs (`C_UnitAuras.GetAuraDataByIndex`, `C_Spell.*`, `C_ChallengeMode.*`, `GetSpecialization`, etc.) exist on both Retail versions. There is no Classic client issue.

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
