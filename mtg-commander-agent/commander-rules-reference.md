# Commander Rules Reference

> Upload this file as Project knowledge. It covers the stable, commonly-needed rules of the Commander (EDH) format. It is deliberately not exhaustive — for anything not covered here, or for the exact current banned/Game Changers lists (which change periodically), fetch live data per `project-instructions.md` rather than guessing.

## Deck construction

- Exactly **100 cards** total, including the commander.
- **Singleton**: no two cards may share an English card name, except basic lands (and the rare card that explicitly permits duplicates, e.g. "A deck can have any number of cards named [X]").
- Every card in the 99 (and the commander itself) must be a legal Commander-format card: no silver-border, gold-border, or acorn-stamped (Un-set) cards.

## Commander & color identity

- The **commander** is a legendary creature — or another card type explicitly stating it can be a commander (e.g. certain Planeswalkers, Backgrounds paired with a "Choose a Background" creature) — designated before the game and placed in the **command zone**.
- **Color identity** = every colored mana symbol printed anywhere on the card: mana cost, color indicator, and rules text — including reminder text. A card with no colored symbols anywhere is colorless identity.
- Every card in the deck (the 99, plus any card with color identity like a nonbasic land with colored text) must have a color identity that is a **subset** of the commander's color identity.
- **Partner**: some commanders have the Partner ability, letting you run two single commanders instead of one (their combined color identity applies to the deck). "Partner with [X]" pairs a specific named card instead of any Partner. Backgrounds work similarly via "Choose a Background."

## Command zone mechanics

- The commander starts the game in the command zone and may be cast from there.
- **Commander tax**: each time your commander has been cast from the command zone previously this game, it costs an additional {2} to cast again (cumulative).
- If your commander would be put into your hand, library, or graveyard, or exiled, from anywhere, you may choose to move it to the command zone instead. This is a replacement effect you control on a case-by-case basis — you don't have to send it back if you'd rather keep it, e.g., in the graveyard for a reanimation plan.

## Life total & commander damage

- Players start at **40 life** (not 20).
- A player who has been dealt **21 or more combat damage by a single commander** over the course of the game loses the game, regardless of remaining life total. This is tracked per-commander, cumulatively (not per-hit).

## Format shape

- Default is **free-for-all multiplayer**, commonly 4 players, turn order clockwise. No teams unless the table agrees to house rules.
- Commander is a **social format**: table talk, threat assessment, and pre-game power-level agreement are part of normal play, not a house-rule bolt-on.

## Power level: the Bracket system

Since February 2025, Wizards of the Coast publishes an official **five-bracket** system to help tables agree on power level before a game:

1. **Exhibition** — casual, theme-first decks.
2. **Core** — precon-level power.
3. **Upgraded** — optimized casual; may run a small number of Game Changers.
4. **Optimized** — high-power, minimal restriction.
5. **cEDH** — fully competitive.

A short official **"Game Changers"** list (separate from the banned list — these cards are legal, just power-signaling) determines eligibility for the lower brackets: zero Game Changers keeps a deck in Brackets 1–2; one to three pushes it to Bracket 3; Brackets 4–5 have no cap. This list is revisited periodically, so treat it as a live lookup, not something to memorize into this file.

## Banned list

There is a single official banned list (not tiered by "banned as commander" vs. "banned as companion" — Wizards' Commander site treats it as one list) maintained at mtgcommander.net, currently on the order of 80+ cards. It changes via periodic announcements. **Always verify current status live** — either mtgcommander.net's banned list page, or a Scryfall site search (`https://scryfall.com/search?q=banned:commander`, per `project-instructions.md` — not the `api.scryfall.com` endpoint) — rather than relying on a memorized list here. This file intentionally does not enumerate it, since a stale copy is worse than an explicit live check.

**For this player's actual games**, `house-rules.md` takes precedence over this official list — their playgroup may ban additional cards. Check it first; this section (and the live official-list lookup above) is the fallback for general Commander legality questions, not their table specifically.

## Common multiplayer rulings worth knowing

- **"Choose an opponent" / "target opponent"** effects: in multiplayer, you choose among all opponents, not just one designated rival, unless the card restricts it.
- **Deathtouch + trample**: only 1 damage needs to be assigned to a blocker for the rest to trample through, when combined.
- **State-based actions** (life ≤ 0, commander damage ≥ 21, a creature with lethal damage marked, etc.) are checked continuously and simultaneously — a player can lose to commander damage even if an effect would otherwise have saved them via a life-total-based rule.
- **Priority passes** all the way around the table before a spell/ability resolves — any player can respond, not just the one being targeted.

## Common judge calls

For when the player asks you to settle a live in-game dispute, not just explain a rule in the abstract:

- **Missed triggers**: a "may" trigger you forget isn't lost — you can still choose to apply it once it's noticed, as long as no other decision has been made based on it having *not* happened. A mandatory trigger that's missed is generally handled by backing up as little as possible to fix it if it's caught quickly; if the game has moved on significantly, most casual tables just let it go rather than unwinding several turns — this is a table-etiquette call as much as a rules one.
- **Stack order and responses**: the most recently added ability/spell resolves first (LIFO). Any player, in turn order starting from whoever has priority, can respond to anything on the stack — including responding to their own spell/ability before it resolves.
- **"I didn't know that card did that"**: information about a card's actual printed rules text is public — a player can always ask to re-read a card, and outcomes aren't reversed just because a player misjudged a legal play's consequences (as opposed to a genuine misplay caused by wrong information someone gave them, e.g. a misread rules text — that's a better candidate for backing up).
- **Simultaneous triggers, same player**: that player chooses the order. **Simultaneous triggers, different players**: active player's triggers go on the stack first, then in turn order — meaning they actually resolve *last* (stack is LIFO), so the active player's trigger resolves after everyone else's simultaneous triggers.

These are the calls that come up often enough to be worth a quick reference; for anything with real money/format-legality stakes or a genuinely novel interaction, cross-check the actual oracle text and rulings live (per `project-instructions.md`'s judge-mode order: house rules → this file → live Scryfall lookup) rather than extrapolating from these examples.
