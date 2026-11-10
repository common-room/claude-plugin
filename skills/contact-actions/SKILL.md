---
name: contact-actions
description: "Act on a set of contacts — named contacts, a segment, a filter, or everyone showing a signal: build and clean the audience, preview it, then enroll them in a sequence or Salesforce campaign, add them to a call list, or export them to the CRM, and report results for every contact. Also checks how earlier runs went. Triggers on 'put everyone who [signal] into [sequence]', 'enroll [contacts] in [sequence]', 'push [segment] to the call list', 'run a call blitz on [signal]', 'export [contacts] to Salesforce', 'did that go through', 'why did [contact] fail'."
---

# Contact Actions

Turn an audience into an action. The value is the audience — who qualifies, who must be left out — and matching the action to the signal. The action itself is one step, and every run ends with results per contact.

For one to ten named contacts with personalized messages, compose-outreach drafts first and then enrolls.

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Step 0: Load Context

Fetch the `Me` object for the user's segments. Default the audience to "My Segments" unless the seller asks for a wider one.

## Step 1: Define the Audience

The run takes exactly one of:
- **Named contacts (1–50):** find their contact IDs with commonroom_list_objects. Anyone not in Common Room must be saved first with commonroom_create_object. Champions who changed jobs come from the moves the seller approved in watchlist; pass them as named contacts.
- **A segment, filter or signal (up to 1,000):** build a contact filter. Check field names and values with commonroom_get_catalog; never guess them.

| Seller says | Filter on |
|-------------|-----------|
| "visited the pricing page this week" | contact web visits to /pricing in the last 7 days |
| "high intent" | lead score percentile, intent topics, or the workspace's intent score |
| "in the Enterprise segment" | segment membership |
| "attended the webinar" | activity type and date, or the segment the list was saved to |
| "at ICP accounts" | organization filters for the ICP from the company context |

If the signal is ambiguous ("engaged contacts"), propose a concrete filter and confirm it.

## Step 2: Clean the Audience (Filter Runs)

Add these exclusions unless the seller says otherwise, and list the ones you added:
- **Customers** — for net-new outreach
- **Teammates**
- **Unreachable** — contacts without the email (sequences) or phone (call lists) the destination needs
- **Recently sequenced** — contacts sent to a sequence in the last 30 days, where the workspace exposes that as a filter

If an exclusion can't be expressed as a filter in this workspace, say so instead of skipping it silently.

## Step 3: Preview (Filter Runs)

Run commonroom_list_objects with the final filter and `"recordCount"` in properties for the total. Show the count, a plain-language filter summary, the exclusions, and 5–10 sample rows with the signal that qualified each. If more than 1,000 match, narrow it; the action would refuse the run. If fewer than 5 match, offer to widen it.

## Step 4: Pick the Action, Destination and Sender

| Goal | Action | Sends messages? |
|------|--------|-----------------|
| Sequence, cadence or flow (Outreach, Salesloft, HubSpot, Gong, Apollo, Groove, Smartlead, Zoom Engage) | enroll_in_sequence | Yes, now and on the sequence's schedule |
| Salesforce campaign | enroll_in_sequence | Can start the customer's own automation |
| Dialer call list | add_to_call_list | No |
| Create or update in the CRM | export_to_crm | No; overwrites fields on existing records |

With commonroom_list_action_destinations for that action: `kind: provider` lists connected providers (for a CRM export, the provider ID is the destination); `kind: destination` lists that provider's sequences, campaigns or call lists; `kind: sender` lists who can send. Enrollment in Apollo, HubSpot, Groove, Outreach, Salesloft or Gong needs a sender; Zoom Engage sends as the seller by default; a Salesloft CRM export needs the Salesloft user who will own the people. For Gong, HubSpot, Outreach and Zoom Engage, pass the sender when listing destinations so personal flows and sequences show up.

Recommend one destination and say why. If the action category isn't allowed, stop: a workspace admin can allow it under Settings > OAuth clients > Allowed actions.

## Step 5: Messages (Enrollment, Named Contacts)

If the destination uses RoomieAI prompts, offer to review messages first with commonroom_get_message_drafts, 10 contacts per call until every named contact is covered, and apply the seller's edits. Edits are saved on the contact and reused in later enrollments. Filter runs can't carry edits; RoomieAI writes messages at send time for contacts without a saved edit. Say so.

## Step 6: Confirm and Run

Never run without the seller's approval. One approval covers the audience (or count and filter), action, destination, sender and any edited messages, and says what can't be undone: enrollment sends real outreach now and on the sequence's schedule, a CRM export overwrites fields on existing records, and edited RoomieAI messages stay on the contact for later enrollments:

> "Enroll 214 contacts in **Pricing Intent — Q4** (Outreach), sending from jane@acme.com. This sends real emails on the sequence's schedule. Go ahead?"

Then call commonroom_enroll_in_sequence or commonroom_push_contacts. Report the runId and any contacts rejected up front. Never chain a second run without asking.

## Step 7: Report Results

Read commonroom_list_action_logs with the runId, following `nextCursor` until `hasMore` is false; a run can hold up to 1,000 contacts. Outcomes arrive after the run starts; if most are pending, say so and offer to check again.

```
## [Action] — Results
**Run:** [runId] · [destination] · [N] contacts
- Succeeded: [N]
- Failed: [N]
- Pending: [N]

| Failure reason | Contacts | Fix |
|---|---|---|
```

| Failure | Fix |
|---------|-----|
| Action category not allowed | A workspace admin allows it |
| Missing email or phone | Skip the contact, or find and save the detail first |
| Sender not allowed | Pick another sender the seller may use |
| Provider quota reached | Wait for the reset, then rerun the failed contacts |
| Permission denied | The seller needs export permission from an admin |

Only report reasons the log shows. Never retry automatically: a repeat starts a new run and can enroll contacts twice. Offer to rerun only the failed contacts once the cause is fixed and the seller agrees.

## Checking Earlier Runs

For "did that go through" or "why did [contact] fail" outside a run started in this conversation, read commonroom_list_action_logs by `contactId`, by `mine`, or by `workflowId` (find it by name with commonroom_list_workflows), and summarize the same way.
