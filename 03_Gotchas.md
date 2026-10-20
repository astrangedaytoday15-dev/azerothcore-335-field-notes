# Gotchas: symptom, cause, fix

**Draft 1.** Traps that cost us time on AzerothCore 3.3.5a. Each line:
**symptom** → cause → fix. Tags: **[Tested]**, **[Source]** (see
[README](README.md)). Spell and talent design traps are explained in more
depth in [02_Custom_Spells.md](02_Custom_Spells.md) and
[01_Fourth_Talent_Tab.md](01_Fourth_Talent_Tab.md); this page is the quick
scan.

## Contents
- [Server start, reloads, crashes](#server-start-reloads-crashes)
- [Talents](#talents)
- [Procs](#procs)
- [Spell data and clones](#spell-data-and-clones)
- [Scripts and threads](#scripts-and-threads)
- [Client files](#client-files)
- [Items](#items)
- [Creatures, loot, instances](#creatures-loot-instances)
- [Playerbots](#playerbots)
- [Building on Windows](#building-on-windows)
- [Database work](#database-work)

---

## Server start, reloads, crashes

- **"The proc / spell change does nothing."** → The worldserver was started
  before the DB writes and keeps the old spell data. → Restart after writes
  to spell tables. **[Tested]**
- **Creature damage rate change has no effect after `.reload`.** →
  `Rate.Creature.*.Damage` is baked into the creature DamageModifier at
  load. → Restart. **[Source]**
- **Server crashes when a shapeshift spell is applied.** → The form id has
  no `SpellShapeshiftForm` row. → Never ship a form spell without its row.
  **[Tested]** **[Source]**
- **Server crashes on a loot roll.** → A loot reference chain loops back to
  itself; there is no recursion guard. → Check references for cycles before
  install. **[Source]**
- **`[1213]` deadlock lines in Errors.log.** → The core retries them. Only
  "Fatal deadlocked" means lost data. **[Source]**
- **Startup error naming a `spell_proc` row.** → Spell DONE flags with
  SpellPhaseMask 0 (the row never fires), unknown spell, no ProcFlags, or
  REQ_SPELLMOD on a spell with no spell mod. → Fix the row; the load does
  not check that the spell has a proc aura at all. **[Source]**

## Talents

- **Server asserts at one character's login after a talent change.** → A
  saved talent whose spell is no longer a talent. → Reset talents and delete
  the `character_talent` rows (server stopped) **before** loading data
  without the talent. **[Tested]**
- **Client crashes on `/inspect` (error #132).** → Classes have different
  talent tab counts. → Same tab count for every class, always; placeholder
  tabs for unfinished trees. **[Tested]**
- **"Interface files are corrupt" popup at world load.** → An edited
  Blizzard UI file in an MPQ. → Use your own addon instead. **[Tested]**
- **Talent learns, but nothing appears in the spellbook.** → Talent.dbc
  Flags is 0. → Set 1 for castable talent spells. **[Tested]**
- **A talent that should teach a spell does nothing.** → The rank spell has
  other effects besides learn-spell, or the taught spell itself tries to
  teach a spell (only one level deep). **[Tested]** **[Source]**
- **"After respec" hook fires too early.** → `OnPlayerTalentsReset` runs
  before the reset. → Use `OnPlayerFreeTalentPointsChanged`. **[Tested]**
- **Talent arrow missing or drawn wrong.** → Arrows only go straight down
  one column or sideways on one tier, with nothing in between. **[Tested]**

## Procs

- **Hand-written proc row never fires from a triggered spell.** → Only
  generated entries get TRIGGERED_CAN_PROC. → Set AttributesMask 0x2.
  **[Tested]**
- **Proc on a talent keyed to Execute never fires.** → Keyed on the
  triggered damage spell (20647) instead of the cast (5308). **[Tested]**
- **Proc keyed to "melee" never fires from Thunder Clap / Hammer of
  Wrath.** → They are ranged damage class (0x100). **[Tested]**
- **"Block" proc never fires.** → A TAKEN proc with no HitMask fires on
  normal and critical hits only. → HitMask 0x40. **[Tested]**
- **A potion proc never fires.** → HIT phase; the item is gone by then. →
  CAST phase. **[Tested]**
- **AoE weapon proc runs away on big pulls.** → A weapon-strike proc spell
  re-rolls item procs per target. → `spellcooldown_N` 1000 on the item.
  **[Tested]**
- **Proc cooldown starts even when the effect does not happen.** → Chance is
  rolled after the cooldown starts. → Chance 100 + roll in `DoCheckProc`.
  **[Tested]**
- **"Procs 5%" seems broken.** → Flat 5% per hit is rare. → Swing ~60 times
  before calling it broken. **[Tested]**
- **A wand with a chance-on-hit effect never procs.** → Wands never proc.
  **[Tested]**

## Spell data and clones

- **Clone deals a random extra 1..N.** → It kept the donor's DieSides. →
  DieSides 1. **[Tested]**
- **Clone silently cannot be cast.** → Leftover class-only fields
  (ShapeshiftMask, Category, ExcludeTargetAuraSpell, Mechanic). **[Tested]**
- **Stock talents keep buffing your clone.** → It kept the donor's
  SpellFamilyFlags. → Give it its own free bit. **[Tested]**
- **Melee ability procs your own weapon effects on yourself.** → A
  self-target effect on a melee-class spell. **[Tested]**
- **Shield refuses to apply next to another buff.** → Stack rule 4 refuses
  the whole aura if one effect is lower. → Split the stacking part into a
  helper. **[Tested]**
- **Stored counter resets.** → An aura refresh resets every effect amount.
  → Keep state in `Object::CustomData`. **[Tested]**
- **Instant shot has a 0.5 s cast.** → Ranged-slot attribute (0x2) on a
  non-auto-repeat spell. **[Tested]**
- **Shot fires from melee range minimum.** → Ranged range rows (Flags 2) add
  melee range to the minimum. → Own range row with Flags 0. **[Tested]**
- **Leap "misses" and the caster stays put.** → JUMP effect 41 needs a clean
  hit roll. → Scripted `MoveJump`. **[Tested]**
- **Action bar locks in a custom form.** → PresetSpell on the form row. →
  Stance-masked passive instead. **[Tested]**

## Scripts and threads

- **Random crash or wrong value from a module-wide map.** → Map updates run
  on several threads. → State on the object (`CustomData`); static guards
  `thread_local`. **[Tested]**
- **Bots trigger your spell's side effect.** → A side effect in CheckCast;
  bots probe spells through it. **[Tested]**
- **Scripted damage ignores all bonuses.** → `SetHitDamage` in OnHit replaces
  the result. → `SetEffectValue` in `OnEffectLaunchTarget`. **[Tested]**
- **Heal hook arguments swapped on ticks.** → `ModifyHealReceived` passes
  (target, caster) for periodic ticks. → Use `OnHeal`. **[Tested]**
- **`GetOwnedAuras()` misses your DoTs on enemies.** → It lists auras on the
  unit, not auras it cast. **[Tested]**
- **Spent resources on a melee hit that then missed.** → The melee damage
  hook runs before the hit roll. **[Tested]**
- **Lua talent veto does not work.** → mod-ale has only the after-learn
  hook; also no "damage taken" hook. → Module C++. **[Source]**

## Client files

- **Client shows the old spell after a change.** → Client Cache. → Clear
  the `Cache` folder after every spell / talent change. **[Tested]**
- **Talent / spell desync between client and server.** → DBC rows differ.
  → Same rows both sides; byte-compare before release. **[Tested]**
- **Texture shows as stripes.** → BLP2 with encoding 3 (uncompressed). →
  DXT1; recolour by moving each block's two end colours and keeping the
  indices. **[Tested]**
- **Replaced texture does not show.** → Base-archive textures cannot be
  replaced from the locale patch. → Ship them in `Data\patch-N.MPQ`.
  **[Tested]**
- **Recoloured effect stays black.** → Effect colour lives in the model
  (M2 colour block, particle tracks); textures are mostly grey masks; a
  black texture stays black. → Give it a white copy. **[Tested]**
- **Creature flame particles ignore the new skin colour.** → They are tinted
  by CreatureDisplayInfo ParticleColorID. **[Tested]**
- **New visuals shift other generated ids.** → New SpellVisual / kit rows
  need their own id range. **[Tested]**

## Items

- **Custom item: "?" icon in bags, right-click use does nothing, Throw
  blocked.** → The item id is not in the client `Item.dbc`. → `/use` macro
  works; ranged custom weapons stay stats-only unless you patch Item.dbc.
  **[Tested]**
- **First Use effect fails on an item.** → The first Use spell is cast
  untriggered, so its power cost applies. **[Tested]**
- **Equip effect disappears after a while.** → The spell has a duration;
  equip spells must be permanent. **[Tested]**
- **Gem fits a socket it should not.** → Stock socket code has no item level
  check. (**Our setup:** we added one in the socket opcode handler.)
  **[Tested]**

## Creatures, loot, instances

- **Creature armor change does not apply on respawn.** → Creature armor is
  set only in UpdateEntry. → Difficulty hooks must set absolute values
  (template × factor). **[Source]**
- **Heroic group gets the normal instance.** → No `(map, 1)` MapDifficulty
  row. Heroic spawns also need spawnMask bit 2 on gameobjects. **[Source]**
- **DamageModifier did not change spell damage.** → It is melee only.
  **[Source]**
- **Map query misses a boss.** → Script-summoned bosses have no creature
  spawn row. → Check the boss list against your query. **[Tested]**

## Playerbots

- **Bots never use a 4th talent tab.** → Parser and spec detection stop at
  3 tabs. → Patch (see the talent tab guide). **[Source]** **[Tested]**
- **New premade spec never picked.** → The spec list stops at the first
  empty `PremadeSpecName` slot. **[Source]** **[Tested]**
- **Bot strategy action silently skipped.** → Action name must equal the
  server spell name exactly. **[Tested]**
- **Bot level log lines show one talent point less.** → They lag one point.
  **[Tested]**
- **Bot `/played` looks slow.** → Only a fraction of bots run AI at once;
  `/played` is mostly idle time. Do not read bot pace from it. **[Tested]**
- **Config keys stop working after a mod-playerbots update.** → Upstream
  master renamed every `AiPlayerbot.*` key to `Playerbots.*` with no
  fallback (as of our read; newer than our pinned commit). **[Source]**
- **Deleting random bot accounts stops the server.** → The module halts the
  worldserver itself after the delete. Set the option back to 0 before the
  next start. **[Source]** **[Tested]**

## Building on Windows

- **`MSB1008` when building from Git Bash.** → Bash rewrites
  `/p:CL_MPCount=2` into a path. → Run the build from PowerShell. **[Tested]**
- **New `.cpp` in a module is not compiled.** → Sources are globbed at
  configure time. → Re-run `cmake .` in the build folder. **[Tested]**
- **`C3064 'uint8'` in a module.** → `SpellAuras.h` / `SpellInfo.h`
  included before `Player.h` / `ScriptMgr.h`. → Include them after.
  **[Tested]**
- **Bash heredoc or `sed` mangles Windows paths.** → Backslashes eaten. →
  Edit files with an editor tool, or build paths in Python with `chr(92)`;
  `"\U"` in a Python string breaks too. **[Tested]**

## Database work

- **Backup will not restore.** → PowerShell `mysqldump ... > file` writes
  UTF-16. → Use `--result-file`. A backup is unproven until loaded once.
  **[Tested]**
- **Half a script applied after an SQL error.** → One file fed to `mysql`
  runs statements before the error. → Check them, restore from the rollback
  file, fix, rerun only that file. **[Tested]**
- **Syntax error from a name like "Blade's Edge".** → Unescaped quote in a
  generated SQL comment. → Escape every name. **[Tested]**
- **Negative result turns into a huge number.** → Unsigned columns. →
  `CAST(x AS SIGNED)` before subtracting. **[Tested]**
- **MySQL 8.4 Error 1235 on `LIMIT` in a subquery.** → Not supported
  there. → Restructure the query. **[Tested]**
- **`INSERT ... SELECT ... UNION ALL` fails.** → Split it into separate
  INSERTs. **[Tested]**
- **`LIKE '[...]'` matches nothing.** → LIKE has no regex brackets. → Use
  REGEXP. **[Tested]**
- **`REPLACE INTO` overwrote a row.** → It overwrites on a duplicate key.
  → Check for existing keys first. **[Tested]**
- **DBeaver runs only half the script (Error 1064).** → A blank line ends a
  statement in DBeaver. → No blank lines in pasted SQL, or Ctrl+A then
  Ctrl+Enter. **[Tested]**
- **Delta tool removed a stock row.** → A `spell_script_names` row on a
  stock spell id, and the tool deletes by spell id. → Never script stock
  ids that way. **[Tested]**
