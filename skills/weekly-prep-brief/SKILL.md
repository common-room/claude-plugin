---
name: weekly-prep-brief
description: "Generate a comprehensive weekly briefing for all external calls in the next 7 days. Triggers on 'weekly prep brief', 'prepare my week', 'what calls do I have this week', 'Monday prep', or any weekly planning request."
---

# Weekly Prep Brief

## Briefing Process

### Step 1: Get the Week's External Meetings

**Option A — Calendar connected:**
Use the `~~calendar` connector to fetch all meetings scheduled in the next 7 days (or a user-specified range). Filter to keep only external meetings — those with attendees from outside your organization. Discard internal-only meetings, one-on-ones with colleagues, and recurring internal syncs.

For each external meeting, note the company, the date and time, and the external attendees' names and emails.

**Option B — No calendar connected:**
Ask the user: "To build your weekly prep brief, I'll need your upcoming external calls. Please list them: company name, date/time, and attendee names."

### Step 2: Confirm the Meeting List

Before researching, show the meetings found (company, day, time, attendees) and ask whether anything is missing or should be excluded.

### Step 3: Research Each Meeting

For each confirmed meeting, in parallel where possible:
1. **Account research** — account snapshot using the account-research skill
2. **Contact research** — profile for each external attendee using the contact-research skill

Then a recency check per company, secondary to the Common Room data: search `"[company name]" news` for the last 7 days and, for executive attendees, their name for recent posts or interviews. Include only noteworthy findings (funding, leadership changes, major press).

Research high-priority accounts (large, open opportunity, renewal risk) at full depth; give lower-priority or short meetings 3–4 bullets each.

### Step 4: Synthesize the Weekly Brief

Compile one document sorted by meeting date and time. Open with a week overview that flags accounts with urgent signals (at risk, trial expiring, expansion opportunity), meetings needing special preparation or executive involvement, and the external call count with estimated time.

## Output Format

```
# Weekly Prep Brief — Week of [Date]

## Week Overview
[2–4 bullets: key themes, flagged priorities, call count]

---

## [Monday / Tuesday / etc.]

### [Company Name] — [Time]
**Attendees:** [Names and titles]
**Meeting type:** [Discovery / QBR / Renewal / Expansion / etc. — say if inferred]

**Company Snapshot**
[4–5 bullets: account status, top signals, recent activity]

**Attendee Profiles**
- **[Name]** ([Title]): [2–3 bullets on their signals, persona, conversation angle]
- [Repeat per attendee]

**Top Signals This Week**
[2–3 most relevant signals for this specific call]

**This Week's News** [If notable news found]
[Only genuinely noteworthy findings — funding, leadership changes, major press]

**Recommended Objectives**
[1–2 sentences: what to accomplish in this meeting]

---

[Repeat per meeting, sorted by date/time]
```

**Sparse data:** when Common Room returns little for a meeting's account or attendees, mark that meeting "Limited Data", show only what was returned plus any web findings, and say what's missing so the rep can check before the call. Don't fill the template; every fact must come from a tool call.

## Reference Files

- **`references/briefing-guide.md`** — guidelines for structuring briefings, prioritization logic, and how to handle edge cases (cancelled meetings, new accounts with no data, etc.)
