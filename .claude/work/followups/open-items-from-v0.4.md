# Open items carried in from the v0.4 Desktop workshop

Source: `.claude/work/sessions/2026-08-15-meal-plan-v0.4-desktop-workshop.md`

Item 1 from that summary ("create the repo … and land the skill") is **done**,
but was corrected in the process — see below.

## 1. Repo split — RESOLVED, and the original framing was wrong

The v0.4 summary said to create one private repo under a household-controlled
account and land the skill in it. That fuses two different repos.

Landing the skill in the private household repo means it can't be shared with
another household without handing over that family's data — which defeats the
"generic by construction" premise the whole design rests on.

The correct shape, now implemented:

| | repo | visibility | owner | created by |
|---|---|---|---|---|
| **skill** | `gruntwork-household-skills` | public | GruntworkAI | this scaffold |
| **state** | e.g. `smith-household` | private | household-controlled account | the skill's `setup` mode, at first run |

The state repo should not live under `~/Code/` at all — it isn't a development
project. The skill itself already got this boundary right (setup step 4 tells
the user "the skill itself stores nothing"); only the summary's open-item
framing was off.

## 2. Sheet backend — partly addressed in v0.4.1, one hard question still open

Expectation is the Google Sheet, not GitHub: most households want data their
family can open in a tab without repo literacy.

**Addressed in v0.4.1:**

- Backend ordering now leads with `sheet` in the config enum and the backends
  reference. (Setup step 1 already led with the sheet — an earlier read of this
  saying otherwise was wrong.)
- Setup step 1 now **preflights** what the runtime can reach before
  recommending, so a household is never steered toward a backend that fails at
  the sentinel write.
- Config-on-sheet representation is now specified: dotted-path keys, 1-based
  lists, reindex-on-delete, whole-tab atomic rewrites, blank means unset, no
  escaping in cells. Previously the skill said only "config as key/value rows,"
  which permitted at least three incompatible flattenings — a real hazard given
  that multiple contributors share one store.

**🔴 Still open, and it's the important one: there may be no Sheets write path.**

Connector inventory taken in Claude Code on 2026-08-15:

| Connector | What it exposes |
|---|---|
| Google Calendar | Full CRUD, 11 tools |
| Google Drive | `authenticate` / `complete_authentication` only, and unauthenticated |
| Google Sheets | Does not exist |

Drive is not Sheets: Drive grants file-level access, while this backend needs
cell-level read and write in named tabs. So in Claude Code as configured, the
recommended backend cannot be written to at all.

**Unverified: whether Claude Desktop exposes richer Google tooling.** That is
where the users are, so it is the thing to test first. If Desktop also lacks
spreadsheet write, the sheet backend is aspirational and the honest options are
(a) lead with `local`/`github` until tooling exists, or (b) design the
paste-back mode into a real first-class path rather than a degraded fallback.

Settle this before recommending the skill to anyone outside the household.

## 3. Calendar integration untested

**Open.** Specified but never exercised: event-per-dinner with prep notes,
shopping block, events updated on audibles. The first real week will test it.

Prior art worth reading before debugging: `gruntwork-travel-skills` learned that
all-day events need a full midnight ISO timestamp (`2026-06-22T00:00:00`), not a
bare date. Same calendar tooling, same trap.

## 4. Deferred by design — not backlog

Each has a config-level home when wanted; none blocks phase one. Listed so they
don't get "discovered" later as gaps:

- real authentication
- per-person meal ratings
- lunches and snacks
- expansion beyond three covered days
- delivery-service ordering integration

## 5. Version drift between SKILL.md and plugin.json — RESOLVED

`SKILL.md` frontmatter carried `metadata.version: "0.4"`, an artifact of being
authored standalone in Desktop, while `plugin.json` carries `0.4.0`. Two sources
of truth with nothing validating them against each other.

Resolved at scaffold time by **deleting the frontmatter field**: `plugin.json`
owns versioning outright. Recorded in the CLAUDE.md release checklist (step 2)
and gotchas table. If a future Desktop-authored revision reintroduces the field,
strip it on the way in.

## 6. Skill triggering on Claude Desktop is unverified

**Open.** Whether Skills auto-trigger on the Desktop app is an open question
across the gruntwork plugins generally (same unknown tracked against the
lastmilefirst plugin). Meals is a Desktop-first product in a way the dev-tooling
plugins aren't, so this matters more here. Test explicitly on first Desktop
install rather than assuming parity with Claude Code.

## 7. Personal detail in shipped example values — RESOLVED

Two instances, both fixed at scaffold time before the first commit:

- `household.name` used a real family name as its illustrative value. Now
  `"The Smith household"`.
- `acquisition.sources` named a specific real grocery pairing. Now
  `"grocery delivery service"`.

**Standing rule:** example values in a shipped skill are generic placeholders,
never a real person's name and never the author's actual vendors — the config
comments are read by every installing household. Same principle as the
synthetic-fixtures lesson in travel-skills.

A full sweep (emails, handles, phone numbers, URLs, absolute paths, family
specifics, brands, stray artifacts) ran clean before the initial commit. Re-run
it on any revision authored outside this repo, since Desktop-workshopped drafts
naturally pick up the author's real details as examples.
