---
name: compose-outreach
description: "Generate personalized outreach messages using Common Room signals, and enroll the contact in a sequence when the seller wants to send. Triggers on 'draft outreach to [person]', 'write an email to [name]', 'compose a message for [contact]', 'enroll [person] in [sequence]', 'send this', 'add them to a sequence', or any outreach drafting request."
---

# Compose Outreach

**Company context** means what the user's company sells and who it sells to (its ICP). Load it only when a step uses it (fit, ICP or positioning), not for plain lookups. `Me.company` gives the company's name, domains and a summary; if that's empty, look the company up as an Organization, then search the web; if neither helps, ask the user once per session.

## Outreach Process

### Step 1: Look Up the Target

Retrieve the target company and/or contact from Common Room: recent product activity and engagement, community activity, third-party intent signals (job postings, news, funding) and relationship history (prior contact, meetings, email opens).

If the user named a person, research that contact. If only a company was given, pick the best contact by title, engagement and role.

### Step 2: Web Search for External Hooks (If CR Signals Are Thin)

When Common Room returned strong signals, use them and skip web search. When they're thin, or the user asks for external hooks, search:
- `"[company name]" funding OR acquisition OR launch OR announcement` — last 30 days
- `"[contact full name]" "[company name]"` — recent articles, interviews, posts or talks

Prefer hooks under two weeks old that are public and signal change (growth, new role, new product, new market).

### Step 3: Spark (If Available)

Read the contact's Spark summary (`sparkSummary`) for persona, background and influence signals, and use it to set tone and angle.

### Step 4: Identify the Best Hooks

Pick the 1–3 strongest hooks, ranked by recency (last 7–14 days), specificity (a concrete action they took, such as a community question or a new hire wave, not "I noticed you're a customer" or an industry trend) and relevance to what the user's product delivers.

### Step 5: Generate All Three Formats

Write an email, a call script and a LinkedIn message from the strongest hooks, clearly labeled, following `references/outreach-formats-guide.md`. Warm and conversational for inbound or community signals; more formal for cold or executive outreach. With company context, ground the value bridge in the user's product and positioning.

### Step 6: Annotate Your Choices

After the drafts, add 2–4 sentences: which signals were used and why, any assumptions, and alternative angles.

Then offer: "Want me to enroll [Name] in a sequence with this angle?"

### Step 7: Enroll (When the Seller Wants to Send)

Only when the seller asks to send or enroll. Never enroll unasked.

1. **Check the contact is saved.** Enrollment needs a Common Room contact ID. If the target came from web search or Prospector, offer to save them with commonroom_create_object first.
2. **Pick the sequence.** List enrollment destinations with commonroom_list_action_destinations (action enroll_in_sequence). Recommend the one whose name or steps fit the hook from Step 4, and say why. Ask for the sending mailbox when the provider needs one; Zoom Engage sends as the seller by default. If the Sequences category is disabled, stop and say a workspace admin can allow it.
3. **Reconcile the messages.** If the destination uses RoomieAI prompts, get each contact's draft with commonroom_get_message_drafts (up to 10 contacts per call). Compare each draft with the hook. Propose edits that work the hook in while keeping the admin's prompt and voice. Show the final snippets, and say that they fill RoomieAI slots in the sequence's own template (not the whole email), and that edits stay on the contact and are used in later enrollments too. If the destination has no RoomieAI prompts, say the sequence's existing content is sent, not the email drafted here.
4. **Enroll.** One approval covers the contacts, sequence, sender and final snippets. Call commonroom_enroll_in_sequence with contactIds and the edited messages. Say that this sends real outreach on the sequence's schedule.
5. **Check.** Read commonroom_list_action_logs with the runId and report each contact's outcome: succeeded, failed (with the reason) or pending. Don't retry automatically: a repeat starts a new run and can enroll contacts twice. Offer to rerun only the failed contacts, after the seller agrees.

More than 10 contacts, or a filter instead of names: hand off to contact-actions.

## Output Format

```
## Outreach for [Name / Company]

### 📧 Email

**Subject:** [Subject line]

[Email body — 3–5 sentences]

---

### 📞 Call Script

**Opening:**
[Opening line — conversational, 1–2 sentences]

**Value Bridge:**
[Why you're calling and why now — 2–3 sentences tied to a signal]

**Ask:**
[Single, low-friction ask — e.g., 15-minute call, specific question]

---

### 💼 LinkedIn Message

[Under 300 characters. Warm, personal, no pitch.]

---

### Signal Notes
[2–4 sentences: which signals were used, why, and any alternative angles]
```

After enrollment (Step 7), add:

```
### Enrollment
**Sequence:** [name] ([provider]) · **Sender:** [mailbox]
**Why this sequence:** [one sentence tied to the hook]

| Contact | RoomieAI field | Final message | Edited? |
|---|---|---|---|

**Result:** [succeeded / failed with reason / pending] per contact
```

## When Signal Data Is Sparse

If Common Room returns little on the target (no activity, no scores, no Spark), web search becomes the personalization source. If that also returns little, don't draft: show only what was found, say what's missing, and ask the user for prior-conversation context or a reason to reach out now. Never reference a signal that Common Room or web search didn't return, and don't offer enrollment without a real hook.

## Reference Files

- **`references/outreach-formats-guide.md`** — detailed format rules, examples, and tone guidelines for each channel
