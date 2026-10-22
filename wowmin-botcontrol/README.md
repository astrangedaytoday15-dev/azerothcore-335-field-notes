# Bot Control for WoWMin: command your playerbots, steer your group

**Draft 1.** Patches for [WoWMin](https://github.com/Jellypowered/wowmin)
(a web admin for AzerothCore) and its
[mod-wowmin-telemetry](https://github.com/Jellypowered/mod-wowmin-telemetry)
module that let the server owner give
[mod-playerbots](https://github.com/mod-playerbots/mod-playerbots) bots
orders from the browser, and lead a dungeon group with clicks: move your own
character (as a self-bot) and the group follows; mark the next pack and the
group pulls it.

It builds on [World 3D](../wowmin-world3d/README.md) (the open-world 3D view)
from this repo. Install that first.

Tags: **[Tested]** passed a live test on our server; **[Built]** built and
unit-tested, not tried live; **[Source]** read in source at the revisions
in section 2.

## Contents
1. [What you get](#1-what-you-get)
2. [Requirements](#2-requirements)
3. [Install](#3-install)
4. [Use](#4-use)
5. [Console commands](#5-console-commands)
6. [What is allowed (and what is not)](#6-what-is-allowed-and-what-is-not)
7. [How it works](#7-how-it-works)
8. [Test checklist](#8-test-checklist)
9. [Troubleshooting](#9-troubleshooting)
10. [Roll back](#10-roll-back)
11. [Known limits](#11-known-limits)
12. [License and credits](#12-license-and-credits)

---

## 1. What you get

| Feature | Status |
|---|---|
| **Command panel** for a selected bot on the Live Map, in World 3D and in Instance Watch: Stay, Grind, Flee, Reset AI, Release | **[Tested]** |
| Follow, Summon, Attack, Tank attack for a bot in **your** group | **[Tested]** |
| **Strategy chips**: the bot's live strategies; click one to remove it, "+" adds one from a short list; a change shows in yellow when the next live sample has it | **[Tested]** |
| A text box for one allowed command, greyed out for anything the server would refuse; the server's reply under it | **[Tested]** |
| **Group bar** in World 3D and Instance Watch: **Leader** mode (your character as a self-bot leads, the group follows) and **Squad** mode (the bots hold and you place them) | **[Tested]** |
| **Move**: click the floor (3D view, or the flat dungeon map) and the leader, the selected bot or the whole squad walks there (pathfinding, never a teleport) | **[Tested]** |
| **Mark**: click a mob to put a skull on it; **Mark nearest** works like Tab (nearest live hostile mob to you); bots attack the skull | **[Tested]** |
| Hold, Regroup, Clear mark | **[Tested]** |
| Every command checked on the server: bots only, an allowlist, your own group only, a rate limit, logged | **[Tested]** |
| Two off switches: a WoWMin checkbox (no restart) and a module setting (off by default) | **[Tested]** |
| Every sent command logged by WoWMin to `bot-commands.jsonl` | **[Tested]** |
| **Point box**: click the ground in World 3D or Instance Watch (no tool armed) for its x y z, copied as is, as `moveto x y z` for the text box, or as `go x;y;z` to whisper a bot in game | **[Built]** |

What it does **not** do: no items, gold, mail, trades, teleports, gear or
talent changes, nothing for other players' characters or their bots, and
nothing runs by itself.

## 2. Requirements

Tested on:

| Part | Version |
|---|---|
| Client | WotLK 3.3.5a (12340) |
| Core | `mod-playerbots/azerothcore-wotlk`, branch `Playerbot`, `f19a18799a35` |
| mod-playerbots | `037c01418b5d` |
| mod-wowmin-telemetry | `804b896` + World 3D's module patches 0001 and 0002 |
| WoWMin | `27ed100` + World 3D's 5 patches |
| OS / tools | Windows 11, MSVC, Node 24, MySQL 8.4 |

You need:

- A worldserver built from source with mod-playerbots and
  mod-wowmin-telemetry (the module patch is C++ and needs a rebuild).
- WoWMin set up with its console (SOAP) connection: a game account with GM
  level 3, `SOAP.Enabled = 1` in `worldserver.conf`. WoWMin's own README
  covers this.
- Your game account's **id** (not the name):
  `SELECT id, username FROM acore_auth.account WHERE username = 'YOURNAME';`
  (use your auth database's name).
- For Leader mode: self-bots allowed for your account.
  `AiPlayerbot.SelfBotLevel` in `playerbots.conf`: 0 off, 1 GM accounts only
  (the default), 2 all players, 3 on at login. A GM account works with the
  default.

## 3. Install

### 3.1 The module (server side)

```
cd <azerothcore>/modules/mod-wowmin-telemetry
git checkout 804b896
git apply <field-notes>/wowmin-world3d/mod-wowmin-telemetry/0001-continent-radius-stream.patch
git apply <field-notes>/wowmin-world3d/mod-wowmin-telemetry/0002-older-playerbots-compat.patch
git apply <field-notes>/wowmin-botcontrol/mod-wowmin-telemetry/0001-bot-control-and-group-steering.patch
```

The World 3D compat patch (0002) is for older mod-playerbots revisions like
ours. The bot control patch also applies without it, but we have only built
and tested it **with** it. On a newer mod-playerbots, check that these exist
before you build: `sPlayerbotsMgr.GetPlayerbotAI`, `PlayerbotAI::HandleCommand`,
`IsRealPlayer`, `IsSelfBot`, `AiObjectContext::GetSupportedStrategies`, the
`"position"` value (`PositionValue.h`), `sPlayerbotAIConfig.commandPrefix`
and `.sightDistance` **[Source]**.

Rebuild the worldserver. Only the module's one source file changes, so it is
short (about 2 minutes for us with MSVC, target `worldserver`). Keep the old
`worldserver.exe` (and `.pdb`) next to the new one under another name, to roll
back.

Add to your live `configs/modules/mod-wowmin-telemetry.conf` (the
`.conf.dist` from the patch documents them):

```
WowMinTelemetry.BotCommand.Enable = 1
WowMinTelemetry.BotCommand.OwnerAccount = <your account id>
# optional, the default is 60 (5-250); raise it if long moves come back
# "too far" (we run 250):
WowMinTelemetry.BotCommand.MoveMaxYards = 60
```

`WowMinTelemetry.Enable` must be 1 as well (WoWMin's telemetry already needs
it). Restart the worldserver. Check it in the worldserver console:

```
wowmin botcmd
```

It answers with a usage line. `WBOTCMD|disabled` means `Enable` is still 0.

### 3.2 WoWMin (web side)

```
git clone https://github.com/Jellypowered/wowmin
cd wowmin
git checkout 27ed100
git am <field-notes>/wowmin-world3d/wowmin/*.patch
git am <field-notes>/wowmin-botcontrol/wowmin/*.patch
npm ci
npm run build
```

Add to WoWMin's `.env` (or your WoWMin settings file):

```
WOWMIN_BOT_OWNER_ACCOUNT=<your account id>
```

This is how WoWMin knows which character is yours: it shows the master
buttons for bots in your group and finds the leader for the group bar.
Without it, any real player grouped with a bot counts for the master
buttons, and the group bar does not appear. The server checks
`OwnerAccount` either way.

Start WoWMin (`npm start`, or your usual start script).

### 3.3 Turn it on

Live Map → **Options** → tick **Bot commands**. It is off after a fresh
install. WoWMin keeps the setting in its data folder (`bot-commands.json`), so
it survives restarts. Unticking it hides every panel and makes WoWMin refuse
to send, without a restart.

## 4. Use

### 4.1 Command panel (one bot)

Select a bot:
- **Live Map:** click it. A "Commands · Name" box appears above the bottom
  bar.
- **World 3D:** follow it. The box is at the top right.
- **Instance Watch:** click it in the party list. The box is a card under
  "Bot decisions".

In the box:
- **Stay, Grind, Flee, Reset AI, Release** work on any bot.
- **Follow, Summon, Attack, Tank attack** (yellow border) show only for a
  bot in your group while your character is online. They act through your
  character: Attack means "attack what I have targeted".
- **Strategy chips** need live data: the bot followed in World 3D, or in a
  watched instance. "In combat" chips are combat strategies (`co`), "Out of
  combat" ones the rest (`nc`). Click a chip to remove it; pick from "+" to
  add one. A chip that changes turns yellow for a few seconds.
- **Text box:** one command from the allowlist (section 6), for example
  `nc +food` or `moveto -10238.6 1219.9 42.2`. The box turns red and Send
  greys out for anything not allowed.
- The **reply line** shows what the server said: `ok: stay`,
  `refused: needs the owner as master`, and so on.

Each send is also written to WoWMin's activity log and to
`<data dir>/bot-commands.jsonl` (time, bot, command, reply, source).

### 4.2 Group steering (your group)

**Leader mode** (the default) **[Tested]**:

1. In game, group with your bots and turn your character into a self-bot:
   type `.playerbots bot self` in chat. The same command turns it off. While
   it is on, your character plays itself: keep your hands off the keyboard.
2. In WoWMin, follow your character in World 3D (Live Map → select → 3D), or
   open Instance Watch in a dungeon. The **Group** box shows your name and
   how many bots follow you.
3. **Move**, then click the floor: your character walks there and the bots
   follow. Esc cancels an armed tool.
4. **Mark nearest** (like Tab): a skull goes on the nearest live hostile mob
   within 60 yards of you. Your character and the bots attack it. That is
   your pull button.
5. **Mark**, then click a mob: the skull goes on that one instead.
   **Clear mark** removes it.
6. **Hold**: your character stops where it is. **Move** again to go on.

**Squad mode** (your character stays put) **[Tested]**:

1. Click **Squad**. Every bot in your group gets "stay".
2. **Move**, then click the floor: the bots walk there (spread on a small
   ring) and stay there. Follow one bot in World 3D, or select it in
   Instance Watch, and Move moves only that bot.
3. **Hold**: every bot stays where it is.
4. **Regroup**: every bot follows you again, and the bar goes back to Leader
   mode.

### 4.3 Send a bot anywhere: copy a point **[Built]**

Click the ground with no group tool armed: in World 3D anywhere on the
terrain, in Instance Watch on the floor (not on a player). A small box shows
the point's `x y z`, already selected for Ctrl+C. Its buttons copy:

- **Copy**: `x y z`, for notes or the console (`wowmin botcmd <bot> moveto x y z`);
- **moveto**: `moveto x y z`, to paste into a bot's text box (section 4.1);
- **go**: `go x;y;z`, to whisper a bot of yours in game; it walks there
  with pathfinding.

Esc or x closes the box; the next click replaces it. On the flat dungeon map
the height is your character's (the module finds the ground near it).

A walk through a dungeon, Leader mode: Mark nearest → the group kills the
pack → Move to the next room → Mark nearest → ... Use Hold when you want the
group to wait, and Squad when a bot should stand somewhere on its own (a
healer behind a corner, say).

## 5. Console commands

All three are module commands at administrator level, for the worldserver
console or WoWMin's SOAP console (no dot). In game, a GM (level 3) can type them in
normal chat with a dot: `.wowmin botcmd ...`. Whispering them to a bot does
nothing: the bot reads them as chat **[Tested]**.

```
wowmin botcmd <bot> <command>
wowmin botcmd <bot> moveto <x> <y> <z>
wowmin botmark <yourCharacter> <icon> <entry> <unitId>
wowmin botmark <yourCharacter> <icon> clear
```

- `<command>`: one from the allowlist (section 6).
- `moveto`: three plain numbers separated by **spaces** (not `;`). Use
  `.gps` in game to read a position.
- `<icon>`: star, circle, diamond, triangle, moon, square, cross or skull.
  Bots attack the **skull** by default.
- `<entry>` and `<unitId>` identify the creature as WoWMin's live stream
  does: the unit id is the creature's low GUID with a type tag in the top
  byte. In practice, mark mobs from WoWMin; by hand, `clear` is the useful
  form.

Replies:

| Reply | Meaning |
|---|---|
| `WBOTCMD\|ok\|<bot>\|<command>` | Sent to the bot (a `moveto` shows the ground height it used) |
| `WBOTCMD\|refused\|<reason>` | Not sent; reasons below |
| `WBOTCMD\|disabled` | `BotCommand.Enable` (or `WowMinTelemetry.Enable`) is 0 |
| `WBOTMARK\|ok\|<icon>\|<entry>\|<unitId>` | Icon set for your group |
| `WBOTMARK\|cleared\|<icon>` | Icon removed |
| `WBOTMARK\|refused\|<reason>` / `WBOTMARK\|disabled` | As above |

Refusal reasons: `not allowed` (not on the allowlist), `no such bot online`,
`not a bot` (a real player, or someone else's self-bot), `bot of another
player`, `needs the owner as master` (a master command or a move for a bot
that is not in your group, or you are offline), `the leader is its own
master` (a master command to your self-bot), `unknown strategy`, `bad point`,
`too far`, `no ground there`, `rate limit`; for marks also `not the owner`,
`not in a group`, `no such player online`, `bad creature`, `no such
creature`, `a player's pet`, `unknown icon`.

## 6. What is allowed (and what is not)

The **module** decides; WoWMin only mirrors the list to grey out buttons.

| Kind | Commands | Sent from |
|---|---|---|
| Self | `stay`, `grind`, `flee`, `reset botAI`, `release` | the bot itself |
| Strategy | `co +<name>`, `co -<name>`, `nc +<name>`, `nc -<name>`, one at a time; the name must be one the bot supports | the bot itself |
| Master | `follow`, `summon`, `attack`, `tank attack` | your character (the bot's master) |
| Move | `moveto <x> <y> <z>`, sent as `go x;y;z` | your character, or your self-bot itself |
| Mark | `wowmin botmark` | your character |

Rules **[Tested]**:

- **Bots only.** A real player's character is refused. So is a self-bot,
  except your own (the leader).
- **Your bots only** for master commands and moves: the bot's master must be
  your character, online, on `OwnerAccount`. Random bots take the self and
  strategy commands only; they have no master.
- **Text** may hold letters, digits, spaces, `+` and `-` (plus `.` in a
  move). No `;` `\` `|` or anything else, so one line cannot chain commands.
- **Moves:** at most `MoveMaxYards` (60 by default, up to 250) from the bot, on its map, and the
  point must be within 8 yards of the ground; the command uses the ground
  height.
- **Rate limit:** 1 command per bot per second, 30 per minute in all.
  WoWMin's group bar spaces a bot's second command a second apart.
- **Log:** each command goes to the worldserver log (logger `module`) and to
  WoWMin's `bot-commands.jsonl`.

Refused on purpose, for example: `cheat`, `maintenance`, `autogear`,
`destroy`, `drop`, `sendmail`, `mail`, `bank`, `trade`, `sell`, `teleport`,
`taxi`, `go` (typed; only the module builds it), `leave`, `logout`,
`talents`, `spells`, `invite`, `guild ...`, `debug`, `do <action>`. None of
these change anything but behaviour, so there is no balance effect.

## 7. How it works

**A whisper without a whisper.** A whisper reaches a bot as
`PlayerbotAI::HandleCommand(type, text, fromPlayer)`; most commands are
queued and run in the bot's own AI update **[Source]**. Playerbots' security
check passes when `from` is the bot itself or a GM, and refuses a null
sender, so a console command must pass a real `Player*` **[Source]**. The
module calls `HandleCommand(CHAT_MSG_WHISPER, prefix + command, from)` with
`from` = the bot for self commands and = the bot's master for master
commands. SOAP and console commands run on the world thread, where a real
whisper is handled too **[Tested]** (no trouble in our runs).

**Self-bots.** `.playerbots bot self` gives your own character a
`PlayerbotAI` whose master is itself **[Source]**. Two consequences:
`IsRealPlayer()` is **false** for a self-bot (it only checks "has no
PlayerbotAI"), so an "is the master a person" check must also accept
`IsSelfBot()`, or master commands to your group fail while you lead
**[Source + Tested]**. And WoWMin's telemetry counts bots by headless
sessions, so your self-bot still shows as a real player: that is how WoWMin
finds the leader **[Tested]**.

**Moves.** Playerbots' `go x;y;z` needs a master, walks with pathfinding
(`MoveNear` in line of sight, else `MoveTo`) and never teleports; the `;`
form has no distance cap of its own **[Source]**, so the module adds one.
A bot told **stay** saves a "stay" position (and a "return" one) and walks
back to it **[Source + Tested]**: sent somewhere with `go`, it walked back
to where it was told to stay. Playerbots' own `SeeSpellAction` moves the
stay position before it sends a staying bot to a point; the module does the
same for `moveto` **[Tested]**. A bot that follows walks to the point and
then comes back to its master **[Tested]**; that is why Squad mode sends
"stay" first.

**Marks.** A bot's raid-icon setting (`rti`) defaults to `skull`, and its DPS
and tank target choice takes the group's skull first **[Source]**. The module
sets the icon with `Group::SetTargetIcon`, as a player marking in game. With
the self-bot on, a skull on an idle mob made the leader and the bots attack
it **[Tested]**. We did not test marks with the self-bot off.

**Finding the mob.** WoWMin's live stream names creatures by a unit id (low
GUID plus a type tag) and sends entry, position, alive, hostile, scenery and
pet flags. A click picks the nearest drawn mob within 16 pixels; Mark
nearest picks the nearest live hostile, non-scenery, non-pet mob to the
leader. The module rebuilds the full GUID from entry + unit id and finds the
creature on your map.

**Where a click lands.** In a 3D view, the scene's own ray pick gives the
floor point. On the flat dungeon map, WoWMin inverts the map projection at
the leader's height and the module finds the real ground near it (within 8
yards). On a different floor of a dungeon the flat map can miss: the module
then answers `no ground there`.

**Replies over SOAP.** A command handler that returns `false` comes back over
SOAP as a Fault ("Failed to parse SOAP response" in WoWMin's console), so the
module's usage lines return `true` **[Tested]**.

## 8. Test checklist

What we ran on the live server (all passed). Use a quiet low-level spot, one
random bot (`<Random>`) and one bot of your own in your group (`<Mine>`,
added with `.playerbots bot add <name>`); `<You>` is your character.

**Module (console)**
1. `wowmin botcmd` → usage line, no SOAP error in WoWMin's console.
2. `wowmin botcmd <Random> stay` → `ok`; the bot stops. `grind`,
   `reset botAI` → `ok`.
3. `wowmin botcmd <Random> follow` → `refused|needs the owner as master`.
4. `wowmin botcmd <Random> cheat` → `refused|not allowed`; a line with `\`
   → `not allowed`; an offline name → `no such bot online`.
5. The same command to one bot twice within a second → the second
   `refused|rate limit`.
6. `wowmin botcmd <Mine> nc -food`, then whisper the bot `nc ?` in game →
   food is gone; `nc +food` brings it back.
7. `<Mine>`: `stay`, `follow`, `attack` (with a target), `summon`, and
   `release` after `.die` on it → all `ok` and they happen.
8. `.gps` in game (note X Y Z), walk 20 yards off,
   `wowmin botcmd <Mine> moveto X Y Z` → it walks there, then follows you
   back.
9. Whisper `<Mine>` `stay`, then the same `moveto` → it walks there and
   stays.
10. `moveto` 100 yards off → `too far`; `moveto 1 2` → `bad point`;
    `moveto` on `<Random>` → `needs the owner as master`.
11. `.playerbots bot self`, then `wowmin botcmd <You> stay` → `ok`;
    `wowmin botcmd <You> moveto X Y Z` → you walk there, `<Mine>` follows;
    `wowmin botcmd <You> follow` → `the leader is its own master`;
    `wowmin botcmd <Mine> follow` → `ok`. `.playerbots bot self` again to
    stop.
12. `wowmin botmark <You> skull clear` → `cleared`; another player's
    character → `not the owner`.

**WoWMin**
13. Live Map → Options → Bot commands off: no Commands box. On: the box
    shows for a selected bot.
14. Stay, Reset AI on `<Random>` → `ok`; no master row for it; `cheat` in the
    text box → red, Send grey.
15. World 3D on a bot: strategy chips appear; add one from "+" → `ok` and a
    yellow chip; click it → removed.
16. `<Mine>` selected: the master row shows; Follow, Summon work.
17. Group bar, Leader: Move + floor click → you walk, bots follow; Mark
    nearest → skull, the group attacks; Clear mark; Mark + click a mob;
    Hold.
18. Squad: Move → the bots walk there and stay; follow one bot in 3D → Move
    moves only it; Regroup → they follow, Leader again.
19. Self-bot off, Move in Leader mode → "not a self-bot: type .playerbots bot
    self in game".
20. Instance Watch in a dungeon: the group card shows; Move on the 3D floor;
    Mark nearest.
21. `bot-commands.jsonl` has a line per command, with source button, text
    or group.

Not run live yet (patch 0004, point box):

22. World 3D, no tool armed, click the ground: a box with x y z; Copy,
    moveto and go each say "Copied"; paste the moveto line into a bot's
    text box and Send -> the bot walks there. Esc closes the box.
23. Instance Watch (3D), click the floor: the same box; click a player:
    it closes and the player is selected.

## 9. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Every command answers `refused\|not allowed`, even `moveto` | The old worldserver is still running. Check the exe you started is the rebuilt one. |
| `WBOTCMD\|disabled` | `WowMinTelemetry.BotCommand.Enable = 0`, or `WowMinTelemetry.Enable = 0`. Config changes need a worldserver restart. |
| `moveto` with `;` → `bad point` | Use spaces: `moveto X Y Z`. |
| Master buttons never show | `WOWMIN_BOT_OWNER_ACCOUNT` missing or wrong in WoWMin's `.env`, or your character is on another continent than the Live Map shows. |
| `needs the owner as master` | The bot is not in your group, `OwnerAccount` is not your account id, or your character is offline. |
| Group bar says "not a self-bot" | Type `.playerbots bot self` in game. `SelfBot is restricted` means your account is not a GM and `SelfBotLevel` is 1. |
| A squad bot walks back after a move | It was following, not staying: click Squad (sends stay) first. |
| `no ground there` | The point is in the air, or on another floor of a flat dungeon map. Click in the 3D view, or nearer the leader's floor. |
| A strategy you set comes back or goes away | Playerbots changes some strategies on its own (for example the random bot manager on random bots). [Unverified how often.] |
| Chips say "No live strategies" | Chips need the live stream: follow the bot in World 3D, or watch its instance. |
| Whispering `.wowmin botcmd` to a bot does nothing | Type it in normal chat (GM), or use the console. |

## 10. Roll back

- **WoWMin:** untick Bot commands (instant), or check out your previous
  WoWMin commit and rebuild.
- **Module:** set `WowMinTelemetry.BotCommand.Enable = 0` and restart, or
  stop the worldserver and put the old `worldserver.exe` / `.pdb` back.

## 11. Known limits

- The leader is your character; one group, one owner account.
- Group bar actions show only while your character is in view: the followed
  unit in World 3D is you or one of your group's bots, or you are in the
  watched instance.
- Mark nearest needs the live stream (World 3D follow or Instance Watch) and
  looks 60 yards round you.
- The flat dungeon map has no heights: a click on another floor fails
  (section 7).
- Marks with the self-bot off, and how long a strategy change lasts on a
  random bot, are not tested.
- Tested on Windows with our revisions only (section 2).

## 12. License and credits

**The real work is upstream.** The web admin, its live map, Instance Watch,
the 3D scene viewer and the telemetry stream are
[WoWMin](https://github.com/Jellypowered/wowmin) (maintained by Jellypowered,
originally by scarecr0w12) and
[mod-wowmin-telemetry](https://github.com/Jellypowered/mod-wowmin-telemetry)
and their contributors. The bots, their commands, strategies, self-bots and
raid-icon targeting are [mod-playerbots](https://github.com/mod-playerbots/mod-playerbots)
and its contributors, on [AzerothCore](https://www.azerothcore.org/). These
patches only wire existing playerbot commands to WoWMin's buttons, with a
server-side allowlist; if they are useful, the credit belongs upstream.

WoWMin and mod-wowmin-telemetry are MIT; these patches change them and are
MIT as well (`LICENSE-CODE` at the top of this repo). This README is
CC BY 4.0, like the rest of these notes.

Designed, written and built with Claude Code (Anthropic's coding assistant)
and tested on a home server. No game files are included.
