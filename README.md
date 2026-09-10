# Bellhaven CRM sync

Keeps facility → parent-company links in the CRM accurate by checking
Bellhaven Senior Living's public website against the CRM sandbox, proposing
fixes, and only ever writing to the CRM once a human approves each one.

## What it does

1. **`app/scraper.py`** — scrapes every community currently listed on
   Bellhaven's website (name, city/state, care type, and best-effort
   address/zip/phone from each community's detail page).
2. **`app/matcher.py`** — pure logic, no network calls. Compares scraped
   communities against current CRM accounts and classifies each into:
   - a confident match (nothing to do)
   - `RENAME` — parent is already right, but the CRM name is stale/rebranded
   - `REPARENT` / `REPARENT_CHOW` — parent is wrong or missing (see SOP below)
   - `CREATE` — website lists it, CRM has nothing for it at all
   - `DUPLICATE` — two CRM accounts for the same facility
   - `NEEDS_REVIEW` — CRM account is parented to Bellhaven but no longer
     appears on the website (possible divestiture — flagged, never auto-changed)
   - `MANUAL_REVIEW` — multiple same-city CRM accounts plausibly match; the
     matcher deliberately refuses to guess
3. **`app/store.py`** — SQLite decision log. Every proposal gets a
   deterministic ID from its underlying condition, so re-running the
   pipeline never re-inserts something already decided (approved or
   rejected). This is what makes daily re-runs safe.
4. **`app/review_app.py`** — Streamlit app. Shows each pending proposal with
   its evidence (scraped data + CRM data side by side). Approve writes to
   the CRM immediately via `app/executor.py`; reject just closes it out.
   Nothing writes without a click.
5. **`app/pipeline.py`** — the daily job: scrape → fetch CRM → match → store.
   Never writes to the CRM.

## The CHOW SOP (billing-preservation rule)

When a facility's parent needs to change, `matcher.propose_reparent` checks
`lifetime_revenue` and `outstanding_ar` on the existing account first:

- **Both > 0** → the old account is left completely untouched. A new account
  is created under the correct parent, and the old account's
  `chow_current_account` is set to the new account's id. Two API calls,
  bundled as one proposal in the review queue.
- **Otherwise** → the existing account is `PATCH`ed directly (`parent_id`,
  and `name` if it's also stale).

## Duplicates

No merge/delete exists in this API, so `find_duplicates` picks a survivor
(prefers whichever record has real billing history, then most recently
updated) and proposes setting `duplicate_of_account` + `status: Inactive`
on the loser.

## Running it

```bash
pip install -r requirements.txt

# macOS/Linux
export CRM_API_TOKEN="bh_..."
# Windows PowerShell
$env:CRM_API_TOKEN = "bh_..."

python app/pipeline.py          # scrape + match + queue proposals
streamlit run app/review_app.py # review, approve/reject
```

Re-run `python app/pipeline.py` any time (that's the "daily job") — it will
only add genuinely new proposals.

## Scheduling

- `.github/workflows/daily-sync.yml` — runs `pipeline.py` daily at 09:00 UTC
  (proposal generation only; approvals always require the review app, so
  this workflow never writes to the CRM). Not deployed for this exercise,
  provided as the config I'd use.
- `crontab.txt` — plain-cron equivalent if running from a machine that stays on.

## Known limitations / what I'd harden next

- The scraper's selectors were written against the site's structure as
  observed manually, not against a live test run from this environment
  (no outbound network to the sandbox host from where this was built) — see
  `SELECTOR NOTE` comments in `scraper.py`. Matching/store/CHOW logic *was*
  tested end-to-end against a real pull of the CRM data and the real
  website community list (see `WRITEUP.md`).
- Name-similarity matching is a normalized token-overlap heuristic
  (`difflib.SequenceMatcher`), not embeddings — good enough here, would
  reconsider for a much larger/noisier dataset.
- `MANUAL_REVIEW` cases (e.g. three same-city candidates with no clear
  winner) intentionally stop short of an automated decision — the review
  app surfaces the evidence but doesn't offer a picker UI to choose among
  candidates; you'd resolve those directly in the CRM today.
