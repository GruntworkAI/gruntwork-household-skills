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

All state lives in three documents in the household's configured storage backend. Read them at the start of any mode; write them back at the end of any mode that changed something. Never hold changes only in conversation, because the whole point of this skill is that state survives between sessions.

### 1. config (the household)

Stored as `config.yaml` on the `local` and `github` backends, and as the `config` tab on the `sheet` backend, where there is no YAML involved at all. The structure below is the shape; see **Config on the sheet backend** for how it flattens.

```yaml
household:
  name: ""                # display name, e.g. "The Smith household"
  members:
    - name: ""
      role: adult | kid | agent   # agent = a household AI assistant that runs this skill with its own credentials
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
  location: ""            # repo (owner/name) or sheet URL or directory path
```

#### Config on the sheet backend

A spreadsheet tab is flat and the config is nested, so the nesting is carried in the key. **Dotted-path keys are the single representation.** Do not maintain a second, prettier copy of the config anywhere in the sheet: the sheet is hand-editable by design, so a derived table is a table someone will edit, and their edit would vanish silently on the next config write.

The `config` tab has three columns:

| key | value | note |
|---|---|---|
| `household.name` | The Smith household | display name |
| `household.members.1.name` | Alex | |
| `household.members.1.role` | adult | adult, kid, or agent |
| `household.members.2.name` | Sam | |
| `household.members.2.role` | kid | |
| `household.members.2.notes` | no mushrooms | dietary needs, dislikes, portions |
| `cadence.planning_day` | Saturday | when the week gets planned |
| `planning.consistency_dial` | 4 | 1 = maximize variety, 5 = run the proven rotation |

The `note` column carries the human gloss that the comments above hold, so a family member who opens the tab can tell what a key means without asking. Notes are documentation only — never read a note as data, and never let a note's absence change behavior.

Rules that keep two contributors from producing configs the other cannot read:

- **Lists are 1-based.** `members.1` is the first member. Someone reading the tab counts from one.
- **Deleting a list item reindexes, and rewrites the whole tab.** Removing member 2 of 3 makes the old member 3 the new member 2. Never leave an index gap — a gap leaves the next reader guessing whether that item is absent or merely blank. The config is small and config writes are rare (setup and reconfigure only), so always rewrite the tab whole rather than patching rows; that also makes the write atomic.
- **A blank value means unset.** If a field ever needs to mean "deliberately empty," it gets an explicit sentinel word, never an empty cell.
- **Do not escape anything in a sheet cell.** Cells hold commas, quotes, and newlines natively. Escaping rules apply only to the CSV documents on the `local` and `github` backends, where a value containing a comma must be quoted.

When the user wants to see the config, render a readable view **in the conversation**. A generated view costs nothing and cannot be edited into a phantom state that disagrees with the stored rows.

### 2. meals.csv (the meal library)

One row per distinct meal the household has ever made or seriously considered. This is what powers both halves of the core requirement: avoid accidental repetition, and deliberately repeat the winners.

```
name, tags, protein, prep, effort, rating, times_made, last_made, source, notes
```

- `tags`: comma-free labels separated by `|` (e.g. `weeknight|kid-favorite|one-pan`)
- `prep`: scratch | semi | prepared. Prepared and semi-prepared meals (a store rotisserie chicken, frozen dumplings the kids love) are first-class citizens of the library, not compromises. Many households run their best weeknights on them, and the skill should never imply that scratch cooking is the goal.
- `effort`: quick | moderate | project (weeknight plans should skew quick)
- `rating`: keep | mixed | retire. A meal rated retire is never proposed again unless the user asks. A meal rated keep with a `last_made` older than the repeat window is a prime candidate.
- `source`: recipe link, cookbook page, or "house standard"

### 3. log.csv (the weekly log)

One row per covered day per week. This is the reality record, and it must record what happened, not what was intended. The plan is a hypothesis; the log is the data.

```
week_of, day, planned_meal, actual_meal, status, by, notes
```

- `status`: as-planned | swapped | substituted | bailed (ordered out or skipped)
- `by`: who made or last updated this entry. Self-declared and optional; see Multiple contributors below.
- When an audible happens, update the row rather than adding a new one. `planned_meal` keeps the original intent so plan-versus-actual is always visible.

## Storage backends

The skill logic is identical across backends; only reading and writing differ. Check `storage.backend` in config and use the matching approach. If the backend is unreachable, say so plainly and offer to proceed with a local copy that the user must sync later. Do not silently fork state.

- **sheet**: a Google Sheet with three tabs named `config`, `meals`, `log`, mirroring the structures above (config as key/value rows). Use the available Google Drive or Sheets tooling. If the current environment can read but not write the sheet, fall back to producing the updated rows in the conversation for the user to paste, and say that this is a degraded mode.
- **github**: the three documents live at the root of a private repo. Read and write via the GitHub API or git, whichever is available in the current environment. Commit messages should name the mode and week (e.g. `plan: week of 2026-08-17`).
- **local**: a directory path, for Claude Code use inside a repo the user controls.

## Multiple contributors

The state store is shared infrastructure, and more than one contributor may read and write it: two adults each running the skill from their own Claude, and potentially a household AI agent running it with credentials of its own. The skill does not authenticate anyone. Identity lives at the storage layer: each contributor reaches the backend with their own credentials (their Google account for a sheet, their GitHub account or a machine user for a repo), and access control is whatever the backend grants those credentials. This keeps the skill simple and makes an agent contributor structurally identical to a human one.

What the skill does do:

- **Ask who's driving.** If config lists more than one contributor of role adult or agent, ask once at the start of a session and stamp the `by` column on any log rows written. Self-declared attribution is sufficient; the goal is context ("dad logged the bail"), not security.
- **Re-read before writing.** Another contributor may have written since the session started. Immediately before any write, re-read the affected document and apply changes against the current state rather than the state loaded earlier. If something changed underneath (tonight's row was already updated), say so and reconcile with the user instead of overwriting.
- **Never delete another contributor's data silently.** Reconciliation can update rows, but removing history requires the user to ask.

## Mode: setup (first run)

Run this when no config exists, or when the user asks to reconfigure. Storage comes first, because everything else in setup produces answers that need somewhere to live.

**Step 1: provision storage.** Do not assume a backend exists or that the user has thought about this. Walk them through it:

1. **Preflight what this runtime can actually reach, before recommending anything.** Check which storage tooling exists in the current environment: spreadsheet read/write tooling for `sheet`, GitHub API or git access for `github`, filesystem access for `local`. Recommend only from what is present, and say plainly what you checked.

   This ordering matters. Recommending a backend and then discovering at the verification step that it cannot be written is a bad first impression, and the user will have already invested in the choice. Note in particular that a Google **Drive** connector is not the same as spreadsheet tooling — Drive grants file-level access, while this backend needs to read and write cells in named tabs. Presence of Drive alone does not make `sheet` workable.

   If the household clearly wants a backend this runtime cannot reach, say exactly that, name what they would need to enable or connect, and offer a working alternative rather than proceeding toward a wall. A household that wants the sheet but is running somewhere without spreadsheet tooling is better served by being told so now.

2. Recommend from what preflight found, matched to the household: a Google Sheet for households that want family members to open the data directly, which is most of them; a private GitHub repo for households comfortable with repos or planning agent contributors; local files for Claude Code use inside a directory the user controls.
3. Guide creation concretely. For a sheet: create it (via available tooling, or walk the user through creating one) with three tabs named `config`, `meals`, `log`; head the `config` tab with `key`, `value`, `note` and the other two with their column names from the state model; then have the user share it with every contributor, with edit access rather than view access. For a repo: a new private repository under an account the household controls, deliberately separate from anyone's work or development accounts, shared with each contributor. For local: a directory path.
4. Verify before proceeding: write a sentinel value to the store, read it back, and confirm out loud that read and write both work. If write fails (a read-only connector, missing permissions), say exactly what failed and what the user can change, and offer the degraded paste-back mode for sheets. Never continue setup on unverified storage, because an interview whose answers cannot be saved is wasted goodwill.

**Step 2: interview.** Cover every config field conversationally rather than as a form, explaining briefly why each answer matters (covered days set scope, acquisition sources shape the shopping lists). Reasonable defaults to offer: covered days of Monday through Wednesday for a first phase, dinner only, a repeat window of 21 days.

Ask where the household sits on the consistency dial, and explain the tradeoff in one sentence: high consistency runs the proven rotation and is often the right call for a new or limited-day plan; high variety works the repeat window hard. There is no correct answer, only a household preference.

**Step 3: seed the meal library.** Ask for five to ten meals the household already makes and likes, with quick ratings, and explicitly ask which prepared or semi-prepared items are house staples (store rotisserie chicken, frozen dumplings, and the like), since users tend to omit these unless prompted, treating them as "not real meals" when they are often the most reliable rows in the library. A cold-start library makes the first plan session dramatically better because the interview can anchor on proven meals instead of guessing.

**Step 4: write and confirm.** Write all three documents to the verified backend, and confirm to the user where their data lives, who can access it, and that the skill itself stores nothing.

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
