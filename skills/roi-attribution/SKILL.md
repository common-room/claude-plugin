---
name: roi-attribution
description: "Tie Common Room rep activity to Salesforce outcomes (opportunities, pipeline, closed won). Agrees the customer's definitions first (stages, opp types, amount, match rule, timing), then counts outcomes and attributes them to rep actions where the data allows, and can save the attributed accounts as a segment. Use for ROI, attribution, 'did Common Room source or influence this pipeline', play or rep impact on deals, and QBR/EBR value reports. For activity-only questions use manager-report."
---

# ROI and Attribution

Every customer defines "pipeline", "won" and "touched by Common Room" differently. The definitions are the deliverable: agree them before computing, and repeat them with every number.

Tools: `commonroom_get_catalog`, `commonroom_list_objects` (CRM records), `commonroom_query_analytics` (rep actions), `commonroom_list_action_logs` (per-contact workflow history; only where contact actions are enabled).

## What can be answered

| Question shape | Path | Result |
| --- | --- | --- |
| Outcome counts and amounts (won, created, by month/type/stage) | Step 3 | Exact |
| One rep / one segment / one month of actions → which accounts opened opps | A | Account-level attribution |
| ≤ 10 named opps → was Common Room on them first | B | Contact-level, workflow actions only, needs action logs enabled |
| "How much of all our pipeline/won did Common Room source" | C | Outcomes + activity trend, labelled correlation |

Rep action rows can't be filtered by account, contact or domain, so whole-funnel attribution isn't possible yet. Say so up front when the question needs it, and offer the closest path above. Never estimate attribution from a sample and present it as a total.

## Step 0 — Basics for every analytics call

- `filters` needs `personaIds` or `memberIds`. Whole workspace: `list_objects` objectType `Persona`, every id with null `parentId`. Team names may be abbreviated ("BDR", "AE").
- Windows: at most one year; `start`/`end` with the user's offset (`2026-09-01T00:00:00-04:00`) and `timeZone`.
- Two call shapes. Per-rep counts: `metrics: ["recordsAddedToSep"]` (etc.), no `repMetrics`/`sepDestinations` filters. Action counts and rows: `metrics: ["actions"]` or `["records"]` with `filters.repMetrics` (values: `contactsResearched`, `recordsActioned`, `recordsAddedToSep`, `tasksMarkedDone`, `tasksDisqualified`) and optionally `sepDestinations`. Mixing them errors with "applies only to repAction metrics".
- Destination names in use: `actions` by `["sepDestination"]` with `repMetrics: ["recordsAddedToSep"]`. Sequences are `addTo…Sequence`; `exportTo…` and `create…Prospect` are CRM/record creation, often sent alongside the sequence — count sequences only unless the user says otherwise.
- A play is a segment: `list_objects` objectType `Segment`, `stringFilter` on `segmentName` `like`. Near-duplicate names are common: ask, or state which one you used.

## Step 1 — Find the outcome data

1. `commonroom_get_catalog` — the response is large; extract only the `objectTypeId` (`cot_…`) of the Opportunity object ("CRM Opportunity records"), Account ("CRM Account records") and any "Opportunity Contact Role" junction. Event/Task objects, if present, are meeting candidates. No Opportunity object → stop: Salesforce opportunities aren't mapped.
2. `list_objects` objectType `CustomField`, filter `entityType` eq `<cot_id>`, properties `["id","name"]`, for the Opportunity and then the Account. Opportunity: stage, amount, type / record type, created date, close date. Account: the domain/website field. No created-date field → "sourced" timing can't be checked; say so.

Field ids go in `properties` as `cf_…`; values come back under `customFields.cf_…`.

## Step 2 — Show real values, propose a definition, confirm

1. Filter to the opp type the user means first (most workspaces are mostly renewals). Read stage values from a small list: properties `[stage]`, `limit` 50, created in the last 6 months. Then surface the rest with `stringListFilter none [stages seen so far]` until it returns nothing, and repeat once for an older window — stage names drift ("7 - Closed Won" → "Closed Won").
2. Confirm candidate values with counts: same filter + `stringListFilter any [value]`, properties `["recordCount"]`, `limit` 1; read `total`.
3. If the question is window-wide (a quarter or more, many reps), size the actions now (Step 4) so you don't ask the user to confirm timing rules that can't be applied.

Propose in one message and ask the user to confirm or edit:

```yaml
definition: <customer> v1
outcome_object: Opportunity (cot_…)
fields: { created: cf_…, close: cf_…, amount: cf_…, stage: cf_…, type: cf_… }
opp_types: [New Business]
pipeline_stages: [...]
won_stages: [...]            # every naming variant
excluded: { name_contains: ["test"] }
match: Account domain (cf_…) = action's company domain, else contact email domain
sourced_actions: recordsAddedToSep to [sequence destinations]   # and/or tasksMarkedDone
influenced_actions: [contactsResearched, recordsActioned]
timing: { sourced: first sourced action before opp created, within 90 days;
          influenced: any action before close }
```

Keep it as text the user can paste into a later session. Every answer starts with `Definition: <name>` plus any per-question overrides.

## Step 3 — Outcome numbers

- Bucket ≤ 100 opps: one call with properties `[stage, amount, created, close, "recordCount"]`, `limit` 100; count and sum yourself.
- Larger buckets: `recordCount` with `limit` 1 per bucket (month × stage group × type); amounts by paging only the amount field.
- CRM date filters: `between`/`after`/`before` with ISO dates; use the day after the window's last day as `max` (for an end-exclusive window, its `end` date itself). `op: "in"` with a duration returns nothing on these fields.
- Report $0/blank amounts separately, and compare created timestamps: several opps created in the same second are a bulk import or migration, so "created after the action" is weak evidence for them.

## Step 4 — Attribution

Size first: `query_analytics` `metrics: ["actions"]`, no `groupBy`, with the slice's filters (`personaIds` or `memberIds`, plus `segmentIds` for a play, `repMetrics`, and `repMetrics: ["recordsAddedToSep"]` with `sepDestinations` when limiting to destinations).

**A. Action-first domain match** — slice up to ~150 actions (6 pages). Bigger: narrow (one rep, one segment, one month, sequence destinations only) and tell the user what was left out.
1. Rows: `metrics: ["actions"]`, `groupBy: ["action"]`, `limit` 25 (larger pages exceed the response size), follow `cursor` to the end. Only `status: "succeeded"` rows can source (`started` and `failed` can't); a contact can appear several times.
2. Domain per row: lowercase, strip any scheme, `www.` or path (`acme.com/legal` → `acme.com`). Match on `domain` (the company); use the `contactEmail` domain only when `domain` is null, and label those matches weak — for job changers the email is often the previous employer. Drop free-mail. Keep every action per domain (`startedAt`, rep, action, `status`); step 5 labels from them, and the first touch reported is the earliest action that qualifies. Count rows with neither.
3. Matched accounts: `list_objects` objectType `Object` with the Account `cot_`, `stringListFilter any` on its domain field (≤ 50 domains per call) **and** a nested group `{"type":"and","target":"Object","objectConfigId":"<opp cot>","targetAssocPaths":null,"clauses":[type/stage filters, created after window start]}`. Properties: name, domain field.
4. Their opps: opportunity `Object` filtered by created after window start with a nested Account group, domain `any [matched domains]`. Properties: name, stage, amount, created, close.
5. Label: **sourced** = a successful sourced action on the domain before created and within the lookback; **influenced** = any action before close; else not attributed. This is account-level: the opp is at a company the rep acted on, not necessarily from those contacts.
6. Upgrade to contact-level for the matched opps with B.1: if a contact the rep acted on (`recordId`) is on the opp, say so — that's the strongest evidence this path gives.

**B. Outcome-first spot check** — ≤ 10 named opps.
1. Contacts on opps (works without action logs): `list_objects` Contact whose filter is a group with `target: "Object"`, the Opportunity Contact Role `cot_`, containing a group with `target: "Object"`, the Opportunity `cot_`, containing `stringFilter` `objectName` `eq <opp name>` (an `or` group of these for several opps; `stringListFilter` on `objectName` errors). Put the junction in explicitly; skipping it returns nothing. Names repeat ("Renewal", "Expansion"): first count opportunities with that name; if more than one matches, add the Account group (its domain) to the filter, or ask which one.
2. Only if `list_action_logs` works (test one contact; "Contact actions are not enabled" → stop here): `contactId`, `since` = created − lookback, following `nextCursor` until `hasMore` is false. Earliest `succeeded` log before created whose action matches the agreed `sourced_actions` = sourced candidate; ignore other workflow steps. Covers workflow sends and segment changes, not research. About 2 calls per contact.

**C. Correlation** — outcomes by month (Step 3) beside activity by month (`records` by `["month"]` with the sourced `repMetrics`), labelled **correlation, not attribution**. These counts include failed and in-progress sends, so label them attempts, not successful actions. A CRM lead-source field can be shown as the customer's own channel label, never as Common Room attribution. Offer path A on the top rep or play.

## Step 5 — Present

1. `Definition: …` line.
2. Headline, e.g. "4 of 25 accounts the rep sequenced in September opened New Business opps afterwards ($130k open pipeline)."
3. Per-opp table: opp, domain, created, stage, amount, label, first touch (rep, action, date), days from first touch to opp.
4. Coverage: path, slice and what it excludes, actions counted (successful only), domain caveats (multiple domains per company, subsidiaries, job changers, unmatched rows), $0 amounts, bulk-created opps, test records.
5. For "is this play worth it": there's no control group, so say whether the evidence is thin or strong and what would firm it up (contact-level matches, a second month, a comparable play) rather than a yes/no.

## Step 6 — Save the result as a segment

Offer it when the attribution produced a list worth acting on: "accounts that opened opps after the Job Change play" (a lookalike seed for prospect) or "accounts the team sequenced that never opened one" (a re-engagement list).

1. Find the matched Account records' organizations in Common Room by domain with commonroom_list_objects (objectType Organization).
2. Create the segment with commonroom_create_segment (entityType organization, no filter), named after the definition, slice and window; add the organizations with commonroom_update_object, up to 100 per call.
3. Ask before creating: show the name and count, and check for an existing segment of that name with commonroom_list_objects. Then report what was added and anything rejected, with the reason.
