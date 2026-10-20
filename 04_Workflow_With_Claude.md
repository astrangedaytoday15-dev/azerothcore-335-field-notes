# Running a private-server project with Claude Code

**Draft 1.** How we run a long AzerothCore project with an AI coding
assistant (Claude Code): one owner who decides, one assistant that reads,
writes and builds. Nothing here is AzerothCore-specific except the examples.
It is what worked for us over ~100 sessions; take what fits.

## Contents
1. [The problem this solves](#1-the-problem-this-solves)
2. [Docs as memory](#2-docs-as-memory)
3. [The project instructions file](#3-the-project-instructions-file)
4. [Confidence tags](#4-confidence-tags)
5. [Safe database work](#5-safe-database-work)
6. [Builds: generate, preview, approve once](#6-builds-generate-preview-approve-once)
7. [C++ by certainty](#7-c-by-certainty)
8. [Spikes before features](#8-spikes-before-features)
9. [Test lists the owner can run](#9-test-lists-the-owner-can-run)
10. [Research waves and source briefs](#10-research-waves-and-source-briefs)
11. [Session rhythm](#11-session-rhythm)
12. [What we would do again](#12-what-we-would-do-again)

---

## 1. The problem this solves

Every session starts with no memory. A long project dies from three things:
decisions get re-argued, facts get re-guessed, and a wrong fact gets written
into a doc and then trusted. Everything below is a guard against one of
those.

## 2. Docs as memory

A small set of plain-text files, read in a fixed order, carries the project.
The local folder is the master copy; a Git repo is the cloud copy.

| Doc | Holds | Rule |
|---|---|---|
| **Master reference** | Server facts; section 0 = SETTLED / PARKED / DEFERRED; section 0.1 = the **decision log** | Read 0 and 0.1 first. Nothing there is re-raised without new evidence, and the evidence is named |
| **Decision log** (in the master reference) | One numbered line per owner decision (`D57`, `D58` ...): what, why, date | Every other doc refers to a decision by number only, never restates it |
| **Timeline** | One entry per session; a CURRENT STATE block | The session number = newest entry + 1 |
| **Project directory** | File list, a quick-facts card, **solved lookups**, the next-session block | After opening a file to settle a question, add a one-line "Solved" note under that file, so nobody opens it again for the same question |
| **Working context** | The owner's style and the technical traps | Updated the same session a trap is found; read every session |
| **Sub-project docs** | One per workstream: GOAL / STATUS / DECISIONS (numbers only) / FACTS / RUNBOOK / GOTCHAS / OPEN ITEMS | Same template everywhere |

Two habits make it work:
- **Same-session upkeep.** A closed item moves into section 0 / 0.1 in the
  session it closes. End of session: spec updates, Timeline entry, master
  reference edits, directory (solved notes, file list, next-session block).
  Back up each doc before editing it.
- **Say what a decision means.** When a decision is filed, write in plain
  words what it changes, not just "D176 filed".

## 3. The project instructions file

Claude Code reads a `CLAUDE.md` at the project root every session. Ours holds
only rules, not facts (facts live in the docs). The sections that earned
their place:

- **Source of truth:** which docs win over the instructions file and over
  memory, and in which order to read them.
- **Database access:** client path, host, and that credentials live in a
  stored login (`mysql_config_editor` login paths), never on the command
  line. A **read-only** login for exploring; the owner login only for
  approved writes.
- **Reads you may do on your own:** SELECT, SHOW CREATE TABLE, SHOW COLUMNS,
  EXPLAIN. Run SHOW CREATE TABLE before writing SQL against a table not
  already documented. Read big CSVs filtered, never whole.
- **Writes the owner approves** (section 5).
- **Server state:** assume the worldserver is running; never start, stop or
  kill it; character DB writes wait until the owner stops it.
- **Accuracy:** say "I'm not sure" rather than guess; mark unverified items;
  read module source, do not recall it; verify GM commands in the core's
  command source before giving them; always give spell ids, not just names.
- **Precision first:** if names, folders, counts or results do not match the
  plan, stop and report before running, editing or writing anything. Never
  write a mistake into the docs.
- **Style:** short replies; one question round per batch; put every table
  and question in the **final** message of a turn (the owner may only see
  that one).

## 4. Confidence tags

Every fact in a reference doc carries a tag, and nothing untagged is
trusted:

| Tag | Meaning |
|---|---|
| Verified (session) | In-game test or a source read, session named |
| Verified (file:line @commit) | Read in source at a pinned commit |
| Data-checked | Reproduced from real DB rows |
| Unsure | Recalled, not read. A hypothesis. The doc lists which file would settle it |

An "unsure register" lists every open hypothesis with what would settle it;
when a build or a source read answers one, the line is updated with the
evidence. Pasted claims from other AI chats or old forum posts count as
unsure until checked: some were wrong.

## 5. Safe database work

The rule set that has never cost us data:

1. **One write at a time.**
2. Before it: show the statement, the **expected row count**, and the SELECT
   that previews the affected rows. Run the preview first. If its count
   differs from the expectation, **stop and report**.
3. After it: compare the affected-row count with the expectation; stop on a
   mismatch.
4. Every generated write ships with a **rollback file**, generated with it.
5. **Character data only with the worldserver stopped.** The owner stops
   it; the assistant never starts, stops or kills a server.
6. Config changes need a restart; the assistant says so, the owner restarts.
7. SQL handed to the owner for a GUI client has no blank lines (DBeaver
   treats a blank line as the end of a statement).
8. Backups at milestones (project start, before a migration, a sign-off),
   not between every batch, in one location. A backup is unproven until it
   has been loaded once.

## 6. Builds: generate, preview, approve once

Content is never hand-written into the database. A **spec file** (JSON) is
written from the design doc, a **generator** turns it into SQL + client
files + rollback + a report, and a **validator** checks it before anything is
shown. Never hand-edit generated files; change the spec and regenerate.

A **build** is the full set of generated writes shown together in one preview
table (statement, expected count, preview result). One owner "yes" covers
that table. Each write still runs alone, preview first, and a mismatch still
stops the build. Anything added after the table was shown is a new build.

Every build also says whether players need a new client file
(**Client: YES (file)** or **no change**). Client changes collect in a
pending list and ship together as the next client version, so players update
rarely.

## 7. C++ by certainty

Core and module C++ is allowed item by item. For each one the assistant
states **CERTAIN** / **SPIKE FIRST** / **NO**, naming the source file, the
function and the existing pattern it follows; the owner approves each item.
"Certain" means read in source, not remembered. Local core edits live in one
patch per repository, exported after every change, with the pinned upstream
commits listed beside them, so the live server can be rebuilt from public
source plus our patches.

## 8. Spikes before features

Before a big feature, the smallest test that can fail: the 4th talent tab
was first proven with one throwaway tab using existing spells only, with a
written PASS / FAIL list and a named fallback if it failed. Only then was
tooling written, and only then the real trees. The same for each new
mechanic: one spell id, one test, then reuse.

## 9. Test lists the owner can run

After every install the owner gets **one** numbered list covering every
untested build:
- plain headings per area, one check per line, each line saying what to do
  and what should happen;
- all `.additem` (or similar) commands first, as one copy-paste block;
- details only for things that need focus (first-use spells, new
  mechanics), not one line per ordinary item;
- quest tests use chain starters with no prerequisites, NPC and zone named;
- ends with "reply with the numbers that fail, or all pass".

Stat-only content of a known type: test a random sample of 2 per batch (seed
shown); any failure means checking the whole batch. Anything new gets a full
check.

## 10. Research waves and source briefs

Big read-only questions ("how does the loot system really work in our
build?") go to parallel research agents, each with a **brief**: one topic,
the exact pinned commits to read, our patches applied, output format,
citations required on every fact (`file:line @commit`), unknowns marked
unsure, never filled from memory. A common-rules file is read first by every
brief. Results are merged into one reference doc locally, after spot-checking
citations and checking table / column names against the live DB.

Practical limit: about six parallel agents; more hit session limits.

## 11. Session rhythm

1. Read the instructions file, then section 0 / 0.1, CURRENT STATE, working
   context, and only the doc sections the goal needs.
2. One goal per session, stated in one sentence.
3. Plan → questions in one message → dry-run the whole batch → one preview →
   one yes → install → test list.
4. After the owner's notes: reprint only a compiled old → new change list,
   not the whole table.
5. A consistency pass over code, config and docs before offering a commit:
   every doubt is checked and fixed or reported, never left unsaid.
6. Docs upkeep, then stop when the goal is done.

## 12. What we would do again

- The decision log. It ended re-arguing.
- Solved lookups. They ended re-reading the same files.
- Preview count before every write. It caught real mistakes before they
  landed.
- Confidence tags, with an "unsure register". They made it safe to use
  research written by other sessions.
- Pinned commits + exported patches. The server can be rebuilt anywhere.
- Spikes with a written fallback.
