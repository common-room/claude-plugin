---
name: manager-report
description: "Help a sales manager see how their team uses Common Room — a weekly or monthly team snapshot, 1:1 prep for one rep, coaching flags (inactive, researching but not sending, task backlog), which plays the team worked, and trends — and turn any list in the report into a segment. Also serves admins reporting across teams. Use for 'how's my team doing', rep activity, adoption, leaderboards and play activity. For pipeline, meetings or revenue tied to that activity, use roi-attribution."
---

# Manager Report

Answers come from `commonroom_query_analytics`, which totals on the server: one call returns a small table. Never page through action rows to count things yourself.

## 1. Find "my team" (once per conversation)

1. `commonroom_list_objects` objectType `Me` → the caller's `id` and `personas`.
2. Teams they manage: objectType `Persona`, filter `{"type":"and","clauses":[{"type":"stringFilter","field":"managedBy","params":{"op":"eq","value":"<Me.id>"}}]}`. Pass those ids as `filters.personaIds`; sub-teams are included.
3. None returned → they manage no team in Common Room. Fall back to the team in `Me.personas` only if they say it's theirs, or ask which team.
4. Managers can query only the teams they manage and their sub-teams; anything else is denied. Say so plainly if a request fails that way. Admins and owners can query any team; for the whole workspace, list Personas (no filter, limit 1000) and pass every id with null `parentId`.

Other names: a team by name → list Personas and match, allowing abbreviations ("BDR" → "Business Development Representative"). A rep by name → `groupBy: ["rep"]` on the team and read `repId`. A play by name → objectType `Segment`, `stringFilter` on `segmentName` `like`; near-duplicates are common, so ask or say which you used.

## 2. Time

Pass `time.timeZone` (the user's zone). "This week" / "last month" → `start`/`end` with the zone's offset at each edge, end exclusive. The offset can differ between the two edges across a daylight-saving change. Rolling → `range` (`P7D`, `P30D`). Max one year. Managers think in weeks: default to the last full week, Monday to Monday, unless asked.

## 3. Manager questions

**Team snapshot** — "how's my team doing?"
1. Context: `actions` by `["rep","week"]` over the last 8 weeks including the current partial week. This shows who is new, dormant, bursty or collapsing. Read it before flagging anyone.
2. Per rep: `contactsResearched`, `recordsActioned`, `tasksMarkedDone`, `tasksDisqualified` by `["rep"]`. Lists every rep with `lastActiveAt` (latest page view) and `lastActionedAt` (latest action), both not limited to the window.
3. Sequence sends per rep: `records` by `["rep"]`, `filters.repMetrics: ["recordsAddedToSep"]`, `sepDestinations` = the sequence destinations (see **Sends**). These are distinct contacts, so they're lower than an `actions` count.
4. Open tasks: `tasksOpen` by `["rep"]`, `time.asOf` today and again a week ago, for the change.
5. Calls 2–3 for the previous period, for the change column.
Show one table: rep · researched · actioned · sent to sequence · tasks done · disqualified · open tasks (change) · change vs last period · last action. List reps with no activity for 90+ days below the table instead of in it, and leave them out of medians. Then the flags, then two or three sentences.

**Coaching flags** (name the rep and the number; compare to the team median, not the top rep):
- New or returning: no actions in the 8-week context before the last two weeks. The context can't tell a new rep from one back after a long gap, so say "new or returning"; don't flag them for low numbers.
- Dormant: no action for 90+ days (`lastActionedAt`). May have left the team; suggest checking.
- Active but not acting: page views in the window, 0 on every metric.
- Big drop: under half their own average of the previous 4 weeks, not just under last week.
- Researching but not sending: `contactsResearched` at or above the median and sequence sends under a quarter of the median.
- Disqualifying a lot: `tasksDisqualified` more than half of `tasksMarkedDone`.
- Backlog growing: `tasksOpen` up week over week while tasks done are below the median. Open-task totals are often whole segments assigned automatically, so the change matters, not the size.
- Outlier: a value over 10× the median usually means bulk or automated activity. Say "verify before reading as performance".
These are prompts for a conversation, not verdicts; say so once.

**1:1 prep** — one rep (if asked for "whoever is most behind", exclude new or returning and dormant reps and say who you picked and why):
1. Reuse their rows from the team snapshot if you have it; otherwise run snapshot steps 2–5 with `memberIds: [rep]`.
2. Trend: `records` by `["week","repMetric"]` with `repMetrics: ["contactsResearched","tasksMarkedDone"]` over the last 8 weeks, and sequence sends separately: `records` by `["week"]` with `repMetrics: ["recordsAddedToSep"]` and `sepDestinations` = the sequence destinations (see **Sends**). Fill missing weeks with 0 and list them oldest first.
3. What they worked: `records` by `["segment"]` with `repMetrics: ["recordsActioned"]` over the last 4 weeks.
4. Five recent actions (see **Drill-down**), with names and titles.
Output: their numbers beside the team median, the trend in one line, top plays, the recent actions, and 2–3 talking points drawn from the numbers.

**Which plays my team worked**: `records` by `["segment"]` with `repMetrics: ["recordsActioned"]` (or sequence sends per **Sends**), sort desc, limit 10; then `records` by `["segment","rep"]` with `segmentIds` for the top 3. Null segment = work outside any play.

**Trend**: `records` by `["week"]` or `["month"]`, plus `["rep"]` or `["repMetric"]`, with `filters.repMetrics`. One call, not one per period.

**Adoption** ("how many of my reps use it"): `repsActive`, `repCount`, `groupBy: []`, per team. There is no per-rep grouping; for one rep use `memberIds`.

## 4. Metric reference

| Metric | Counts | Groupings |
| --- | --- | --- |
| `contactsResearched` | Contacts the rep viewed, opened profiles of, copied email/phone of, emailed | `rep`, `team`, `segment`, pairs |
| `recordsActioned` | Records the rep messaged, sent somewhere, changed segment status on, completed a task for | same |
| `tasksMarkedDone` / `tasksDisqualified` | Segment tasks the rep completed / disqualified | same |
| `tasksOpen` / `tasksAssigned` | Tasks assigned now (latest daily snapshot) | `rep`, `team`; `time.asOf` |
| `recordsAddedToSep` | Contacts sent to each destination, success or not | must include `sepDestination` |
| `repsActive`, `repCount` | Reps with any action or page view / reps covered | `[]` (`repCount` also `team`) |
| `actions`, `records` | Action count / distinct records, filtered by `repMetrics` | any of rep, team, segment, sepDestination, repMetric, actionType, recordType, month, week |

`filters.repMetrics` and `sepDestinations` work only with `actions`/`records`; passing them with another metric errors with "applies only to repAction metrics".

**Sends.** A workflow often sends one contact to several destinations (an Outreach sequence plus a Salesforce export). Get the destination names in use once: `actions` by `["sepDestination"]` with `repMetrics: ["recordsAddedToSep"]`. Sequences are `addTo…Sequence`; `exportTo…` and `create…Prospect` create CRM records. Report sequence sends unless asked otherwise, and never add sequence and CRM numbers together.

**Reading results.** A missing rep/week/segment row means 0. `team` rows are each rep's own team, so sub-teams appear separately — roll them up if the manager thinks of one team. Team totals add per-rep values (a contact two reps worked counts twice). `records` counts a record once per rep per period, so weekly rows can sum to more than the whole window. On a 400, the message names the allowed combination; fix and retry once.

## 5. Drill-down

Only when asked, or five rows for 1:1 prep: `metrics: ["actions"]`, `groupBy: ["action"]`, same filters, `limit` 25 at most (larger pages exceed the response size). Rows: date, rep, action type, play, destination, `status` (failed sends included — show it), contact email, domain, `recordId`. For names and titles, `list_objects` with `id` and `properties: ["fullName","title"]`, one call per contact, so only for the rows shown.

## 6. Present

- Lead with one sentence a manager can act on ("Two of your eight reps sent nothing to sequences last week."), then the table, then flags.
- State the window, time zone and which teams are covered.
- Say what a metric counts when it matters ("sends count attempts, successful or not").
- Offer a chart for trends, as a chart or doc artifact if the client supports one.
- Activity isn't an outcome. For pipeline or meetings, hand off to roi-attribution.

## 7. Turn a list into a segment

When a drill-down gives the manager a list worth acting on — "contacts Sam researched but never sent", "records the team actioned in the Job Change play" — offer to save it as a segment.

1. Collect the `recordId`s from the drill-down rows (page with the cursor until the list is complete). Keep contacts and organizations separate; one segment holds one record type.
2. Create the segment with commonroom_create_segment (no filter, `entityType` `contact` or `organization` to match the records, named after the report and window), then add the records with commonroom_update_object, up to 100 per call.
3. Ask before creating: show the name, record type and count, and check for an existing segment of that name with commonroom_list_objects. Then report what was added and anything rejected, with the reason.
4. Offer the next step: enrolling the segment's contacts or adding them to a call list (contact-actions).
