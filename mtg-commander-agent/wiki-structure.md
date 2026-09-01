# Wiki Structure

This defines the compounding-memory layer of the agent, based on the "LLM wiki" pattern: raw sources are immutable; the wiki is a set of markdown pages the agent owns and updates over time; an index and a log keep the wiki navigable and auditable. All of it lives in the Google Drive folder `MTG Commander Agent/`, alongside `raw/`, `price_log.csv`, and `games_log.csv` (see `manabox-drive-workflow.md`, `price-tracking-workflow.md`, and `game-log-workflow.md`).

## Layout

```
MTG Commander Agent/
  raw/
    manabox_Collection.csv      (immutable — see manabox-drive-workflow.md)
  wiki/
    index.md                    (catalog of every wiki page)
    log.md                      (append-only chronological journal)
    knowledge-gaps.md           (append-only backlog of stale/missing curated knowledge)
    card-cache.md               (verified static card facts — not prices, not legality, not synergy)
    <topic>.md                  (one file per deck / evaluation / insight; deck pages carry a Stats section — see game-log-workflow.md)
  price_log.csv                 (structured — see price-tracking-workflow.md)
  games_log.csv                 (structured — see game-log-workflow.md)
```

## `wiki/index.md`

A flat catalog, one entry per wiki page, newest-relevant first. Each entry is a single line or two:

```markdown
# Wiki Index

- **mono-black-aristocrats.md** — Sac-value Commander deck built around [Commander], budget ~$150. In progress: finalizing mana base.

  (The exact phrase **"In progress"** matters beyond readability: it's also the marker the weekly scheduled refresh task uses to pick which decks to touch — see `scheduled-tasks.md`. Keep using it precisely while a deck is active, and drop it once a deck is finished/shelved.)
- **selesnya-counters.md** — +1/+1 counters deck idea, shelved pending better payoffs.
- **card-eval-smothering-tithe.md** — Notes on why Smothering Tithe was cut from the aristocrats build (doesn't fit the sac subtheme).
```

Update this file whenever a page is created, meaningfully changed, or retired. It should always be a true reflection of what's in `wiki/`.

## `wiki/log.md`

Append-only. Never edit or delete past entries — only add new ones at the bottom, newest last. One line per event:

```markdown
# Log

- 2026-08-13: Created mono-black-aristocrats.md; drafted initial 100-card list from collection + owned staples.
- 2026-08-20: Logged price snapshot (14 cards) to price_log.csv.
- 2026-09-02: Updated mono-black-aristocrats.md — swapped in Blood Artist, cut Zulaport Cutthroat (redundant).
```

This is the audit trail — if a wiki page's current state seems surprising, `log.md` explains how it got there.

## `wiki/knowledge-gaps.md` — how the agent "updates itself"

Project Knowledge files (`commander-rules-reference.md`, `deckbuilding-framework.md`, `community-resources.md`) are **read-only** from inside a Project chat — the agent cannot edit them directly, whether they were uploaded manually or via the GitHub connector (see `README.md`'s "why GitHub and Google Drive split the way they do"). So the agent can't literally keep those files current on its own. What it *can* do is notice when they're wrong or incomplete and externalize that into the one place it can write: the wiki.

Append-only, same discipline as `log.md`, but scoped specifically to gaps in the curated docs:

```markdown
# Knowledge Gaps

- 2026-09-10: commander-rules-reference.md's Bracket section doesn't mention [new mechanic/errata]; confirmed via live Scryfall rulings lookup.
- 2026-09-10: community-resources.md doesn't list [new community site] — came up twice this week when asked for strategy articles.
```

Trigger this whenever a live lookup (Scryfall, EDHREC, Commander Spellbook, or general web search) reveals something that contradicts or is missing from the curated docs — say so in the response, and log an entry here rather than letting the discrepancy pass silently.

**Reconciling the backlog** (a periodic, user-initiated action — the agent can't do this part unattended): when asked to "review knowledge gaps" or similar, read `wiki/knowledge-gaps.md` and summarize what should change in which curated doc, specific enough that you (or a future Claude Code session working on the `mtg-commander-agent/` repo files) can fold it back in and re-upload/re-sync. Once reconciled, note it as resolved with a dated follow-up line rather than deleting the original entry — the log stays append-only.

## `wiki/card-cache.md` — verified static card facts

A single shared cache of card facts that don't change over time (or only change via erratum, not drift): color identity, mana cost, card type, oracle text. Written only after a live Scryfall verification in the current session — never from recollection — with the verification date recorded. A card listed here can be trusted without re-fetching; a card not listed must be looked up live. See `project-instructions.md`'s Role 4 for exactly when to check this before fetching, and when to write back to it after a live lookup.

**Explicitly excluded — always fetch live, never cache:** prices, banned/legality status, and EDHREC synergy figures. These change over time independent of the card's printed text; caching them would go stale silently.

Format: one `### Card Name` entry per card, grouped under a `## Verified <date>` heading, listing color identity/mana cost/type/oracle text and any notable caveat worth remembering (e.g., why a card was rejected from a build, if that came up during verification):

```markdown
## Verified 2026-08-14

### Ragavan, Nimble Pilferer
- Color identity: R | Cost: {R} | Type: Legendary Creature — Monkey Pirate
- Dash {1}{R} (optional). On combat damage to a player: create a Treasure, exile their top card, may cast it until end of turn.
```

## `wiki/<topic>.md` pages

Created on demand, not from a fixed template — but each should generally cover: what it is, current state (e.g. a decklist, or a conclusion), why it's the way it is, and open questions. Keep one topic per file (a deck, a specific card evaluation, a standing insight about the collection) rather than merging unrelated topics — this keeps pages independently readable and citable from `index.md`.

**One file per deck, forever — never a new file per revision.** When a deck gets a substantial revision, edit the existing `<topic>.md` in place; don't create `<topic>-v2.md`, `<topic>-v3.md`, etc. If a changelog is valuable, keep it as a `## Revision History` (or `## Changes from vN`) section growing *inside* that same file — a table of out/in cards with reasons works well, same shape as any other decklist table on the page. A new file per revision is exactly what produces stale `index.md` pointers (an old version still referenced, or referenced instead of the current one) and is a second source of the duplicate-file failure mode below, on top of the write-protocol issue — every extra file is another chance for it to strike.

**A deck page's decklist carries a per-card justification and synergy note, not a bare card-name list — this is mandatory, not a nice-to-have.** For each card, the "Why included" column states, concretely: what it does for the game plan, and whichever of mana-cost/value rate or opportunity cost against a real alternative for the slot actually drove the pick (per `deckbuilding-framework.md`'s Step 3) — not a vague pointer like "fits the theme" or "good card." Ownership and price are never pick-drivers (Step 3, question 5); if noted at all they go in a separate annotation or a dedicated Ownership section, not the "Why included" reason. The "Synergizes with" column names the specific other card(s) or mechanism in *this* deck, not the archetype in the abstract. A table works well:

```markdown
| Card | Why included | Synergizes with |
|---|---|---|
| Blood Artist | Cheap, always-on drain on any creature death | Zulaport Cutthroat (redundant drain trigger), any sac outlet in the deck |
```

This is what lets a later session actually reconsider a specific include/cut with real reasoning, instead of re-deriving it from scratch or re-fetching the same evaluation.

**Deck pages additionally carry a `## Stats` section** (record, win rate, breakdown by pod size, notable matchup patterns) once at least one game has been logged for that deck — see `game-log-workflow.md` for the exact format and refresh discipline. This is a standing summary refreshed incrementally as games are logged, not something recomputed from `games_log.csv` on every read.

## Maintenance rules

- **Read before you write.** Before starting substantive work on something that might already have a page, check `index.md` and read the matching page rather than starting cold.
- **Never blind-create.** Google Drive does not enforce unique filenames in a folder — two files can share the exact same name, unlike a normal filesystem. Before writing *any* wiki file (a meta-file like `index.md`/`log.md`/`knowledge-gaps.md`/`card-cache.md`, or an existing `wiki/<topic>.md` page), search Drive for that exact filename first and capture its file ID. If found, write using that file's ID — an update-in-place call, never a fresh "create." Only issue a create call when the search genuinely returns zero results. This is the single highest-leverage habit against the duplicate-file failure mode (`maintenance-workflow.md`'s "Known limitation: concurrent writes") — most real occurrences of it trace back to a write that skipped this check, not to genuine concurrent-session races.
- **Write only when it's worth remembering.** Don't create a page for a one-off rules lookup or a question with no lasting state. Do create/update one for deck progress, a non-obvious card cut/include decision, or a standing insight about the collection.
- **Keep `index.md` and `log.md` synchronized with reality** — every page write is followed by an `index.md` touch-up (if new or materially changed) and a `log.md` line.
- **Never rewrite `log.md` or `knowledge-gaps.md` history** — only append.
- **Occasional lint pass**: if asked to "clean up the wiki," check for pages `index.md` references but that no longer exist (or vice versa), and flag anything that looks stale (e.g. a deck page that hasn't been touched in months next to a much-changed collection).
