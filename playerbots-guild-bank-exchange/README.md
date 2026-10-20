# Guild Bank Exchange for mod-playerbots

**Draft 1.** A patch for [mod-playerbots](https://github.com/mod-playerbots/mod-playerbots)
that lets guild bots use the guild bank like members do: they drop off loot,
swap for things they can wear, answer "need" requests, share a little gold,
and repair from the bank. Every option is **off by default**.

Tags: **[Tested]** passed an in-game test on our server; **[Log]** seen
working in the server logs on our server, no formal test yet; **[Built]**
compiled and running, not tested yet; **[Source]** read in source.

## Contents
1. [What bots do](#1-what-bots-do)
2. [Install](#2-install)
3. [Options](#3-options)
4. [How it works](#4-how-it-works)
5. [Test checklist](#5-test-checklist)
6. [Known limits](#6-known-limits)
7. [License](#7-license)

---

## 1. What bots do

| Feature | Option | Status |
|---|---|---|
| Now and then swap one random bag item with one guild bank slot. Empty slot = deposit, occupied slot = the bot takes that item in return | `Enabled` | **[Tested]** **[Log]** (every exchange matched the core's bank log) |
| `gb <item>` works without a nearby guild vault | `RemoteCommand` | **[Tested]** |
| Give away armor / weapons the bot cannot use first | `SmartGive` | **[Built]** (running since the first build) |
| One guild chat line per exchange | `ChatChance` | **[Log]** |
| Bots deposit a share of looted gold | `GoldSharePct` | **[Log]** |
| Per-member ledger (items in / out, gold in / out, repairs); whisper a guild bot `ledger` for the top 5 | `Ledger` | **[Tested]** |
| Aim swaps at bank items the bot would wear | `SmartTakeChance` | **[Built]** |
| Equip that upgrade right away (normal auto-equip choice, never forced) | `EquipAfterTake` | **[Built]** |
| `need <item link>` in guild chat: a bot carrying it deposits it after a delay | `NeedRequests` | **[Tested]** |
| A bot asks guild chat for an upgrade of its weakest piece and thanks whoever deposits one | `WantChance` | **[Built]** |
| Repair from the guild bank first (rank rights and daily limit apply) | `GuildRepair` | **[Built]** |
| Full tab: take one grey item out to sell later | `CleanupGreys` | **[Built]** |

## 2. Install

Made against mod-playerbots commit `037c01418b5d` on the
`mod-playerbots/azerothcore-wotlk` `Playerbot` branch (`f19a18799a35`).

```bash
git -C modules/mod-playerbots checkout 037c01418b5d01506917a3db9b44fd56ac5f965c
```

```bash
git -C modules/mod-playerbots apply guild-bank-exchange_mod-playerbots_037c0141.patch
```

Then re-run CMake (the patch adds new `.cpp` files) and rebuild. The SQL
update in `data/sql/playerbots/updates/` (ledger table + chat texts) is
applied by the normal DB updater at the next start.

- The patch applies cleanly to a clean checkout of that commit. **[Tested]**
- It does not need any other patch, but we have only built and run it
  together with our other playerbots changes. Building it alone has not been
  tried.
- **Newer mod-playerbots:** upstream master renamed every config key from
  `AiPlayerbot.*` to `Playerbots.*`. Rename the keys (and expect some
  context drift) before you use it on master.

Guild setup: buy at least one bank tab, and give the bots' rank deposit and
(if you want swaps) withdraw rights with a small daily cap. For `GuildRepair`,
the bots' rank also needs a daily repair allowance (guild rank money per
day above 0) and the bank needs gold; with the allowance at 0 the bot just
repairs with its own gold. A bot without
rights simply does nothing. On our server, guild bots were only online while
a real member of their guild was, so expect few exchanges in a quiet guild.

## 3. Options

All in `playerbots.conf` under **GUILD BANK EXCHANGE**. Defaults are the
values written in the patch.

| Key (`AiPlayerbot.GuildBank.`) | Default | Meaning |
|---|---|---|
| `Enabled` | 0 | Master switch |
| `MinInterval` / `MaxInterval` | 3600 / 7200 | Per-bot timer, seconds (random between) |
| `Chance` | 50 | % chance per timer hit |
| `Tab` | 0 | Bank tab used (0 = first; the guild must own it) |
| `MinQuality` / `MaxQuality` | 2 / 4 | Armor / weapon quality range (green .. epic) |
| `TradeGoods` / `ArmorWeapons` | 1 / 1 | Item kinds allowed |
| `PreferEmptyChance` | 50 | % to use the first free slot instead of a random one |
| `ChatChance` | 30 | % for a guild chat line |
| `RemoteCommand` | 0 | `gb <item>` without a vault |
| `AltBots` | 0 | Also non-random bots |
| `SmartGive` | 0 | Give unusable armor / weapons first |
| `SmartTakeChance` | 0 | % per swap to aim at a wearable upgrade |
| `EquipAfterTake` | 0 | Auto-equip after taking an upgrade |
| `Ledger` | 0 | Per-member totals + `ledger` whisper |
| `NeedRequests` | 0 | Answer `need <item link>` |
| `NeedDelayMin` / `NeedDelayMax` | 60 / 600 | Delay before answering, seconds |
| `NeedUsable` | 1 | 0 = bots keep items they would wear |
| `WantChance` | 0 | % per timer hit to ask for an upgrade |
| `GoldSharePct` | 0 | % of looted gold deposited |
| `GuildRepair` | 0 | Repair from the guild bank first |
| `CleanupGreys` | 0 | Free a slot on a full tab |

Values we run: `Enabled 1`, `SmartGive 1`, `SmartTakeChance 30`,
`EquipAfterTake 1`, `Ledger 1`, `NeedRequests 1`, `WantChance 3`,
`GoldSharePct 1`, `GuildRepair 1`, `CleanupGreys 1`.

## 4. How it works

- **Thread safety.** A real player's bank move is handled on the world
  thread (the bank swap opcode is marked thread-unsafe), while bot AI runs on
  map update threads. So bot AI never touches the bank directly. A
  `WorldScript` checks bots every 10 s on the world thread and swaps there;
  `gb` from bot AI queues an operation on the existing playerbots
  world-thread processor. **[Source]**
- **The core does the rest.** The swap uses the core guild bank swap, which
  checks rank rights, writes the bank log and the DB. It has no distance
  check, so no vault trip is needed. **[Source]**
- **Which item.** Bags only (never equipped, bank or keyring items). Trade
  goods, or armor / weapons in the quality range. Never soulbound, quest,
  conjured or limited-duration items. Whole stacks.
- **Which slot.** With `PreferEmptyChance` the core picks the first free slot;
  otherwise a random slot. If nothing moved, one retry into a free slot.
- **Reading the bank.** The core keeps bank tab contents private, so
  `SmartTakeChance` reads `guild_bank_item` from the characters DB, which the
  core keeps in step with every move. No core edit. **[Source]**
- **Ledger.** The core's `guild_bank_eventlog` keeps only 25 rows per tab, so
  the ledger is its own table (`playerbots_guild_ledger`), filled from bot
  exchanges and from any member's bank moves (`GuildScript::OnBankEvent`).
- **Texts.** Chat lines come from `ai_playerbot_texts` (the SQL update adds
  10, localizable), with a built-in default if a text is missing.
- **Logging.** `LOG_INFO` line per exchange: bot, item given, item taken,
  guild, tab, and whether it equipped the take.

## 5. Test checklist

1. `Enabled 0`: nothing happens.
2. Guild with no bank tab: no errors.
3. Short interval (e.g. 60 s), `PreferEmptyChance 100`: deposits show in the
   bank log under the bot's name.
4. Occupied slot + withdraw rights: the bot gets that item, the bank gets
   the bot's; the daily withdraw cap holds.
5. No withdraw rights: free slot used, or nothing happens. Never an error.
6. Soulbound / quest / conjured items never move.
7. A bot logs out or leaves the guild while queued: the entry is dropped.
8. `ChatChance 100`: one guild line per exchange.
9. `RemoteCommand 1`: `gb <item>` works away from a vault.
10. Each **[Built]** option above, one at a time.

On our server the basic exchange, `gb`, the ledger whisper and need
requests passed an in-game test, and the logs match the core's bank log one
to one. The options still tagged **[Built]** are under test.

## 6. Known limits

- Account-bound (BoA) items cannot be deposited; `gb` on one is refused.
  We think the core refuses any item that cannot be traded, but have not
  confirmed it in source.
- `Tab` is a fixed tab number: the core `Guild` class has no public tab
  count, so "last purchased tab" is not possible without a core edit. A
  missing tab is a silent no-op.
- In a 13-hour run with mostly empty bank slots, only 2 of 22 exchanges were
  real swaps; `SmartTakeChance` exists to fix that and is not tested yet.

## 7. License

This patch modifies mod-playerbots, which is GPL-2.0, so the patch is
released under the **GNU GPL v2 or later** (see `COPYING`, the same terms as
the files it changes; new files carry the mod-playerbots header). This
folder's README is CC BY 4.0 like the rest of the repository.
