# Scheduled Tasks

> For you, not for Project upload — like `manabox-drive-workflow.md`, this documents how to configure Claude Projects' own recurring-task scheduler, a feature separate from Instructions, Knowledge, and native Project memory. Whatever prompt you paste into the scheduler runs as if you'd sent that message in a fresh chat in this Project, so it has the same access to Instructions, Knowledge, and Drive as any normal conversation.

## Weekly: price + EDHREC refresh for in-progress decks

**Cadence:** weekly.

**Scope:** only decks flagged "in progress" — see `wiki-structure.md`'s `wiki/index.md` format; a deck is in scope if its index entry contains the phrase "In progress". If none are currently in progress, the task should say so and do nothing rather than guessing which decks to touch.

**Task prompt** (paste as-is into the Project's scheduled task):

> Run the weekly deck refresh. Read `wiki/index.md` and find every deck entry containing "In progress". For each one:
> 1. Read that deck's `wiki/<topic>.md` page for its current decklist and commander.
> 2. Per `community-resources.md`, fetch the commander's EDHREC page and note any new high-synergy cards not already in the decklist or already rejected there — add a short "Possible additions (EDHREC, [date])" note to the deck's wiki page rather than rewriting the list yourself; don't add cards to the actual decklist without being asked.
> 3. Per `price-tracking-workflow.md`, log a price snapshot to `price_log.csv` — but only for a representative sample (the 15-20 highest-value-looking or most-recently-added cards in the deck), not an exhaustive 100-card sweep, consistent with that file's pacing guidance. Say explicitly that it's a sample, not a full valuation. First check whether a snapshot dated today already exists for this deck in `price_log.csv` — if so, skip logging again for it this run (avoids duplicate same-day rows if this task is accidentally re-run).
> 4. Append one `wiki/log.md` line per deck touched, noting what was refreshed.
> 5. If a deck's EDHREC or price data can't be fetched, say so for that deck and continue with the others rather than stopping the whole run.
>
> Finish with a one-line summary: how many decks were in scope, how many succeeded, and a short list of anything worth a look (new synergy suggestions, notable price moves).

**Why sampled prices, not exhaustive:** a full 100-card sweep per in-progress deck, every week, one Scryfall page fetch at a time, is exactly the workload `setup-guide.md`'s "If you outgrow this" section already flags as the reason to eventually build the batch-capable Level 3 system (`mtg-agent/`). Until that exists, sampling keeps this task fast and token-light; if it becomes a real pain point, that's a concrete signal to prioritize `mtg-agent` sub-project 1.

## Weekly: wiki hygiene check (duplicate files + index drift)

**Cadence:** weekly (Claude Projects' scheduler has a one-week minimum interval — can't be set to monthly even though the underlying problem doesn't need weekly attention). This is fine in practice: the check itself is cheap (list a Drive folder, compare filenames — the heavier read/compare work only runs if it actually finds a duplicate), and catching a duplicate within a week beats catching it within a month.

**Why this exists:** Google Drive doesn't enforce unique filenames in a folder — unlike a normal filesystem, two files can share the exact same name. The read-modify-write pattern every write in this package uses (see `maintenance-workflow.md`'s "Known limitation: concurrent writes") has, in practice, occasionally created a new file instead of updating the existing one, leaving duplicate-named files sitting side by side with no automatic way to notice. This task is the periodic check for that, plus a check that `wiki/index.md` still matches what's actually in `wiki/`.

**Task prompt** (paste as-is into the Project's scheduled task):

> Run the weekly wiki hygiene check. This audits the wiki folder's own structural health, not any deck's content.
> 1. List every file in `wiki/` via the Drive integration. Group by exact filename.
> 2. For any filename with more than one file present, this is the known duplicate-write failure mode:
>    - Read all copies. If one copy's content is a clean superset of the others (contains everything the older ones do, plus more — e.g. an append-only file like `log.md` or `knowledge-gaps.md` that simply grew), keep the newest/most complete copy and move the rest to Drive Trash (not permanent delete).
>    - If the copies have genuinely diverged — real content in one that's actually missing from the others, not just "older and shorter" — do NOT auto-resolve. Instead, log an entry in `wiki/knowledge-gaps.md` naming both file IDs and summarizing the difference, and leave all copies in place for the player to review.
> 3. Cross-check `wiki/index.md` against the files actually present in `wiki/`: log a `knowledge-gaps.md` entry for any wiki file with no index entry, and any index entry pointing to a file that no longer exists.
> 4. Append one `wiki/log.md` line summarizing this run — how many duplicate sets found, how many auto-resolved, how many flagged, how many index-drift issues found.
> 5. If nothing is wrong, say so plainly and stop — don't manufacture busywork or touch files that don't need it.
>
> Finish with a one-line summary: duplicate sets found/cleaned/flagged, and index-drift issues found.

**Why this is fine at weekly, even though the underlying problem doesn't need it:** the check itself is nearly free most weeks (a folder listing plus a filename comparison, ending immediately at step 5 if nothing's wrong) — the cost only shows up on a week a duplicate actually exists, which is exactly the week you want it caught. If duplicates turn out to appear almost every week, that's itself worth noticing (a sign the underlying write pattern needs a closer look), not just routine output.

**What this doesn't cover**: the doc/architecture-level review of the *curated files themselves* (`project-instructions.md`, `deckbuilding-framework.md`, etc.) — cross-reference bugs, token-budget shape, Knowledge-upload completeness, and so on — is out of scope for this task. That review needs to edit files in the GitHub repo directly, which the live Project agent cannot do (Project Knowledge from GitHub is read-only from chat, same reason `wiki/knowledge-gaps.md` exists at all). That kind of review stays a periodic Claude Code session task — see `maintenance-workflow.md`'s "Periodic doc/architecture audit" note.

## Other candidates (not set up yet)

Documented here for later if wanted — ask to add one:
- **Knowledge-gaps reminder**: periodic nudge to review `wiki/knowledge-gaps.md` and reconcile it into the curated docs (currently a manual trigger, per `maintenance-workflow.md`).
- **Banned/Game Changers list check**: periodically re-verify the official lists against what's referenced in `commander-rules-reference.md`/`house-rules.md`, flagging drift.
- **ManaBox export reminder**: periodic nudge to re-export/upload the collection CSV, per `manabox-drive-workflow.md`'s refresh-cadence guidance.
