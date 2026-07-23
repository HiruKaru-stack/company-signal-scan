---
name: company-signal-scan
description: Given a company's website and LinkedIn URL (or a CSV of many companies), deep-researches which growth/buying signals are active right now — hiring, funding, leadership changes, expansion, tech/tool adoption, or press coverage — using Exa MCP search and fetch. Discards any signal not evidenced within the last 3 months (a live careers page always counts as current). Use to qualify a lead/company as a warm outbound target, either one at a time or as a batch pass over a lead list.
user_invocable: true
---

# Company Signal Scan

## Goal

Answer: **is this company (or are any companies in this list) showing signs of growth or change right now that make them a good outbound target?** Checks several signal categories — not just hiring — and reports which are real (dated evidence) vs. stale (old, not counted).

Two modes:
- **Single-company mode** — user gives one website + LinkedIn URL, wants an answer in chat. See "Signal categories" and "Output format" below.
- **Batch mode** — user gives a CSV of many companies, wants the same checks run per row. See "Batch mode (CSV input)" below — it reuses everything in single-company mode per row, it just adds how to loop, sample, and write results back out.

## Inputs

**Single-company mode:**
- Company website (e.g. `coldlabs.ai`)
- Company LinkedIn URL (e.g. `linkedin.com/company/cold-labs`)

If the user only gives one, find the other via `web_search_exa` before proceeding.

**Batch mode:**
- A CSV with at least a domain column and a LinkedIn URL column (other columns — location, industry, headcount, whatever the list-builder produced — are carried through untouched)
- Which signal categories to check this run (any subset of the 6 below). If not specified, ask before running the full file — see "Batch mode" for why.

## Tools

- `mcp__exa__web_fetch_exa` — read known/likely URLs (careers page, LinkedIn company page, ATS board)
- `mcp__exa__web_search_exa` — find pages not at a guessable URL, and find dated posts/news per signal category

No custom script needed — Exa MCP is the execution layer for this skill.

## Recency rule (applies to every category below)

- A page you fetch live right now (careers page, ATS board) counts as **current** by definition — no date needed, you're looking at it today.
- Everything else (news articles, LinkedIn posts, press releases) only counts as a real signal if Exa's `Published` date is within the **last 3 months** of today. Anything older is background context — report it separately as stale, never blend it into the verdict.

## Identity-match rule (applies to every category below)

Short or generic company names collide with unrelated companies constantly — e.g. searching "Cold Labs" also surfaces "ColdAI," "Coldi," "ColdSend," "Coldlytics," and an unrelated Australian company literally named "CoLabs." Before counting ANY search result as a signal, confirm it's actually about the target company:

- Domain matches (the result links back to, or explicitly names, the target's website), OR
- Founder/exec name matches someone already confirmed on the target's LinkedIn/team page, OR
- Location + industry both match (e.g. same city, same niche).

If a result doesn't clearly clear one of these checks, exclude it — don't count it, and don't feel obligated to explain every excluded result in the output. Only note excluded results if they're the kind of near-miss the user might otherwise assume was a hit (e.g. a same-named company in a related industry).

## Proof-link rule (applies to every category below)

**A signal is only reported as YES if it comes with a working URL the user can click to verify it themselves.** Not just a source name like "LinkedIn" or "careers page" — the actual link (e.g. `https://coldlabs.ai/careers`, the exact LinkedIn post URL, the exact news article URL).

- If you can't produce a real link for a claim, it doesn't count as YES — downgrade it to `UNKNOWN — no verifiable link found` instead of reporting it as confirmed.
- This applies to every category, not just hiring: a funding claim, a leadership change, a tech-adoption badge — all of it needs a link, every time.
- Never fabricate or guess a URL to fill this requirement. If Exa's result doesn't include a usable link, treat the claim as unverified.

## Which categories to run

If the user asks about one specific thing ("is this company hiring", "any funding news"), run only that category. Run all six only when they ask for a general/overall signal check.

## Signal categories

1. **Hiring**
   - Fetch the careers page: `{domain}/careers`, `{domain}/jobs`, `{domain}/join`, or ATS boards (`jobs.{domain}`, `talent.{domain}`, `boards.greenhouse.io/{company}`, `jobs.lever.co/{company}`, `jobs.ashbyhq.com/{company}`, `{company}.workable.com`). If none resolve, `web_search_exa("{company} careers jobs")`.
   - If Greenhouse/Lever/Ashby, prefer the public JSON API (e.g. `boards-api.greenhouse.io/v1/boards/{company}/jobs`) — gives per-role `updated_at` dates.
   - Extract open role titles, employment type, location.
   - Corroborate with `web_search_exa("{company} hiring")` / `("{company} is hiring")`, applying the recency rule.

2. **Funding / investment**
   - `web_search_exa("{company} funding round raised investment")` and `("{company} raises")`.
   - Look for amount, round stage (seed/Series A/etc.), investors, and date.

3. **Leadership change**
   - `web_search_exa("{company} new CEO OR CMO OR Head of Growth OR VP Sales appointed OR joins")`.
   - New execs often precede new tool/agency/vendor decisions — worth flagging even without a hiring page match.

4. **Expansion**
   - `web_search_exa("{company} opens new office OR expands to OR launches in")`.
   - New market or office = growth signal.

5. **Tech / tool adoption**
   - `web_search_exa("{company} adopts OR partners with OR built on")`.
   - Also check the company's own homepage/LinkedIn "about" text for tool-stack mentions (e.g. Cold Labs' homepage says "Built on Clay") — these are self-reported and count as current if still live on the page today, same as the careers-page rule.

6. **Press / product launch**
   - `web_search_exa("{company} launches OR announces OR press release")`.

For each category: run the targeted search, check the top few results' `Published` dates, keep only what's within 3 months (or live-fetched pages), and explicitly set aside older matches as stale rather than silently dropping or using them.

## Output format

```
Company: {name} ({domain})
Overall: {N} active signal(s) found (of 6 checked)

Hiring — {YES/NO/UNKNOWN}: {roles found, or "no live openings"} — {date} — {proof URL}
Funding — {YES/NO/UNKNOWN}: {amount/round, or "none found"} — {date} — {proof URL}
Leadership change — {YES/NO/UNKNOWN}: {who, new role, or "none found"} — {date} — {proof URL}
Expansion — {YES/NO/UNKNOWN}: {what, or "none found"} — {date} — {proof URL}
Tech/tool adoption — {YES/NO/UNKNOWN}: {what, or "none found"} — {date} — {proof URL}
Press/launch — {YES/NO/UNKNOWN}: {what, or "none found"} — {date} — {proof URL}

Stale mentions (>3 months, not counted toward verdict):
- {summary} — {date} — {proof URL}
```

Every YES line must carry a real, clickable URL — that's the proof-link rule above. If no link exists for an otherwise-plausible claim, report it as UNKNOWN instead of YES. For a single-category ask, just report that one line plus its evidence/date/link — don't pad the answer with the other five as "none checked."

## Edge cases

- **Small/private companies:** funding will often be "none found" — that's a valid, honest answer. Don't stretch other categories to compensate for a quiet one.
- **Conflicting employee counts across sources:** report the LinkedIn figure as primary (self-reported, closest to source); only flag a discrepancy if it's large.
- **Careers page says no openings:** Hiring = NO, even if old "we're hiring" posts exist elsewhere.
- **Same/similar-named unrelated companies:** apply the identity-match rule above — this is common with short or generic names (e.g. "Cold Labs" vs. "ColdAI," "Coldi," "CoLabs Australia") and easy to miss if you only check the headline.
- **Unverifiable database claims** (e.g. a funding tracker citing a dollar figure under a slightly different company name, with no date, contradicted by another source): don't count it — flag it as low-confidence and explain why in one line, rather than silently including or silently dropping it.

## Batch mode (CSV input)

Same rules as single-company mode (recency, identity-match), just looped over rows. Follows the same sample-first-then-scale pattern as `/icp-prompt-builder` in this toolkit — don't skip the sample step even if the user seems in a hurry, since Exa search noise (name collisions, stale posts) is exactly the kind of thing that's cheap to catch on 5 rows and expensive to catch on 500.

### 1. Confirm the signal set before running anything

Exa MCP calls are free on this setup — no per-call spend to manage. The reason to still confirm scope up front is time and rate limits, not cost: checking all 6 categories means roughly 6× the Exa calls of checking 1, and hundreds of calls back-to-back can hit rate limiting or just take a long time to finish. Before running the file:
- If the user named specific signals ("just check hiring and funding"), use only those.
- If they didn't say, ask which categories to check — don't silently default to all 6 on a large file.
- For a large file (hundreds of rows), mention the rough scale (e.g. "200 rows × 3 categories is ~600-1000 calls, this will take a while and may need to run in waves") so the user knows what to expect — not as a cost warning, just a time-and-pacing one.

### 2. Sample first (5 rows)

Take the first 5 rows. Launch one Task sub-agent per row, in parallel, each one running the single-company steps above (fetch/search only the confirmed signal categories, applying the recency + identity-match rules) for that row's domain/LinkedIn.

### 3. Show the sample, get a sanity check

Present the 5 results as a table (one column per requested signal). Ask the user to spot-check before scaling — this is where they catch things like a signal that's really name-collision noise, or a category that isn't turning up anything useful across the board and should be dropped.

### 4. Scale in waves

Once approved, continue in waves of 10-20 rows, one Task sub-agent per company, run in parallel within a wave. Don't run more than ~20 companies at once — matches the batch discipline used elsewhere in this toolkit.

### 5. Write the output CSV

Same filename with a `-signals` suffix (e.g. `leads.csv` → `leads-signals.csv`). Original columns preserved, plus one column per requested signal and a `signal_count` column:

```
domain,linkedin_url,location,...,signal_hiring,signal_hiring_link,signal_funding,signal_funding_link,signal_count
coldlabs.ai,linkedin.com/company/cold-labs,Sofia Bulgaria,...,"YES: 4 roles, 2026-07-19",https://coldlabs.ai/careers,NO,,1
```

Each signal category gets two columns: the verdict cell (`NO`, `UNKNOWN — {reason}`, or `YES: {short evidence}, {date}`) and a matching `_link` column with the exact proof URL. A `YES` with an empty link column is not allowed — per the proof-link rule, downgrade it to `UNKNOWN — no verifiable link found` instead. `NO` and `UNKNOWN` rows can leave the link column blank.

### Batch-specific edge cases

- **Row missing a LinkedIn URL:** look it up via `web_search_exa` first, same as single-company mode — but if no confident match is found, mark it `UNKNOWN — no LinkedIn match found` rather than guessing.
- **Domain unreachable / site down:** mark all requested signals `UNKNOWN — site unreachable`, not `NO`. `NO` means "checked, found nothing"; unreachable means the check couldn't run.
- **Duplicate domains in the CSV:** dedupe before running — no reason to pay for the same company twice.

## Notes

- Pull "today's date" from the current session context, don't assume or hardcode it.
- Single-company mode is the building block; batch mode is just that block run per row with sampling discipline on top.

## Checklist

- [ ] Relevant categories identified (all 6, or just the one(s) asked about)
- [ ] Each requested category's live page fetched and/or search run
- [ ] Each result's date checked against the 3-month cutoff
- [ ] Each result's identity checked against the target (domain/founder/location match), not just the headline
- [ ] Stale mentions separated out, not blended into the verdict
- [ ] Every YES has a real, working proof URL attached — no link means it's UNKNOWN, not YES
- [ ] Output format followed, with source + date + link cited per category

**Batch mode only:**
- [ ] Signal set confirmed with the user (not silently defaulted to all 6) and scale/runtime expectation given for large files
- [ ] 5-row sample run and shown to the user before scaling
- [ ] Duplicate domains removed before running
- [ ] Output CSV includes evidence + date + proof link per signal, not bare YES/NO
- [ ] Unreachable domains / unresolved LinkedIn marked UNKNOWN, not NO
