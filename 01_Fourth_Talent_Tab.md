# Adding a native 4th talent tab (AzerothCore 3.3.5a)

**Draft 1.** Tags: **[Tested]** in-game, **[Source]** read in source,
**[Data]** checked against DB rows (see [README](README.md)).

We gave every class a fourth talent tree that sits in the normal talent
window, uses normal talent points, works with dual spec and respec, shows on
inspect, and is used by playerbots. Ten trees, about 270 talents, all
in-game tested. This file is the method.

## Contents
1. [What it takes (overview)](#1-what-it-takes)
2. [Server: the core edit](#2-server-the-core-edit)
3. [Server: data rows](#3-server-data-rows)
4. [Client: data files and the addon](#4-client-data-files-and-the-addon)
5. [Rules the client forces on you](#5-rules-the-client-forces-on-you)
6. [Talents that teach spells, and the spellbook](#6-talents-that-teach-spells-and-the-spellbook)
7. [Choice tiers (pick one of N)](#7-choice-tiers-pick-one-of-n)
8. [Playerbots](#8-playerbots)
9. [Changing a live tree safely](#9-changing-a-live-tree-safely)
10. [Build order that worked](#10-build-order-that-worked)
11. [Test checklist](#11-test-checklist)
12. [Open questions](#12-open-questions)

---

## 1. What it takes

| Layer | Change | Size |
|---|---|---|
| Core C++ | Raise the tab limit from 3 to 4 | 2 lines |
| World DB | Rows for the tab, its talents, a skill line, and the talent spells | per tree |
| Client | Same DBC rows in an MPQ patch, 4 background textures per tree, a small addon | per tree + once |
| mod-playerbots | Arrays and loops from 3 tabs to 4 | small patch, once |
| Your module (optional) | Scripts for talents data cannot express; choice tiers | per talent |

No Blizzard UI file is edited. No talent-point formula changes.

## 2. Server: the core edit

Two lines. **[Tested]** **[Source]**

| File | Change |
|---|---|
| `src/server/shared/DataStores/DBCStructure.h` | `#define MAX_TALENT_TABS 3` → `4` |
| `src/server/game/DataStores/DBCStores.cpp` | `static uint32 sTalentTabPages[MAX_CLASSES][3];` → `[MAX_CLASSES][MAX_TALENT_TABS]` |

Why the second line matters: a TalentTab row with tab page 3 (the fourth
page, zero-based) writes past the end of the stock `[3]` array and corrupts
the next class's first page. **[Source]**

What we checked and did **not** need to change: the tier gate in
`Player::LearnTalent` (it counts points in the same tab), talent reset and its
cost, dual spec activation, the `character_talent` table (it stores spell id +
spec mask, no tab), and the talent packets (no tab count). **[Source]**
**[Tested]** for all of these in play.

Build note: rebuild with the worldserver stopped.

## 3. Server: data rows

AzerothCore loads these DBC override tables from the world database. Each new
tab needs: **[Tested]**

| Table | One row per | Notes |
|---|---|---|
| `talenttab_dbc` | tab | Class mask, tab page 3 (= 4th), internal name (also the texture name, section 4) |
| `talent_dbc` | talent | Tier, column, rank spell ids, prerequisite |
| `skillline_dbc` | tab | Category 7 (class). We cloned one of the class's talent skill lines |
| `skillraceclassinfo_dbc` | tab | Class mask for the skill line. We copied flags from the stock rows |
| `spell_dbc` | talent rank | The spells the ranks grant. Must also ship in the client `Spell.dbc` |

The skill line is not optional: without a SkillLine + SkillRaceClassInfo pair
the spells granted by the tab do not survive the character load. **[Source]**

ID choice: pick ranges above the stock maximum and write the scheme down
before the first build, because talent ids end up in players'
`character_talent` rows (section 9). Our scheme, for reference:

```
TalentTab   800 + k          k = tree number 1..10
Talent      3000 + 100k + i  i = talent number in the tree
Spell       200000 + 1000k + 10i + r   r = rank
Helpers     200000 + 1000k + 900..999
```

Talent points: `Rate.Talent` in `worldserver.conf` scales the number of points
only (`Player::CalculateTalentsPoints`); nothing scales talent effects.
**[Source]** **[Tested]** (52 points at level 61 with rate 1.0).

## 4. Client: data files and the addon

### 4.1 DBCs in an MPQ **[Tested]**
Ship these in a patch MPQ (we used `Data\enUS\patch-enUS-4.MPQ`), with the
exact same rows as the server tables:

`Talent.dbc`, `TalentTab.dbc`, `SkillLine.dbc`, `SkillRaceClassInfo.dbc`,
`Spell.dbc` (and `SkillLineAbility.dbc` once you tag spells into the
spellbook, section 6).

- Client and server DBC rows must match. We byte-compare the server
  `data/dbc` files with the patch files before release.
- Clear the client `Cache` folder after every spell or talent change.
- Copy the MPQ only while the client is closed.

### 4.2 Background art **[Tested]**
Four BLP files per tree under `Interface\TalentFrame\`, named after the
TalentTab internal name: `<name>-TopLeft.blp`, `-TopRight`, `-BottomLeft`,
`-BottomRight`. Until you have art, point a tree at the class's stock third
tab background.

### 4.3 Do not edit Blizzard UI files **[Tested]**
The 3.3.5a client rejects any edited Blizzard interface file shipped in an
MPQ ("Your game interface files are corrupt", at world load). That rules out
editing `TalentFrameBase.lua`, `Blizzard_TalentUI.*` or `Blizzard_InspectUI.*`.
If you ever get that popup, bisect: no MPQ, then DBC-only, then UI files.

### 4.4 The addon (ours) **[Tested]**
A plain addon in `Interface\AddOns\` does the UI work at runtime. It is the
same file for every class and tree: a class only shows the 4th tab when its
TalentTab class mask matches. What it does:

- On `ADDON_LOADED` of `Blizzard_TalentUI`: set `MAX_TALENT_TABS = 4`, move the
  Glyphs tab from id 4 to id 5 (`GLYPH_TALENT_TAB = 5`, also re-name it
  `PlayerTalentFrameTab5`), create a new tab from `PlayerTalentTabTemplate` as
  tab 4, set the frame to 5 tabs.
- On `ADDON_LOADED` of `Blizzard_InspectUI`: add `InspectTalentFrameTab4`
  from `TabButtonTemplate`.
- Inspect fix: the inspect frame keeps its selected tab between inspects.
  Reset it to tab 1 when the selected tab is above the inspected unit's tab
  count (see 5.1 for why this matters).

```lua
-- 4th talent tab addon (ours, written for this project). No Blizzard file is modified.
-- LegendsTalent4.toc:  ## Interface: 30300  /  LegendsTalent4.lua
local function PatchPlayerTalentUI()
	if ( LT4_PlayerDone or not PlayerTalentFrame or not PlayerTalentFrameTab4 ) then return end
	LT4_PlayerDone = true
	MAX_TALENT_TABS = 4
	local glyph = PlayerTalentFrameTab4
	glyph:SetID(5)
	GLYPH_TALENT_TAB = 5
	PlayerTalentFrameTab5 = glyph
	local t = CreateFrame("Button", "LegendsTalentTab4", PlayerTalentFrame, "PlayerTalentTabTemplate")
	t:SetID(4)
	t:SetPoint("LEFT", PlayerTalentFrameTab3, "RIGHT", -15, 0)
	PlayerTalentFrameTab4 = t
	glyph:ClearAllPoints()
	glyph:SetPoint("LEFT", t, "RIGHT", -15, 0)
	PanelTemplates_SetNumTabs(PlayerTalentFrame, 5)
end
local function PatchInspectTalentUI()
	if ( LT4_InspectDone or not InspectTalentFrame or not InspectTalentFrameTab3 ) then return end
	LT4_InspectDone = true
	MAX_TALENT_TABS = 4
	local t = CreateFrame("Button", "InspectTalentFrameTab4", InspectTalentFrame, "TabButtonTemplate")
	t:SetID(4)
	t:SetText("OK")
	t:SetPoint("LEFT", InspectTalentFrameTab3, "RIGHT", 0, 0)
	t:SetScript("OnClick", function(self)
		PanelTemplates_Tab_OnClick(self, InspectTalentFrame)
		InspectTalentFrame_Update()
		TalentFrame_Update(InspectTalentFrame)
		PlaySound("igMainMenuOptionCheckBoxOn")
	end)
	PanelTemplates_SetNumTabs(InspectTalentFrame, 4)
end
-- Inspect keeps its selected tab between inspects; a tab the inspected
-- class does not have crashes the client, so fall back to tab 1.
local LT4_TalentFrame_Update = TalentFrame_Update
function TalentFrame_Update(TalentFrame)
	if ( TalentFrame and TalentFrame.inspect ) then
		local sel = PanelTemplates_GetSelectedTab(TalentFrame) or 1
		if ( sel > 1 and sel > (GetNumTalentTabs(true) or 0) ) then
			PanelTemplates_SetTab(TalentFrame, 1)
		end
	end
	return LT4_TalentFrame_Update(TalentFrame)
end
local f = CreateFrame("Frame")
f:RegisterEvent("ADDON_LOADED")
f:SetScript("OnEvent", function(self, event, name)
	if ( name == "Blizzard_TalentUI" ) then PatchPlayerTalentUI() end
	if ( name == "Blizzard_InspectUI" ) then PatchInspectTalentUI() end
end)
if ( IsAddOnLoaded("Blizzard_TalentUI") ) then PatchPlayerTalentUI() end
if ( IsAddOnLoaded("Blizzard_InspectUI") ) then PatchInspectTalentUI() end
```

Keep the addon loose (not inside the MPQ): you can edit it and `/reload`
without rebuilding the archive.

## 5. Rules the client forces on you

### 5.1 Every class needs the same number of tabs **[Tested]**
If some classes have 3 tabs and others 4, inspecting across them crashes
`Wow.exe` (error #132). The client resizes its inspect talent store when the
tab count differs and then reads freed memory; no addon can fix it. So ship
all classes at once. A tree that is not ready ships as a **placeholder tab**
(real names, tiers and ranks; each rank a passive dummy spell with a "no
effect yet" note), never as a missing tab.

Pet talent tabs use a separate store. Not tested.

### 5.2 Arrows must be drawable **[Tested]**
The 3.3.5a talent window draws a prerequisite arrow only straight down one
column or sideways on one tier, with no talent in between. Lay out the tree so
every hard prerequisite fits, or drop the hard prerequisite and keep it as
tooltip text. Our generator refuses a tree with an undrawable arrow.

### 5.3 Above-level spells
Spells the design borrows from higher levels are templates only. A talent
rank spell must be learnable at the level the talent unlocks.

## 6. Talents that teach spells, and the spellbook

| Want | How | Tag |
|---|---|---|
| A talent that is itself a castable spell | Talent.dbc `Flags` = 1 (the core reads it as `addToSpellBook`). Without it the talent learns but nothing shows in the spellbook | **[Tested]** **[Source]** `Player::addTalent` |
| A talent that teaches a second spell | The rank spell holds **only** learn-spell effects (effect 36), nothing else in the same spell. One level deep: a learned spell may not learn spells. Taught spells go away on unlearn / respec. Stock example: 65139 | **[Tested]** **[Source]** |
| The new spells in a spellbook tab | Add a `SkillLineAbility` row (server table + client DBC) that tags the spell into an existing class skill line (we tag by theme, e.g. a time-magic tree into Arcane). Passives and helpers need no row | **[Tested]** |
| A new weapon type for a class | `SkillRaceClassInfo` row + `SkillLineAbility` class mask + `playercreateinfo_skills`. The core hands out missing default skills at every login | **[Tested]** |

Hooks around respec: `OnPlayerTalentsReset` runs **before** the reset; for
"after the reset" use `OnPlayerFreeTalentPointsChanged`. **[Source]**
**[Tested]**

## 7. Choice tiers (pick one of N)

One of our trees has tiers where you may take only one of several talents.
**[Tested]** **[Source]**

- Do it in a C++ module: `PlayerScript::OnPlayerCanLearnTalent` returning
  `false` vetoes the learn (called from `Player::LearnTalent`). It also blocks
  command learns. No core edit.
- Eluna (mod-ale) cannot do this: it registers only the after-learn hook.

## 8. Playerbots

Stock mod-playerbots parses talent links and counts spec points for 3 tabs
only; a bot never touches a 4th tab without a patch. **[Source]**

What our patch changes (all **[Tested]**; patch is ours, against
mod-playerbots `037c01418b5d`):

| Area | File (mod-playerbots) | Change |
|---|---|---|
| Spec link parser | `PlayerbotAIConfig.cpp` | Loops and arrays to `MAX_TALENT_TABS` |
| Talent sort | `Mgr/Talent/Talentspec.cpp` (`SortTalents`) | Tab arrays `{0,1,2}` → `{0,1,2,3}` |
| Spec detection | `Bot/Factory/AiFactory.cpp` (`GetPlayerSpecTabs`, `GetPlayerSpecTab`) | Count the 4th tab; a bot with most points there plays a stock "proxy" role tab (its rotation) |
| Level-up | `Bot/Factory/PlayerbotFactory.cpp` | A bot built in its 4th tree keeps that premade spec when it levels |
| Spec output | `Actions/ChangeTalentsAction.cpp` | Prints the 4th tab's points |

Then add one premade spec per class in `playerbots.conf` whose talent link
fills the 4th tab. Bots that pick it get the tree's passives and procs and
play the proxy tab's rotation. Teaching bots to cast the new active
abilities is separate work (we have not done it yet).

Two traps: **[Source]** **[Tested]**
- The spec list stops at the first empty `PremadeSpecName` slot, so number
  your new spec right after the last used slot.
- Picking a spec by chat needs the exact full name.

## 9. Changing a live tree safely

- **Never delete a live talent.** A saved talent whose spell is no longer a
  talent makes the server assert at that character's login (`_LoadTalents`).
  **[Tested]** **[Source]**
- To retire one: reset talents, then with the worldserver stopped delete the
  `character_talent` rows for it, confirm 0 rows, and only then load the data
  without it.
- Talent ids follow position in your design list if you generate them by
  index. Once characters hold talents, add new talents at the end of a tree
  or give the generator fixed ids first.
- Restart the worldserver after DB writes to spell tables. A server started
  before the writes keeps the old spells, and it looks like "the proc is
  broken". **[Tested]**

## 10. Build order that worked

1. **Source read first.** Read the talent code paths (learn, reset, dual
   spec, load, inspect packet), the client talent UI Lua, and the bot talent
   code. Rate each risky item: certain / spike first / no.
2. **Spike one tab with existing spells only.** One class, 6-8 talents, 3
   tiers, a prerequisite, a 3-rank talent; rank spells = existing spell ids,
   so no spell work. Prove the tab, learn, respec, dual spec, inspect, bots.
3. **Tooling.** A spec file per tree (tier, column, ranks, prerequisite,
   implementation class) parsed from the design doc; a validator (counts,
   tiers 0-10, columns 0-3, ranks 1-5, drawable arrows, capstone reachable); a
   DBC writer and an SQL writer that also writes the rollback; an MPQ packer.
   Never hand-edit generated files; change the spec and regenerate.
4. **All classes at once as placeholders** (section 5.1).
5. **Per tree:** data-only talents first (stat auras, spell mods, proc rows,
   stacking groups); install and test that playable subset; then scripted
   talents one at a time, smallest first; then a power check on the built
   tree; then bots.

Sort each talent into one class before building: **A** DBC only (stat auras,
spell mods, granted auras), **B** DBC + SQL (proc rows, stacking groups,
server spell rows), **C** script, **D** core edit. Most of a tree is A and B.
How those are built: [02_Custom_Spells.md](02_Custom_Spells.md).

## 11. Test checklist

Passed on our server for the spike and then for every tree: **[Tested]**

- Tab shows with its name; icons and tooltips show.
- Tier lock follows 5 points per tier, counted in the same tab.
- Ranks add and remove; points persist after relog.
- `.reset talents` works; dual spec keeps both builds; Glyphs tab opens in
  both specs.
- `/inspect` across classes, with and without points in the 4th tab: no
  crash.
- Bots of that class log in and spec normally.
- Point total at the level cap unchanged.
- No Lua errors.

## 12. Open questions

Not verified; do not build on these yet.

- Pet talent tabs with a 4-tab player store: not tested.
- Bots casting 4th-tree active abilities: not built.
- Background art from a base archive: base-archive textures cannot be
  replaced from the locale patch; ship them in `Data\patch-N.MPQ`.
  **[Tested]** for spell textures; not tried for talent backgrounds.
