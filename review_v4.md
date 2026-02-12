# NaowhQOL Code Review v4

**Version:** v1.0.0 Stable
**Interface:** 120000 (Retail TWW 12.0), 110207 (Retail TWW 11.0.7)
**Review Date:** 2026-02-12
**Scope:** Full codebase review (~75 files, 29 modules, ~30,133 lines of Lua)
**Compared Against:** review_v3.md (NaowhQOL_v3 codebase)

---

## Changes Since v3

### Structural Changes
- **SavedVariables changed** from account-wide (`SavedVariables: NaowhQOL`) to per-character (`SavedVariablesPerCharacter: NaowhQOL`) with new `SavedVariables: NaowhQOL_Profiles` for profile system
- **MouseCursor split**: `Config/MouseCursor.lua` reduced from 1,526 lines to 344 lines; display code moved to `Modules/MouseRingDisplay.lua` (549 lines)
- **Dead code removed**: `BuffTrackerDisplay.lua`, `PoisonReminderDisplay.lua`, `RaidAlerts.lua`, `DragonRidingDisplay.lua`, `Config/RaidAlerts.lua` all deleted
- **New modules added**: `EquipmentReminder`, `CRezDisplay`, `PetTrackerDisplay`, `DeathReleaseProtection`
- **New utilities**: `Data/StaticPopups.lua` (66 lines), `Data/LayoutUtil.lua` (178 lines)
- **New config**: `Config/Profiler.lua` (339 lines), `Config/EquipmentReminder.lua`, `Config/CRez.lua`, `Config/PetTracker.lua`
- **SettingsIO expanded**: 528 -> 736 lines (added profile system, type schema validation)
- **Total codebase**: ~30,133 lines across 75 Lua files (excluding libs)

### Fixes Applied from v3 Review
12 of 52 issues fixed or mostly fixed. See "v3 Issue Status" section below for full tracking.

---

## NEW Issues (Not in v3 Review)

### CRITICAL Severity

### N1. UNIT_DIED Is Not a Registerable WoW Event -- CRezDisplay Silently Fails
**File:** `Modules/CRezDisplay.lua` line 152

```lua
eventFrame:RegisterEvent("UNIT_DIED")
```

`UNIT_DIED` is NOT a standard WoW event that can be registered via `RegisterEvent()`. Deaths are detected via `COMBAT_LOG_EVENT_UNFILTERED`, parsing the `UNIT_DIED` subevent from `CombatLogGetCurrentEventInfo()`. Registering `"UNIT_DIED"` directly will silently fail -- the event never fires. The entire death tracking feature in CRezDisplay is **non-functional**.

**Fix:** Register `COMBAT_LOG_EVENT_UNFILTERED` and parse the subevent:
```lua
eventFrame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
-- In handler:
local _, subevent, _, _, _, _, _, destGUID = CombatLogGetCurrentEventInfo()
if subevent == "UNIT_DIED" then OnUnitDied(destGUID) end
```

### N2. SavedVariables Changed from Account-Wide to Per-Character -- DATA LOSS Risk
**File:** `NaowhQOL.toc` lines 4-5

v3: `## SavedVariables: NaowhQOL`
v4: `## SavedVariablesPerCharacter: NaowhQOL`

All existing user settings from v3 will be **lost** for every character except the one loaded last when v3 was active. The WoW client stores `SavedVariables` and `SavedVariablesPerCharacter` in different files. Users upgrading from v3 to v4 lose their configurations with no migration path, no warning, and no way to recover.

### N3. EquipmentReminder Config Sets Global Frame Name to nil -- Orphans Local Reference
**File:** `Config/EquipmentReminder.lua` lines 95-98

```lua
onChange = function(val)
    db.iconSize = val
    if _G["NaowhQOL_EquipmentReminder"] then
        _G["NaowhQOL_EquipmentReminder"]:Hide()
        _G["NaowhQOL_EquipmentReminder"] = nil
    end
end,
```

This nils the global name, but the module-local `equipmentFrame` in `EquipmentReminder.lua` line 197 still holds a reference to the old frame. The guard at line 125 (`if equipmentFrame then return equipmentFrame end`) will return the stale hidden frame, and a new frame will never be created. Icon size changes never take effect visually.

---

### HIGH Severity

### N4. MouseRingDisplay mouseWatcher OnUpdate Runs Every Frame Regardless of Enable State
**File:** `Modules/MouseRingDisplay.lua` lines 348-359

```lua
local mouseWatcher = CreateFrame("Frame")
mouseWatcher:SetScript("OnUpdate", function()
    local db = GetDB()
    if not db.hideOnMouseClick then return end
    ...
end)
```

This `OnUpdate` runs every single frame regardless of whether the mouse ring is enabled. Even when `db.enabled = false`, the handler fires, calls `GetDB()` (which does nil-guard table creation), and checks `hideOnMouseClick`. A perpetual per-frame cost for a potentially disabled feature.

### N5. GCD Tracker Default Key Mismatch Between Defaults Table and InitializeDB
**File:** `Core.lua`

Line 133 (`GCD_TRACKER_DEFAULTS`): `showDowntimeSummary = true`
Line 612 (`InitializeDB`): `downtimeSummaryEnabled = false`

The defaults table uses key `showDowntimeSummary` but the actual DB key used by the config panel (line 60) and display module (lines 710, 733) is `downtimeSummaryEnabled`. `RestoreModuleDefaults` will restore the wrong key (`showDowntimeSummary = true`) while leaving the actual key (`downtimeSummaryEnabled`) as `nil`. The feature's default also conflicts: the defaults table says `true` but `InitializeDB` says `false`.

### N6. Dragonriding v4 Removed BCDM PowerBar Hiding -- Functional Regression
**File:** `Modules/Dragonriding.lua` lines 371-378

v3's `HideCooldownManager()` hid `BCDM_PowerBar` and `BCDM_SecondaryPowerBar` (popular cooldown manager frames). v4 removed this logic entirely. Users with BCDM will see power bars overlapping the dragonriding HUD when `hideCdmWhileMounted` is enabled. `ShowCooldownManager()` also no longer restores these bars.

### N7. Dragonriding Events Remain Registered When Feature Is Disabled
**File:** `Modules/Dragonriding.lua`

v4 added `RegisterDynamicEvents()` / `UnregisterDynamicEvents()` (lines 80-102), but `UnregisterDynamicEvents()` is only called from `ns:HideDragonridingPreview()` (line 604), which runs when the config panel closes. When the user **disables** dragonriding via the checkbox, the 5 ACTIONBAR/GLIDE events remain registered. The event handler's `not IsEnabled()` guard prevents the updater from starting, but the events still fire and invoke the handler unnecessarily.

### N8. Duplicate `ns:OptimizeNetwork()` Definitions -- Function Shadowing
**File:** `Core.lua` line 853 and `Config/Optimizations.lua` line 529

Two completely different implementations of `ns:OptimizeNetwork()` exist. The Core.lua version (sets `SpellQueueWindow`, `reducedLagTolerance`, `MaxSpellQueueWindow`) is silently overwritten by the Config/Optimizations.lua version (sets `advancedCombatLogging`, `chatBubbles`, `chatBubblesParty`) because Config loads after Core in the TOC. Same issue exists for `ns:ApplyFPSOptimization()` (Core.lua line 817 vs Config/Optimizations.lua line 270). The Core.lua versions are dead code that will never execute.

---

### MEDIUM Severity

### N9. Default Values Changed Between v3 and v4 with No Migration
**File:** `Core.lua`

Multiple defaults silently changed. The `if x == nil then x = v end` pattern means these changes **never apply to existing users** -- only new installs see the new defaults:

| Module | Key | v3 Default | v4 Default |
|--------|-----|-----------|-----------|
| combatTimer | enabled | `true` | `false` |
| combatLogger | enabled | `true` | `false` |
| dragonriding | enabled | `false` | `true` |
| buffMonitor | enabled | `false` | `true` |
| consumableChecker | enabled | `false` | `true` |
| consumableChecker | iconPoint/iconY | CENTER/150 | TOP/-140 |
| talentReminder | enabled | `true` | `false` |
| misc.fasterLoot | | `false` | `true` |
| misc.suppressLootWarnings | | `false` | `true` |
| misc.durabilityWarning | | `false` | `true` |
| combatAlert | y/width/height | 200/300/80 | 100/200/50 |
| emoteDetection | point/y | CENTER/0 | TOP/-50 |

`RestoreModuleDefaults()` will restore to v4 defaults, which differ from what v3 users expect.

### N10. PetTrackerDisplay Felguard Detection Is English/German Only
**File:** `Modules/PetTrackerDisplay.lua` lines 104-106

```lua
local lowerFamily = petFamily:lower()
if not lowerFamily:find("felguard") and not lowerFamily:find("teufelswache") then
    return true
end
```

Only English ("felguard") and German ("teufelswache") creature family names are checked. Players using French, Spanish, Italian, Portuguese, Russian, Korean, Chinese, or other locales will always get a false "Wrong Pet" warning for Demonology Warlocks with a Felguard.

### N11. PetTrackerDisplay Uses Hardcoded Spec Indices
**File:** `Modules/PetTrackerDisplay.lua` lines 31-33

```lua
local DEMONOLOGY_SPEC = 2
local UNHOLY_SPEC = 3
local FROST_MAGE_SPEC = 3
```

These spec index values are correct currently but are fragile. If Blizzard reorders specs (as they have done historically), all pet tracking detection silently breaks. Should use spec IDs from `GetSpecializationInfo()` instead.

### N12. CRezDisplay Uses Combat Rez Charges API Outside Applicable Content
**File:** `Modules/CRezDisplay.lua` line 51

```lua
local charges = C_Spell.GetSpellCharges(REBIRTH_SPELL_ID)
```

Called every 0.1s in the OnUpdate handler whenever `encounterActive = true`. In non-M+/non-raid content (e.g., world boss encounters), `C_Spell.GetSpellCharges(20484)` returns `nil`, and the UI shows "--:--" and "0" charges, which is misleading.

### N13. Config/BuffTracker.lua Is Orphaned -- Module Removed but Config Remains
**File:** `Config/BuffTracker.lua` (241 lines, in TOC)

The display module `Modules/BuffTrackerDisplay.lua` was removed from v4, but the config file remains in the TOC. It creates `NaowhQOL.buffTracker` saved variable data that serves no purpose, and references `ns.RefreshBuffTracker` which is `nil` since the display module doesn't exist. All sliders and checkboxes in the config panel call a no-op `refresh()`. The sidebar entry for Buff Tracker also doesn't exist, so users can't reach this panel anyway -- but the initialization code still runs.

### N14. EquipmentReminder Duplicate Default Initialization with Conflicting Values
**File:** `Modules/EquipmentReminder.lua` lines 271-289 and `Core.lua` lines 186-195

Core.lua initializes `NaowhQOL.equipmentReminder` at `ADDON_LOADED` time with `enabled = false`. The module's own `PLAYER_LOGIN` handler re-initializes the same table with `enabled = true`. Since `ADDON_LOADED` fires first, the `if x == nil` guard means Core.lua's `false` takes precedence. But if Core.lua's initialization is ever removed, the module would silently change the default. The dual initialization is confusing and the `enabled` defaults conflict.

### N15. C_InstanceEncounter.IsEncounterInProgress() May Error
**File:** `Modules/CRezDisplay.lua` line 161

```lua
encounterActive = C_InstanceEncounter and C_InstanceEncounter.IsEncounterInProgress() or false
```

The nil-check on the namespace is good, but if `IsEncounterInProgress` doesn't exist within the namespace (it's a relatively new API), this will error. Should be:
```lua
encounterActive = C_InstanceEncounter and C_InstanceEncounter.IsEncounterInProgress
                  and C_InstanceEncounter.IsEncounterInProgress() or false
```

---

### LOW Severity

### N16. MouseRingDisplay Events Registered at File Load Before PLAYER_LOGIN
**File:** `Modules/MouseRingDisplay.lua` lines 362-373

Nine spell-related events (`UNIT_SPELLCAST_START/STOP/FAILED/INTERRUPTED`, `CHANNEL_START/STOP`, `SPELL_UPDATE_COOLDOWN`, combat events) are registered at file load time before knowing if the module is enabled. Unlike Dragonriding (which added dynamic registration), MouseRingDisplay events fire frequently during gameplay even when `db.enabled = false`.

### N17. EquipmentReminder Doesn't Use PerfMonitor:Wrap
**File:** `Modules/EquipmentReminder.lua` line 270

Most new modules (CRezDisplay, PetTrackerDisplay) wrap their event handlers with `ns.PerfMonitor:Wrap()`. EquipmentReminder skips this, making it invisible in the addon profiler's performance diagnostics.

### N18. CRezDisplay: IsGUIDInGroup Compatibility
**File:** `Modules/CRezDisplay.lua` line 129

```lua
local inGroup = IsGUIDInGroup and IsGUIDInGroup(unitGUID)
```

`IsGUIDInGroup` was added in a recent WoW patch. The nil-check prevents errors, but when it doesn't exist, `inGroup` is `nil` (falsy), and the death warning **only works for the player's own death**, never for group members. The feature silently degrades without notification.

### N19. Core.lua Has Dead Code from Shadowed Functions
**File:** `Core.lua` lines 817-880

`ns:ApplyFPSOptimization()` (line 817) and `ns:OptimizeNetwork()` (line 853) are defined in Core.lua but silently overwritten by Config/Optimizations.lua. These ~63 lines of Core.lua are dead code that will never execute. They should be removed to avoid confusion about which implementation is active.

---

## v3 Issue Status Tracker

### FIXED (12 issues)

| v3 # | Issue | Status | Notes |
|-------|-------|--------|-------|
| 1 | SetCVar MaxSpellQueueWindow override | **Effectively fixed** | Core.lua version is dead code (shadowed by Config/Optimizations.lua). SpellQueueWindow is opt-in via slider. But duplicate function definitions need cleanup (see N8). |
| 3 | ClearFrame O(n^2) select pattern | **Fixed** | Now captures children into local table first. O(n) complexity. |
| 4 | Dead code files on disk | **Mostly fixed** | All 4 module files + Config/RaidAlerts.lua removed. Config/BuffTracker.lua still exists as orphan (see N13). |
| 11 | FasterLoot undefined config key | **Fixed** | `lootThrottle` removed; uses hardcoded local `0.2`. |
| 16 | Deprecated GetItemInfo | **Fixed** | Both files use `C_Item.GetItemInfo()`. |
| 17 | RaidAlerts UNIT_SPELLCAST not filtered | **N/A** | Module removed entirely. |
| 18 | Widgets picker duplication (550+ lines) | **Fixed** | Consolidated into single implementations in Widgets.lua. |
| 19 | FontList/BarStyleList structural duplication | **Fixed** | Separate clean data modules. |
| 20 | ZoneUtil fires callbacks with invalid data | **Fixed** | Proper retry logic with max retries and cancellation. |
| 21 | MouseCursor.lua is 1526 lines | **Fixed** | Now 344 lines. Display code moved to MouseRingDisplay.lua. |
| 35 | Old DragonRidingDisplay.lua duplicates active module | **Fixed** | File deleted. |
| 41 | Global function leaks in Config/SlashCommands.lua | **Fixed** | All functions properly namespaced. |

### PARTIALLY FIXED (6 issues)

| v3 # | Issue | Status | Notes |
|-------|-------|--------|-------|
| 2 | UNIT_SPELLCAST not unit-filtered | **Partially fixed** | GcdTrackerDisplay.lua fixed (uses `RegisterUnitEvent`). EmoteDetection.lua still uses `RegisterEvent` (lines 217-218). |
| 13 | C_Timer.NewTicker never cancelled | **Partially fixed** | DurabilityDisplay fixed. BuffMonitorDisplay ticker is anonymous (uncancellable). ConsumableCheckerDisplay stores ticker but doesn't cancel on disable. |
| 14 | Dragonriding magic numbers | **Partially fixed** | Magic numbers 11/650 still bare literals, but `C_PlayerInfo.GetGlidingInfo()` now used as fallback (line 144). |
| 22 | Defaults tables duplicated 3x | **Partially fixed** | Reduced to 2 copies (Core.lua + Config/BuffTracker.lua). |
| 25 | Locale strings for dead features | **Partially fixed** | POISON_* locale strings removed. RAIDALERTS_* strings (17 strings) still present in enUS.lua lines 461-477. SIDEBAR_TAB_BUFF_TRACKER and SIDEBAR_TAB_RAID_ALERTS still defined but unused. |
| 36 | BuffTracker has own InitializeDB separate from Core | **Partially fixed** | BuffTrackerDisplay.lua removed. Config/BuffTracker.lua still has its own defaults (see N13). |

### NOT FIXED (34 issues)

| v3 # | Issue | Severity | Notes |
|-------|-------|----------|-------|
| 5 | Multiple inconsistent color systems | HIGH | Now **8 separate** COLORS tables across files. `ns.COLORS` exists but 5+ modules define their own local copies. |
| 6 | Massive code duplication | HIGH | PlaceSlider: **12 copies** across Config files. FRAME_BACKDROP: 4 copies. SetFrameUnlocked: 3 copies. GetSpellName/Icon: 3 copies. Only MakeSlot partially consolidated. |
| 7 | ChatConfigCheckButtonTemplate (deprecated) | HIGH | **92 uses** across 16 files. Only 1 use of UICheckButtonTemplate (in Widgets.lua). |
| 8 | UIDropDownMenuTemplate in BuffMonitor | HIGH | 6 uses of legacy API. Zero uses of MenuUtil anywhere. |
| 9 | Inconsistent InitializeDB defaults pattern | HIGH | Only ~7/17 modules use `ApplyDefaults()`. ~250 manual `if x == nil` checks remain despite `_DEFAULTS` tables existing for all modules. |
| 10 | Sidebar missing entries for active modules | HIGH | AuctionHouseFilter, BuffTracker, PoisonReminder still have no sidebar entries. Locale keys SIDEBAR_TAB_BUFF_TRACKER and SIDEBAR_TAB_RAID_ALERTS still unused. |
| 12 | OnUpdate handlers run when disabled | MEDIUM | GcdTrackerDisplay, FocusCastBarDisplay, RangeCheckDisplay, CombatTimerDisplay all set OnUpdate unconditionally, never call `SetScript("OnUpdate", nil)` when disabled. |
| 15 | Hardcoded shapeshift form indices | MEDIUM | Form indices still hardcoded magic numbers. No named constants. |
| 23 | No runtime toggle for irreversible hooks | MEDIUM | HideUIClutter uses permanent `hooksecurefunc`. DisableLootWarnings has no unregister path. Requires `/reload` to toggle off. |
| 24 | Undocumented "secret value" APIs | MEDIUM | FocusCastBarDisplay uses `UnitCastingDuration`, `SetVertexColorFromBoolean`, `SetAlphaFromBoolean`, `SetTimerDuration` -- undocumented 11.1+ APIs. |
| 26 | Unreachable `if not ns` guard | MEDIUM | Both Init.lua and SettingsIO.lua. Harmless but dead code. |
| 27 | PerfMonitor is entirely no-op | LOW | Still 12 lines of stubs. Every `PerfMonitor:Wrap()` call adds closure overhead for zero benefit. |
| 28 | SettingsIO custom serializer | LOW | Grew from 528 to **736 lines** with profile system additions. Still fully custom (no LibSerialize/LibDeflate). |
| 29 | SlashCommands chat edit box hack | LOW | Still manipulates ChatFrame1EditBox. |
| 30 | Crosshair textures at file scope | LOW | 12 textures + frame created at file load before PLAYER_LOGIN, regardless of enable state. |
| 31 | Redundant DB access in CombatAlertDisplay | LOW | `db` and `db2` still both reference `NaowhQOL.combatAlert` (lines 50, 56). |
| 34 | RelayoutSections duplicated | LOW | Now in **15 Config files** (was 18 -- 3 dead config files removed). |
| 37 | StaticPopupDialogs redefined per click | LOW | `NAOWHQOL_RESTORE_DEFAULTS` still defined inside OnClick handler (Widgets.lua line 2302). |
| 38 | GroupUtil ForceRefresh bypasses change detection | LOW | Still unconditionally fires callbacks. |
| 39 | Hardcoded difficulty ID 8 for M+ | LOW | CombatLoggerDisplay.lua line 88. No named constant. |
| 42 | Config files missing localization | LOW | CombatLogger, TalentReminder, ImportExport still have no `local L = ns.L`. |
| 43 | Config rebuilds without frame pooling | LOW | CombatLogger and TalentReminder still create frames without recycling. |
| 44 | Optimizations 26+ OnUpdate timers | LOW | 26 individual CVar-row OnUpdate handlers in Config/Optimizations.lua. Only active when panel is visible. |
| 49 | FocusCastBar CreateColor every tick | MEDIUM | Still creates 2 `CreateColor` objects per tick (~30fps) in UpdateBarColor (line 142). |
| 50 | SlashCommands Cleanup doesn't unregister | LOW | `Cleanup()` wipes internal table but doesn't remove `_G["SLASH_*"]` or `SlashCmdList` entries. |
| 51 | hasSecretInterruptible dead variable | LOW | FocusCastBarDisplay.lua line 330. Set at lines 348, 394, 647, 655 but never read. |
| 52 | Font path hardcoded everywhere | LOW | `"Interface\\AddOns\\NaowhQOL\\Assets\\Fonts\\Naowh.ttf"` appears in **46 locations** across 16 files. Core.lua defines local `NAOWH_FONT` but it's not shared as `ns.DEFAULT_FONT`. |

---

## Full Summary Table

### New Issues (v4-specific)

| # | Severity | Issue | File(s) |
|---|---|---|---|
| N1 | CRITICAL | UNIT_DIED not a registerable event -- CRez death tracking non-functional | CRezDisplay.lua:152 |
| N2 | CRITICAL | SavedVariables changed to per-character -- data loss on upgrade | NaowhQOL.toc:4-5 |
| N3 | CRITICAL | Config sets _G frame name to nil, orphans module-local reference | Config/EquipmentReminder.lua:95-98 |
| N4 | HIGH | mouseWatcher OnUpdate runs every frame even when module disabled | MouseRingDisplay.lua:348-359 |
| N5 | HIGH | GCD Tracker showDowntimeSummary vs downtimeSummaryEnabled key mismatch | Core.lua:133 vs 612 |
| N6 | HIGH | Dragonriding removed BCDM PowerBar hiding -- functional regression | Dragonriding.lua:371-378 |
| N7 | HIGH | Dragonriding events remain registered when feature disabled | Dragonriding.lua |
| N8 | HIGH | Duplicate ns:OptimizeNetwork() / ns:ApplyFPSOptimization() -- function shadowing | Core.lua:817,853 vs Config/Optimizations.lua:270,529 |
| N9 | MEDIUM | Default values changed between v3/v4 with no migration logic | Core.lua |
| N10 | MEDIUM | PetTracker Felguard detection is English/German only | PetTrackerDisplay.lua:104-106 |
| N11 | MEDIUM | PetTracker uses hardcoded spec indices | PetTrackerDisplay.lua:31-33 |
| N12 | MEDIUM | CRez uses combat rez charges API outside applicable content | CRezDisplay.lua:51 |
| N13 | MEDIUM | Config/BuffTracker.lua orphaned -- module removed but config in TOC | Config/BuffTracker.lua |
| N14 | MEDIUM | EquipmentReminder duplicate defaults with conflicting enabled value | EquipmentReminder.lua:271 vs Core.lua:186 |
| N15 | MEDIUM | C_InstanceEncounter.IsEncounterInProgress() missing function guard | CRezDisplay.lua:161 |
| N16 | LOW | MouseRingDisplay events registered at file load before PLAYER_LOGIN | MouseRingDisplay.lua:362-373 |
| N17 | LOW | EquipmentReminder doesn't use PerfMonitor:Wrap | EquipmentReminder.lua:270 |
| N18 | LOW | IsGUIDInGroup compatibility -- silent degradation | CRezDisplay.lua:129 |
| N19 | LOW | Core.lua has ~63 lines of dead code from shadowed functions | Core.lua:817-880 |

### Carried Forward from v3 (Not Fixed)

| v3 # | Severity | Issue | File(s) |
|---|---|---|---|
| 5 | HIGH | 8 inconsistent color definition systems | Core, Widgets, Sidebar, CombatLogger, Profiler, EquipmentReminder, TalentReminder |
| 6 | HIGH | Massive code duplication (PlaceSlider 12x, FRAME_BACKDROP 4x, SetFrameUnlocked 3x) | ~15 files |
| 7 | HIGH | ChatConfigCheckButtonTemplate used 92 times (deprecated) | All Config files |
| 8 | HIGH | UIDropDownMenuTemplate in BuffMonitor (deprecated) | Config/BuffMonitor.lua |
| 9 | HIGH | Inconsistent InitializeDB: ~250 manual nil checks vs ApplyDefaults | Core.lua |
| 10 | HIGH | Sidebar missing entries for active modules | Sidebar.lua |
| 12 | MEDIUM | OnUpdate handlers run when disabled (4+ modules) | GcdTracker, FocusCastBar, RangeCheck, CombatTimer |
| 15 | MEDIUM | Hardcoded shapeshift form indices | StealthReminderDisplay.lua |
| 23 | MEDIUM | No runtime toggle for irreversible hooks | DisableLootWarnings, HideUIClutter |
| 24 | MEDIUM | Undocumented "secret value" APIs | FocusCastBarDisplay.lua |
| 49 | MEDIUM | CreateColor objects every tick (~30fps GC pressure) | FocusCastBarDisplay.lua:142 |
| 52 | LOW | Font path hardcoded in 46 locations across 16 files | 16 files |
| 34 | LOW | RelayoutSections duplicated in 15 Config files | All Config files |
| 42 | LOW | 3 config files missing localization | CombatLogger, TalentReminder, ImportExport |

---

## Statistics

| Category | Count |
|----------|-------|
| v3 issues fixed | 12 |
| v3 issues partially fixed | 6 |
| v3 issues not fixed | 34 |
| New issues (v4) | 19 |
| **Total open issues** | **59** |

| Severity | New (v4) | Carried (v3) | Total Open |
|----------|----------|--------------|------------|
| CRITICAL | 3 | 0 | 3 |
| HIGH | 5 | 6 | 11 |
| MEDIUM | 7 | 5+ | 12+ |
| LOW | 4 | 3+ | 7+ |

---

## Recommended Priority Order

1. **Fix UNIT_DIED registration** in CRezDisplay (N1) -- entire feature is non-functional
2. **Add SavedVariables migration** path from v3 account-wide to v4 per-character (N2) -- prevents data loss on upgrade
3. **Fix EquipmentReminder frame nil** issue (N3) -- icon size changes silently break
4. **Fix GCD Tracker key mismatch** (N5) -- RestoreDefaults will corrupt settings
5. **Remove dead code** in Core.lua from shadowed functions (N8, N19)
6. **Remove orphaned Config/BuffTracker.lua** from TOC (N13) and clean up dead locale strings
7. **Fix mouseWatcher OnUpdate** to only run when enabled (N4)
8. **Fix Dragonriding BCDM regression** (N6) and event registration cleanup (N7)
9. **Add unit filtering** to EmoteDetection UNIT_SPELLCAST events (v3 #2)
10. **Consolidate InitializeDB** to use ApplyDefaults for all modules (v3 #9)
11. **Extract duplicated code** (PlaceSlider, FRAME_BACKDROP, SetFrameUnlocked, RelayoutSections) into shared utilities (v3 #6, #34)
12. **Stop OnUpdate/tickers** when features are disabled (v3 #12, #13)
13. **Update deprecated templates** (ChatConfigCheckButtonTemplate, UIDropDownMenuTemplate) (v3 #7, #8)
14. **Consolidate color definitions** into single ns.COLORS (v3 #5)
15. **Centralize font path** as ns.DEFAULT_FONT (v3 #52)
