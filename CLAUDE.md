# gruntwork-household-skills

Claude skills for running a household. Published by **GruntworkAI** under MIT.

## Archetype: Usable

A distributable skill bundle others install into a Claude runtime. Optimize for
the installer/end-user experience, not just our dev loop.

## Workspace

- Org: gruntwork (`~/Code/gruntwork/gruntwork-household-skills`)
- Repo: `GruntworkAI/gruntwork-household-skills` (GitHub, public)

## Why this repo exists as a family

Meals is one skill in one plugin. It could have had its own marketplace; it
doesn't, deliberately.

Strip the food out of `meal-plan/SKILL.md` and what remains is a household
shared-state substrate: a storage-backend abstraction with sentinel-write
verification, an identity-at-the-storage-layer contributor model, config created
at first run, and a planned-versus-actual log. None of that is about food. Any
second household skill — chores, home maintenance, kid activities, budgeting —
needs all of it verbatim.

Adding a marketplace costs the user a one-time `/plugin marketplace add` per
marketplace. Sizing the repo for the family now means the second skill costs
them nothing.

**Do not extract the substrate yet.** One instance isn't enough to find the
seams. When household skill #2 lands, that's the moment to pull the shared
storage/contributor material into `plugins/*/core/` or a shared reference doc —
and it's a separate, deliberate decision, not a refactor to slip into a feature
branch. (travel-skills has held `core/` empty past its own rule-of-three bridge
for the same reason.)

## Layout

The repo is a **self-contained marketplace**: the root `marketplace.json`
indexes the plugins, and each plugin dir is what an install ships (Claude Code
and the claude.ai consumer app both install from it).

```
README.md                          # family-level entry point (install + data model)
LICENSE                            # MIT
.claude-plugin/marketplace.json    # marketplace index (one entry per plugin)
plugins/
  gruntwork-meals/                 # THE PLUGIN — everything an install ships
    .claude-plugin/plugin.json     # plugin manifest (name, version, license)
    README.md                      # plugin-specific detail
    skills/
      meal-plan/SKILL.md           # the skill (four modes: setup, plan, shop, cook)
docs/                              # reference docs (for users/deployers)
.claude/work/                      # plans, sessions, followups (for developers — not shipped)
```

## Invariants (do not break)

- **This repo never holds household data.** A household's `config.yaml`,
  `meals.csv`, and `log.csv` live in that household's own separate private state
  store, created by the skill's `setup` mode. Those filenames are gitignored
  here as defense in depth, because the `local` backend accepts an arbitrary
  directory path and a user could point it at a checkout of this repo. If you
  ever find yourself wanting to commit a real household's data "as an example,"
  build a synthetic fixture instead.
- **The skill is generic by construction.** No family, schedule, store, or
  dietary preference is hardcoded. If an instruction would only make sense for
  one household, it belongs in that household's config or meal library. This is
  the premise the whole design rests on — violate it once and the skill stops
  being shareable.
- **No `userConfig` in `plugin.json`.** Unlike travel-skills, household config
  lives entirely in the state store, because it must be shared across
  contributors and readable by the family. Adding plugin-level `userConfig`
  would create a second config surface competing with `config.yaml`. Resist it.
- **Reality over intention.** `planned_meal` and `actual_meal` stay separate
  columns; an audible updates a row rather than adding one. Don't add code or
  prose that collapses them or editorializes about bails — a log that gets
  argued with stops getting used.
- **Re-read before writing.** The state store is shared infrastructure. Every
  write re-reads the affected document first and reconciles rather than
  overwriting. Never silently delete another contributor's rows.
- **Storage is verified before setup proceeds.** Write a sentinel, read it back,
  and never continue an interview on unverified storage.

## Conventions

- snake_case across config keys and any Python (org convention).
- README at root = the family entry point; each plugin's README = domain detail.
- Skills are prose-only today. If helper scripts arrive, they go under the
  skill's own `scripts/` and stay deterministic — each SKILL.md must still stand
  alone, because self-contained runtimes like Claude Desktop have no shared
  filesystem.

## Change management

- Solo project: code/skill changes go through a branch + PR; doc and
  session-note touch-ups may go direct to `main`.
- Public repo — run `/run-scan-secrets` before pushing. Household data is the
  specific risk here, not API keys.

## Releases & packaging (marketplace-native)

The repo is the source of truth **and** the distribution channel: users add the
marketplace and install the plugin from it. Updates flow through the native
`/plugin update` mechanism.

**Claude Code** resolves the marketplace from the **default branch** — merging a
version bump to `main` IS the release for Claude Code.

🔴 **The consumer surface needs a GitHub release/tag.** The claude.ai / Claude
**Desktop** install resolves the plugin via the repo's GitHub **release/tag**,
not the default branch. If there's no release for the current version, or the
latest release predates the current layout, **the consumer install 404s**. This
bit gruntwork-travel-skills on 2026-07-14: "Latest" was a pre-restructure tag
and Desktop couldn't find `plugins/…`. Merging to `main` alone is **not** a
complete release.

### Release checklist

Run every step. Steps 1–3 are one commit.

1. **Bump the version in all three places together:**
   - `plugins/<plugin>/.claude-plugin/plugin.json` → `version`
   - `.claude-plugin/marketplace.json` → the plugin's `version` **and** the
     top-level `metadata.version`
   - `README.md` → the plugin table row

   A `plugin.json`-only bump looks stale to `/plugin update`, because the
   marketplace index is what clients actually read.

2. **Do not add a version to `SKILL.md` frontmatter.** `plugin.json` owns
   versioning outright. The skill carried its own `metadata.version` when it was
   authored standalone in Claude Desktop; that field was deliberately removed at
   scaffold time rather than kept in sync forever. A skill-level version is a
   second source of truth that drifts silently, because nothing validates the
   two against each other.

3. **Update `README.md`** if user-facing behavior changed.

4. **Merge to `main`** (branch + PR for skill changes).

5. **Cut the GitHub release and tag at `main`, marked latest:**

   ```
   gh release create vX.Y.Z --target main --latest \
     --title "vX.Y.Z" --notes "…"
   ```

   Required for the consumer-app install path. Skipping this ships to Claude
   Code and silently breaks Desktop.

6. **Keep stale/old-layout releases from being "Latest."** If an old tag is
   marked latest, the consumer install resolves to it.

7. **Verify both surfaces.** Claude Code: `/plugin update`. Desktop: install
   fresh from the marketplace and confirm the skill triggers. Desktop is the one
   that breaks; check it.

**Release-note discipline:** these notes are public. Describe the class of
change, never a specific household, meal, or state-store location.

## Gotchas

| Issue | Symptom | Cause / fix |
|-------|---------|-------------|
| **Merging to `main` is not a full release** | Claude Code updates fine; Desktop/claude.ai install 404s. | The consumer surface resolves via GitHub release/tag, not default branch. Always `gh release create vX.Y.Z --target main --latest` after the version-bump merge. See the checklist above. |
| **A version reappears in SKILL.md frontmatter** | `plugin.json` says 0.5.0, the skill's frontmatter says something else. | The skill was authored standalone in Claude Desktop and carried its own `metadata.version`; it was removed at scaffold time so `plugin.json` is the single owner. If a Desktop-authored revision reintroduces it, strip it on the way in rather than syncing it. |
| **`local` backend can point inside this repo** | A household's `config.yaml` / `meals.csv` / `log.csv` appear in `git status`. | The `local` backend takes any directory path. Those filenames are gitignored defensively — if you see them staged, stop and move the state store out of the repo entirely. |
| **Skill triggering on Desktop is unverified** | Skill doesn't fire on "what's for dinner" in the consumer app. | Whether Skills auto-trigger on the Desktop app is still an open question across the gruntwork plugins (same open item as the lastmilefirst plugin). Test explicitly on first Desktop install rather than assuming parity with Claude Code. |
| **Sheet backend may be read-only** | Setup's sentinel write fails against a Google Sheet. | The available connector may read but not write. The skill has a specified degraded paste-back mode — confirm it works rather than assuming, since the sheet is the backend most households will choose. |
