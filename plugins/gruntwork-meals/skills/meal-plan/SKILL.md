---
name: meal-plan
description: Plan, shop for, and execute a household meal plan on a recurring weekly cadence, with persistent memory of what the family has eaten, what they loved, and what actually happened versus what was planned. Use this skill whenever the user wants to plan meals or menus for the week, build a grocery or shopping list from a meal plan, decide what to cook tonight, log that dinner went differently than planned, review meal history, or set up meal planning for their household for the first time. Trigger even on casual phrasings like "what's for dinner", "let's plan the week's meals", "make the shopping list", or "we ordered pizza instead".
---

# Meal Plan

A skill for running a household meal plan as a repeating loop: plan the covered days, acquire the food, cook the meals, and record what actually happened so next week's plan is smarter than this week's.

The skill is generic. Nothing about a specific family, schedule, or store is hardcoded. All of that lives in the household's own state store, created at first run. The skill has four modes:

- **setup**: first-run onboarding. Interview the user, create the state store, write the config.
- **plan**: guided interview that converges on a menu for the covered days.
- **shop**: turn the current plan into acquisition-ready lists.
- **cook**: day-of execution, including audibles (swaps, substitutions, bailouts), all logged as reality.

Infer the mode from what the user says. "Let's plan next week" is plan. "Build the shopping list" is shop. "What's for dinner" or "we're ordering out instead" is cook. If no state store exists yet, run setup first regardless of what was asked, then continue into the requested mode.

## State model

All state lives in three documents in the household's configured storage backend. Never hold changes only in conversation, because the whole point of this skill is that state survives between sessions.

**Read only the documents a mode needs, not the whole store.** `setup` and `plan` need all three; `shop` needs config and the current week of the log; `cook` needs the log and, for prep notes, meals. The log grows every week, so reading it wholesale on every interaction gets steadily more wasteful for no gain. Where only recent history matters, read the recent rows.

### 1. config (the household)

Stored as `config.yaml` on the `local` and `github` backends, and as the `config` Sheet on the `sheet` backend, where there is no YAML involved at all. The structure below is the shape; see **Config on the sheet backend** for how it flattens.

```yaml
household:
  name: ""                # display name, e.g. "The Smith household"
  members:
    - name: ""
      role: adult | kid | agent   # agent = a household AI assistant that runs this skill with its own credentials
      email: ""           # REQUIRED for adult and agent, omitted for kid. This is how the
                          # store gets shared with them; a name cannot be granted access.
      notes: ""           # dietary needs, strong dislikes, portion notes (n/a for agents)
  goals:
    shared: ""            # e.g. "generally healthy, high protein"
    adults: ""            # optional divergent goals
    kids: ""              # optional divergent goals

cadence:
  planning_day: ""        # e.g. Saturday
  shopping_day: ""        # e.g. Sunday
  covered_days: []        # e.g. [Monday, Tuesday, Wednesday]
  meals_covered: []       # e.g. [dinner]; may grow to lunches, snacks

planning:
  consistency_dial: 1-5   # 1 = maximize variety, 5 = run the proven rotation
  repeat_window_days: 21  # advisory freshness signal; the dial decides how much it matters

acquisition:
  sources:
    - name: ""            # e.g. "grocery delivery service"
      type: delivery | in-person
      default_for: ""     # e.g. "staples and bulk"
    - name: ""            # e.g. "local market"
      type: in-person
      default_for: ""     # e.g. "produce and meat"

calendar:
  enabled: true | false
  calendar_name: ""       # which calendar to write to
  write_meal_events: true | false     # one event per covered dinner
  write_shopping_block: true | false  # block on shopping day

storage:
  backend: sheet | github | local
  location: ""            # sheet: the FOLDER id holding the documents (not a document id)
                          # github: repo (owner/name).  local: directory path
  archive_keep: 20        # sheet backend: how many superseded copies of EACH document to
                          # retain in archive/ before trashing the oldest. Default 20; do
                          # not interview for it. Sized for the log, which churns fastest
                          # at roughly five writes a week, so 20 is about a month of
                          # history there and years for config. Raising it costs almost
                          # nothing; lowering it quietly discards the backup the archive
                          # exists to provide.

meta:
  last_updated: ""        # ISO timestamp of the last config write
  last_updated_by: ""     # which contributor drove that write; see "Who last touched this"
```

#### Config on the sheet backend

A spreadsheet tab is flat and the config is nested, so the nesting is carried in the key. **Dotted-path keys are the single representation.** Do not maintain a second, prettier copy of the config anywhere in the sheet: the sheet is hand-editable by design, so a derived table is a table someone will edit, and their edit would vanish silently on the next config write.

The `config` Sheet has three columns:

| key | value | note |
|---|---|---|
| `household.name` | The Smith household | display name |
| `household.members.1.name` | Alex | |
| `household.members.1.role` | adult | adult, kid, or agent |
| `household.members.1.email` | alex@example.com | how the store is shared with them |
| `household.members.2.name` | Sam | |
| `household.members.2.role` | kid | |
| `household.members.2.notes` | no mushrooms | dietary needs, dislikes, portions |
| `cadence.planning_day` | Saturday | when the week gets planned |
| `cadence.covered_days` | Monday;Tuesday;Wednesday | semicolon-separated list |
| `planning.consistency_dial` | 4 | 1 = maximize variety, 5 = run the proven rotation |

The `note` column carries the human gloss that the comments above hold, so a family member who opens it can tell what a key means without asking. Notes are documentation only — never read a note as data, and never let a note's absence change behavior.

Rules that keep two contributors from producing configs the other cannot read:

- **Lists are 1-based.** `members.1` is the first member. Someone reading it counts from one.
- **Deleting a list item reindexes, and rewrites the whole document.** Removing member 2 of 3 makes the old member 3 the new member 2. Never leave an index gap — a gap leaves the next reader guessing whether that item is absent or merely blank. The config is small and config writes are rare (setup and reconfigure only), so always rewrite the document whole rather than patching rows; that also makes the write atomic.
- **A blank value means unset.** If a field ever needs to mean "deliberately empty," write an explicit word saying so, never an empty cell.
- **Do not escape anything in a sheet cell.** Cells hold commas, quotes, and newlines natively. Escaping rules apply only to the CSV documents on the `local` and `github` backends, where a value containing a comma must be quoted.

When the user wants to see the config, render a readable view **in the conversation**. A generated view costs nothing and cannot be edited into a phantom state that disagrees with the stored rows.

#### What survives a sheet read

Verified against a live sheet, so trust these rather than defending against them: blank cells hold their position (a blank `value` does not shift the `note` after it), commas inside a cell stay within that cell, and numbers and ISO dates come back exactly as written rather than reformatted. The dotted-path scheme above is safe on this backend.

**Booleans are the exception.** Sheets coerces them to its own boolean type, so a written `true` reads back as `TRUE`. Compare booleans case-insensitively, and never match on the exact string `true`. Everything else round-trips as written.

Two cautions. Google states the returned text form may change over time, so never depend on its incidental formatting — the phantom leading row, the alignment markers, or the backslash-escaping of `_` and `|`. Depend only on the header row and the cell values.

And **a read returns no tab names at all.** With one document per table this does not arise, because each document is identified by its own filename. It matters only for a legacy store that kept all three tables as tabs in a single Sheet: there, identify each table by its header row (`key,value,note` / `name,tags,…` / `week_of,day,…`), never by position, since any contributor can reorder tabs invisibly.

### 2. meals (the meal library)

One row per distinct meal the household has ever made or seriously considered. This is what powers both halves of the core requirement: avoid accidental repetition, and deliberately repeat the winners.

```
name, tags, protein, prep, effort, rating, times_made, last_made, source, notes
```

- `tags`: comma-free labels separated by `;` (e.g. `weeknight;kid-favorite;one-pan`). Not `|` — on the sheet backend the reader returns tables in a pipe-delimited form, and a pipe inside a value is escaped rather than kept literal. The value survives, but any parser that splits on `|` shreds the row. The same rule applies to every multi-value field, including `cadence.covered_days`.
- `prep`: scratch | semi | prepared. Prepared and semi-prepared meals (a store rotisserie chicken, frozen dumplings the kids love) are first-class citizens of the library, not compromises. Many households run their best weeknights on them, and the skill should never imply that scratch cooking is the goal.
- `effort`: quick | moderate | project (weeknight plans should skew quick)
- `rating`: keep | mixed | retire. A meal rated retire is never proposed again unless the user asks. A meal rated keep with a `last_made` older than the repeat window is a prime candidate.
- `source`: recipe link, cookbook page, or "house standard"

### 3. log (the weekly log)

One row per covered day per week. This is the reality record, and it must record what happened, not what was intended. The plan is a hypothesis; the log is the data.

```
week_of, day, planned_meal, actual_meal, status, by, notes
```

- `status`: as-planned | swapped | substituted | bailed (ordered out or skipped)
- `by`: who made or last updated this entry. Self-declared and optional; see Multiple contributors below.
- When an audible happens, update the row rather than adding a new one. `planned_meal` keeps the original intent so plan-versus-actual is always visible.

## Storage backends

The skill logic is identical across backends; only reading and writing differ. Check `storage.backend` in config and use the matching approach. If the backend is unreachable, say so plainly and offer to proceed with a local copy that the user must sync later. Do not silently fork state.

- **sheet**: a shared Google Drive **folder** holding three separate Sheets — `config`, `meals`, `log` — plus an `archive/` subfolder. One document per Sheet, not three tabs in one document. See **The sheet backend** below for the full mechanics; it is the most involved backend and the one most households will choose.
- **github**: the three documents live at the root of a private repo. Read and write via the GitHub API or git, whichever is available in the current environment. Commit messages should name the mode and week (e.g. `plan: week of 2026-08-17`).
- **local**: a directory path, for Claude Code use inside a repo the user controls.

## The sheet backend

The store is a Drive **folder** shared with every contributor, holding three Sheets and an `archive/` subfolder:

```
Household Meal Plan/          <- storage.location is THIS folder's id
  config                      <- one tab, key/value/note
  meals                       <- one tab
  log                         <- one tab
  archive/                    <- superseded copies, newest first
    log 2026-08-18T00-07Z by-alex
    log 2026-08-10T18-22Z by-sam
```

Separate documents rather than one document with three tabs, for two reasons: each is addressable by filename, and each mode reads only what it needs. `cook` has no use for the config or a year of log history, and dragging the whole store through every "what's for dinner" gets worse every week.

### Writing: replace, don't edit

The Drive connector **cannot edit an existing file** — it creates files and changes metadata, nothing more. Writes therefore work by replacement:

1. **Resolve** the live document by searching the folder for its title.
2. **Read** it. This is also the re-read that the shared-store guarantee requires; do it immediately before writing, not at the start of the session.
3. **Build** the new full contents in memory.
4. **Create** the replacement in the same folder, same title, **uploading the whole document as CSV text and letting Drive convert it to a Sheet.** Create the file with a CSV content type and the full contents as text; do not create a file with the Google spreadsheet type and expect to fill it in afterwards. That produces an empty Sheet, and there is no way out of it — the connector cannot edit an existing file, so an empty document stays empty. This is the step the whole backend depends on. The replacement inherits the folder's sharing automatically, so every contributor keeps access without anything being re-shared.

   Because the upload transport is CSV, **quote any value containing a comma, a quote, or a newline on the way up**. This is not in tension with "do not escape anything in a sheet cell" below: that rule is about what ends up stored in the cell, and this one is about getting it there intact. `no mushrooms, no olives` is written to the CSV as `"no mushrooms, no olives"` and lands in the cell unquoted, as one value.
5. **Archive** the old one: move it into `archive/` and rename it `<title> <ISO timestamp> by-<contributor>`.
6. **Prune** archived copies of that document beyond `storage.archive_keep` (default 20), oldest first. Count per document, not across the archive as a whole — otherwise the fast-churning log would evict the config's only history.

Never trash a superseded document instead of archiving it. The archive is what makes a bad write survivable, and it costs nothing.

### Detecting a concurrent write

**Compare the file id, not a timestamp.** Replacement gives every write a new id, so if the id you resolved in step 1 differs from the one you read earlier in the session, another contributor wrote in between. This is exact — no clock skew, no granularity problem, nothing to parse.

On a conflict, say so and reconcile with the user rather than overwriting. Their version is in `archive/` either way, so nothing is lost; say that too, because it is the difference between an alarming message and a manageable one.

If a read ever finds **two** documents with the same title in the folder, a previous run died between steps 4 and 5. Self-heal: take the most recently created, archive the other, and mention it once.

### Who last touched this

Drive's own `modifiedTime` records when a file changed but **cannot record who** — every write goes through the one account the store lives on, so metadata attributes all of them to that account no matter which contributor drove the session. Attribution has to live in content.

- **config** carries `meta.last_updated` and `meta.last_updated_by` rows. Set both on every config write.
- **log** already attributes per row via the `by` column.
- **meals** needs no document-level stamp; its history is the archive, and archived filenames carry both timestamp and contributor.

### When the runtime cannot write to Drive

Preflight decides. If the runtime can read Drive but not create files, fall back to reading natively and handing the user the changed rows to paste, and say plainly that this is happening. Re-read immediately before producing the paste block so the state is current, and **never report the write as done until the user confirms they pasted it** — in this mode the write genuinely has not happened yet.

A connector with true cell-level write (Google's own Sheets MCP server, for instance) can update in place and skip the replace-and-archive cycle entirely. It requires a cloud project and OAuth client, so treat it as an upgrade a household may choose, never a prerequisite.

## Multiple contributors

The state store is shared infrastructure, and more than one contributor may read and write it: two adults each running the skill from their own Claude, and potentially a household AI agent running it with credentials of its own. The skill does not authenticate anyone. Identity lives at the storage layer: each contributor reaches the backend with their own credentials (their Google account for a sheet, their GitHub account or a machine user for a repo), and access control is whatever the backend grants those credentials. This keeps the skill simple and makes an agent contributor structurally identical to a human one.

What the skill does do:

- **Ask who's driving.** If config lists more than one contributor of role adult or agent, ask once at the start of a session and stamp the `by` column on any log rows written. Self-declared attribution is sufficient; the goal is context ("dad logged the bail"), not security.
- **Re-read before writing.** Another contributor may have written since the session started. Immediately before any write, re-read the affected document and apply changes against the current state rather than the state loaded earlier. If something changed underneath (tonight's row was already updated), say so and reconcile with the user instead of overwriting. On the sheet backend, compare file ids to detect this exactly — see **Detecting a concurrent write**.
- **Never delete another contributor's data silently.** Reconciliation can update rows, but removing history requires the user to ask.

## Mode: setup (first run)

Run this when no config exists, or when the user asks to reconfigure. Storage comes first, because everything else in setup produces answers that need somewhere to live.

**Step 1: provision storage.** Do not assume a backend exists or that the user has thought about this. Walk them through it:

1. **Preflight what this runtime can actually reach, before recommending anything.** Check which storage tooling exists in the current environment: spreadsheet read/write tooling for `sheet`, GitHub API or git access for `github`, filesystem access for `local`. Recommend only from what is present, and say plainly what you checked.

   This ordering matters. Recommending a backend and then discovering at the verification step that it cannot be written is a bad first impression, and the user will have already invested in the choice. Note in particular that a Google **Drive** connector is not the same as spreadsheet tooling — Drive grants file-level access, while this backend needs to read and write cells in named tabs. Presence of Drive alone does not make `sheet` workable.

   If the household clearly wants a backend this runtime cannot reach, say exactly that, name what they would need to enable or connect, and offer a working alternative rather than proceeding toward a wall. A household that wants the sheet but is running somewhere without spreadsheet tooling is better served by being told so now.

2. Recommend from what preflight found, matched to the household: a Google Sheet for households that want family members to open the data directly, which is most of them; a private GitHub repo for households comfortable with repos or planning agent contributors; local files for Claude Code use inside a directory the user controls.
3. Guide creation concretely.

   For a **sheet**: establish the folder that will hold the store and an `archive/` subfolder inside it. Record the **folder's** id as `storage.location` — not any document's id, since documents are replaced and their ids change while the folder's does not. Do not create the three documents yet; they are created once they have content, at step 4.

   Then collect contributors and arrange access. Ask for the email address of every adult and of any household agent, and explain that this is how they get access.

   **Attempt the share, and expect it to fail when the household owns the folder.** If the folder belongs to a household member rather than to the account this skill is running as, Drive will refuse — an editor cannot grant access to a folder it does not own. That is the permission model working correctly, and it is the arrangement to prefer, because access to a family's data should require the owner rather than a service account.

   When it fails, hand the user exact instructions instead: name the folder, the address, and the role (Editor). Then record the share as outstanding and carry it into step 4. **Never report setup complete with a member configured but not actually granted access** — that is precisely the failure the email requirement exists to prevent, and it is invisible from the inside.

   Sharing the folder rather than the documents is what makes replacement writes safe: a newly created document inherits folder access automatically, so nobody loses access when a document is replaced. Sharing propagates to documents that already exist as well as to ones created later, so the order does not matter.

   For a **repo**: a new private repository under an account the household controls, deliberately separate from anyone's work or development accounts, shared with each contributor. For **local**: a directory path.

4. Verify before proceeding, using a **throwaway test document** rather than the real ones. Write it, read it back, confirm out loud that reading and writing both work, then delete it. Put a value containing a comma in it and leave one cell deliberately blank, so the check covers the two things most likely to corrupt a row rather than merely proving the connector answers at all.

   Use a throwaway rather than creating the real documents empty. Creating them here and filling them in later means every document is written twice and archived once before the household has done anything, which is noise in a history whose whole purpose is recovering from real mistakes.

   If writing fails (a read-only connector, missing permissions), say exactly what failed and what the user can change, and offer the paste-back mode for sheets. Never continue setup on unverified storage, because an interview whose answers cannot be saved is wasted goodwill.

**Step 2: interview.** Cover the config fields that reflect a household choice, conversationally rather than as a form, explaining briefly why each answer matters (covered days set scope, acquisition sources shape the shopping lists). Reasonable defaults to offer: covered days of Monday through Wednesday for a first phase, dinner only, a repeat window of 21 days.

Do not interview for mechanical fields that have sound defaults and no household opinion attached — `storage.archive_keep` and the `meta` stamps among them. Set them and move on. Every question spent on plumbing is one the user has to answer before reaching the part they came for.

Ask where the household sits on the consistency dial, and explain the tradeoff in one sentence: high consistency runs the proven rotation and is often the right call for a new or limited-day plan; high variety works the repeat window hard. There is no correct answer, only a household preference.

**Step 3: seed the meal library.** Ask for five to ten meals the household already makes and likes, with quick ratings, and explicitly ask which prepared or semi-prepared items are house staples (store rotisserie chicken, frozen dumplings, and the like), since users tend to omit these unless prompted, treating them as "not real meals" when they are often the most reliable rows in the library. A cold-start library makes the first plan session dramatically better because the interview can anchor on proven meals instead of guessing.

**Let this step be deferred.** Recalling what a household actually eats takes real thought, and users often want to come back to it rather than improvise a list on the spot. That is a reasonable instinct, so honor it: finish setup without the library, do not create an empty `meals` document, and record it as outstanding. Say that the first plan session will be much better once it exists, and that at a high consistency dial it matters more still, because there the library *is* the rotation. Then stop asking.

**Step 4: write, then report honestly.** Write each document that has content to the verified backend — config always, log with just its column names since no week has happened yet, and meals only if the library was seeded. Creating a document with nothing in it buys nothing and costs an archived copy on the household's first day.

Then confirm where their data lives, who can access it (name them), and that the skill itself stores nothing.

**Say what is still outstanding, and who has to do it.** Setup rarely finishes completely on the first pass, and a report that only describes success leaves the user believing they are done when they are not. List each open item plainly — a share the household owner has to grant, a meal library not yet seeded — with the specific action and whose it is. An outstanding item named out loud gets handled; one quietly omitted becomes a puzzle weeks later, when the household has forgotten this conversation and only knows that something does not work.

**Adding a contributor later.** When reconfigure adds an adult or agent, share the store with their address as part of that change. A member added to config but never granted access is the failure this is written to prevent: everything looks configured, and they silently cannot read a thing.

## Mode: plan

The goal is a committed menu for each covered day, reached by guided interview, not generated unilaterally. The user (often the whole family) makes the calls; the skill's job is to make the calls easy by bringing the right candidates.

1. Read config, meals, and the last few weeks of the log.
2. Open with context, not questions: note anything relevant from last week's log (what got bailed on and why matters for this week's effort levels), and note which keepers are freshest versus longest unmade.
3. Propose a candidate slate shaped by the consistency dial. At 5, propose the proven rotation nearly unchanged, with at most one optional new or rotated-in slot. The rotation is defined as the most recent completed week's plan; if no prior week exists, use the highest-rated keepers with the most `times_made`. Households in an early phase or with limited covered days often want exactly this, and the skill should treat "same as last week" as a fully legitimate plan, not a failure of imagination. At 1, lean hard on the repeat window and fill most slots with meals not made recently or genuinely new ideas. Between, blend proportionally. Regardless of dial position: every proposal fits the household goals (for a high-protein household, name the protein), effort stays realistic for weeknights, and prepared or semi-prepared meals appear in the slate on equal footing with scratch cooking.
4. Interview to converge: which nights have time pressure, who is home, any ingredients on hand to use up. Let the user swap, veto, and add freely.
5. Commit the plan: write one log row per covered day with `planned_meal` filled and `actual_meal` empty. Add any genuinely new meals to the meal library with `times_made` 0.
6. If calendar integration is enabled, create one event per covered dinner (meal name as title; prep and defrost notes in the description) so the plan is visible to the household without opening the state store.

## Mode: shop

The goal is food acquired, not a list admired. Output must be directly actionable against the household's real sources.

1. Read config and the current week's plan. If no plan exists, offer to run plan first.
2. Build the full acquisition list across all planned meals. Explode scratch and semi meals into ingredients, consolidated (one entry for chicken thighs across two recipes, with total quantity). Prepared meals go on the list as the item itself (a rotisserie chicken is one line, not a recipe), plus any sides or additions the plan named.
3. Pantry check before finalizing: walk the list and ask what is already on hand, staples first. Rebuying staples is the most common waste in grocery delivery, so this step earns its keep.
4. Split the final list by acquisition source using the `default_for` hints in config, and confirm the split (the user may want to shift items between sources this week).
5. Deliver each list in a paste-ready format for its source: a flat item-per-line list for delivery services, a checklist for in-person trips. Offer to create per-item reminders for the in-person list if reminder tooling is available.
6. If calendar integration is enabled and `write_shopping_block` is true, place a block on the shopping day.

## Mode: cook

The goal is tonight executed and reality recorded. This mode must be fast; nobody wants a ceremony at 5pm on a Tuesday.

**Normal path**: read the plan, state tonight's meal, and surface anything time-sensitive (defrost now, marinate by 4pm, preheat notes). After dinner, or next time the skill runs, confirm it happened and mark the log row `as-planned`, incrementing `times_made` and updating `last_made` in the meal library.

**Audibles**: honor them without friction and record them without judgment. The log exists to present reality, and a log that gets argued with stops getting used.

- Swap two nights: `planned_meal` stays untouched on both rows (the original intent is history). Fill each row's `actual_meal` with what was actually eaten that night, and set `status` swapped on both.
- Substitute a different meal: fill `actual_meal` with what was actually made, `status` substituted.
- Bail (ordered out, skipped, cereal for dinner): fill `actual_meal` with what happened, `status` bailed, and capture the reason in notes if offered. Never lecture. A bail with a recorded reason (late meeting, sick kid) is useful planning data; a guilt trip is not. A bail never creates a meal library row on its own; if the household wants "pizza night" as a real planned option, that is a decision the user makes explicitly, not an inference the skill draws from a Tuesday.

Update calendar events to match audibles when calendar integration is enabled, so the calendar never shows a fiction.

**End of covered days**: when cook runs after the last covered day (or when the user asks "how did the week go"), present plan versus actual for the week in two or three sentences, note any meal that earned a rating change, and ask if any rating should be updated. This closes the loop that makes next week's plan mode smarter.

## Principles

- State is the product. Every mode ends with the state store true. If a write fails, tell the user exactly what did not persist.
- Shared means shared. Re-read before writing, attribute with `by`, and treat every contributor (human or agent) as a peer at the storage layer.
- Reality over intention. The log records what happened. Plan-versus-actual is presented neutrally, as data for better planning, never as a scorecard.
- The household decides. Plan mode proposes and interviews; it does not dictate. Respect vetoes immediately and without relitigating.
- Generic by construction. If an instruction only makes sense for one specific family, it belongs in their config or their meal library, not in this skill.
