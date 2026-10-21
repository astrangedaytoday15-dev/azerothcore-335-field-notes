# AzerothCore 3.3.5a: a 4th talent tree and custom spells (field notes)

**Draft 1.** Notes from modding a home WotLK 3.3.5a server (AzerothCore),
played by a few friends on a local network. The core of it is a native fourth talent tab for every class,
built and tested in-game (ten trees). The rest is what we learned on the way:
how custom spells, procs and scripts behave, a catalog of traps, and how we
run the project with Claude Code.

## Contents

| File | What it covers |
|---|---|
| [01_Fourth_Talent_Tab.md](01_Fourth_Talent_Tab.md) | Adding a native 4th talent tab: core edit, DBC rows, client files, spellbook, bots, the build loop |
| [02_Custom_Spells.md](02_Custom_Spells.md) | How spells are built and how they interact: the layers, procs, stacking, spell mods, scripts, math |
| [03_Gotchas.md](03_Gotchas.md) | Symptom / cause / fix catalog: traps that cost us time |
| [04_Workflow_With_Claude.md](04_Workflow_With_Claude.md) | How the project runs with an AI coding assistant: docs as memory, safe DB writes, tests |
| [playerbots-guild-bank-exchange/](playerbots-guild-bank-exchange/README.md) | Extra: a standalone mod-playerbots patch that lets guild bots use the guild bank (GPL-2.0) |
| [wowmin-world3d/](wowmin-world3d/README.md) | Extra: patches for WoWMin and mod-wowmin-telemetry that show the open world in 3D round one bot, with live models and NPCs (MIT) |
| [wowmin-botcontrol/](wowmin-botcontrol/README.md) | Extra: command playerbots from WoWMin (allowlisted, server-checked) and steer your dungeon group with clicks: lead as a self-bot, move, mark, pull (MIT) |

## How much to trust a line

Every technical claim carries one tag. Anything we only remembered or did not
read is left out, or listed under "Open questions" as unverified.

| Tag | Meaning |
|---|---|
| **[Tested]** | Built and passed an in-game test on our server. |
| **[Source]** | Read in the AzerothCore / module source at the commit below. Not an in-game test. |
| **[Data]** | Cross-checked against real rows in a stock 3.3.5a world database. |

Source references name the file and function, so you can find them on other
revisions. Line numbers drift; we leave them out.

## Baseline (what this was tested on)

| Part | Version |
|---|---|
| Client | WotLK 3.3.5a, build 12340, enUS |
| Core | `mod-playerbots/azerothcore-wotlk`, branch `Playerbot`, commit `f19a18799a35` |
| mod-playerbots | `037c01418b5d` |
| mod-ale (Eluna) | `bd74eae623ca` |
| Database | MySQL 8.4 |

Our own changes sit on top: a small core patch, a playerbots patch, and our own
module with the talent scripts. Lines that depend on those changes say so
(**Our setup:**). On another revision, re-check any **[Source]** line before
you build on it.

## Scope and status

- First public draft. Wording and structure will change; facts tagged
  **[Tested]** will not, unless a newer test says otherwise.
- Not covered yet (later drafts): AzerothCore systems notes (loot, creature
  difficulty, playerbots behaviour), client patching in depth, the item
  generator kit.
- Game terms mean game things: "bots" are mod-playerbots AI players, "patch"
  is a source diff or a client data archive.

## License and credits

**Text:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (see
`LICENSE`). Use it, copy it, change it, build on it; just credit
**astrangedaytoday15-dev** with a link to this repo.

**Code samples** written for this project (the addon in guide 01): MIT (see
`LICENSE-CODE`).

**The playerbots patch** in `playerbots-guild-bank-exchange/` modifies
mod-playerbots, so it is GPL-2.0-or-later (its own `COPYING`).

**The WoWMin patches** in `wowmin-world3d/` and `wowmin-botcontrol/` modify
WoWMin and mod-wowmin-telemetry (both MIT, by their authors); the patches are
MIT too.

**Third-party projects:** AzerothCore (GPL-2.0-or-later, per its file
headers), mod-playerbots (GPL-2.0) and mod-ale (GPL-3.0) belong to their
authors. These notes cite their files and functions; they copy no source
beyond identifiers and one-line changes.

**How it was written:** researched, built and written with Claude Code
(Anthropic's coding assistant). The tags above say how each fact was
checked; nothing untagged is presented as fact.

## Disclaimer

Not affiliated with or endorsed by Blizzard Entertainment. World of
Warcraft and Wrath of the Lich King are trademarks of Blizzard
Entertainment. No game files are distributed here: no DBCs, MPQs,
textures or client data. This is a hobby project for a home server shared
with a couple of friends; no public server, no money involved. For
educational use.
