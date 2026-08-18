# gruntwork-household-skills

Claude skills for running a household — the recurring domestic loops that are
too small to hire for and too persistent to hold in your head.

Every skill here shares one idea: **plan it, do it, and record what actually
happened**, so next cycle starts from evidence instead of memory.

| Plugin | What it does | Version |
|---|---|---|
| [`gruntwork-meals`](plugins/gruntwork-meals) | Plan the week's meals, build the shopping lists, cook, and log reality | 0.4.3 |

---

## Your data does not live here

This repo ships the skill. **It never holds your household's data.**

The first time you run the skill, its `setup` mode creates a *separate* state
store that you own and control — a Google Sheet, a private GitHub repo, or a
local directory. Your family roster, your meal library, and your weekly log all
live there. The skill stores nothing.

That separation is the point: it's what lets the skill be shared with another
household without handing over yours.

```
this repo (public)                 your state store (private, yours)
├── the skill                      ├── config     household, cadence, goals
└── no household data              ├── meals      the meal library
                                   ├── log        what actually happened
                                   └── archive/   superseded copies, kept as backup
```

On Google Drive those are separate Sheets in one shared folder; on GitHub or a
local directory they're `config.yaml`, `meals.csv`, and `log.csv`.

Every change replaces a document rather than editing it, and the superseded copy
moves to `archive/` stamped with when it changed and who changed it. So a bad
week is recoverable, and two people editing at once can't quietly lose each
other's work.

Most households pick the **Google Drive** backend — the data stays in a shared
folder anyone in the family can open and edit directly, without needing to know
what a repo is. Setup asks for each adult's email address and shares the folder
with them. Pick GitHub if you're already comfortable with repos, or if you plan
to have a household AI agent contributing with its own credentials.

---

## Install

### Claude Code

```
/plugin marketplace add GruntworkAI/gruntwork-household-skills
/plugin install gruntwork-meals@gruntwork-household-skills
```

### Claude Desktop / claude.ai

Settings → Customize → Plugins → Add marketplace, then paste:

```
GruntworkAI/gruntwork-household-skills
```

Install `gruntwork-meals` from the marketplace listing.

### First run

Just talk to it. Say *"let's set up meal planning"* — or simply *"what's for
dinner"* — and it will run setup first, walk you through creating your state
store, verify it can actually read and write, and then interview you.

Setup takes about ten minutes, most of which is naming five to ten meals you
already make.

---

## Using it

The skill infers what you want from what you say. There are no commands.

| You say | It does |
|---|---|
| "Let's plan next week" | Proposes a slate from your library, interviews you to a committed menu |
| "Build the shopping list" | Explodes the plan into consolidated lists, split by where you shop |
| "What's for dinner?" | Tells you tonight's meal and anything time-sensitive (defrost now, marinate by 4) |
| "We ordered pizza instead" | Logs the audible without argument, keeps the original plan visible as history |
| "How did the week go?" | Plan versus actual, neutrally, and asks if any ratings changed |

---

## Design commitments

These aren't preferences. They're why the skill behaves the way it does, and
changing one changes the product.

- **Generic by construction.** Nothing about one family is in the skill. If an
  instruction would only make sense for one household, it belongs in that
  household's config.
- **Reality over intention.** The log records what happened, not what was
  planned. Both columns are kept. Plan-versus-actual is planning data, never a
  scorecard.
- **Prepared food is first-class.** A store rotisserie chicken and frozen
  dumplings are legitimate library entries, proposed on equal footing with
  scratch cooking. Plenty of households run their best weeknights on them.
- **Consistency is a dial, not a failing.** Set it to 5 and "same as last week"
  is a complete, legitimate plan. Set it to 1 and it works the repeat window
  hard. There's no correct setting, only your household's.
- **Shared means shared.** Two adults and a household agent can all write to one
  state store. The skill re-reads before every write and never silently deletes
  another contributor's data.

---

## License

MIT. See [LICENSE](LICENSE).
