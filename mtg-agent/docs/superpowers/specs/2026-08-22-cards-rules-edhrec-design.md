# mtg-agent — Sub-project 1: cards + rules (+ EDHREC synergy cache) — Design

## Context

`mtg-commander-agent/` (sibling folder) is a working **Level 1** system: a Claude.ai Project, pure markdown + Google Drive, no hosted infrastructure, zero cost beyond a Claude Pro subscription. It stays exactly as-is and remains the daily driver on phone/web while this new system is built alongside it.

`mtg-agent/` is a **Level 3** system: a real monorepo with a database, an MCP server, data ingestion pipelines, and Claude Code dev subagents — built incrementally, sub-project by sub-project, only superseding Level 1 once it's proven reliable. The target top-level shape (user-specified):

```
mtg-agent/
├── CLAUDE.md
├── .claude/{agents/, skills/}
├── apps/mcp-server/
├── modules/{cards, rules, collection, prices, decks, games, statistics, commander, metagame}/
├── ingestion/{scryfall, mtgjson, wizards, topdeck, spellbook}/
├── db/{schema/, migrations/}
├── evaluations/{rulings, legality, deckbuilding, statistics}/
└── tests/
```

This spec covers **sub-project 1 only**: `cards` + `rules`, plus an EDHREC synergy cache folded in per this session's decision. Motivation (all apply): Level 1's per-card Scryfall API calls can't do reliable batch lookups; rules/legality checks are currently LLM reasoning instead of deterministic code; stats/collection analytics need a real query layer eventually. This sub-project is the foundation the rest (`decks`, `collection`, `games`, `statistics`, `metagame`) build on.

## Goals

- Local, queryable card database (name, oracle text, mana cost, color identity, per-format legality, prices), refreshed from Scryfall on a schedule the user controls — no per-card live API calls during normal use.
- Deterministic deck-legality checking (singleton, 100-card count, color-identity subset, per-format legal/banned) as real code, not LLM inference.
- A minimal, cached EDHREC synergy lookup (per-commander recommended cards, grouped by category, with synergy score and deck-inclusion count), since the user wants this available now rather than deferred to a later sub-project. Average/sample decklists are **not** included — see "EDHREC response shape" below.
- An MCP server exposing all of the above as tools, runnable locally via Claude Code/Desktop now; hosting for mobile/Project access is an explicit later decision (not solved here).

## Non-goals (deferred to later sub-projects)

- `collection`, `prices` (beyond Scryfall's bundled price fields), `decks`, `games`, `statistics`, `commander` modules.
- `mtgjson`, `wizards`, `topdeck`, `spellbook` ingestion sources.
- Hosting/deployment of `apps/mcp-server` or the DB anywhere reachable from mobile.
- House rules / custom ban list integration into the DB (stays in Level 1's `house-rules.md` for now).
- A full EDHREC bulk crawl (explicitly rejected this session — see Risks).

## Data source decisions

**Scryfall — bulk data, not per-card API.** Scryfall publishes downloadable bulk JSON dumps (`oracle_cards`, `rulings`), refreshed ~daily, explicitly intended for exactly this kind of local-dataset use (as opposed to sustained per-card API traffic). Ingesting these once (and re-running on demand) turns every card/legality/price/ruling query into an instant local SQL read.

**EDHREC — on-demand, cached, hand-written client — not a bulk crawl, not an external dependency.** Researched this session:
- No official API or bulk export exists; `json.edhrec.com` is undocumented and can change without notice (confirmed via EDHREC's own FAQ, which only documents that *they* source from Archidekt/Moxfield/Scryfall — nothing about third-party API terms).
- `edhrec.com/robots.txt` doesn't blanket-disallow crawling, but there's no stated bulk-use policy — community norm (per every tool found) is "interactive-use request rates," not bulk crawling.
- Three existing options were evaluated and rejected as dependencies: `edhrec-mcp` (0 stars, 1 fork, no license/version info), `pyedhrec` (PyPI, last release Feb 2024 — stale), `mightstone` (MIT, actively committed, broader scope including MTGJSON/rules — worth revisiting for *later* sub-projects, but 0 stars/8 open issues is too thin to build sub-project 1 on).
- **Decision**: write a small first-party client (a few functions, not a library) that fetches one commander's data at a time, only when asked (deck-building/analysis for that commander), and caches the result in the DB with a fetch timestamp. Refetch when stale (EDHREC states data is "reflected within a few days" of underlying changes, so a ~7-day TTL is a reasonable default, tunable). This is deliberately the low-risk option: minimal request volume, no dependency on an unmaintained external project, and a small enough surface that if the endpoint shape changes, it's a quick fix rather than a blocked wait.

**EDHREC response shape (verified this session, not guessed)**: `GET https://json.edhrec.com/pages/commanders/<slug>.json` (slug = lowercase, hyphenated commander name, same slug format as the `edhrec.com/commanders/<slug>` page Level 1 already uses) returns a JSON object whose `container.json_dict.cardlists` key is a list of category panels — e.g. `{"header": "High Synergy Cards", "tag": "highsynergycards", "cardviews": [{"name": "...", "synergy": 0.27, "num_decks": 29032}, ...]}` — one such panel per category (creatures, instants, high-synergy, etc.). **There is no average/sample decklist in this response**, and a guessed `average-decks.json` path returned HTTP 403 — average decklists are out of scope for this sub-project rather than built on an unverified endpoint. If they're wanted later, that's a small follow-up once the correct endpoint (if any) is found.

## Data model (`db/schema`)

SQLite for local dev (zero ops, no hosting decision required yet); **SQLAlchemy Core** (not full ORM — this domain is bulk-upsert + read-heavy, not transactional business objects) for the query layer; **Alembic** for migrations. Both SQLAlchemy Core and Alembic support Postgres with the same code, so the eventual SQLite→Postgres move (once a host is chosen) doesn't require a rewrite. The `.sqlite` file itself is gitignored — only schema/migrations are committed; the file is generated by running ingestion.

Tables:
- **`cards`** — `scryfall_id` (PK), `oracle_id`, `name`, `mana_cost`, `cmc`, `colors`, `color_identity`, `type_line`, `oracle_text`, `power`, `toughness`, `loyalty`, `set_code`, `collector_number`, `rarity`, `price_usd`, `price_usd_foil`, `price_eur`, `price_tix`, `raw_json` (full Scryfall object, catch-all for anything not modeled above).
- **`card_legalities`** — `card_id` (FK), `format`, `status` (`legal`/`not_legal`/`banned`/`restricted`) — normalized so "commander-legal, not banned" is a real filter, not a JSON scan.
- **`rulings`** — `oracle_id` (FK), `published_at`, `source`, `comment`.
- **`edhrec_synergy_cache`** — `commander_slug` (PK), `cardlists_json` (the full list of category panels — header/tag/cardviews — as returned), `fetched_at`. Read-through cache: a lookup checks this table first; only calls EDHREC if missing or `fetched_at` older than the TTL.
- **`ingestion_runs`** — `source` (`scryfall`/`edhrec`), `started_at`, `finished_at`, `record_count`, `status` — observability for "when did I last refresh this."

## Ingestion (`ingestion/`)

- **`ingestion/scryfall/`**: calls Scryfall's `/bulk-data` endpoint for current download URLs, pulls `oracle_cards` + `rulings`, upserts into `cards`/`card_legalities`/`rulings`, writes an `ingestion_runs` row. Run manually via a CLI entry point (`python -m ingestion.scryfall.run`); no scheduling infra yet — that's a hosting-dependent decision, deferred.
- **`ingestion/edhrec/`**: not a batch job — a `get_commander_synergy(commander_name)` function used by `modules/metagame`, checking `edhrec_synergy_cache` first, fetching+parsing+caching on a miss/stale hit. Failure mode: if the fetch fails or the response shape doesn't match expectations, return/raise clearly rather than silently returning stale or fabricated data — same "say exactly that, don't guess" philosophy as Level 1.

## Modules

- **`modules/cards`**: query functions — get by name/id, search/filter by color/type/text/format-legality, batch price lookup.
- **`modules/rules`**: deterministic checks — singleton, 100-card count, color-identity subset (commander → deck), per-format legality via `card_legalities`. This is the actual judge logic: real code, not an LLM re-deriving it each time.
- **`modules/metagame`**: wraps `ingestion/edhrec/`'s cached lookup — "what pairs with this commander," synergy scores and deck-inclusion counts by category.

## `apps/mcp-server`

Exposes: `get_card`, `search_cards`, `get_rulings`, `check_deck_legality`, `get_commander_synergy`. Runs locally via Claude Code/Desktop config for development and testing. **Not yet reachable from a Claude Project or mobile** — that requires a remote/hosted deployment, an explicit later decision per this session's "not yet, keep it open" answer. Ingestion stays a manual/CLI operation, not an MCP tool (it's maintenance, not a per-conversation query).

## Evaluations & tests

- **`evaluations/legality`**: fixed set of known-good/known-bad decklists (wrong color identity, non-singleton, banned card, wrong card count) asserting `check_deck_legality` gets each one right. This is the one evaluation category scoped for this sub-project — `rulings`/`deckbuilding`/`statistics` wait for their own modules.
- **`tests/`**: ingestion parsing (fixture Scryfall JSON → expected DB rows), `modules/cards` and `modules/rules` query/logic tests, EDHREC cache hit/miss/stale behavior against a mocked response, MCP tool handlers.

## Dev subagents (`.claude/agents/`)

Scaffolded now: **`data-engineer.md`** (owns `ingestion/scryfall`, `ingestion/edhrec`, `db/schema`, `db/migrations`), **`rules-engineer.md`** (owns `modules/rules`, `evaluations/legality`), **`test-reviewer.md`** (reviews test coverage across both). **`deck-engineer.md`** is listed in the target tree but out of scope until the `decks` sub-project — not created yet.

## `CLAUDE.md`

Top-level instructions for Claude Code sessions working in this repo: module boundaries above, the stack choices (Python, SQLite→Postgres, SQLAlchemy Core, Alembic), a pointer to this spec and to `mtg-commander-agent/` explaining the Level 1/Level 3 relationship (Level 1 is the live daily driver; this repo is built incrementally alongside it), and the token/reliability philosophy carried over from Level 1 (say plainly when data is missing/stale rather than guessing).

## Risks

- **EDHREC endpoint is undocumented** and could change shape or access policy without notice — mitigated by minimal request volume (on-demand, cached, not bulk) and a small first-party client instead of an external dependency; if it breaks, the fallback is the same live-page-fetch approach Level 1 already uses, manually, until the client is fixed.
- **SQLite → Postgres migration** is deferred but not blocked — SQLAlchemy Core + Alembic were chosen specifically to keep that path open; no SQLite-only features are used in the schema.
- **Scope crept once already this session** (EDHREC folded in after the initial cards+rules-only proposal) — worth actively resisting further additions (e.g. `mtgjson`, `wizards` rules ingestion) until this sub-project is actually working end-to-end.

## Verification

1. Run `ingestion/scryfall` against real Scryfall bulk data; confirm `cards`/`card_legalities`/`rulings` populate correctly and `ingestion_runs` logs the run.
2. Run `evaluations/legality`'s fixed decklist set against `check_deck_legality`; all expected verdicts must pass.
3. Exercise `get_commander_synergy` for a commander with no cache entry (miss → fetch → cache) and again immediately after (hit, no re-fetch) — confirm via `edhrec_synergy_cache.fetched_at`.
4. Start `apps/mcp-server` locally via Claude Code/Desktop and manually exercise all five tools from a chat.
5. `tests/` suite passes end-to-end (`pytest` or equivalent).
