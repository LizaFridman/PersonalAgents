# ManaBox → Google Drive Workflow

> This defines how your ManaBox collection gets into the agent. There's no free official ManaBox API, so this is a manual export/upload step you repeat periodically — see `setup-guide.md` for the one-time setup and `project-instructions.md` / `wiki-structure.md` for how the agent treats this file (read-only ground truth, never edited by Claude).
>
> The collection answers "do I own this" and feeds buy lists. It is **not** a deckbuilding input — per `deckbuilding-framework.md` whether a card is owned never affects whether it goes in a deck.

## Exporting from ManaBox

1. In the ManaBox app, open the **Collection** tab.
2. Open the menu in the top right and choose **Export → CSV** for the whole collection (you can also export a single binder/list the same way from within that binder if you ever want a scoped export instead).
3. This produces a CSV including your card properties (name, set, quantity, foil status, condition, etc.) and the binder/list name.

ManaBox's documented CSV columns (used for both import and, per its own docs, export) include at minimum: card name, set code/name, quantity, foil, card number, language, condition, purchase price, purchase currency, and Scryfall ID. **Treat the actual header row in your exported file as authoritative** — don't assume a fixed column layout; the agent should read whatever headers are actually present rather than guessing positions.

## Uploading to Drive

1. In Google Drive, create (once) a folder named `MTG Commander Agent` at a location you'll remember, with a `raw` subfolder inside it (see `wiki-structure.md` for the full folder layout).
2. Each time you export from ManaBox, upload the new CSV into `MTG Commander Agent/raw/` as **`manabox_Collection.csv`**, replacing the previous file (don't accumulate dated copies here — this folder holds the *current* collection only; historical change tracking belongs in the wiki/log if it's ever worth recording, not in duplicate raw files).

## Refresh cadence

There's no live sync — refresh it when it'll actually matter:
- After a notable purchase, trade, or organizing session.
- Before a deck-building conversation where "what do I already own" materially affects the recommendation.

There's no harm in it being a few weeks stale for casual questions; just re-export before anything where accuracy matters (e.g. "am I done buying for this deck").

## What the agent does with it

Per `project-instructions.md`, `raw/manabox_Collection.csv` is read-only ground truth for ownership questions. The agent should:
- Match cards primarily by name + set code (and Scryfall ID when present), since that's what uniquely identifies a specific printing.
- Never write to this file. If it needs to record something derived from the collection (e.g. "this deck is 80% buildable from owned cards"), that goes in a `wiki/<topic>.md` page instead.
- Tell you plainly if the file is missing, empty, or looks stale relative to what you're asking, rather than silently working from an outdated collection.
