---
name: watchlist
description: "Monitor the seller's accounts and champions for what changed since the last check — intent and web visits, score jumps, new people engaging, hiring and news, opportunity changes, buyers who left, champions who joined a new company — rank what needs action, and hand off the follow-up. Works as a scheduled run. Triggers on 'what changed on my accounts', 'anything new on my accounts this week', 'which of our champions changed jobs', 'watch these accounts', 'track my champions', 'what should I work on today'."
---

# Watchlist

It suits a scheduled run, for example every Monday morning, when the scheduled task's prompt names the window.

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Step 0: Define the Watchlist

Fetch the `Me` object. The watchlist is accounts, champions, or both:
- **Accounts:** default to "My Segments" or the accounts the seller owns; or a segment or list of domains they name.
- **Segments:** a segment in `Me.segments` holds contacts or organizations. Scope account queries with `groupSegmentId`. For contacts at those accounts, nest an Organization group (`target` "Organization") with `groupSegmentId` in the Contact filter; `memberSegmentId` only matches a segment that holds contacts, such as a champion list; if one comes back empty, check the segment's `entityCount` before reporting no changes.
- **Lead score:** sort by the score the seller uses for accounts. List the scores with objectType LeadScore; ask once if it isn't obvious, and reuse it.
- **Champions:** propose what the workspace has: contacts tagged "Champion" or similar (look up tags with commonroom_list_objects, objectType Tag); a Champion or Economic Buyer Spark persona; contacts on closed-won opportunities, if Salesforce opportunities are mapped; power users by product usage; or a segment the seller names.

If the list is defined by filters, offer to save it as a segment with commonroom_create_segment so it stays current and later runs start from it (with the seller's approval). Reuse a saved watchlist segment when one exists.

Window: the last 7 days, unless the seller gives a start date. Say the window in the heading. Nothing records when the last run happened. For a scheduled run, the scheduled task's prompt should name its window (for example "since last Monday" or "the last 7 days"), matching how often it runs.

## Step 1: Account Changes

Check commonroom_get_catalog for the fields the workspace has; never guess them. Don't look accounts up one at a time. Run one commonroom_list_objects query per kind of change, scoped to the whole watchlist (its segment with `groupSegmentId` or `memberSegmentId`, or its domains with `companyWebsite`) and to the window:

| Change | Query |
|---|---|
| Visits to high-intent pages (pricing, docs, security, integrations) | Organizations with `groupWebVisitLastSeen` in the window, filtered by `groupWebVisitPagesViewed` for those pages. The result carries no visit date, so report these as "visited [pages] in the window" rather than with a date |
| New contacts engaging, especially senior titles or a buying role the account lacked | Contacts at watchlist accounts (a nested Organization group with `groupSegmentId` or `companyWebsite`) with `memberLastActivity` or `memberWebVisitLastSeen` in the window, requesting `recentActivities` for dated activity. Contact results carry no visit date, so report a web visit as "visited in the window" |
| A champion or buyer leaving, or joining a new company | Contacts in the champion set from Step 0 (its tag with `memberLabelId`, or a contact segment that isn't defined by current company, with `memberSegmentId`) with `memberJobChangedAt` in the last 90 days, requesting `currentCompany`, `previousCompany` and `jobChangeDate` in properties (they aren't returned by default). Never scope this by the watched accounts: a contact who left no longer matches their old company. With no champion set, departures can't be found; say so. Step 2 reuses this |
| Hiring, funding and news | **Hiring:** Organizations with a nested `"JobOpening"` group (`jobOpeningFirstSeen` in the window), sorted by the lead score from Step 0. Don't request `recentJobOpenings`; it carries full job descriptions. For the top 10 accounts, read postings with a JobOpening query (`organizationId` `eq` that account, sorted by `firstSeenAt` descending, limit 5), which leaves descriptions out. **News:** Organizations with a nested `"NewsItem"` group (`newsPublishedAt` in the window), sorted by the same lead score; don't request `recentNews` across many accounts (about 5KB per account). For the top 10 accounts, read articles with a NewsItem query (`organizationId` `eq` that account, sorted by `publishedAt` descending, limit 5). List matching accounts beyond the top 10 by name as "hiring in the window" or "in the news in the window", and say how many weren't read in detail |
| Score or intent jumps; opportunity stage or amount changes | Only where a dated activity or signal in the window shows them |

Add `"recordCount"` to each query's properties for the total, and fetch at most the top 50 rows, sorted by the lead score from Step 0; Organization can't sort by date. Without a score, say the rows aren't ranked. When more match, say how many were left out and offer to narrow the watchlist (one segment, or only the accounts the seller owns). Beyond those detail lookups for the top accounts, never fall back to one query per account.

Only report changes the data shows, each with its date. Scores and CRM fields come back as current values with no history, so never present one as old → new; call it a change only when a dated activity or signal in the window shows it, and otherwise show the current value as context.

## Step 2: Champion Moves

1. **Find the moves** in the `memberJobChangedAt` results from Step 1; `previousCompany` and `currentCompany` give the old and new company. Keep name, previous company, new company and domain, new title, and `jobChangeDate`. That is the month the new role started, not when the move was detected; detection often comes weeks or months later, which is why the query looks back 90 days. Report every move as "role started [Month YYYY]", newest first, and never call one new or recent. A scheduled task's prompt can list moves already reported so later runs skip them. Never infer a move from a title alone.
2. **Check the new company.** In Common Room: owner, segments, customer status and any open opportunity. Not in Common Room: find it in Prospector (`ProspectorCompany`, `groupWebsite` `eq` its domain from `currentCompany`, requesting `description`, `employees`, `revenueRange`, `subIndustry`, `technologies` and `location`). Rate fit Strong, Possible or Poor against the company context and the seniority of the new title, naming the fields that decided it.

## Step 3: Rank and Present

Lead with the three to five changes most worth acting on, ranked by recency, signal strength, fit and deal stage.

```
## Watchlist — [window]

### Act on these
| # | Account / Contact | What changed | When | Why it matters | Next step |
|---|---|---|---|---|---|

### Champion moves
| Contact | From | To | New title | Role started | Fit | Next step |
|---|---|---|---|---|---|---|

### Also changed
[One line per account with a smaller change]

**No change:** [N] accounts, [N] champions (omit when any query hit the 50-row cap; the count can't be worked out then)
```

Next steps:
- **Intent or engagement spike:** research the account (account-research) or draft outreach to the person engaging (compose-outreach)
- **A buyer or champion left:** a note on the account and a buying committee check (account-research)
- **Champion moved, Strong fit, not a customer:** save them and the new company, then enroll in a champion-moved sequence (contact-actions)
- **Champion moved to a customer or an account with an open opportunity:** tell the owner; no sequence without them
- **Champion moved, Possible fit:** save and add to a "Champions — moved" segment for review
- **Several contacts showing the same signal:** enroll them or push them to a call list (contact-actions)

## Step 4: Write What's Approved

Write only the rows the seller picks, after they approve what will be written, and report what was written and anything rejected, with the reason:
- Update a moved champion's title and company with commonroom_update_object, or create them with commonroom_create_object; save the new company if it isn't in Common Room.
- Add contacts or accounts to a segment with commonroom_update_object; create the segment with commonroom_create_segment if needed.
- Add a note on an account that lost a champion with commonroom_create_object (objectType Note, organizationId), saying only what the data shows, with dates.

Sequences, call lists and CRM exports go through contact-actions.
