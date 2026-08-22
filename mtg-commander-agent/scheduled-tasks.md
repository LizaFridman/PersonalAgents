# Scheduled Tasks

> For you, not for Project upload — like `manabox-drive-workflow.md`, this documents how to configure Claude Projects' own recurring-task scheduler, a feature separate from Instructions, Knowledge, and native Project memory. Whatever prompt you paste into the scheduler runs as if you'd sent that message in a fresh chat in this Project, so it has the same access to Instructions, Knowledge, and Drive as any normal conversation.

## Weekly: price + EDHREC refresh for in-progress decks

**Cadence:** weekly.

**Scope:** only decks flagged "in progress" — see `wiki-structure.md`'s `wiki/index.md` format; a deck is in scope if its index entry contains the phrase "In progress". If none are currently in progress, the task should say so and do nothing rather than guessing which decks to touch.

**Task prompt** (paste as-is into the Project's scheduled task):

> Run the weekly deck refresh. Read `wiki/index.md` and find every deck entry containing "In progress". For each one:
> 1. Read that deck's `wiki/<topic>.md` page for its current decklist and commander.
> 2. Per `community-resources.md`, fetch the commander's EDHREC page and note any new high-synergy cards not already in the decklist or already rejected there — add a short "Possible additions (EDHREC, [date])" note to the deck's wiki page rather than rewriting the list yourself; don't add cards to the actual decklist without being asked.
> 3. Per `price-tracking-workflow.md`, log a price snapshot to `price_log.csv` — but only for a representative sample (the 15-20 highest-value-looking or most-recently-added cards in the deck), not an exhaustive 100-card sweep, consistent with that file's pacing guidance. Say explicitly that it's a sample, not a full valuation.
> 4. Append one `wiki/log.md` line per deck touched, noting what was refreshed.
> 5. If a deck's EDHREC or price data can't be fetched, say so for that deck and continue with the others rather than stopping the whole run.
>
> Finish with a one-line summary: how many decks were in scope, how many succeeded, and a short list of anything worth a look (new synergy suggestions, notable price moves).

**Why sampled prices, not exhaustive:** a full 100-card sweep per in-progress deck, every week, one Scryfall page fetch at a time, is exactly the workload `setup-guide.md`'s "If you outgrow this" section already flags as the reason to eventually build the batch-capable Level 3 system (`mtg-agent/`). Until that exists, sampling keeps this task fast and token-light; if it becomes a real pain point, that's a concrete signal to prioritize `mtg-agent` sub-project 1.

## Other candidates (not set up yet)

Documented here for later if wanted — ask to add one:
- **Knowledge-gaps reminder**: periodic nudge to review `wiki/knowledge-gaps.md` and reconcile it into the curated docs (currently a manual trigger, per `maintenance-workflow.md`).
- **Banned/Game Changers list check**: periodically re-verify the official lists against what's referenced in `commander-rules-reference.md`/`house-rules.md`, flagging drift.
- **ManaBox export reminder**: periodic nudge to re-export/upload the collection CSV, per `manabox-drive-workflow.md`'s refresh-cadence guidance.
