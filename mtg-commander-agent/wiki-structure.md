# Wiki Structure

This defines the compounding-memory layer of the agent, based on the "LLM wiki" pattern: raw sources are immutable; the wiki is a set of markdown pages the agent owns and updates over time; an index and a log keep the wiki navigable and auditable. All of it lives in the Google Drive folder `MTG Commander Agent/`, alongside `raw/` and `price_log.csv` (see `manabox-drive-workflow.md` and `price-tracking-workflow.md`).

## Layout

```
MTG Commander Agent/
  raw/
    manabox_collection.csv      (immutable — see manabox-drive-workflow.md)
  wiki/
    index.md                    (catalog of every wiki page)
    log.md                      (append-only chronological journal)
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

## `wiki/<topic>.md` pages

Created on demand, not from a fixed template — but each should generally cover: what it is, current state (e.g. a decklist, or a conclusion), why it's the way it is, and open questions. Keep one topic per file (a deck, a specific card evaluation, a standing insight about the collection) rather than merging unrelated topics — this keeps pages independently readable and citable from `index.md`.

## Maintenance rules

- **Read before you write.** Before starting substantive work on something that might already have a page, check `index.md` and read the matching page rather than starting cold.
- **Write only when it's worth remembering.** Don't create a page for a one-off rules lookup or a question with no lasting state. Do create/update one for deck progress, a non-obvious card cut/include decision, or a standing insight about the collection.
- **Keep `index.md` and `log.md` synchronized with reality** — every page write is followed by an `index.md` touch-up (if new or materially changed) and a `log.md` line.
- **Never rewrite `log.md` history** — only append.
- **Occasional lint pass**: if asked to "clean up the wiki," check for pages `index.md` references but that no longer exist (or vice versa), and flag anything that looks stale (e.g. a deck page that hasn't been touched in months next to a much-changed collection).
