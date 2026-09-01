# Maintenance Workflow (Claude Code, not for Project upload)

> For you, or a future Claude Code session working on this repo — not uploaded to the Project (same reason `manabox-drive-workflow.md` and this file's siblings aren't: they're about maintaining the package, not content the agent consults mid-conversation).

## The token budget that matters here

Project Knowledge files (`commander-rules-reference.md`, `deckbuilding-framework.md`, `community-resources.md`, `house-rules.md`, `game-log-workflow.md`) load **in full, every conversation**, on both web and mobile. Drive files (`raw/manabox_Collection.csv`, `price_log.csv`, `games_log.csv`, `wiki/*`) are fetched on demand instead. That asymmetry is the whole ops lever:

- When editing a Project Knowledge file, ask whether the addition earns its place in every single conversation, or whether it's better as a live-fetched role (already the pattern for card data, EDHREC, combos) or a wiki page (already the pattern for deck-specific accumulated knowledge).
- A Project Knowledge file that's grown long is a bigger problem than a Drive file that's grown long — Drive files are read selectively (a deck's Stats section, a filtered `games_log.csv` query), Knowledge files aren't.
- When in doubt, keep curated docs at the "framework/checklist" altitude they're already written at (see `deckbuilding-framework.md`'s style) rather than exhaustive reference — exhaustive belongs in a live lookup.

## Reconciling `wiki/knowledge-gaps.md`

`wiki-structure.md` already defines the loop: the agent logs stale/missing curated knowledge there instead of silently working around it. Periodically (whenever it's grown a handful of entries, or before a session where accuracy matters):

1. Pull the current `wiki/knowledge-gaps.md` content from Drive (ask the user to paste it, or read it directly if you have Drive access in this session).
2. For each entry, decide which curated doc it belongs in and draft the change.
3. Edit the file(s) in this repo, commit.
4. Tell the user which files need re-upload per `setup-guide.md` (Claude Code can't upload to a Claude Project directly).
5. Note in `wiki/log.md` (via the user, next Project session) that the gap was reconciled — don't delete the original `knowledge-gaps.md` entry; append a resolved follow-up line instead, per that file's append-only discipline.

## If `games_log.csv`, `price_log.csv`, or `card-cache.md` get large

Per `game-log-workflow.md`, deck win-rate stats live primarily in each deck's `wiki/<topic>.md` Stats section, not recomputed from the full log each time — so `games_log.csv` growth mostly doesn't cost anything mid-conversation. If it ever does become a real drag (very large multi-season history, cross-deck queries getting slow to reason about), the documented move is archiving older rows into a dated `games_log_archive_<year>.csv` and keeping the live file recent — same shape as `setup-guide.md`'s existing "if you outgrow this" note for pricing, not something to solve preemptively.

`price_log.csv` grows under the same pressure, arguably faster — the weekly scheduled task (`scheduled-tasks.md`) appends a sampled batch per in-progress deck every week. Same fix if it ever becomes a real drag: archive older rows into a dated `price_log_archive_<year>.csv`, keep the live file recent.

`wiki/card-cache.md` is different — a single shared file, read in full whenever a card lookup checks it, so its cost is felt immediately rather than only on cross-cutting queries. If it grows large enough to matter, split it (alphabetically, or by color identity) into multiple files with an index note in `wiki-structure.md` pointing to the split — not something to do preemptively while it's still a reasonable size.

## Known limitation: concurrent writes

Every CSV/wiki write is read-modify-write with no locking — Drive doesn't even enforce unique filenames in a folder, so two sessions writing around the same time (a phone session and a desktop session, or a scheduled task overlapping a manual price/game log) can race and, worst case, produce duplicate-named files instead of one clean append. Not worth engineering a fix for a single-user setup — but avoid manually logging prices or games while a scheduled task is running, and if you ever notice two files with the same name in `wiki/`, treat it as this known failure mode: read both, keep the one that's the superset, trash the other. `scheduled-tasks.md`'s weekly wiki hygiene check exists specifically to catch this automatically.

## Periodic doc/architecture audit

Unlike the wiki hygiene check above (which the live Project agent runs on a schedule against Drive), a review of the curated doc package itself — cross-reference bugs between files, whether the required-Knowledge-upload list is still complete and accurate, token-budget shape, archived-vs-live growth policy, DRY violations between files — needs repo write access and can only be done in a Claude Code session, not a Project-scheduled task (Project Knowledge from GitHub is read-only from chat — see `README.md`'s "why GitHub and Google Drive split the way they do"). There's no mechanism to trigger this automatically; do it periodically (a natural cadence: every few months, or whenever the package feels like it's drifted — new files added without updating cross-references, a workflow doc that's grown past "framework altitude," etc.). The 2026-09-01 session's review (repo git log) is the template to follow: read every file in the package, check cross-references actually resolve, check every file referenced by a required Knowledge file is itself accessible, verify Drive filenames matched across docs, and check growth/archival policy exists for anything that accumulates over time.

## General shape of a maintenance pass

1. Read the relevant file(s) in this repo before editing — don't assume from memory what a doc currently says.
2. Keep each file's single responsibility intact; if an addition doesn't fit any existing file's stated purpose, that's a signal to add a new small file (as this session did for `house-rules.md` and `game-log-workflow.md`) rather than overloading an existing one.
3. Update cross-references (`README.md`'s file list, `project-instructions.md`'s role list, `setup-guide.md`'s upload/verification steps) in the same pass — a new file that isn't wired into those three is invisible to the actual running Project.
4. Commit with a message describing the *why*, not just the file list — future-you (or a future session) reading `git log` is the audit trail for decisions that aren't captured in `wiki/log.md`.
