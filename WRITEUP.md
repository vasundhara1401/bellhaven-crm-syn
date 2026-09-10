# Writeup

## Time spent
<!-- Fill in honestly, e.g. "2h 40m" -->

## What I found

I validated the matcher's logic against a real pull of your CRM data (121
accounts) and the real list of communities on Bellhaven's website (35
communities) before finalizing it. It produced 17 proposals:

| Community / Account | Issue | Fix |
|---|---|---|
| Bellhaven of Tiffin | parent stuck on old owner (Cedar Trail); has revenue **and** outstanding AR | CHOW: preserve old account, create new under Bellhaven, link via `chow_current_account` |
| Bellhaven of Marietta | same — parent stuck on Cedar Trail; revenue **and** AR > 0 | CHOW: preserve + create + link |
| Bellhaven Crossings of Lima | parent stuck on Harborview; has revenue but AR = 0 | direct reparent |
| Bellhaven of Zanesville | CRM record still named/parented "Cedar Trail of Zanesville"; no revenue | direct reparent |
| Bellhaven at Union Square | CRM record "Union Square Senior Living" under Juniper Point; no revenue | direct reparent |
| Bellhaven Meadows of Findlay | orphaned (no parent) — the site's own homepage calls this out as a recent acquisition; has revenue but AR = 0 | direct reparent |
| Bellhaven of Chagrin Falls | CRM has "Riverbend Manor Care Center" already under Bellhaven — full rebrand, name just never updated | rename only |
| Bellhaven Willow Creek (Portage, MI) | CRM has "Sunny Acres Retirement Home" already under Bellhaven | rename only |
| Bellhaven of Chesterton | CRM has "Chesterton Senior Commons" already under Bellhaven | rename only |
| Bellhaven of Batavia | no CRM account in that city at all | create new |
| Bellhaven of Carlisle | no CRM account in that city at all | create new |
| Amberly Manor (Hudson, OH) | no CRM account in that city — note the CRM *does* have several unrelated "Amberly ___" accounts elsewhere under Juniper Point/Stonebridge; those are decoys, not this facility | create new |
| Bellhaven of Owosso | two identical CRM records under Bellhaven | mark the less-complete one `duplicate_of_account` + Inactive |
| Bellhaven Care Center of Alliance, OH | parented to Bellhaven, no longer on the website | flag `Needs Review`, don't touch parent |
| Bellhaven of Coldwater, MI | same | flag `Needs Review` |
| Bellhaven of Sandusky, OH | same | flag `Needs Review` |
| Bellhaven of Kettering | 3 same-city CRM accounts (Harborview / unparented / Cedar Trail), none named "Bellhaven of Kettering" — no name similarity clearly wins | surfaced as ambiguous, not auto-resolved |

## Design decisions worth calling out

- **CHOW rule is checked per-account, not per-community.** The SOP says
  "if the account has revenue history AND outstanding AR greater than
  zero" — I read that as strictly both conditions, which is why Lima
  (revenue but zero AR) and Findlay (revenue but zero AR) get a direct
  reparent rather than the preserve-and-create path, while Tiffin and
  Marietta (both > 0) do.
- **A city can only have one Bellhaven-owned facility, so I prioritize an
  already-Bellhaven-parented same-city record over pure name matching.**
  Without that rule, "Sunny Acres Retirement Home" → "Bellhaven Willow
  Creek" and "Riverbend Manor Care Center" → "Bellhaven of Chagrin Falls"
  would have been misclassified as brand-new facilities (name similarity
  ~0.2) instead of renames, which would have created duplicate accounts.
- **Ambiguity is surfaced, not guessed.** Kettering has three CRM
  candidates and none of them is a decisive name match — rather than
  picking one and risking a wrong parent/billing assignment, the matcher
  flags it for a human. I'd rather under-automate here than silently
  misroute a facility's ownership.
- **Divestiture detection never auto-changes `parent_id`.** Alliance,
  Coldwater, and Sandusky are parented to Bellhaven but don't appear on the
  current site. That could mean they were sold, or that the website simply
  isn't exhaustive — I don't have enough signal to tell those apart
  automatically, so these get `status: Needs Review` and a note, not a
  parent change.

## What I actually approved

<!--
Fill this in after running `streamlit run app/review_app.py` for real and
clicking through the queue. The instructions are explicit that an
unapproved queue scores the same as doing nothing — so this section should
describe the real end state of your CRM copy, not just what the pipeline
proposed.
-->
