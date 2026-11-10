---
name: call-prep
description: "Prepare for a customer or prospect call using Common Room signals, and log what came out of it afterwards. Triggers on 'prep me for my call with [company]', 'prepare for a meeting with [company]', 'what should I know before talking to [company]', 'log my call with [company]', 'what came out of my call with [company]', 'update Common Room after my meeting', or any call preparation or follow-up request."
---

# Call Prep

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Prep Process

### Step 1: Identify the Account and Attendees

The company name is required; attendee names are optional.

**Calendar lookup:** If a `~~calendar` connector is available, search for upcoming meetings with the named company to automatically surface attendee names, meeting time, and any meeting notes or agenda. Use this to fill gaps the user didn't provide.

If neither attendees nor a calendar match can be found, ask: "Who will be on the call from [Company]? I can research each attendee to make your prep more useful."

### Step 2: Run Account Research

Follow the account-research skill. For call prep, prioritize recent product signals, open opportunities or the renewal timeline, risk signals (declining usage, support tickets, churned seats) and key recent events (funding, executive change, new hire).

In the activity history, prioritize Gong and call recording activities; they show what was said in earlier conversations. Do not filter out call recordings by activity origin.

### Step 3: Run Contact Research for Each Attendee

Follow the contact-research skill for each external attendee. Focus on their role and influence in the buying process, their own activity, recent signals of their current priorities, and their Spark persona if available.

### Step 4: Synthesize Talking Points and Objectives

- The **call objective** (discovery, demo, expansion, renewal, QBR)
- **3–5 talking points**, each grounded in a specific signal and, with company context, tied to the user's product
- **2–3 likely objections or topics** the customer may raise
- A **recommended outcome** for the call

### Step 5: Recency Check (Web Search)

After the Common Room research, check for anything newer than its data. Common Room drives the prep; web search only adds recency.

- **Company news:** `"[company name]" news`, last 14 days: funding, launches, leadership changes, layoffs, partnerships.
- **Attendees:** `"[full name]" "[company name]"`: recent articles, posts, talks or podcasts.

Put a significant item (a new round, a major hire) in Signal Highlights; mention the rest briefly.

### Step 6: After the Call

When the seller asks to log the call or what came out of it. Nothing is written without the seller's approval.

1. **Get the source.** In order: the transcript or call analysis from `~~conversation intelligence` (Zoom, Gong) if connected; call recording activity in Common Room; the seller's notes. If there is none, ask for notes. Never infer what happened from the calendar entry alone.
2. **Match attendees.** Look up each external attendee by email with commonroom_list_objects. Offer to save anyone not found with commonroom_create_object. External attendees only; never log activities on teammates.
3. **Pull out what happened,** with a quote or timestamp for each: meeting type, each attendee's role (only when the source shows it), product feedback and feature requests, objections, commitments with owners and dates, and the next meeting if one was booked.
4. **Propose updates, grouped by record:**
   - An activity per attendee with commonroom_create_object: demo → UserInputAttendedDemoAdded; other meeting → UserInputAttendedMeetingAdded; next meeting booked → UserInputBookedMeetingAdded; product feedback → UserInputProvidedProductFeedbackAdded; feature request → UserInputSubmittedFeatureRequestAdded. Body: one or two sentences from the source; occurredAt: the meeting time.
   - One note on the organization: summary, objections, next steps with owners and dates.
   - Role tags such as "Champion": look up existing tags with commonroom_list_objects (objectType Tag) and reuse them; create a missing one with commonroom_create_object (objectType Tag, entityType contact); apply them with commonroom_update_object.
5. **Write what's approved,** one approval per group, and report what was written and anything rejected, with the reason. Then offer a follow-up email (compose-outreach) or adding attendees to a segment.

## Output Format

Include only sections you have real data for. Never invent deal context: no proposals, competitor comparisons, pricing, trial terms or objections that a tool call didn't return.

```
## Call Prep: [Company] — [Date/Time if known]

**Meeting Context**
[Attendees, meeting type, and any known agenda]

---

### Company Snapshot
[4–6 bullets: key account status, signals, and recent activity]

---

### Attendee Profiles

**[Attendee Name] — [Title]**
[3–4 bullets: role, recent activity, Spark persona if available, personal hook]

[Repeat for each attendee]

---

### Signal Highlights
[Top 3 signals most relevant to this specific call]

---

### Talking Points
1. [Point tied to a specific signal]
2. [Point tied to a specific signal]
3. [Point tied to a specific signal]

### Likely Topics / Objections to Prepare For
- [Topic or objection + suggested response]
- [Topic or objection + suggested response]

### Recommended Call Outcome
[1–2 sentences: what success looks like for this meeting]
```

**Sparse data** (few properties returned, no activity, null `sparkSummary`): show only what was returned plus any web findings, list what's missing, and say what you can look up next. Don't fill the template.

**After the call (Step 6):**

```
## [Company] — [Meeting type], [date]

**Activities**
| Attendee | Activity type | Body | Source |
|---|---|---|---|

**Account note**
[Summary · Objections · Next steps with owners and dates]

**Tags**
| Attendee | Tag | Evidence |
|---|---|---|
```

## Reference Files

- **`references/call-types-guide.md`** — guidance for different call types (discovery, expansion, renewal, QBR) and how to tailor prep accordingly
