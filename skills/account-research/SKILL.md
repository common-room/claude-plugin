---
name: account-research
description: "Research a company using Common Room data, including companies not yet in your workspace. Triggers on 'research [company]', 'tell me about [domain]', 'pull up signals for [account]', 'what's going on with [company]', 'what's the tech stack at [domain]', 'who's on the buying committee at [company]', 'who are we missing at [company]', or any account-level question."
---

# Account Research

When a company isn't in Common Room, look it up in Prospector instead of stopping.

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Step 0: Load User Context (Me)

Fetch the `Me` object for the user's title, role, personas and segments. Default all queries to the user's own segments ("My Segments") unless they ask for a broader view.

## Step 1: Identify the Interaction Pattern

- **Pattern 1 — Full overview** ("Tell me about Datadog"): fetch the full property set and produce the briefing in Step 6.
- **Pattern 2 — Targeted question** ("Who owns the Snowflake account?"): fetch only the relevant properties and answer in 1–3 sentences, not a brief.
- **Pattern 3 — Sparse or missing data:** say what Common Room has. No record → Prospector (Step 3.5); limited data → web search (Step 4).
- **Pattern 4 — Combined reasoning** ("Is Stripe a good fit?"): fetch the data, then reason over it (Step 5).
- **Pattern 5 — Buying committee** ("Who are we missing at acme.io?"): Step 5.5.

## Step 2: Look Up the Account

Find the Organization by domain (`companyWebsite`) or name (`companyName`) across the whole workspace, not just My Segments; if it isn't in My Segments, say so. Exact match first; if nothing, try a partial match and confirm with the user before proceeding.

If there's still no match, the company isn't in the workspace. Go to Step 3.5 with its domain; if the user gave only a name, ask for the domain.

## Step 3: Fetch the Right Properties

Call commonroom_get_catalog for the Organization properties this workspace has, then commonroom_list_objects with only the `properties` the question needs:

- `leadScores` — `{scoreId, score, percentile}` per configured score. Report the raw score or percentile, never a label.
- `researchResults` — RoomieAI research answers, each with its `promptName`; `isPrimaryPrompt` marks the primary account summary.
- `topContacts` — up to 5 names and titles, with no scores, and it can still list people who have left. Use it for a quick "who's there" and as the departure check below, not as the ranked contact list.

**Contacts at the account:** a second commonroom_list_objects call on Contact with `companyWebsite` `eq` the domain, sorted by a contact lead score (`ls_*`; check the LeadScore list, since organization scores also appear) descending where the workspace has one (otherwise `latest_activity`), limit 5–10, requesting `title`, `leadScores`, `jobChangeDate`, `previousCompany` and `recentActivities`. Flag anyone whose `jobChangeDate` is recent (a new arrival). This query only returns current employees, so for departures fetch `currentCompany` for any `topContacts` entry it didn't return; anyone now at another company has left — flag them, especially a former champion.

**Recent activity at the account** (full overview): commonroom_list_objects on Activity with `organizationId`, sort `activityTime` descending, limit 15. `EmploymentStarted` entries show contacts taking new roles; when the previous role at this company has no `finishedAt`, it may be a side role rather than a departure, so say which the data shows.
- `profiles` — linked CRM records; `cf_*` — CRM and other custom fields (owner, stage, ARR). Find their IDs with objectType `CustomField`.
- `about`, `employees`, `subIndustry`, `location`, `technologies`, `revenueRangeMin`, `revenueRangeMax`
- `recentNews`, `recentJobOpenings`, `surgingTopics`

| User query type | Request |
|-----------------|---------|
| Full account overview | All of the above, plus the contact and activity queries |
| "Who owns this account?" | `profiles` and the owner `cf_*` field |
| "Is this company a good fit?" | `about`, `employees`, `subIndustry`, `technologies`, `leadScores` |
| "What signals is this account showing?" | `leadScores`, `researchResults`, `surgingTopics`, `recentNews`, `recentJobOpenings` |
| "Who are the top contacts?" | The contact query |
| "What does RoomieAI say about them?" | `researchResults` |
| "Find engineers at this account" | Contacts with `companyWebsite` `eq` the domain and a `title` filter; for people not in the workspace, `ProspectorContact` with `companyDomain` and `title` |

## Step 3.5: Look Up Companies Not in Common Room

When the company isn't in the workspace (Step 2), find it in Prospector: `ProspectorCompany` with `groupWebsite` `eq` the domain (exact match; `like` returns unrelated companies), requesting `description`, `employees`, `revenueRange`, `subIndustry`, `technologies` and `location`. Prospector holds only companies that aren't in your workspace. Reading it adds nothing to the workspace.

Label Prospector fields as Prospector, separate from Common Room data. Prospector has no signals, scores or activity — don't infer any.

## Step 4: Web Search (Sparse Data Only)

Skip web search when Common Room returns rich data. When it's sparse (few properties, no activity, no scores), or the user asks for news or external context, search `"[company name]" news` for the last 30 days: funding, acquisitions, launches, executive changes.

## Step 5: Apply Reasoning (Pattern 4)

Compare the account's size, industry, tech stack and hiring to the ICP from the company context, and note timing signals (funding, trial status, an activity spike). Say which returned values each conclusion rests on.

## Step 5.5: Buying Committee (Pattern 5)

1. **List known contacts** at the account with commonroom_list_objects (contacts filtered by the organization): title, Spark persona, scores and recent activity.
2. **Map roles:** Champion, Economic Buyer, Technical Evaluator, End User or Gatekeeper. Use the Spark persona when present; otherwise infer from title and activity, and say it's inferred.
3. **Name the gaps** against the roles the deal needs, from the company context, or ask the user. Flag a champion who has left the company.
4. **Find candidates** for each gap among Prospector contacts at the account's domain, filtered by title and seniority. Show two or three per gap with title and why they fit.
5. **Save the chosen ones** with commonroom_create_object (prospectorContactId) and add them to a segment if asked, once the user approves the names (see Saving below). Then offer outreach (compose-outreach).

Output: a role map table (Role | Contact | Title | Engagement | Source of role) followed by Gaps and Candidates.

## Step 6: Produce Output

Every fact must come from a tool call. Include only sections Common Room returned data for; omit the rest.

```
## [Company Name] — Account Overview

**Snapshot**
[2–3 sentences: what they do, plan/stage, relationship status]

**Key Details**
[Employee count, industry, location, domain, revenue]

**CRM & Ownership** [If CRM fields returned]
[Owner, opp stage, ARR]

**Scores** [If scores returned]
[All available scores as raw values or percentiles]

**Signal Highlights** [If activity/signals exist]
[3–5 most important signals with dates]

**Top Contacts** [If contacts returned]
[Name | Title | Score | Latest activity — top 5 from the contact query; flag new arrivals and anyone who left]

**RoomieAI Research** [If `researchResults` returned]
[Primary summary; names of the other research topics]

**Recommended Next Steps**
[2–3 specific, signal-backed actions]
```

**Sparse or missing data:** show only what was returned and list what's missing (or "Not in your workspace"); don't fill the template. Add Prospector fields and web findings under their own labels. If the company isn't in Common Room, ask "Want me to save [Company] to Common Room?" and, if the user agrees, create the organization with commonroom_create_object, passing prospectorCompanyId when Prospector found it, otherwise the domain.

**Saving.** Ask before every write and show what will be written: the records, the fields and values, and any segment. Contacts and organizations upsert, so saving one that already exists updates it. Before creating a segment, look for an existing one with commonroom_list_objects. Report what was saved and anything rejected, with the reason.

## Reference Files

- **`references/signals-guide.md`** — signal type taxonomy and interpretation guide
