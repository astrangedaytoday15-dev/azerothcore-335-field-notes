# Custom spells on AzerothCore 3.3.5a: how they work and interact

**Draft 1.** Tags: **[Tested]** in-game, **[Source]** read in source,
**[Data]** checked against DB rows (see [README](README.md)).

How to design spells on purpose instead of by trial and error: where each
behavior lives, how procs are checked and fired, what stacks, what a script
can do. Only tested or source-read facts are stated; open points are listed
at the end.

## Contents
1. [The layers: what lives where](#1-the-layers)
2. [Four ways to make a custom spell](#2-four-ways-to-make-a-custom-spell)
3. [Aura procs (`spell_proc`)](#3-aura-procs-spell_proc)
4. [Item procs (`item_template` spell slots)](#4-item-procs)
5. [The two-layer pattern (wrappers)](#5-the-two-layer-pattern)
6. [Stacking and spell groups](#6-stacking-and-spell-groups)
7. [Spell modifiers](#7-spell-modifiers)
8. [Chaining spells](#8-chaining-spells)
9. [Scripts (SpellScript / AuraScript / module hooks)](#9-scripts)
10. [Values, scaling and damage math](#10-values-scaling-and-damage-math)
11. [Targeting](#11-targeting)
12. [Cloning a stock spell](#12-cloning-a-stock-spell)
13. [Ranged attacks for non-hunters](#13-ranged-attacks-for-non-hunters)
14. [Test protocol](#14-test-protocol)
15. [Open questions](#15-open-questions)

---

## 1. The layers

| Layer | Owns | How you change it |
|---|---|---|
| Client `Spell.dbc` (MPQ patch) | Name, icon, tooltip, visuals, cast bar, buff icon | Client patch **[Tested]** |
| Server `Spell.dbc` (same file, server copy) | All gameplay data: effects, values, auras, targets, durations, cooldowns, attributes, proc flags | `spell_dbc` table rows **[Tested]** |
| World DB tables | Overrides around a spell: `spell_proc`, `spell_group`, `spell_bonus_data`, `spell_script_names`, `spell_linked_spell`, item spell slots | SQL + restart **[Tested]** (except `spell_linked_spell`, see 8) |
| Core / module C++ | SpellScripts, AuraScripts, dummy handlers, hard-coded rules | Compile **[Tested]** |
| Lua (mod-ale / Eluna) | Event hooks | Script; limited (9.4) **[Tested]** |

- An item's "Equip / Use / Chance on hit" line is the **client's** spell
  description. A server-only change never changes the tooltip; it can make it
  wrong. **[Tested]**
- Tooltip `$s1` follows spell mods in the client. **[Tested]**

## 2. Four ways to make a custom spell

| Path | What | Proven | Limit |
|---|---|---|---|
| 0 | Reuse stock spell ids only | **[Tested]** | Only what Blizzard shipped; zero client work |
| 1 | Re-tune with DB tables (`spell_proc`, `spell_group`, `spell_bonus_data`, `spell_script_names`) | **[Tested]** on custom spells | Server only; restart needed; affects every user of that spell id |
| 2 | New spell ids on the server via `spell_dbc` | **[Tested]** together with path 3 | The client does not know the spell without path 3 |
| 3 | Client + server `Spell.dbc` patch | **[Tested]** | Rows must match on both sides; clear the client Cache |
| 4 | C++ scripts in a module | **[Tested]** | Needs your own build |

The server loads `spell_dbc` rows as part of Spell.dbc, so no server DBC file
is edited. **[Source]** (`DBCStores.cpp`)

Habit that saved tests: one new mechanic enters with **one** chosen spell id;
later content reuses it, so one test proves it for the project.

## 3. Aura procs (`spell_proc`)

The engine behind "when struck", "on cast", heal procs, kill procs, and any
aura with proc flags. Not the item chance-on-hit path (section 4).

### 3.1 Which table
`spell_proc` is live. `spell_proc_event` is a dead leftover; never edit it.
**[Source]** A row overrides the Spell.dbc proc data for that spell id.

With no row, the core **generates** an entry at startup for every spell that
has a trigger-type aura **and** DBC ProcFlags (hit phase, default hit mask,
chance and charges from the DBC, no cooldown). A spell with neither a row
nor a generated entry never procs. **[Source]** (`SpellMgr::LoadSpellProcs`,
`Aura::GetProcEffectMask`)

### 3.2 Event flow **[Source]**
1. Something happens (hit, cast, heal, tick, kill, death). The core calls
   `Unit::ProcSkillsAndAuras` with the actor's DONE flags and the target's
   TAKEN flags.
2. **Check round.** Every aura on the actor, then every aura on the target,
   is checked. Nothing fires yet. Per aura, in order:
   1. has a proc entry
   2. the event's spell was not triggered by this same aura
   3. the triggered-spell rule (3.4)
   4. item-cast block (attribute 0x100)
   5. charges left
   6. internal cooldown clear (`Cooldown`)
   7. row filters: ProcFlags, school, family + mask, SpellTypeMask,
      SpellPhaseMask, HitMask
   8. REQ_SPELLMOD: the spell was changed by this aura's spell mod
   9. `conditions` rows of source type "spell proc"
   10. script gate `DoCheckProc`
   11. per effect: DisableEffectsMask, `DoCheckEffectProc`
   12. passive aura with EquippedItemClass: the item is equipped
   13. **last:** the roll (PPM or Chance)
   A passing aura is prepared at once: one charge taken, cooldown started.
3. **Fire round.** Actor's auras fire, then the target's: `OnProc` → each
   effect → `AfterProc` → remove at 0 charges, or drop one stack with
   USE_STACKS_FOR_CHARGES.

Consequences:
- The cooldown starts only on a successful roll (unless the spell has
  AttributesEx2 PROC_COOLDOWN_ON_FAILURE). **[Source]**
- All checks of one event finish before any proc fires, so an aura added by
  one proc is not seen by another proc's check on the same hit. **[Tested]**
- To start a cooldown only when your effect really happens: Chance 100 and
  roll the real chance in `DoCheckProc`. **[Tested]**

### 3.3 Columns **[Source]** (`SpellMgr.h`, `SpellMgr::LoadSpellProcs`)

| Column | Meaning |
|---|---|
| `SpellId` | Spell carrying the proc aura. Negative = first rank, covers all ranks |
| `SchoolMask`, `SpellFamilyName`, `SpellFamilyMask0-2` | Filters on the triggering spell (0 = any) |
| `ProcFlags` | Which event kinds. 0 = use the spell's own DBC flags |
| `SpellTypeMask` | 1 damage, 2 heal, 4 neither |
| `SpellPhaseMask` | 1 cast, 2 hit, 4 finish |
| `HitMask` | Outcome filter. 0 = default: TAKEN = normal + crit; DONE = normal + crit + absorb |
| `AttributesMask` | 0x1 REQ_EXP_OR_HONOR, 0x2 TRIGGERED_CAN_PROC, 0x4 REQ_MANA_COST, 0x8 REQ_SPELLMOD, 0x10 USE_STACKS_FOR_CHARGES, 0x80 REDUCE_PROC_60, 0x100 CANT_PROC_FROM_ITEM_CAST |
| `DisableEffectsMask` | Bit per effect to ignore |
| `ProcsPerMinute` | Chance from attack speed instead of a flat % |
| `Chance` | Flat % per event. 0 = DBC chance (only when PPM is 0 too) |
| `Cooldown` | Internal cooldown, ms |
| `Charges` | 0 = the spell's DBC ProcCharges |

Hard rules, all **[Tested]**:
- **SpellPhaseMask is required for spell DONE flags.** With it 0 the proc
  never fires; the error log names the row at startup.
- A TAKEN proc with no HitMask fires on normal and critical hits only. A
  blocked hit needs HitMask 0x40; dodge / parry triggers use 0x10 / 0x20 on
  the melee TAKEN flags.
- Check the triggering spell's damage class before picking ProcFlags.
  Thunder Clap (6343) and Hammer of Wrath (24275) are **ranged** class
  (0x100), not melee.
- Key a proc on the spell the player **casts**, not on its triggered damage
  spell: Execute 5308, not 20647.
- A proc on an item cast (a potion) must use the CAST phase; in the HIT
  phase the item is already gone.
- Kill procs run before the victim's auras are removed, so a kill script can
  still read them.
- A proc aura can sit on the **enemy** and fire on its casts; the script then
  makes the owner cast.

### 3.4 The triggered-spell rule **[Source]** **[Tested]**
A triggered cast cannot proc an aura unless one of these holds:
- the aura's spell has AttributesEx3 CAN_PROC_FROM_PROCS
- its `spell_proc` row has TRIGGERED_CAN_PROC (0x2)
- the event is an auto attack (flags 0x4 / 0x8 / 0x40 / 0x80); an
  auto-repeat shot cast fully triggered still procs
- the event is a kill / killed / death
- the triggered spell itself has AttributesEx3 NOT_A_PROC

Also: an aura never procs from a spell it triggered itself. Periodic ticks
are passed without a spell cast, so the rule does not block procs from DoT /
HoT ticks. Generated entries get 0x2 by themselves for some aura types; **a
hand-written row gets nothing: set 0x2 yourself.**

This is the main brake on proc loops. If layer B must proc layer C, set it
on purpose and loop-test it.

### 3.5 Chance and PPM **[Source]**
- PPM chance % = speed_ms × PPM / 600. Speed = the weapon's attack time for
  auto attacks, melee- and ranged-class spells; otherwise the spell's cast
  time, at least 1500 ms. A slow weapon procs more per swing, the same per
  minute.
- Spell mod 26 changes PPM; spell mod 18 changes the final chance.
- REDUCE_PROC_60 lowers the chance by 1/30 per level **above** 60 only.

### 3.6 Worked example **[Data]**
Val'anyr's stock row (spell 64415): ProcFlags 0x44400 (done periodic + done
positive magic spell + done positive spell without damage class),
SpellTypeMask 2 (heals), SpellPhaseMask 2 (hit), Chance 0 (DBC 10%),
Cooldown 45000. Reads as: "when you land a heal, 10%, at most once per 45 s".
A good template for any heal proc.

## 4. Item procs

Item `spellid_N` with `spelltrigger_N` = 2 ("Chance on hit") runs through
`Player::CastItemCombatSpell`, **not** `spell_proc`. **[Source]** **[Tested]**

| Fact | Tag |
|---|---|
| All equipped slots are scanned, not only weapons: a head item with trigger 2 procs on the wearer's melee hits. Armor procs work | **[Tested]** |
| Wands never proc | **[Tested]** |
| `spellppmRate_N` set = chance from PPM and weapon speed; 0 = the spell's own ProcChance | **[Tested]** |
| `spellcooldown_N` on a trigger-2 spell acts as the internal cooldown (per spell, per player). It is a different gate from `spell_proc.Cooldown` | **[Tested]** |
| Item chance-on-hit procs ignore `spell_proc`; the triggered spell's cooldown is the ICD; "self" effects hit the wearer | **[Source]** |
| If the proc spell is itself a weapon strike (Cleave 15496, Whirlwind 52028), every target it hits re-rolls the item's procs. Brake: `spellcooldown_N` 1000 on AoE procs | **[Tested]** |
| The **first** Use spell on an item is cast untriggered (normal cost + checks); a first Use spell with a power cost fails. Later Use spells are fully triggered | **[Tested]** |
| Equip spells (trigger 1) are fully triggered casts; a spell with a duration **expires**, so equip effects must be permanent auras | **[Tested]** |

## 5. The two-layer pattern

Many procs are two spells: **[Data]** **[Tested]**
- **Layer 1, the wrapper:** a passive aura of type 42 (proc trigger spell)
  with a trigger spell. Proc flags, chance and ICD live here (its DBC data or
  its `spell_proc` row).
- **Layer 2, the effect:** the triggered spell that does the damage / buff.
  The strength lives here.

Always open the triggered spell: 16428 looks small but fires 16427 (a DoT
worth 360 damage per proc at 25%). Stock examples: 4245 → 4244, 9233 → 9057,
5364 → 5363, 27867 → 27868.

A third layer (an aura with its own proc entry, or a spell mod that works
only while layer 2 is up) gives "combo" designs. Write down each layer's
flags, ICD and "may proc / may not proc" before building.

## 6. Stacking and spell groups

| Fact | Tag |
|---|---|
| `StackAmount` = max stacks of the same spell from the same caster; a recast refreshes | **[Tested]** |
| `ProcCharges` = proc uses before the aura ends | **[Tested]** |
| A spell-mod aura with charges loses **all** stacks at 0 charges: a free "spend all stacks on the next cast" | **[Tested]** |
| A refresh resets every effect's base amount: never keep state in an aura amount; use the caster's `Object::CustomData` | **[Tested]** |
| Stack rules (`spell_group_stack_rules`): 0 default, 1 exclusive, 2 exclusive from the same caster, 3 exclusive for the same effect, 4 exclusive, highest wins | **[Source]** (`SpellMgr.h`) |
| Rule 3 keeps only the highest of the same effect; the aura itself is never refused | **[Tested]** |
| Rule 4 (e.g. group 1062, attack power) refuses the **whole** aura if one effect is lower. Split the part that must stack into a helper spell | **[Tested]** **[Source]** (`Aura::CanStackWith`) |
| Spell haste aura 65 stacks unless a spell group says otherwise; Icy Veins and Netherwind are in no group | **[Source]** |
| Run speed: only MOD_SPEED_ALWAYS multiplies; other speed auras take the maximum. Mounted speed auras are a separate bucket | **[Tested]** |

## 7. Spell modifiers

Aura 107 (flat) and 108 (percent): "for spells matching
`EffectSpellClassMask` + family, change property X by N". Class-specific by
construction: dead for other classes. **[Source]** **[Tested]**

`EffectMiscValue` = the operation (`SpellModOp`, in `SpellDefines.h`):
**[Source]**

| Op | Property | | Op | Property |
|---|---|---|---|---|
| 0 | damage | | 16 | resist-miss chance |
| 1 | duration | | 17 | jump targets |
| 2 | threat | | 18 | chance of success |
| 3 | effect 1 | | 19 | activation time |
| 4 | charges | | 20 | damage multiplier |
| 5 | range | | 21 | global cooldown |
| 6 | radius | | 22 | dot |
| 7 | crit chance | | 23 | effect 3 |
| 8 | all effects | | 24 | bonus multiplier |
| 9 | not lose casting time | | 26 | procs per minute |
| 10 | casting time | | 27 | value multiplier |
| 11 | cooldown | | 28 | resist dispel chance |
| 12 | effect 2 | | 29 | crit damage bonus 2 |
| 13 | ignore armor | | 30 | cost refund on fail |
| 14 | cost | | | |
| 15 | crit damage bonus | | | |

**Tested in a talent:** ops 0, 1, 3, 5, 6, 7, 9, 10, 11, 14, 15, 19, 22, 23.
Op 19 on a DoT changes its tick time. DoT damage uses op 22, direct damage op
0. Spell pushback talents use 107 / op 9. No spell mod changes a stock
spell's school.

Notes, all **[Tested]**:
- A "next cast is free" buff (spell mod + 1 charge) needs no `spell_proc` row;
  set ProcTypeMask to the **target** spell's damage class (a ranged shot =
  0x100).
- A spell-mod aura with charges drops on use only when a cast used the mod.
- Chain targets: op 17 adds jumps; DamageMultiplier compounds per jump (50%,
  25%).
- `ManaCostPct` is a whole number.

## 8. Chaining spells

| Way | Notes | Tag |
|---|---|---|
| Effect 64 TRIGGER_SPELL, 142 TRIGGER_SPELL_WITH_VALUE; aura 42 PROC_TRIGGER_SPELL, 231 PROC_TRIGGER_SPELL_WITH_VALUE, 23 PERIODIC_TRIGGER_SPELL | Data-only | **[Tested]** (42, 231, 23) **[Source]** (numbers, `SharedDefines.h`, `SpellAuraDefines.h`) |
| Aura 23 with a period | "Every N s cast X on the holder": the data-only heartbeat | **[Tested]** |
| A clone's free third effect pulsing with aura 23 | A "while active" effect | **[Tested]** |
| One ground cloud that hurts enemies and heals allies | Side is picked per effect from TargetB | **[Tested]** |
| `spell_linked_spell` | Pure SQL; see the trap below | **[Source]** only |

`spell_linked_spell` per the source (`SpellMgr::LoadSpellLinked`): type 0 = on
cast (or, with a negative trigger, on aura removal); type 1 = on hit; type 2
= aura-on-aura while A is up. Negative effect = remove / immunity instead of
cast. **ID trap:** the core stores type 1 under spell id + 200000 and type 2
under id + 400000 in the same list as type 0, so a type 1 row on stock spell
N also fires when spell N + 200000 is cast. Check your custom id ranges.
Not tested in play.

## 9. Scripts

### 9.1 Dummies do nothing on their own **[Tested]**
Aura 4 (dummy), 226 (periodic dummy) and effect 3 (dummy) need C++ that
handles the spell. Check for a script before you pick a dummy spell.
Register with `spell_script_names` (spell_id, ScriptName; negative id = all
ranks). Never put a `spell_script_names` row on a **stock** spell id if your
tooling deletes by spell id: it would take the stock row with it.

### 9.2 Hooks **[Source]** (`SpellScript.h`)
- SpellScript: BeforeCast, OnCast, AfterCast, OnCheckCast, OnEffectLaunch,
  OnEffectLaunchTarget, OnEffectHit, OnEffectHitTarget, BeforeHit, OnHit,
  AfterHit, target-select hooks.
- AuraScript: apply / remove / periodic / calc amount / calc periodic / calc
  spell mod / absorb / mana shield / split / dispel / proc hooks
  (DoCheckProc, DoCheckEffectProc, DoPrepareProc, OnProc, AfterProc,
  OnEffectProc, AfterEffectProc). `DoAfterCheckProc` is declared but nothing
  calls it.
- Module hooks we used: `AllSpellScript::OnSpellCheckCast` (cast gate),
  `UnitScript::ModifySpellDamageTaken` (every direct spell hit),
  `PlayerScript::OnPlayerCanLearnTalent`, `OnHeal`.

### 9.3 Script facts, all **[Tested]**
- **CheckCast must have no side effects:** playerbots probe spells through it.
- To act right after a spell finishes applying, queue a 1 ms event on the
  caster.
- Scripted damage: set the base with `SetEffectValue` in
  `OnEffectLaunchTarget`. `SetHitDamage` in OnHit **replaces** the result and
  drops every damage done / taken bonus. For a % bonus on top of all others,
  multiply `GetHitDamage()` in OnHit.
- The CalcAmount hook runs **before** the stack multiply.
- A helper that re-deals damage already holding the caster's bonuses needs
  "no caster bonuses" (AttributesEx3 0x20040000).
- An absorb with no fixed limit: amount -1, the script sets the absorb.
- "When the shield breaks": a used-up absorb is removed with
  `AURA_REMOVE_BY_ENEMY_SPELL`. **[Source]** (`Unit.cpp`)
- No "dispel succeeded" hook on the spell side: count the target's auras
  before and after.
- `ModifyHealReceived` gets (healer, target) for direct heals but (target,
  caster) for periodic ticks; `OnHeal` is consistent.
- `Unit::GetOwnedAuras()` = auras **on** the unit, not auras it cast.
- State between events lives on the object (`Object::CustomData`), not in a
  module-wide map: map updates run on several threads. A static guard must be
  `thread_local`.
- The melee damage hook runs **before** the hit roll: do not spend anything
  there.
- Aura 64 mana leech is capped by the core; a real drain is a script.
- `OnPlayerCanEquipItem` can only refuse, never allow. Letting a class equip
  something new is a core edit (CanEquipItem, the two-hand check and the
  auto-unequip of the off-hand all need the same gate).
- Summoned helpers: `SummonCreature` + an AI copied from the Army of the Dead
  ghoul AI (attacks only enemies already fighting the owner); set stats in
  `IsSummonedBy`; keep the lifetime as an AI timer so a death / fade hook can
  still run.

### 9.4 Lua (mod-ale) limits **[Source]** **[Tested]**
- Cannot veto a talent learn (only the after-learn hook).
- Its damage hooks fire for damage / heal a player **deals**; there is no Lua
  hook for damage a player **takes**. Use module C++.
- A bool handler returning nil keeps the hook's default.
- `GetTalentTreePoints` never sees a 4th tab. `SpellInfo` has no id getter.

## 10. Values, scaling and damage math

Effect value: **[Source]** (`SpellEffectInfo::CalcValue`)
```
value = BasePoints
      + int( RealPointsPerLevel * (L - max(BaseLevel, SpellLevel)) )
      + die roll
L = caster level, capped at MaxLevel (if > 0), raised to BaseLevel if below
die roll: DieSides 0 -> +0, 1 -> +1, N > 1 -> random 1..N
then + PointsPerComboPoint * combo points, then the caster's spell mods
```
The level term goes **negative** below SpellLevel when SpellLevel >
BaseLevel. A value passed in by a script or aura 231 replaces BasePoints and
is exact.

| Fact | Tag |
|---|---|
| A clone keeps the donor's DieSides and rolls it on top of your amount: set DieSides 1 | **[Tested]** |
| An effect's strength lives in the spell, not the item; item level does not scale a spell | **[Tested]** |
| Spell power / AP coefficients: DBC EffectBonusMultiplier, the default cast-time formula, or a `spell_bonus_data` row (`ap_bonus` tested) | **[Tested]** |
| Player AoE cap: over 10 targets each takes 10/N of the damage (area effects, chains of non-magic class) | **[Source]** (`Spell::DoAllEffectOnTarget`) |
| Level penalty (players only): `min(1, (SpellLevel + 6) / casterLevel) × (1 - P)`, P = (20 - SpellLevel) × 3.75% below SpellLevel 20. No penalty when SpellLevel is 0 or ≥ MaxLevel. Multiplies the bonus only, not the base value | **[Source]** (`Unit::CalculateLevelPenalty`) |
| A finisher that lasts longer per combo point: AttributesEx 0x400000 + a SpellDuration row (base + (max - base) × points / 5) | **[Tested]** |
| `Rate.Creature.*.Damage` is baked into creature DamageModifier at load: needs a restart, not `.reload` | **[Source]** |

## 11. Targeting

- Implicit target ids used in our builds: 1 caster, 6 enemy target, 15 / 16
  area around source / destination (enemy), 20 caster-area party, 21 ally
  target, 22 source = caster, 25 any target, 31 area around destination
  (ally), 53 destination = enemy target, 87 destination = chosen spot.
  **[Source]** (`SharedDefines.h`) **[Tested]** (in use)
- For a triggered spell, the caster is the unit holding the proc aura. Its
  explicit target is the event's target on a DONE event, and the
  **attacker** on a TAKEN event. **[Source]** **[Tested]**
- Never give a melee-class spell a self-target effect: the caster becomes a
  second target and weapon procs fire on it. **[Tested]**
- Always run a friendly-NPC test on the first build of any AoE proc.

## 12. Cloning a stock spell

All **[Tested]**:
- Clear the class-only fields: ShapeshiftMask, Category,
  ExcludeTargetAuraSpell, Mechanic. A leftover one silently blocks the cast.
- Clear class masks on effects you do not use.
- A clone keeps the donor's SpellFamilyFlags, so every stock talent that
  modifies the donor keeps modifying the clone. Give it its own bit to opt
  out; pick and record free family bits per class **before** a build (some
  stock bits carry core meaning).
- An immunity spell's AttributesEx 0x8000 (immunity purges effects) comes
  along with a clone; drop it when re-purposing.
- Weakened Soul comes from Power Word: Shield's ExcludeTargetAuraSpell; a
  ward that must not cause it cannot use that column.
- Aura 147 takes a preset id in MiscValue, not a mechanic mask; one aura 77
  per mechanic gives real immunity.
- A custom disease (Dispel = disease, periodic damage) counts for every "per
  disease" bonus, but stock code that names specific disease ids ignores it.
- Channel while moving: clear 0x8 (and 0x4) from ChannelInterruptFlags and
  set AttributesEx5 0x1.
- JUMP effect 41 runs only on a clean hit roll; a melee-class leap can
  "miss" and not move. A scripted `MoveJump` is reliable.
- Aura 262 lifts an aura-state gate for every target, and the client
  follows it.
- A PresetSpell on a shapeshift form row locks the action bar; use a
  stance-masked passive instead. A form id with no `SpellShapeshiftForm` row
  crashes the server on apply.
- The server reads SpellVisual only for missiles, so client-only look rows
  are safe; check that the donor spell has the visual kit first.
- A helper's `$s1` in its hover shows the helper's own base value, not the
  value the talent passed in: no `$s` in such a hover.

## 13. Ranged attacks for non-hunters

All **[Tested]** unless tagged:
- The client will not fire a bow / gun / crossbow weapon shot without ammo
  in the paper doll (Death Knights have no ammo slot). Make the shot a plain
  school-damage spell with no item requirement; a script checks the bow and
  sets the damage.
- A passive aura is never sent to the client, so a talent that grants an
  equip spell (ammo) does nothing client-side. **[Source]**
- Only Auto Shot (75) keeps auto-repeating while moving; a custom auto shot
  needs a core edit.
- A non-auto-repeat spell with the ranged-slot attribute (0x2) gets +500 ms
  cast time; drop the flag for instant shots.
- Your own character draws the bow only for ranged-slot spells;
  `SetSheath(RANGED)` shows it to other players only.
- Hunter shot ranges have Flags 2 (ranged): the core adds melee range to the
  minimum. A custom range with Flags 0 and min 1 shoots almost point blank.
- Hunters carry a hidden +15% ranged haste passive (34082); other classes
  with the same bow are 15% slower.
- Auto-repeat shots run on the ranged attack timer: auras 140 and 192 speed
  them up, melee haste aura 138 does not.

## 14. Test protocol

Every new **construct** (layering, new flags, new proc row) is new, even if
each spell id is known. Per construct:

1. `.list auras` on self: wrapper present, right id, right duration.
2. Tooltip on item and buff bar: record what the client shows.
3. Proc rate: ~60 swings / casts for a 5% proc; count procs.
4. ICD: no two procs inside the window (combat log).
5. Layer B / C: `.list auras` on self and target during the buff.
6. Loop test: 4+ mob pull; no back-to-back procs inside 1 s.
7. Friendly-fire test next to a friendly NPC.
8. Unequip / unlearn: all auras gone.
9. Server-side cast check: `.cast <spell>` separates client blocks from
   server blocks.

GM commands used here (verified in `cs_*.cpp`): `.additem`, `.go creature
id`, `.list auras`, `.cast`, `.learn`, `.setskill`, `.reset talents`.

## 15. Open questions

Not verified on our server; treat as hypotheses.
- A `spell_proc` row on a **stock** spell id (we only wrote rows on custom
  spells).
- `spell_linked_spell` and `conditions` in play.
- Server-only custom spells (path 2 without the client patch): expected no
  tooltip, no icon, maybe a blank buff; every spell we built shipped both.
- Spell mod ops 4, 8, 12, 13, 17 as a talent op, 18, 20, 21, 24, 26-30.
- Whether item-fired class spells gain spell power.
- Two items with the same **passive** equip spell: the source says two auras
  coexist; whether the effect doubles depends on the aura type. Not tested.
