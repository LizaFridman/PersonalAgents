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

## If `games_log.csv` gets large

Per `game-log-workflow.md`, deck win-rate stats live primarily in each deck's `wiki/<topic>.md` Stats section, not recomputed from the full log each time — so log growth mostly doesn't cost anything mid-conversation. If it ever does become a real drag (very large multi-season history, cross-deck queries getting slow to reason about), the documented move is archiving older rows into a dated `games_log_archive_<year>.csv` and keeping the live file recent — same shape as `setup-guide.md`'s existing "if you outgrow this" note for pricing, not something to solve preemptively.

## General shape of a maintenance pass

1. Read the relevant file(s) in this repo before editing — don't assume from memory what a doc currently says.
2. Keep each file's single responsibility intact; if an addition doesn't fit any existing file's stated purpose, that's a signal to add a new small file (as this session did for `house-rules.md` and `game-log-workflow.md`) rather than overloading an existing one.
3. Update cross-references (`README.md`'s file list, `project-instructions.md`'s role list, `setup-guide.md`'s upload/verification steps) in the same pass — a new file that isn't wired into those three is invisible to the actual running Project.
4. Commit with a message describing the *why*, not just the file list — future-you (or a future session) reading `git log` is the audit trail for decisions that aren't captured in `wiki/log.md`.
