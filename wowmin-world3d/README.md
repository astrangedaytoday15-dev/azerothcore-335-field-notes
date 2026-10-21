# World 3D for WoWMin: the open world in 3D, round one bot

**Draft 1.** Patches for [WoWMin](https://github.com/Jellypowered/wowmin)
(a web admin for AzerothCore) and its
[mod-wowmin-telemetry](https://github.com/Jellypowered/mod-wowmin-telemetry)
module. WoWMin's Instance Watch already shows dungeons and battlegrounds in
3D. These patches add the same for the **open world**: pick a player or bot
on the Live Map, press **3D**, and the view follows it through the
continent, with live character models, NPCs, mobs and combat around it.

Tags: **[Tested]** passed a live test on our server; **[Built]** built and
checked against a stand-in data stream, not on the live server;
**[Source]** read in source.

## Contents
1. [What you get](#1-what-you-get)
2. [Install](#2-install)
3. [Use](#3-use)
4. [How it works](#4-how-it-works)
5. [Test checklist](#5-test-checklist)
6. [Known limits](#6-known-limits)
7. [License and credits](#7-license-and-credits)

---

## 1. What you get

| Feature | Status |
|---|---|
| Live Map: **3D** on the selected player opens a 3D view over the map that follows it | **[Tested]** |
| The 3 x 3 map tiles round the unit (about 1,600 yards square), built from your own 3.3.5a client the first time they are asked for, then cached | **[Tested]** |
| Tiles load and unload as the unit moves; the last row left behind stays loaded for turning back | **[Tested]** |
| Player models in their gear, NPC and mob models, health bars, combat swings and spell effects (same model code as Instance Watch) | **[Tested]** |
| Radius picker: models and NPCs within 30 / 50 / 100 / 250 yards of the unit (default 30; 100 is a player's own view distance) | **[Tested]** |
| Nameplates, the selected bot's waypoint as a dashed line, trails of where units walked while the view was open | **[Tested]** |
| Q / E rotate, scroll zoom (about 16 to 960 yards across), Esc closes | **[Tested]** |
| A unit that takes a boat or portal is followed to its continent; in a dungeon the status line points to Instance Watch | **[Built]** |
| Server cost only while someone watches: the module looks at nothing outside the radius | **[Source]** our patch; no lag seen in our live test |

## 2. Install

Tested on: WoWMin `27ed100`, mod-wowmin-telemetry `804b896`, AzerothCore
`mod-playerbots/azerothcore-wotlk` Playerbot branch `f19a18799a35`,
mod-playerbots `037c01418b5d`, Windows, Node 24, MSVC.

**WoWMin** (4 commits, `wowmin/`):

```
git clone https://github.com/Jellypowered/wowmin
cd wowmin
git checkout 27ed100
git am <this folder>/wowmin/*.patch
npm ci
npm run build
```

WoWMin needs `WOWMIN_CLIENT_DATA` (your 3.3.5a `Data` folder) to build tiles
and models, and its console (SOAP) connection for the live stream. Tiles are
cached under `assets/world/v2/<map>/` (or `WOWMIN_WORLD_TILE_DIR`). To build
tiles ahead of time: `npm run extract:world -- --source <Data> --map 0 --around 32 32`.

**mod-wowmin-telemetry** (`mod-wowmin-telemetry/`):

```
cd <azerothcore>/modules/mod-wowmin-telemetry
git checkout 804b896
git apply <this folder>/mod-wowmin-telemetry/0001-continent-radius-stream.patch
```

`0002-older-playerbots-compat.patch` is only for older mod-playerbots
revisions like ours that have no `PlayerbotAI::GetCurrentEngine()`, and for
MSVC "narrowing conversion" errors (C2397). Apply it after 0001 if the module
does not build for you. Then rebuild the worldserver: only the module's one
source file changes, so the rebuild is short (2 minutes for us). Keep a copy
of the old `worldserver.exe` to roll back.

Without the module patch, World 3D still works with markers instead of models,
and the status line says the models are unavailable.

## 3. Use

1. Live Map: pick the continent, select a player or bot, press **3D**.
2. The status line at the bottom says how many tiles are ready (the first
   visit to an area builds them, a few seconds each) and whether the models
   are live.
3. Toolbar: rotate, zoom, the radius picker, Trails, and close.

The module command behind it, for console use:
`wowmin stream <mapId> 0 <playerName> <yards>` (a continent, a player who is
online, 10 to 250 yards; administrator level, like the module's other
commands). WoWMin renews it every few seconds while the view is open.

## 4. How it works

**Tiles.** A continent is a 64 x 64 grid of ADT tiles, each 533.33 yards
square. The tile file `<map>_<x>_<y>.adt` runs x with the server's Y axis
and y with its X axis, both counting down from 32 at the map centre
**[Tested]**. WoWMin's own instance scene exporter builds one tile at a time
(it is handed only that tile's ADT), in a child process, on the first
request; a tile took 4 to 10 seconds and 6 to 9 MB in Hillsbrad
**[Tested]**. Scenes keep server coordinates, so tiles line up with each
other and with the telemetry positions without conversion.

**Seams.** Every ADT a building or prop overlaps lists it, so neighbouring
tiles would draw it twice. Each placement goes to exactly one tile, chosen by
its uniqueId: the tile under its origin if that tile lists it, otherwise the
first listing tile round it. In a 3 x 3 Hillsbrad window, 171 of 6,718
placements were listed twice and every tile agreed on the owner
**[Tested]**.

**Viewer.** The scene viewer draws a set of tile scenes at once and frees a
tile's GPU memory when it leaves. The window moves once the unit is 40 yards
past its tile's edge, keeps the last row it left, and asks for at most two
tiles at a time, because the web service holds each request open while it
builds and the browser has only a few connections for everything else.

**Live stream.** The module streams dungeons and battlegrounds at 10 Hz
already. The patch adds a continent stream round one player: each tick it
finds that player on the map and keeps only players, creatures and objects
within the radius (flat distance, checked before anything else), and only
combat events from inside it. It sends nothing while nobody is watching or
while the player is not on that continent. The datagrams are unchanged, with
instance 0, so WoWMin reads them with its existing parser. One continent
stream per continent: a second follower replaces the first.

## 5. Test checklist

What we ran on the live server after installing both patches (all passed):

1. Live Map, Show All, select a bot in the open world, **3D**: within a few
   seconds the status line ends with "models live . N NPCs (30 yd)".
2. Zoom in: the bot and nearby players are character models in their gear.
3. NPCs and mobs within 30 yards are models with nameplates (red bar
   hostile, yellow neutral).
4. The bot fights a mob: both animate attacks, the mob's health falls.
5. Radius picker to 100: more NPCs appear within a few seconds.
6. Back to 30: the far ones go.
7. Instance Watch in a dungeon or battleground still shows models and
   combat as before.
8. The server runs normally while World 3D is open: no lag in game, no
   errors in the worldserver console.

## 6. Known limits

- One followed unit at a time, one stream per continent.
- First visit to an area: tiles fill in over a minute or so (about 70 MB for
  3 x 3); after that they come from the cache.
- A tile that fails to build stays blank until the unit moves far enough
  that it leaves the window and comes back.
- No fade for underground places (Undercity): the floors above are drawn.
- The window is fixed at 3 x 3 tiles in the viewer. `WOWMIN_WORLD3D_TILES`
  is read by the web service but not passed to the viewer yet.
- Tested on Windows with our revisions only (section 2).

## 7. License and credits

**The real work is WoWMin's.** The web admin, the instance scene exporter,
the 3D viewer, the model builder and the telemetry stream all come from
[WoWMin](https://github.com/Jellypowered/wowmin) (maintained by Jellypowered,
originally by scarecr0w12) and
[mod-wowmin-telemetry](https://github.com/Jellypowered/mod-wowmin-telemetry)
and their contributors. These patches only point that machinery at the open
world; if they are useful, the credit belongs upstream.

Both projects are MIT; these patches change them and are MIT as well
(`LICENSE-CODE` at the top of this repo). This README is CC BY 4.0, like the
rest of these notes.

Written and built with Claude Code (Anthropic's coding assistant), tested on
a home server. No game files are included: the tiles and models are built on
your machine from your own client.
