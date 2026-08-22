# Deck-Building Framework

> Upload this file as Project knowledge. It's a working structure for deck-building conversations, not a rules document (see `commander-rules-reference.md` for that). Use it to keep deck discussions consistent and to make card cuts/includes defensible rather than vibes-based.

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
| Lands | 35 | Sweet spot for this player — 37+ is too many by default. Go a couple lower with heavy ramp/a very low curve; only go a couple higher for an unusually greedy, high-curve manabase. |
| Ramp (mana rocks/dorks/ramp spells) | 8–12 | Include in the land-count trade-off above. |
| Card draw / advantage | 8–12 | Recurring advantage engines count double toward "enough." |
| Removal (spot + board wipes) | 8–10 | Split between targeted removal and 1–3 wraths depending on the meta. |
| Protection / interaction (counterspells, hexproof effects) | 3–6 | More at higher brackets. |
| Synergy / payoff pieces | remainder | Everything that executes the actual game plan. |

## Step 3 — Evaluate a candidate card against 5 questions

A card needs to clear these together, not pass just one — fitting the deck's theme doesn't excuse a bad mana-cost-to-value trade-off, and raw power doesn't excuse not fitting the plan. See `community-resources.md`'s "Deckbuilding theory & card evaluation" section for sourced methodology behind questions 2 and 4.

1. **Does it advance the game plan?** If it's just "a good card" but doesn't fit the archetype, it's a cut candidate even if powerful — Commander rewards coherence. Fitting the theme is necessary, not sufficient — see 2.
2. **Is it worth its mana cost on its own terms, in this deck?** Rate = effect delivered ÷ mana invested, judged against this deck's actual curve and speed. A card that complements the theme but underperforms its cost relative to what else is available at that cost is still a weak include — "on theme" doesn't waive this.
3. **Does it do too little on its own?** Cards that only work with 2+ other specific cards in play are combo pieces, not general inclusions — fine in small numbers, risky as the deck's backbone unless the deck *is* a combo deck. Check any such combo against `house-rules.md`'s combo ban before relying on it at all.
4. **What's the opportunity cost against the actual alternatives?** A slot given to this card is a slot not given to every other card that could fill it. Name at least one real alternative for the slot and state explicitly why this card wins the comparison (better rate, better curve fit, higher floor/ceiling for this deck specifically) — don't evaluate a card in isolation and call it done.
5. **Ownership and cost** — check the collection (`raw/manabox_collection.csv`) before recommending a purchase; if the player already owns a reasonable substitute, say so and note the tradeoff instead of defaulting to the "optimal" unowned card.

**On community synergy data specifically**: EDHREC's synergy scores come from the wider Commander population, which plays cards `house-rules.md` bans for this table (fast mana beyond Sol Ring, stax, infinite combos, free counterspells, and more). A high synergy score doesn't automatically transfer here — before recommending a card on synergy strength, check whether the reason behind that score depends on an interaction this pod doesn't allow, and discount it if so.

**Document the answer, per card.** A finalized or updated decklist in the deck's `wiki/<topic>.md` page isn't a bare card-name list — each card carries a one-line justification (which of the above it wins on) and names what it actually synergizes with in this deck, not just "fits the theme." See `wiki-structure.md` for the exact format. This is what lets a later session reconsider a specific include/cut on real reasoning instead of re-deriving it from scratch.

## Common rejection patterns (catch these before finalizing)

Recurring reasons a card gets cut in review — checking for these directly tends to catch problems faster than only running the Step 3 questions in the abstract:

- **Narrow/meta-specific tech without a stated reason.** A card whose value depends on a specific opposing strategy (tribal hate, graveyard hate, artifact hate) is a weak default include — only run it if there's an actual reason tied to this pod's known tendencies, not "just in case."
- **Weak effect in a category that has better options.** "It's removal/draw/ramp, the deck needs some" isn't sufficient on its own — compare it against the other real candidates in that category at similar cost (Step 3, question 4); a small or conditional effect loses to a cleaner one even if both technically qualify.
- **Payoff type doesn't match the actual win condition.** A card that rewards combat damage/attacking is dead weight in a deck whose plan is sacrifice, drain, control, or an alt-wincon that doesn't route through connecting in combat — check the payoff's trigger against Step 1's stated game plan, not just its color or type.
- **A fixed single-mode card loses to a flexible one at equal cost.** Between two cards at the same mana cost, one offering a choice of modes (e.g., ramp *or* draw *or* a body) is usually the better include over a fixed single-effect card, all else equal — flexibility adapts to whatever the game state actually needs.
- **The deck's critical engine has no protection.** If the whole plan routes through one card (a single draw engine, the commander itself), size the protection suite (redirect effects, hexproof/ward, indestructible) to that dependency, not to a flat baseline — see the Synergy checklist below and Step 2's protection quota.

## Step 4 — Power-level calibration

Tie recommendations back to the target bracket:

- **Brackets 1–2**: avoid Game Changers entirely, avoid fast infinite combos, favor thematic/synergy cards over raw efficiency, don't over-optimize the mana base with expensive fast lands unless the player wants that.
- **Bracket 3**: up to 3 Game Changers is normal; still generally avoid a deck whose whole plan is a 2-card infinite combo that ends the game with no interaction window.
- **Brackets 4–5**: optimize freely; consult live card data aggressively rather than relying on this file's general guidance, since the metagame at this level moves fast.

## Step 5 — Mana base sanity check

- Color pip count across the deck (not just card count) is the better guide to how many sources of each color you need — a deck with double-pip commitments in one color needs more sources of that color than an even 3-color split.
- Untapped, colorless-fixing lands (e.g. basics, fetches, duals) are worth more early; taplands are worth less in aggressive/low-curve decks.
- Note when a mana base is aspirational (needs cards the player doesn't own) vs. buildable today from the collection.
- If a specific color is starved despite an adequate pip-to-land ratio, prefer mana-fixing/doubling tools (in-color signets/talismans, filter lands, snow-typed basics paired with a snow payoff like Extraplanar Lens) over adding more lands of that color — there's limited room to fix a color problem by land count alone now that total lands are capped near 35 (Step 2).

## Synergy checklist (for evaluating a whole deck, not just one card)

- Is there a clear A-to-B-to-win path, or is it a pile of good-but-disconnected cards?
- Are there redundant effects for the deck's most important effect (e.g. multiple sac outlets in an aristocrats deck), or direct protection for it (redirect effects, hexproof/ward, indestructible) when redundancy isn't practical for a genuinely unique engine piece — so one removed or countered piece doesn't blank the plan?
- Does the deck have a plan against removal-heavy or counterspell-heavy tables, or is it fully reliant on one resolving spell?
- Is there a way to interact with what opponents are doing, or is this a fully solitaire deck (fine for casual, risky at higher brackets)?

## Table politics & play strategy

Commander is multiplayer and social by design (`commander-rules-reference.md`) — a deck's *deck-building* strengths and its *political* exposure aren't the same thing, and both matter for "how good is this deck" beyond raw power level.

- **Threat assessment**: the table generally targets whoever looks closest to winning or hardest to interact with, not whoever is "strongest" on paper. A deck that telegraphs its win early (a big flashy commander, an obvious combo piece in play) draws removal a low-key deck doesn't — factor this into card evaluation, not just power.
- **Deal-making**: temporary alliances ("don't attack me this turn, I'll not counter your next spell") are normal play, not a house rule — but they're non-binding; advise honoring the spirit of a deal you made unless the game state changed enough to justify breaking it, since a table that stops trusting a player's word changes how everyone plays against them going forward, across games.
- **Kingmaker/spite plays**: when a player is eliminated or clearly can't win, how they spend their last actions (helping decide who *does* win) is a real political lever, not just "well they're out anyway" — worth factoring into how a deck plays if it tends to make enemies along the way.
- **Archetype-specific exposure**:
  - *Group hug / politics* decks buy goodwill but can accidentally accelerate an opponent to a win — pair with some way to close out the game or interact late, not just generosity.
  - *Stax / resource denial* decks draw disproportionate table heat — worth discussing with the player up front whether that's an accepted deck for their table (see `house-rules.md` for their group's power-level agreement) rather than discovering it mid-game.
  - *Voltron* (single-creature beatdown) is an obvious, concentrated threat — protection density matters as much as raw power, since the whole plan dies to one removal spell resolving.
  - *Combo* decks read as either "impressive" or "unfun" depending on the table's stated bracket/expectations — calibrate against the target bracket from Step 1, not just whether the combo is legal.

For deeper reading beyond this framework-level summary, live-fetch Commander's Herald's politics/table-dynamics content per `community-resources.md` rather than expecting this file to cover it exhaustively.
