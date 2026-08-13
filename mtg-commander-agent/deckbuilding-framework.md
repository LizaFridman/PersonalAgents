# Deck-Building Framework

> Upload this file as Project knowledge. It's a working structure for deck-building conversations, not a rules document (see `commander-rules-reference.md` for that). Use it to keep deck discussions consistent and to make card cuts/includes defensible rather than vibes-based.

> Once a commander is chosen (Step 1), pull the EDHREC page for it (`https://edhrec.com/commanders/<slug>` — see `project-instructions.md`'s roles list) for real popularity/synergy signal before free-associating card ideas. Treat it as a starting pool to filter through this framework's questions, not a list to copy wholesale — EDHREC skews toward what's popular, not necessarily what fits this specific budget/collection/bracket.

## Step 1 — Anchor the deck before picking cards

Before recommending cards, establish (and record in the deck's `wiki/<topic>.md` page — see `wiki-structure.md`):

1. **Commander and color identity.**
2. **Archetype / game plan** — what the deck is actually trying to do. Pick one primary plan; a deck trying to do three unrelated things is usually worse than one doing one thing well. Common Commander archetypes: aristocrats (sacrifice value), voltron (single-creature beatdown via auras/equipment), group hug/politics, stax (resource denial), tokens/go-wide, spellslinger, reanimator, ramp-into-big-stuff, tribal synergy, combo.
3. **Target bracket** (1–5, per `commander-rules-reference.md`'s Bracket system) — agreed with the table this deck will actually play at, since this drives how many Game Changers / fast mana / tutors / infinite combos are appropriate.
4. **Budget**, if any — and whether it's a hard cap or a "prefer cheaper when equal" preference.

## Step 2 — Card quota guidelines (100-card deck, adjust for archetype)

These are starting ratios, not hard rules — a low-curve aggro deck runs fewer lands than a ramp deck, a combo deck needs more tutors and less removal, etc. Say explicitly when you're deviating and why.

| Category | Typical count | Notes |
|---|---|---|
| Lands | 36–38 | Lower with more ramp/cheap curve; a precon-level deck should stay near 38. |
| Ramp (mana rocks/dorks/ramp spells) | 8–12 | Include in the land-count trade-off above. |
| Card draw / advantage | 8–12 | Recurring advantage engines count double toward "enough." |
| Removal (spot + board wipes) | 8–10 | Split between targeted removal and 1–3 wraths depending on the meta. |
| Protection / interaction (counterspells, hexproof effects) | 3–6 | More at higher brackets. |
| Synergy / payoff pieces | remainder | Everything that executes the actual game plan. |

## Step 3 — Evaluate a candidate card against 4 questions

1. **Does it advance the game plan?** If it's just "a good card" but doesn't fit the archetype, it's a cut candidate even if powerful — Commander rewards coherence.
2. **Does it do too little on its own?** Cards that only work with 2+ other specific cards in play are combo pieces, not general inclusions — fine in small numbers, risky as the deck's backbone unless the deck *is* a combo deck.
3. **What's the mana cost relative to impact?** In a 100-card singleton deck you rarely draw your best cards on curve — prioritize efficient, flexible cards over narrow high-ceiling ones unless the deck has enough tutors/draw to find them reliably.
4. **Ownership and cost** — check the collection (`raw/manabox_Collection.csv`) before recommending a purchase; if the player already owns a reasonable substitute, say so and note the tradeoff instead of defaulting to the "optimal" unowned card.

## Step 4 — Power-level calibration

Tie recommendations back to the target bracket:

- **Brackets 1–2**: avoid Game Changers entirely, avoid fast infinite combos, favor thematic/synergy cards over raw efficiency, don't over-optimize the mana base with expensive fast lands unless the player wants that.
- **Bracket 3**: up to 3 Game Changers is normal; still generally avoid a deck whose whole plan is a 2-card infinite combo that ends the game with no interaction window.
- **Brackets 4–5**: optimize freely; consult live card data aggressively rather than relying on this file's general guidance, since the metagame at this level moves fast.

## Step 5 — Mana base sanity check

- Color pip count across the deck (not just card count) is the better guide to how many sources of each color you need — a deck with double-pip commitments in one color needs more sources of that color than an even 3-color split.
- Untapped, colorless-fixing lands (e.g. basics, fetches, duals) are worth more early; taplands are worth less in aggressive/low-curve decks.
- Note when a mana base is aspirational (needs cards the player doesn't own) vs. buildable today from the collection.

## Synergy checklist (for evaluating a whole deck, not just one card)

- Is there a clear A-to-B-to-win path, or is it a pile of good-but-disconnected cards?
- Are there redundant effects for the deck's most important effect (e.g. multiple sac outlets in an aristocrats deck), so one removed piece doesn't blank the plan?
- Does the deck have a plan against removal-heavy or counterspell-heavy tables, or is it fully reliant on one resolving spell?
- Is there a way to interact with what opponents are doing, or is this a fully solitaire deck (fine for casual, risky at higher brackets)?
