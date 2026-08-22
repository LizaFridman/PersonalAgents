# Price Tracking Workflow

> Defines how the agent looks up and logs card values. Covers both owned cards (from `raw/manabox_Collection.csv`) and unowned cards you're considering.

## On-demand lookups (no logging)

For "what's this worth" questions, fetch the card's Scryfall page (`https://scryfall.com/search?q=!"<name>"` for an exact match) per `project-instructions.md`'s card-data role — its JSON API (`api.scryfall.com`) is avoided here since it's been observed returning 403s to this agent's fetch tool. Read the price(s) shown on the page: `usd`, `usd_foil`, `eur`, `tix` (not every card has every finish). Report the relevant one(s) given the card's actual finish (check owned quantity/foil status from the collection CSV when the question is about an owned card). Don't log anything to `price_log.csv` unless asked to — on-demand lookups are ephemeral.

**Pacing**: these are individual page lookups, not a batch API (see `project-instructions.md`). That's not a practical constraint for a handful of cards in a chat, but for a large batch (e.g. "price my whole 100-card deck" or a big chunk of the collection):
- Prefer summarizing (e.g. price the top 10–15 most valuable-looking cards by rarity/type rather than all 100) unless the player explicitly wants an exhaustive total.
- If an exhaustive total is genuinely wanted, say you're doing it in batches and spread it across a couple of messages rather than issuing a huge burst of individual fetches in one turn.

## `price_log.csv` schema

Located at `MTG Commander Agent/price_log.csv` in Drive (see `wiki-structure.md` for the folder layout). One row per card per snapshot date:

```csv
date,card_name,set_code,finish,quantity_owned,unit_price_usd,total_value_usd,source
2026-08-13,Sol Ring,cmr,nonfoil,1,1.85,1.85,scryfall
2026-08-13,Smothering Tithe,rna,nonfoil,1,24.10,24.10,scryfall
```

- `quantity_owned` — from the collection CSV; use `0` when logging a price for a card you don't own (e.g. tracking a card you're considering buying).
- `total_value_usd` — `unit_price_usd * quantity_owned` (0 for unowned cards being watched).
- `source` — always record where the price came from (`scryfall`), so a future mixed-source history stays honest.

## Logging a snapshot (append mechanics)

Google Drive's create/read tools available to the agent don't support in-place row insertion into an existing file — appending means a **read-modify-write**:

1. Download/read the current `price_log.csv` content.
2. Append the new dated row(s) to the end (don't reorder or edit existing rows).
3. Write the full updated content back to the same file (overwrite), preserving everything that was already there.
4. Add a line to `wiki/log.md` noting the snapshot was taken and how many cards were covered (per `wiki-structure.md`).

If `price_log.csv` doesn't exist yet, create it with the header row shown above plus the first snapshot's rows.

## Example trigger prompts

- "Log current prices for my [deck name] deck" — look up each card in that deck's `wiki/<topic>.md` decklist, append a dated batch of rows.
- "What's my collection worth right now" — for a full collection this is the exhaustive case above: confirm scope/expectations before firing off a large batch of lookups, since it'll take a while one card at a time.
- "Has [card] gone up since I last checked" — read existing rows for that card from `price_log.csv` rather than re-fetching, if a recent-enough snapshot already exists; otherwise fetch fresh and note there's no prior snapshot to compare against.
