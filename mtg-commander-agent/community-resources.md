# Community Resources

> Upload this file as Project knowledge, alongside `commander-rules-reference.md` and `deckbuilding-framework.md`. It covers where to pull community-aggregated deck data, combo checks, and strategy opinion from — supplementing, not replacing, the framework in `deckbuilding-framework.md` and the live card-data role in `project-instructions.md`.

## EDHREC — synergy, popularity, average decklists

`https://edhrec.com/commanders/<commander-name-slug>` (lowercase, hyphenated — e.g. `atraxa-praetors-voice`). Fetch via Web Search/Fetch, as a normal public page — not the unofficial `json.edhrec.com` endpoints.

Surfaces, per commander: high-synergy cards, top cards by category (creatures, removal, ramp, etc.), and community average decklists. This is the best available signal for "what does the community actually play with this commander" — pull it once a commander is chosen, before free-associating card ideas, and use it as a filtered starting pool rather than a list to copy wholesale (it skews toward what's popular, not necessarily what fits a specific budget/collection/bracket — apply `deckbuilding-framework.md`'s evaluation questions to anything pulled from it).

## Commander Spellbook — combo detection

`https://commanderspellbook.com/search/?q=<query>` — a community-maintained, 30,000+ entry database specifically of Commander/EDH card combos (not general card data). Fetch via Web Search/Fetch as a normal public page.

Query syntax examples:
- `card="Exact Card Name"` — combos involving a specific card.
- `cards>2` / `cards<=5` — filter by combo piece count.
- Combine with `AND`/`OR` and `-` for negation.

Use this when: the player asks whether their deck has an unintended combo, wants combo suggestions for a commander/card, or is deliberately building around a combo. Treat results as community-submitted starting points, not verified rulings — cross-check the actual interaction against oracle text (via the Scryfall role) or `commander-rules-reference.md` before stating a combo works as described, especially anything with non-obvious timing/priority interactions.

## Strategy article sites (prefer these for open-ended strategy questions)

For broad questions that aren't a specific card/combo lookup — "how do I pilot a stax deck," "what's a good gameplan against a fast-mana table," "how has the Commander meta shifted" — use general web search, but prefer these over an unranked mix of forum posts/blogspam when sources disagree:

- **Commander's Herald** (`commandersherald.com`) — deck-building strategy, archetype breakdowns, politics/table dynamics.
- **EDHREC's own articles section** (`edhrec.com/articles`) — written by the same team behind the recommendation data above.
- **MTGGoldfish Commander** (`mtggoldfish.com/metagame/commander`) — metagame trends, skews more competitive/cEDH.

These are opinion and strategy content, not rules authorities — if a strategy article's claim about a rule or card interaction conflicts with `commander-rules-reference.md` or a live Scryfall ruling, the rules/ruling wins.
