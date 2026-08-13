# Setup Guide

One-time setup to turn this file package into a working Claude.ai Project, plus your first refresh cycle. Exact menu wording may drift slightly as claude.ai's UI evolves — look for the nearest equivalent if something's been renamed.

## 1. Create the Project

1. On claude.ai (web or desktop), go to **Projects → New project**.
2. Name it **"The Archon of the Ninety-Nine"**.

## 2. Add the custom instructions

1. Open the Project's settings and find **Instructions** (sometimes called "custom instructions" or "how should Claude approach this project").
2. Paste the entire contents of `project-instructions.md` in as-is.

## 3. Upload knowledge files

> **Known bug — GitHub connector currently broken for personal plans:** completing Settings → Connectors → GitHub on an individual Pro account (not Team/Enterprise) redirects to a "you don't have access to organization settings" error and the connection never finishes. This is a reported, reproduced Anthropic-side bug, not a misconfiguration — see [anthropics/claude-code #78761](https://github.com/anthropics/claude-code/issues/78761). Use manual upload below until it's fixed; the steps for connecting via GitHub instead are kept underneath in case you want to retry later.

1. In the Project's **Knowledge** section, upload:
   - `commander-rules-reference.md`
   - `deckbuilding-framework.md`
2. Don't upload the workflow docs (`manabox-drive-workflow.md`, `price-tracking-workflow.md`, `wiki-structure.md`) or `README.md` here — those are for you/future-you to maintain this package, not for Claude to consult mid-conversation. (Optional: upload them too if you want the agent able to explain its own setup on request — harmless either way, just not required.)
3. Because these are static uploads, re-upload (replacing the old file) whenever you edit `commander-rules-reference.md` or `deckbuilding-framework.md` in the repo — there's no sync to forget about, but also no auto-update.

### Alternative once the GitHub connector bug is fixed

Instead of manual upload, you can connect the `PersonalAgents` repo itself so the Project's knowledge stays tied to the source files:

1. Push this repo to GitHub if it isn't already (public or private both work — a private repo just needs you to grant the Claude GitHub App access to it during connector setup).
2. In the Project's **Knowledge** section, click **+ → GitHub**, authenticate if prompted, and select the `PersonalAgents` repository.
3. Use the file browser to select only `mtg-commander-agent/commander-rules-reference.md` and `mtg-commander-agent/deckbuilding-framework.md` (not the whole repo).
4. **Not live sync**: after editing either file in the repo, the Project keeps using the old version until you click **Sync now** in the Project's Knowledge section — there's no sync control in the iPhone app, so do this from web/desktop before a mobile session where the update matters.

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
  price_log.csv                 ← header row from price-tracking-workflow.md
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
5. Starter `price_log.csv`:
   ```csv
   date,card_name,set_code,finish,quantity_owned,unit_price_usd,total_value_usd,source
   ```
6. Make sure the Drive integration/connector has access to this folder (some connector setups scope to "all Drive" and some ask you to pick folders — grant access to at least this one).

## 6. First ManaBox export

Follow `manabox-drive-workflow.md`: export your collection CSV from the ManaBox app and upload it to `MTG Commander Agent/raw/manabox_Collection.csv`.

## 7. Verify on iPhone

1. Open the Claude iOS app, sign in with the same account.
2. Open **Projects** and confirm your new Project appears with its instructions and knowledge intact.
3. Send a test message (see checks below) and confirm it can reach both the knowledge files and your Drive folder from mobile.

## Verification checklist

Run these from the iPhone app once setup is done:

1. Ask a rules question (e.g. "how much commander damage kills a player?") — should answer from `commander-rules-reference.md` without a web search.
2. Ask about a recent card's legality/oracle text — should fetch it live from `api.scryfall.com`.
3. Ask "what do I own that could go in a [strategy] deck?" — should read `raw/manabox_Collection.csv`.
4. Start a deck-building conversation, then in a later session ask about the same deck — should find/update its `wiki/<topic>.md` page instead of starting over.
5. Ask it to log current prices for a few cards — should append a correct row to `price_log.csv`.

If any of these fail, the likely culprit is a missing connector permission (step 5), the GitHub connector not synced to the latest commit (step 3), or a knowledge file that wasn't selected — check the Project's Knowledge and Connectors sections first.

## If you outgrow this

If pricing a full collection in one go becomes a real pain point (Level 1 has no batch price lookup — see `price-tracking-workflow.md`'s pacing notes), the documented upgrade path is registering a dedicated Scryfall MCP connector, which unlocks batched lookups and still works from the iPhone app once added via web/desktop. That's a separate, later project — not part of this setup.
