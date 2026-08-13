# MTG Commander Agent

A Claude.ai Project setup for a personal Magic: The Gathering Commander (EDH) copilot: rules Q&A, card exploration, deck-building help that's aware of your actual collection (via ManaBox), and card value tracking (owned and unowned) — built entirely on a Claude Pro subscription, no additional API or subscription cost, usable from the iPhone Claude app.

This is **Level 1** of a 3-level design (pure Project, no hosted infrastructure) — see the plan history for the full comparison. If full-collection price lookups ever become a real bottleneck, `setup-guide.md`'s last section points at the documented upgrade path.

## How it fits together

- **`project-instructions.md`** — paste into the Project's custom instructions. The orchestrator: defines the agent's persona and which file/data source to consult for which kind of question.
- **`commander-rules-reference.md`** and **`deckbuilding-framework.md`** — upload as Project knowledge. Static reference content.
- **`manabox-drive-workflow.md`** — how your ManaBox collection gets exported and kept current in Google Drive.
- **`price-tracking-workflow.md`** — how the agent looks up and (on request) logs card values.
- **`wiki-structure.md`** — the compounding-memory layer: how the agent accumulates deck progress and insights across sessions in Google Drive instead of starting fresh every chat, based on Karpathy's "LLM wiki" pattern (raw sources → agent-owned wiki → index/log).
- **`setup-guide.md`** — do this once, start to finish, including a verification checklist.

## Why this shape

Full Magic card data (30k+ cards) and the full comprehensive rules are both far too large to fit in a Claude Project's knowledge base, so card/rules lookups beyond the curated basics happen live against Scryfall's free public API instead of being embedded. ManaBox has no free official API, so collection sync is a manual CSV export/upload rather than a live integration. Both constraints are explained in more detail, with sources, in the plan this package was built from.

Each doc has a single responsibility and the instructions reference *roles* ("authoritative card data", "the collection") rather than hardcoding specific mechanisms — so any one piece (e.g. swapping the Scryfall web-fetch role for a dedicated MCP connector later) can change without rewriting the rest.
