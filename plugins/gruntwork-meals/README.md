# gruntwork-meals

Run a household meal plan as a repeating loop: plan the covered days, acquire
the food, cook the meals, and record what actually happened so next week's plan
is smarter than this week's.

Install instructions are in the [repo README](../../README.md).

## Four modes, no commands

The skill infers the mode from what you say. If no state store exists yet, it
runs `setup` first regardless of what you asked for, then continues into what
you wanted.

| Mode | Triggered by | What it does |
|---|---|---|
| **setup** | first run, or "reconfigure meal planning" | Provisions and verifies your state store, interviews you, seeds the meal library |
| **plan** | "let's plan next week" | Proposes a candidate slate, interviews you to a committed menu, writes the log rows |
| **shop** | "build the shopping list" | Explodes the plan into consolidated lists, checks the pantry, splits by source |
| **cook** | "what's for dinner", "we ordered pizza instead" | States tonight's meal and time-sensitive prep; records audibles as reality |

## State

Three documents in your own store — never in this repo. See
[Your data does not live here](../../README.md#your-data-does-not-live-here).

| Document | Holds |
|---|---|
| `config.yaml` | Household members, goals, cadence, consistency dial, acquisition sources, calendar settings, storage block |
| `meals.csv` | One row per distinct meal: `name, tags, protein, prep, effort, rating, times_made, last_made, source, notes` |
| `log.csv` | One row per covered day per week: `week_of, day, planned_meal, actual_meal, status, by, notes` |

### Backends

| Backend | Best for | Notes |
|---|---|---|
| **sheet** | Most households | A Google Sheet with `config`, `meals`, `log` tabs. Family members read and edit it directly, no repo literacy required. |
| **github** | Repo-comfortable households; agent contributors | Three documents at the root of a private repo. Commit messages name the mode and week. |
| **local** | Claude Code use | A directory you control. Do not point it inside a checkout of this repo. |

## The consistency dial

Config carries a 1-to-5 dial that decides how plan mode behaves.

- **5** — propose the proven rotation nearly unchanged, at most one new slot.
  The rotation is the most recent completed week's plan; with no prior week, the
  highest-rated keepers with the most `times_made`. "Same as last week" is a
  complete plan, not a failure of imagination.
- **1** — work the 21-day repeat window hard, filling most slots with meals not
  made recently or genuinely new ideas.
- **Between** — blend proportionally.

The original draft assumed variety was the goal. Testing against a household
with only three covered days showed consistency is often the point, especially
early on.

## Multiple contributors

Two adults and a household AI agent can share one state store. The skill
authenticates nobody — identity lives at the storage layer, where each
contributor reaches the backend with their own credentials and the backend's
sharing model is the access control.

What the skill does: asks who's driving, stamps a self-declared `by` column on
log rows, re-reads state immediately before any write, and never silently
deletes another contributor's data.

An agent contributor is structurally identical to a human one. That's the point.

## Deliberately deferred

Not oversights — each has a config-level home when wanted, and none blocks
phase one.

- Real authentication (identity stays at the storage layer)
- Per-person meal ratings
- Lunches and snacks
- Expansion beyond three covered days
- Delivery-service ordering integration

## Status

v0.4. Workshopped and revised through four versions, tested against a fixture
household (two adults, two kids, ten seeded meals, one completed week of log).
Calendar integration is specified but **untested** — the first real week will
exercise it.
