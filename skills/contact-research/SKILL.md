---
name: contact-research
description: "Research a specific person using Common Room data, including people not yet in your workspace. Triggers on 'who is [name]', 'look up [email]', 'research [contact]', 'is [name] a warm lead', 'what does [email] do', or any contact-level question."
---

# Contact Research

When the person isn't in Common Room, look them up in Prospector by name and company instead of stopping.

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Step 1: Locate the Contact

Query Contact with commonroom_list_objects:

| What the user gives | Filter |
|---------------------|--------|
| Email address | `memberEmail` `eq` the full address (most reliable) |
| Name + company | `fullName` `like` the name and `companyWebsite` `eq` the company's domain; show the matches if several |
| Name only | `fullName` `like` the name; if several match, show a brief list and ask which |
| LinkedIn, Twitter/X or GitHub handle | No Contact filter takes a handle. Ask for an email, or a name and company |

If no match is found, say: "Common Room doesn't have a record for this person." Then:
- **Name and company given:** go to Step 1.5.
- **Email only:** use its domain as the company and ask for the name, then go to Step 1.5.
- **LinkedIn URL, Twitter/X or GitHub handle, or name only:** ask for the name and company. Prospector can't be searched by email or profile URL.

## Step 1.5: Look Up People Not in Common Room

Search `ProspectorContact` with `fullName` `like` the name and `companyDomain` `eq` the company's domain, requesting `title`, `companyName`, `seniority` and `linkedInHandle`. If several match, show them and ask which. Reading Prospector adds nothing to the workspace.

If found and their company isn't in Common Room, also find it: `ProspectorCompany` with `groupWebsite` `eq` the domain (exact match; `like` returns unrelated companies), requesting `description`, `employees`, `revenueRange`, `subIndustry`, `technologies` and `location`. Then produce the "Not in Common Room" output below. Prospector has no activity, scores or Spark — skip Steps 2–5.

If not found, say so and offer a web search on their name and company.

For a contact already in Common Room, `jobChangeDate` and `previousCompany` show a job change; `currentCompany` carries the new company's `domain`. If they were a champion, check the new company: in Common Room with commonroom_list_objects (owner, customer status, open opportunity), otherwise in Prospector (`ProspectorCompany`, `groupWebsite` `eq` its domain, requesting `description`, `employees`, `revenueRange`, `subIndustry`, `technologies` and `location`). Rate its fit against the company context. If the new company is a customer or has an open opportunity, tell its owner instead of offering outreach. Otherwise, if it's worth acting on, offer outreach (compose-outreach) or enrollment (contact-actions).

## Step 2: Fetch Contact Properties

Call commonroom_get_catalog for the Contact properties this workspace has, then commonroom_list_objects with only the `properties` the question needs:

- `leadScores` — report the raw score or percentile, never a label.
- `recentActivities` — the contact's own activities, newest first, not your team's. For a set window (default: last 60 days), query Activity with `contactId`, `activityTime` and `activityIsUserInitiated` true.
- `recentWebVisitsNumber` and `recentWebPages` — visit total and top 5 pages for the last 12 weeks.
- `sparkSummary` — Spark. Request `recentSparkSummaries` as well when the user asks how engagement has changed over time.
- `segments`, `tags`, `profiles`, `linkedInUrls`, `jobChangeDate`, `previousCompany`, `currentCompany`

## Step 3: Read Spark (If Available)

Spark gives professional background, influence signals and a persona: Champion, Economic Buyer, Technical Evaluator, End User or Gatekeeper. If `sparkSummary` is null but real activity exists (recent actions, website visits), infer a persona from it and say it's inferred. With neither, the persona is Unknown — never guess one from title alone.

## Step 4: Assess Account Context

Pull a short snapshot of the contact's company: open opportunities, expansion or churn signals, whether other contacts there are active, and how this person's engagement compares to theirs.

## Step 5: Identify Conversation Angles

Pick the 2–3 strongest hooks: a recent activity of their own, a page they visited that signals evaluation, a job change or company news, their Spark persona, or their role in an active deal.

## Output Format

Include only sections data was returned for; omit the rest.

```
## [Contact Name] — Profile

**Overview**
[2 sentences: who they are, their role, and relationship status]

**Details**
- Title: [title]
- Company: [company]
- Email: [email]
- LinkedIn: [URL]
- Other profiles: [Twitter/X, GitHub, CRM link if available]

**Scores** [If scores returned]
[All scores as raw values or percentiles]

**Recent Activity** (last 60 days) [If activity returned]
[3–5 bullets with dates; lead with this, and flag it when the newest is over 30 days old]

**Website Visits** (last 12 weeks) [If visit data exists]
[Total visit count + list of pages visited]

**Spark Profile** [If Spark data is non-null]
[Persona type, background summary, influence signals]

**Segments** [If segments returned]
[List of segment names this contact belongs to]

**Account Context**
[1–2 sentences on their company's status]

**Conversation Starters**
[2–3 specific, signal-backed openers]
```

**Sparse data** (for example only name, title, email and tags; `sparkSummary` null): show only what was returned, list what's missing, and add any web findings on their name and company. Don't fill the template.

**Not in Common Room (Step 1.5):**

```
## [Name] — Prospector (not in your workspace)

**Person:** [role, company — only fields returned]
**Company:** [employees, sub-industry, technologies, location — only fields returned]

**Next step:** "Want me to save [Name] as a contact in Common Room?"
```

If the user agrees, create the contact with commonroom_create_object, passing prospectorContactId.

Conversation starters, persona inferences and engagement assessments need real signals; don't produce them from sparse data or Prospector.

**Saving.** Ask before every write, and show the fields and values. Contacts upsert, so saving one that already exists updates it. Never invent contact details. Report what was saved and anything rejected, with the reason.

## Reference Files

- **`references/contact-signals-guide.md`** — full field descriptions, Spark persona guide, and conversation starter principles
