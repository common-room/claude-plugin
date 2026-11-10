# Common Room Plugin

GTM workflows powered by Common Room — account research, contact research, call prep and follow-up, personalized outreach, prospecting, account and champion watchlists, contact actions, manager reporting, ROI attribution, and weekly briefings.

## Overview

This plugin connects Claude to Common Room's MCP server and equips it with ten skills covering the most common rep workflows. Every output is grounded in real Common Room signal data — 1st-party product signals, 2nd-party community signals, 3rd-party intent signals, and research from RoomieAI and Spark.

## Requirements

- **Common Room MCP** (`mcp.commonroom.io/mcp`) must be connected and authenticated. This is the primary data source for all plugin functionality.
- **Calendar connector** (optional) — enables automatic meeting lookup in `call-prep` and `weekly-prep-brief`. If not connected, both skills ask the user for meeting details instead.
- **Conversation intelligence connector** (optional) — Zoom or Gong transcripts for logging calls in `call-prep`. Without one, the skill asks for your notes.
- **Allowed actions** — sequence enrollment, call lists and CRM export need a workspace admin to turn them on under Settings → OAuth clients → Allowed actions.
- **Salesforce opportunities** — `roi-attribution` needs opportunities mapped into Common Room.

## Skills

Skills are triggered conversationally. Describe what you want and Claude will load the right skill automatically.

| Skill | Trigger phrases |
|-------|----------------|
| `account-research` | "Research [company]", "tell me about [domain]", "who's on the buying committee at [company]" |
| `contact-research` | "Who is [name]", "look up [email]", "research [contact]", "is [name] a warm lead" |
| `call-prep` | "Prep me for my call with [company]", "log my call with [company]", "what came out of my call" |
| `compose-outreach` | "Draft outreach to [person]", "write an email to [name]", "compose a message for [contact]" |
| `prospect` | "Find companies that match [criteria]", "build a prospect list", "qualify this attendee list" |
| `watchlist` | "What changed on my accounts", "which of our champions changed jobs", "what should I work on today" |
| `contact-actions` | "Put everyone who [signal] into [sequence]", "push [segment] to the call list", "did that go through" |
| `manager-report` | "How's my team doing", "prep me for a 1:1 with [rep]", "which plays did my team work" |
| `roi-attribution` | "How much closed-won have we had since [date]", "did the [play] lead to pipeline" |
| `weekly-prep-brief` | "Weekly prep brief", "prepare my week", "what calls do I have this week" |

## Commands

Two commands for complex workflows that benefit from explicit invocation:

| Command | Usage |
|---------|-------|
| `/generate-account-plan <company>` | Comprehensive strategic account plan with stakeholder mapping, engagement analysis, opportunities, risks, and action items |
| `/weekly-brief [date range]` | Generate a full weekly prep briefing (defaults to next 7 days) |

## What Each Skill Produces

**Account Research** — Handles four patterns: full overviews, targeted field questions, honest sparse-data responses, and combined MCP data + LLM reasoning. Includes web search for recent news. Automatically scopes to "My Segments." Finds companies not yet in Common Room in Prospector and maps the buying committee, with candidates for the gaps.

**Contact Research** — Lookup by email or name+company. Returns identity, CRM fields, scores, website visits, activity history, Spark analyses, and conversation starters. Finds people not yet in Common Room in Prospector by name and company.

**Call Prep** — Company snapshot, per-attendee profiles, signal highlights, tailored talking points, likely objections, and recommended call outcome. Prioritizes Gong/call recording activity. Calendar-aware if connected. After the call, logs activities, an account note and role tags from the transcript or your notes.

**Compose Outreach** — Three personalized formats (email, call script, LinkedIn message) grounded in Common Room signals and web search hooks. Tailored to the user's company positioning when available. Enrolls the contact in the right sequence, with RoomieAI messages you can review and edit first.

**Prospecting** — Distinguishes between net-new companies (ProspectorCompany) and existing accounts (Organization). Supports iterative refinement and lookalike search ("find companies like [X]"). Prospector firmographics fill in net-new results; paste a list to qualify it against your ICP; save the fits to a segment.

**Weekly Prep Brief** — Full briefing covering every external call in the next 7 days: company snapshot, attendee profiles, signals, and recommended objectives per meeting.

**Watchlist** — Sweeps your accounts and champions for what changed since the last check: intent spikes, new people engaging, buyers who left, champions who moved to a new company. Scores each champion's new company against your ICP, ranks what to act on, and hands the follow-up to research, outreach or Contact Actions. Works well as a scheduled run.

**Contact Actions** — Builds an audience from named contacts, a segment, a filter or a signal; adds exclusions; previews the count and a sample; enrolls them in a sequence or campaign, adds them to a call list, or exports them to the CRM with one approval; and reports results per contact, with fixes for failures.

**Manager Report** — Team snapshot with coaching flags against the team median, 1:1 prep, which plays the team worked, and trends. Any list in the report can become a segment.

**ROI Attribution** — Agrees your definitions of pipeline, won and "touched" first, then counts Salesforce outcomes and attributes them to rep activity by account or contact, labels correlation honestly, and can save the attributed accounts as a segment.

## Setup

1. Ensure the Common Room MCP server is connected and authenticated in your Cowork settings.
2. (Optional) Connect a calendar MCP server for automatic meeting lookup in call prep and weekly briefings.
3. Install this plugin. All skills and commands are available immediately.

## User Context

All skills that scope to a user's territory automatically fetch the `Me` object from Common Room. This provides the user's profile, role, and "My Segments" — ensuring queries default to their territory. See `references/me-context.md` for details.

When company context is available, skills tailor recommendations to the user's product and ICP. See `references/my-company-context.md` for details.

## Customization

See `CONNECTORS.md` for details on the calendar connector and how tool references work.
