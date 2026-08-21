# Game Log & Win-Rate Workflow

> Defines how the agent logs game results and computes win-rate stats per deck. Companion to `price-tracking-workflow.md` (same Drive folder, same read-modify-write append pattern) — read that file first if you haven't; this one only calls out what's different.

## Logging a game result

Trigger prompts like "log a game — played [deck], won/lost, 4-player pod, opponents were [X, Y, Z]" or "I just lost with [deck] to a [commander] combo, 3-player."

Ask for (or infer from what's given) whatever of the following wasn't stated — don't block on optional fields, but don't guess `result` or `deck`:

- `date` — today's date if not given.
- `deck` — must match an existing `wiki/<topic>.md` deck page; if it doesn't, ask whether to create one first (per `wiki-structure.md`) or confirm the name.
- `commander` — the deck's commander(s), for cross-deck queries later (e.g. "how does [commander] do overall").
- `pod_size` — total players including the player, since win rate isn't comparable across pod sizes.
- `opponents_commanders` — semicolon-separated list of opposing commanders/decks, as known; `unknown` is fine if not tracked that game.
- `result` — `win`, `loss`, or `draw`.
- `notes` — short free text: what happened, why it went that way. This is the qualitative record `wiki/<topic>.md` summaries draw on later — don't skip it just because the row would still be valid without it.

## `games_log.csv` schema

Located at `MTG Commander Agent/games_log.csv` in Drive, alongside `price_log.csv`. One row per game:

```csv
date,deck,commander,pod_size,opponents_commanders,result,notes
2026-08-15,mono-black-aristocrats,Teysa Karlov,4,"Atraxa; Korvold; Meren",win,"Blood Artist chain closed it out turn 9"
2026-08-20,mono-black-aristocrats,Teysa Karlov,3,"Muldrotha; unknown",loss,"Got wrathed with no recovery, no board wipes drawn"
```

Quote any field containing a comma (e.g. `opponents_commanders` with multiple entries).

## Append mechanics

Same read-modify-write pattern as `price_log.csv` (see `price-tracking-workflow.md`):

1. Read the current `games_log.csv` content.
2. Append the new row(s) to the end — never reorder or edit existing rows.
3. Write the full updated content back (overwrite), preserving everything already there.
4. Update the deck's `wiki/<topic>.md` Stats section (below) and add a line to `wiki/log.md`.

If `games_log.csv` doesn't exist yet, create it with the header row shown above plus the first logged game.

## Deck Stats section (token-lean win rate)

Don't re-read and recompute from the full `games_log.csv` every time a deck's win rate comes up — that gets expensive as the log grows. Instead, each deck's `wiki/<topic>.md` page carries a small standing summary, refreshed whenever a game for that deck is logged:

```markdown
## Stats
- Record: 6-3-1 (10 games) — 60% win rate
- By pod size: 4-player 4-2 (67%), 3-player 2-1 (67%), 5-player 0-0-1
- Notable: 0-2 vs. fast-combo decks; strong vs. group-hug tables
- Last updated: 2026-08-20 (last game logged)
```

- Recompute this section from `games_log.csv` filtered to that deck whenever a new game for it is logged — it's a cheap incremental update (old record + one new row), not a full rescan.
- If asked a stats question the standing summary doesn't answer (e.g. "how do I do specifically against Muldrotha"), it's fine to read `games_log.csv` and filter/aggregate directly — the Stats section is an optimization for the common case, not the only way to answer.
- If a deck has no logged games yet, say so rather than fabricating a record.

## Example trigger prompts

- "Log a game: [deck], won, 4-player, vs Atraxa and Korvold, closed out with the sac engine."
- "What's my win rate with [deck]?" — read the Stats section first; fall back to `games_log.csv` if it's missing or stale-looking.
- "How am I doing against combo decks?" — this cuts across decks, so read `games_log.csv` directly and filter by `notes`/`opponents_commanders` rather than relying on any single deck's Stats section.
