# Wiki Structure

This defines the compounding-memory layer of the agent, based on the "LLM wiki" pattern: raw sources are immutable; the wiki is a set of markdown pages the agent owns and updates over time; an index and a log keep the wiki navigable and auditable. All of it lives in the Google Drive folder `MTG Commander Agent/`, alongside `raw/` and `price_log.csv` (see `manabox-drive-workflow.md` and `price-tracking-workflow.md`).

## Layout

```
MTG Commander Agent/
  raw/
    manabox_Collection.csv      (immutable — see manabox-drive-workflow.md)
  wiki/
    index.md                    (catalog of every wiki page)
    log.md                      (append-only chronological journal)
    knowledge-gaps.md           (append-only backlog of stale/missing curated knowledge)
    <topic>.md                  (one file per deck / evaluation / insight)
  price_log.csv                 (structured — see price-tracking-workflow.md)
```

## `wiki/index.md`

A flat catalog, one entry per wiki page, newest-relevant first. Each entry is a single line or two:

```markdown
# Wiki Index

- **mono-black-aristocrats.md** — Sac-value Commander deck built around [Commander], budget ~$150. In progress: finalizing mana base.
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

## `wiki/<topic>.md` pages

Created on demand, not from a fixed template — but each should generally cover: what it is, current state (e.g. a decklist, or a conclusion), why it's the way it is, and open questions. Keep one topic per file (a deck, a specific card evaluation, a standing insight about the collection) rather than merging unrelated topics — this keeps pages independently readable and citable from `index.md`.

## Maintenance rules

- **Read before you write.** Before starting substantive work on something that might already have a page, check `index.md` and read the matching page rather than starting cold.
- **Write only when it's worth remembering.** Don't create a page for a one-off rules lookup or a question with no lasting state. Do create/update one for deck progress, a non-obvious card cut/include decision, or a standing insight about the collection.
- **Keep `index.md` and `log.md` synchronized with reality** — every page write is followed by an `index.md` touch-up (if new or materially changed) and a `log.md` line.
- **Never rewrite `log.md` or `knowledge-gaps.md` history** — only append.
- **Occasional lint pass**: if asked to "clean up the wiki," check for pages `index.md` references but that no longer exist (or vice versa), and flag anything that looks stale (e.g. a deck page that hasn't been touched in months next to a much-changed collection).
