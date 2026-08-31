# Findings from the first live setup run (2026-08-25)

> **All defects and gaps below were fixed in v0.4.4.** Kept as the record of what
> a real run found, and of what the skill got wrong before anyone ran it. A
> seventh issue — the word "sentinel" in shipped prose — was caught in review of
> the fix itself and is noted at the end.

First execution of `meal-plan` against a real household and a real Drive store.
Everything below was found by running it, not by reading it.

Store: a Drive folder **owned by the user**, shared to the agent account as
editor. That ownership direction was a deliberate choice (durability — the store
outlives the agent account) and it is what surfaced defects 1 and 5.

---

## Defects — the skill says something that is not true

### D1. Setup tells the skill to share the folder. It cannot. 🔴

Setup step 3: *"share the **folder** with each as an editor."*

```
share_file(folder, <adult email>, writer)
→ "The caller does not have permission"
```

When the user owns the folder, the agent account is an editor, and Drive blocks
editors from granting access (either by default or via "Editors cannot change
permissions and share"). The instruction is unexecutable in the ownership
arrangement the skill otherwise recommends.

Arguably the permission model is right and the skill is wrong: access to a
family's data should require the owner, not the service account.

**Fix:** attempt the share, catch the failure, and hand the user precise
instructions — folder name, exact address, required role. Never report setup
complete with a member configured but not actually granted access; that is the
specific failure the `email` requirement was added to prevent.

### D2. Booleans are type-coerced on the sheet backend

Wrote `true`, read back `TRUE` — Sheets converted it to a native boolean. The
skill claims values return verbatim, but that was verified for strings, numbers,
and ISO dates only. A strict `value == "true"` comparison fails.

**Fix:** compare booleans case-insensitively, or store `yes`/`no` as plain
strings. Say which in the skill rather than leaving it to the reader.

### D3. How content gets into a document is never stated 🔴

The write cycle says *"create the replacement in the same folder, same title"*
and stops there. It never says **how the content gets in**.

Creating a file with mimetype `application/vnd.google-apps.spreadsheet` produces
an **empty** Sheet — and there is then no way to write cells, because the
connector cannot edit existing files. A run following the skill literally walks
into a dead end and has no recovery.

The working mechanism: `create_file` with `contentMimeType: text/csv` and the
full document as `textContent`, which Drive auto-converts to a Sheet.

**Fix:** state the mechanism explicitly in the write cycle. This is the single
most load-critical omission found — the backend does not work without it.

Related: because the transport is CSV, values containing commas **must** be
CSV-quoted on the way up. The skill's "do not escape anything in a sheet cell"
rule is about the stored cell, and reads as contradicting this. Separate the two
rules explicitly.

---

## Gaps — true but incomplete

### G1. Setup creates documents twice

Step 1 says create the three Sheets with headers; step 4 says write all three.
Followed literally, every document is created empty and immediately replaced,
producing a pointless archive entry on day one.

Deviated deliberately: verified storage with a throwaway `__sentinel__` document
(written, read back, trashed), then created each document once with real content.

**Fix:** make that the instruction. Verify with a sentinel; create documents once
they have something to hold.

### G2. The interview assumes the library is ready

Setup runs storage → interview → seed library → write, as one pass. In the real
run the user answered the interview and then wanted time to think about the meal
library, which is entirely reasonable — it is the part that takes actual thought.

Creating an empty `meals` and replacing it later is wasteful; blocking setup on
it is worse.

**Fix:** allow the library to be deferred. Write config and log, say plainly that
`meals` is outstanding, and let a later turn add it. Note that at a high
consistency dial the library *is* the rotation, so it matters more there.

### G3. Setup has no way to report partial completion

Between D1 and G2, this run finished with two things outstanding (Heather's
access, the meal library) and nothing in the skill covers saying so. Step 4 only
describes confirming success.

**Fix:** setup should end with what was done **and** what remains, naming each
outstanding item and who has to do it.

---

## Onboarding observations

### O1. Signals with no config home

Two useful things the user volunteered had nowhere structured to live:

- Sunday is the shopping day, **and** Sunday cooking is possible when it works
  out — that is a real planning affordance, not just a shopping fact.
- Leftovers matter: four adult portions feeds this household with lunch left
  over, and the user would rather be over-portioned than short.

Both were parked in a `cadence` note and in `goals.shared`, where plan mode will
at least see them. Whether portioning deserves a real field is a product
question, not an oversight to fix blindly — but the second one changes which
meals should be proposed, which is more than a note usually earns.

### O2. "The kids" is not one audience

Ages 6.5 and 2.5 differ in portion and texture, not only taste. Recorded in each
kid's notes. Worth watching whether the library eventually needs per-person
ratings — currently deferred by design.

### O3. The interview is long for one turn

Storage, people, goals, cadence, dial, sources, calendar, and the library in a
single pass is a lot. It worked here because the user is technical and motivated.
A first-time household would likely stall. Staging it — storage, then people and
cadence, then the library as its own step — matches how the real run actually
went.

### O4. The library step hands the user a blank page

The real run stalled at exactly one place: the meal library. Not because the
question was unclear, but because "name five to ten meals you make" is cold
recall, and cold recall is the hardest thing to ask of someone ten minutes into
a setup they came to in order to *stop* thinking about dinner.

The first fix attempted was a shipped example library — one generic CSV, the
same for every household. That was rejected in review, correctly: a generic
example is something you read, not something you can act on, and it makes every
household start from a stranger's kitchen.

**Fixed in v0.4.4:** setup now drafts a tailored starter library from the
interview answers and asks the household to correct it. Everywhere else the
skill already proposes and interviews rather than demanding generation — plan
mode most of all — and setup was the one place still asking for composition
from nothing. Correction is a far cheaper cognitive act than composition, and it
returns better data, because "we do that with turkey" is a more precise answer
than anything a blank prompt elicits.

The constraint that makes it safe: a drafted row is a proposal, and only
confirmed rows are written. Seeded rows carry `times_made` 0 and an empty
`last_made` — never invented history. Those columns drive rotation and the
repeat window, so fabricating them would corrupt every plan afterward, dodging
repeats that never happened and treating unmade meals as proven. The generic
example file was deleted rather than kept alongside; two sources of truth for
what a library looks like is one too many.

---

## Not defects, recorded to prevent rediscovery

- **Documents created by the agent inside the user's folder are owned by the
  agent.** The folder survives the agent account; the documents do not. Inherent
  to replacement writes — every write creates a new agent-owned document, so
  transferring ownership once does not hold. Manageable when the user administers
  the agent's domain; worth knowing before assuming folder ownership solves
  durability.
- **Folder-level sharing does propagate**, both to documents created later and to
  documents that already existed when the share was made. Verified twice.

---

## D4. "Sentinel" shipped in user-facing prose

Caught while reviewing the fixes above, not during the run. The skill told Claude
to write a "sentinel" — a term meaning nothing to a family setting up meal
planning, and one that reaches them directly, because SKILL.md prose becomes what
Claude *says* mid-conversation.

It had survived four version bumps and several reviews by reading as unremarkable
to anyone who works with software.

**Fixed in v0.4.4:** now "throwaway test document" throughout, and the same
replacement applied to the blank-value rule, which used "sentinel" in its other
sense. A convention was added to CLAUDE.md: no unexplained jargon in SKILL.md or
shipped READMEs; developer-facing files may use ordinary technical vocabulary.
The line is who ends up hearing it.
