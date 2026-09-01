# Setup Guide

One-time setup to turn this file package into a working Claude.ai Project, plus your first refresh cycle. Exact menu wording may drift slightly as claude.ai's UI evolves — look for the nearest equivalent if something's been renamed.

## 1. Create the Project

1. On claude.ai (web or desktop), go to **Projects → New project**.
2. Name it **"The Archon of the Ninety-Nine"**.

## 2. Add the custom instructions

1. Open the Project's settings and find **Instructions** (sometimes called "custom instructions" or "how should Claude approach this project").
2. Paste the entire contents of `project-instructions.md` in as-is.

## 3. Add knowledge files (GitHub connector)

The GitHub connector is the primary path: connect the `PersonalAgents` repo itself so the Project's knowledge stays tied to the source files instead of static copies you have to remember to refresh.

1. Push this repo to GitHub if it isn't already (public or private both work — a private repo just needs you to grant the Claude GitHub App access to it during connector setup).
2. In the Project's **Knowledge** section, click **+ → GitHub**, authenticate if prompted, and select the `PersonalAgents` repository.
3. Use the file browser to select only:
   - `mtg-commander-agent/commander-rules-reference.md`
   - `mtg-commander-agent/deckbuilding-framework.md`
   - `mtg-commander-agent/in-game-play.md`
   - `mtg-commander-agent/community-resources.md`
   - `mtg-commander-agent/house-rules.md` — fill in your playgroup's actual custom ban list and house rules first (see that file); an empty placeholder is fine too, just less useful until you do.
   - `mtg-commander-agent/game-log-workflow.md`
   - `mtg-commander-agent/price-tracking-workflow.md` — `game-log-workflow.md` explicitly points to this file for its own append mechanics ("same read-modify-write pattern as `price_log.csv`"), so it needs to actually be readable mid-conversation, not just referenced.

   Don't select the whole repo, and don't select the remaining workflow docs (`manabox-drive-workflow.md`, `wiki-structure.md`, `maintenance-workflow.md`) or `README.md` — those are for you/future-you to maintain this package, not for Claude to consult mid-conversation. (Optional: select them too if you want the agent able to explain its own setup on request — harmless either way, just not required.)
4. **Not live sync**: after editing a file in the repo, the Project keeps using the old version until you click **Sync now** in the Project's Knowledge section — there's no sync control in the iPhone app, so do this from web/desktop before a mobile session where the update matters. (This is also where `wiki/knowledge-gaps.md` entries eventually feed back in — see `wiki-structure.md` and `maintenance-workflow.md`.)

### Alternative: manual upload

If the GitHub connector isn't available on your account (there's a reported bug blocking it on some individual Pro accounts — see [anthropics/claude-code #78761](https://github.com/anthropics/claude-code/issues/78761) — though it may not affect yours), upload the same seven files directly in the Project's **Knowledge** section instead. Static uploads have no sync: re-upload (replacing the old file) whenever you edit any of the seven in the repo.

## 4. Enable Web Search

Confirm the Project (or your account-level setting) has **Web Search** enabled — it's how the agent fetches live Scryfall data. This is usually on by default on Pro; check Settings → Features/Tools if unsure.

## 5. Connect Google Drive and create the folder structure

1. If you haven't already, connect the **Google Drive** integration to your Claude account (Settings → Connectors, or add it directly from within the Project). Since you're already using Drive/Gmail/Calendar integrations elsewhere, this is likely just a matter of enabling it for this Project specifically.
2. In Google Drive, create this structure (empty files are fine to start):

```
MTG Commander Agent/
  raw/
    manabox_Collection.csv      ← populated in step 6
  wiki/
    index.md                    ← starter content below
    log.md                      ← starter content below
    knowledge-gaps.md           ← starter content below
    card-cache.md                ← starter content below
  price_log.csv                 ← header row from price-tracking-workflow.md
  games_log.csv                 ← header row from game-log-workflow.md
```

3. Starter `wiki/index.md`:
   ```markdown
   # Wiki Index

   (empty — pages will be added here as decks and insights accumulate)
   ```
4. Starter `wiki/log.md`:
   ```markdown
   # Log

   - 2026-08-13: Project set up.
   ```
   (use today's actual date)
5. Starter `wiki/knowledge-gaps.md`:
   ```markdown
   # Knowledge Gaps

   (empty — the agent logs stale/missing curated knowledge here; see wiki-structure.md)
   ```
6. Starter `wiki/card-cache.md` (see `wiki-structure.md` for the full entry format — this starter carries one worked example so the format is self-documenting the first time the agent writes a real entry):
   ```markdown
   # Card Cache

   Verified card facts only (color identity, mana cost, type, oracle text) — never prices, legality, or EDHREC synergy; those are always fetched live. Written only after a live verification in-session.

   ## Verified 2026-08-13

   ### Sol Ring
   - Color identity: colorless | Cost: {1} | Type: Artifact
   - {T}: Add {C}{C}.
   ```
7. Starter `price_log.csv`:
   ```csv
   date,card_name,set_code,finish,quantity_owned,unit_price_usd,total_value_usd,source
   ```
8. Starter `games_log.csv`:
   ```csv
   date,deck,commander,pod_size,opponents_commanders,result,notes
   ```
9. Make sure the Drive integration/connector has access to this folder (some connector setups scope to "all Drive" and some ask you to pick folders — grant access to at least this one).

## 6. First ManaBox export

Follow `manabox-drive-workflow.md`: export your collection CSV from the ManaBox app and upload it to `MTG Commander Agent/raw/manabox_Collection.csv`.

## 7. Verify on iPhone

1. Open the Claude iOS app, sign in with the same account.
2. Open **Projects** and confirm your new Project appears with its instructions and knowledge intact.
3. Send a test message (see checks below) and confirm it can reach both the knowledge files and your Drive folder from mobile.

## Verification checklist

Run these from the iPhone app once setup is done:

1. Ask a rules question (e.g. "how much commander damage kills a player?") — should answer from `commander-rules-reference.md` without a web search.
2. Ask about a recent card's legality/oracle text — should fetch it live from Scryfall (`scryfall.com`).
3. Ask "what do I own that could go in a [strategy] deck?" — should read `raw/manabox_Collection.csv`.
4. Start a deck-building conversation, then in a later session ask about the same deck — should find/update its `wiki/<topic>.md` page instead of starting over.
5. Ask it to log current prices for a few cards — should append a correct row to `price_log.csv`.
6. Log a game result for a deck (e.g. "log a game: [deck], won, 4-player"), then ask "what's my win rate with [deck]?" — should append a row to `games_log.csv` and answer from that deck's `wiki/<topic>.md` Stats section.
7. If you've filled in `house-rules.md`, ask about a card you banned there — should say it's banned per your house rules, not just check the official list.
8. Ask "what's [card] worth" for a card you haven't discussed yet, then ask to log its price — should complete without the agent saying it can't find the append mechanics (confirms `price-tracking-workflow.md` is actually in Knowledge, not just referenced by `game-log-workflow.md`).
9. Ask about a card that's been in `wiki/card-cache.md` for a while and is central to what you're asking — should mention re-verifying it live rather than trusting the cached entry unconditionally once it's a few months old.
10. Ask a mid-game tactical question (e.g. "I'm on the play with 2 lands, [commander], and a rock — keep or mulligan?") — should answer from `in-game-play.md`'s heuristics, not generic advice.

If any of these fail, the likely culprit is a missing connector permission (step 5), the GitHub connector not synced to the latest commit (step 3), or a knowledge file that wasn't selected — check the Project's Knowledge and Connectors sections first.

## Optional: recurring automation

Claude Projects supports its own task scheduler, separate from everything above. `scheduled-tasks.md` has a ready-to-paste weekly prompt that refreshes prices and EDHREC synergy data for decks you've flagged "In progress" in `wiki/index.md` — set it up there if you want it; nothing above depends on it.

## If you outgrow this

If pricing a full collection in one go becomes a real pain point (Level 1 has no batch price lookup — see `price-tracking-workflow.md`'s pacing notes), the documented upgrade path is registering a dedicated Scryfall MCP connector, which unlocks batched lookups and still works from the iPhone app once added via web/desktop. That's a separate, later project — not part of this setup.
