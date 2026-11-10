---
name: prospect
description: "Build targeted account or contact lists using Common Room's Prospector, and save the ones that fit to Common Room and a segment. Triggers on 'find companies that match [criteria]', 'build a prospect list', 'find contacts at [type of company]', 'show me companies hiring [role]', 'save these to Common Room', 'put these in a segment', 'qualify this attendee list', 'score these domains', 'which of these leads fit our ICP', or any list-building request."
---

# Prospecting

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Critical Distinction: Two Object Types

Settle which one is in play before running a query:

- **`ProspectorCompany`** — companies **not yet in Common Room**. Firmographic and technographic filters and fields only (industry, size, geography, funding, revenue, tech stack); no signals, scores or activity. Use for net-new lists, territory planning and top-of-funnel expansion.
- **`Organization`** — companies **already in the workspace**. Adds signal, score, segment, CRM and custom-field filters. Use for prioritizing warm accounts, expansion candidates and intent within existing pipeline.

When a request could mean either ("Show companies hiring AI engineers this month"), ask:
> "Are you looking for net-new companies not yet in Common Room, or filtering accounts already in your workspace?"

## Step 0: Load User Context (Me)

Fetch the `Me` object to get the user's segments. When prospecting against `Organization` records (accounts already in CR), default to filtering within "My Segments" unless the user asks for a broader search.

## Step 1: Gather Targeting Criteria

If criteria are already provided, proceed. Otherwise ask what they're looking for: company size, industry, job titles, region, signals such as product activity, or intent such as recent funding or job postings.

Check commonroom_get_catalog for the filters each object type has.

**Lookalike search:** If the user asks to "find companies like [X]", first look up the reference company in Common Room. If it isn't there, find it in Prospector (`ProspectorCompany` with `groupWebsite` `eq` the domain (exact match; `like` returns unrelated companies), requesting `description`, `employees`, `revenueRange`, `subIndustry`, `technologies` and `location`), and use web search only if that returns nothing. Extract its industry, employee range, tech stack, funding stage and geography, propose those as filter criteria, and have the user confirm which matter before running the search.

## Step 2: Refine

With 50 or more results, summarize and suggest a filter to narrow by; with fewer than 5, name the criterion to relax. Apply each follow-up as a change to the current filter, not a new search.

## Step 3: Run the Query and Present Results

Sort by signal strength or fit score where available, not alphabetically. Show only what the query returned ("—" for missing fields), and never mix `ProspectorCompany` and `Organization` fields in one list.

**For `ProspectorCompany` (net-new) results:**

| Company | Domain | Industry | Size | Capital Raised | Revenue | Location |
|---------|--------|----------|------|---------------|---------|----------|

**For `Organization` (in CR) results:**

| Company | Industry | Size | Top Signal | Signal Date | Score | CRM Stage |
|---------|----------|------|-----------|-------------|-------|-----------|

Flag any results where data is thin or the most recent signal is older than 90 days.

## Step 3.5: Fill In Net-New Results

For `ProspectorCompany` results, request `primaryWebsite`, `employees`, `revenueRange`, `subIndustry`, `technologies`, `raised` and `location` in the query itself (defaults are only `id` and `name`), so no second call is needed.

Then run a quick web search on the top 3–5 for recent news — funding, product launches, leadership changes — which Prospector doesn't cover. Include findings as brief annotations next to each company.

## Step 4: Offer Next Steps

Offer to save the results to Common Room and a segment (Step 5), draft outreach for the top 3–5, run a full account brief on any of them, refine the criteria, or format the list as CSV.

## Step 5: Save the List (When the Seller Asks)

Nothing is saved or segmented without the seller's approval. Show what will be written: the records, the fields and the segment.

1. **Dedupe.** Check each net-new company by domain with commonroom_list_objects. Mark each result as new or already in Common Room; for existing ones, show the owner and segments.
2. **Check fit.** For each new result, say whether it meets the seller's criteria and the ICP from the company context, naming the fields that decided it. Recommend save or skip.
3. **Save.** Create the approved companies and contacts with commonroom_create_object, passing prospectorCompanyId or prospectorContactId so name, domain, title and firmographics fill in.
4. **Segment.** Ask whether to use an existing segment or create one.
   - If the criteria map to Common Room filters, offer a segment built from those filters with commonroom_create_segment, so it stays current as companies start or stop matching.
   - Otherwise create a segment with no filter and add the saved records with commonroom_update_object, up to 100 per call.
   - For a team segment, ask which teams it's for and which owner field matches records to reps, and whether to pin it to each assignee's Home and to their "My segments" (the tool needs both answers). Set personaIds, assignment, addToHome and addToMySegments.
5. **Report.** Saved, already existed, added to segment, skipped, failed.

## List Mode: Qualify a List the Seller Brings

When the seller pastes or attaches emails, LinkedIn URLs or domains — event attendees, webinar registrants, inbound domains — instead of criteria:

1. **Read the list.** Find one identifier per row: email, LinkedIn URL or domain. Report rows without one; never guess an email or domain from a name.
2. **Split known from new.** Check each row with commonroom_list_objects (contacts by email, organizations by domain). Known records keep their scores, owner, segments and signals.
3. **Look up the new ones in Prospector.** Domains: `ProspectorCompany` with `groupWebsite` `eq` the domain (exact match; `like` returns unrelated companies), requesting `description`, `employees`, `revenueRange`, `subIndustry`, `technologies` and `location`, plus `primaryWebsite`, with up to 25 domains per query as an `or` group of `eq` clauses; match results back to rows by `primaryWebsite`. People: `ProspectorContact` by `fullName` and `companyDomain` when the row has both; Prospector can't be searched by email or LinkedIn URL alone, so label those rows "not looked up". Keep not-found rows, labeled.
4. **Score fit** against the ICP from the company context or the seller's criteria: Strong, Possible, Poor or Unknown, naming the one or two fields that decided it. Never score from a company name alone.
5. **Present** Strong first, with counts:
   | Name / Company | Identifier | In CR? | Fit | Why | Proposed action |
6. **Save** the approved rows with Step 5, creating each from its own email, LinkedIn URL or domain or its Prospector ID when step 3 found one, and offer to enroll the Strong-fit contacts or add them to a call list (contact-actions).

Large lists: work in batches of 25 and report progress.

## Reference Files

- **`references/prospect-guide.md`** — filter types, signal-based sorting, object type distinctions, and list-building strategies
