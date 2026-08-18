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

**RESOLVED 2026-08-17 by live test — see "Sheet backend: measured behavior" below.**

**Superseded: the concern that there may be no Sheets write path.**

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

## Sheet backend: measured behavior (2026-08-17)

Tested by uploading a purpose-built three-tab fixture (column counts 3/10/7,
seeded with blank-cell, comma, numeric, date, and tab-identity probes) to a real
Google Sheet and reading it back through the Drive connector.

**Write is impossible on the plain Drive connector, confirmed at the schema
level** rather than inferred from docs: the `update_file` tool accepts only
`fileId`, `parentId`, and `title`. There is no content parameter. Files can be
created, not edited.

**Read is better than expected, and validates the v0.4.1 config spec:**

| Probe | Result |
|---|---|
| All three tabs reached | ✅ |
| Per-tab column structure | ✅ preserved, not flattened |
| Blank cell mid-row | ✅ holds position; the following column does not shift |
| Comma inside a cell | ✅ stays one value |
| Numbers and ISO dates | ✅ verbatim, no reformatting |
| Tab names | ❌ **not returned at all** |

The blank-cell result is the one that mattered: the dotted-path key/value scheme
depends on it, and it holds.

Two defects found and fixed in v0.4.2: identify tables by header row rather than
tab name, and switch multi-value separators from `|` to `;` because the returned
representation is pipe-delimited and escapes embedded pipes.

**Other read paths considered and rejected:** a CSV export returns cleaner text
(proper quoting, no escaping) but only ever the first tab, with no way to select
another. An `.xlsx` export does carry tab names, but returns base64 and so needs
a decode-and-parse step — available in Claude Code, unavailable on Desktop, which
is the surface that matters. Depending on the returned text's incidental
formatting is unwise regardless: Google documents that it changes over time.

**Consequence for positioning:** there are two tiers, and the zero-setup one is
real. Read-and-paste needs nothing installed and should be the documented default;
native read/write via a Sheets MCP server is for a household with someone willing
to stand up a cloud project once.

## Drive write path: measured and adopted in v0.4.3 (2026-08-18)

The v0.4.2 conclusion — that tier 1 had to be read-and-paste — was wrong, and
folder-level sharing is why. Tested live:

| Behavior | Result |
|---|---|
| File created inside a shared folder inherits the folder's sharing | ✅ |
| A file that already existed gains access when the folder is shared later | ✅ dynamic, so setup order doesn't matter |
| `update_file` moves **and** renames in one call (`parentId` + `title`) | ✅ |

So the one write primitive available — metadata-only `update_file` — is exactly
what archiving needs, and `create_file` with a `parentId` handles replacement
without losing anyone's access. Full cycle: resolve → read → build → create
replacement → move old to `archive/` renamed with timestamp and contributor →
prune per document beyond `archive_keep`.

**Why this beats what it replaced:** native writes with zero setup, family
access preserved, and version history for free. The earlier objection
("create-and-replace breaks sharing") assumed file-level shares and does not
apply to a shared folder.

**Concurrency detection falls out of it.** Every replacement gets a new file id,
so comparing ids is an exact did-someone-else-write check — no clock skew, no
timestamp granularity, nothing to parse. Better than the timestamp approach that
prompted the question.

**Attribution must live in content.** Drive's `modifiedTime` records when but
never who, because every write goes through the single account hosting the store.
Hence `meta.last_updated_by` in config, the existing `by` column in log, and the
contributor stamped into archived filenames.

**Config gained `email` on adult and agent members.** Sharing needs an address; a
name can't be granted access. A member configured without one is silently locked
out, which is the failure the requirement exists to prevent.

**`archive_keep` defaults to 20, and setup must not interview for it.** Sized for
the log at roughly five writes a week — about a month there, years for config.
Pruning counts per document, or the fast-churning log would evict config history.

## 8. Google connectors may share one account grant

Authorizing Drive on a second Google account appeared to move the **Calendar**
connector to that account as well — the calendar list afterward showed only the
new account's calendars. Not proven (no before-snapshot was taken), but worth
knowing before assuming per-product account independence.

Matters here because the household store and the family-facing calendar were
expected to be able to live on different accounts. If one grant covers all Google
products, that assumption is wrong and the calendar integration has to target
whichever account the store uses.

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
