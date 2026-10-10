---
name: caloo
description: Caloo is programmable scheduling - a booking page whose availability is a rules.js file. Use this skill for two jobs. (1) As the owner's agent, set up and maintain their Caloo booking page through the Caloo MCP server - sign in, connect Google calendars, write rules.js against the caloo stdlib, preview, push, change hours, add trips, debug a slot that looks wrong, roll back. (2) Book a time on someone else's Caloo page (https://caloo.ai/<handle>) on behalf of a person, through the public API. Triggers - Caloo, caloo.ai, booking page, booking link, scheduling link, availability rules, rules.js, "book a time with".
version: 1.0.0
license: MIT
---

# Caloo

Caloo turns an owner's calendars plus a short `rules.js` into a booking page at `https://caloo.ai/<handle>`.
Links here use `https://caloo.ai`; if the Caloo server you were given (the MCP server URL, or the booking link)
is on another origin, use that origin instead. Give the owner the URLs tools return (`bookingUrl`), not ones
you build. There is no settings UI: everything an owner would click is a tool call you make. The owner reads rules.js,
so keep it short and plain.

## Which job is this?

```
Is the person asking about THEIR OWN booking page (create, change, fix, share)?
├─ yes → Owner job. You need the Caloo MCP server (https://caloo.ai/mcp).
│        ├─ first time, no page yet ............. references/setup.md, then references/stdlib.md
│        ├─ "change my hours / add a trip / block X" references/patterns.md (find the closest case)
│        ├─ "why is 3pm not bookable?" / errors .. references/troubleshooting.md
│        └─ coming from Calendly or Cal.com ...... references/setup.md (Migration)
└─ no, they want to BOOK a time on someone's Caloo link
         → Invitee job. No account or MCP server needed. references/booking.md
```

References (open only the one you need):

- [setup.md](references/setup.md): owner onboarding and later changes, tool by tool
- [stdlib.md](references/stdlib.md): `caloo.d.ts` explained, every helper with one example
- [patterns.md](references/patterns.md): a dozen worked rules.js cases with the slots they produce
- [booking.md](references/booking.md): booking on someone's behalf through the public API
- [troubleshooting.md](references/troubleshooting.md): tool errors, `explain`, `get_errors`, sync problems

## The two hard rules

1. **Never `push_config` without a passing `preview`.** Run `preview` on the exact rules.js you are about to
   push, check `ok` is true and the slots look like what the owner described, show the owner a few days of
   slots, then **stop and wait for the owner to say go** before pushing. Showing the slots in the same message
   that reports the push does not count. If you edit the file after previewing, preview again.
2. **Never book for a human without their confirmation of the exact time.** Show the slot in their time zone
   (weekday, date, time, zone, duration) and wait for an explicit yes before `POST`ing a booking. "Sometime
   Tuesday afternoon" is not a confirmation.

## Owner workflow, tool by tool

If the Caloo tools are not available, ask the owner to add the MCP server `https://caloo.ai/mcp`
(Claude Code: `claude mcp add --transport http caloo https://caloo.ai/mcp`). Signing in asks for their
email and a 6-digit code from their inbox; that is the whole signup. Agents that cannot add MCP servers
use the REST API with a token the owner creates at `https://caloo.ai/token` (see `https://caloo.ai/llms.txt`).

1. `get_types` — read it first, every session. It returns `caloo.d.ts`, the stdlib reference, a starter
   rules.js, and `skill_version`. If it says this skill is out of date, tell the owner once and carry on.
2. `set_handle` — claim the URL slug (`https://caloo.ai/<handle>`). Ask the owner; suggest their first name.
3. `connect` — `connect("google", "work")` returns a short one-time link. Send it to the owner exactly as
   returned (copy it, never retype it); they pick a Google account and grant access. The label (`work`) is
   how rules.js names the account: `accounts.work`. Repeat per account (`personal`, `client-x`). Then
   `list_connections` to confirm each is `active`.
4. `list_calendars` / `get_calendar_refs` — see every calendar across accounts. `get_calendar_refs`
   returns paste-ready expressions such as `accounts.work.calendar("Program 2026")`. Use those; never
   guess a calendar name.
5. Interview the owner, briefly: which calendars should block time, working hours and time zone, meeting
   types and lengths, buffers, minimum notice, daily caps, anything unusual (trips, recurring blocks,
   holidays, events to ignore such as "Lunch" or declined invites).
6. Write rules.js. Start from the starter in `get_types` or the closest case in patterns.md. Use stdlib
   helpers, not hand-rolled date math.
7. `preview` — `preview(rules_js, 14)` runs it against the owner's real cached events and returns slots per
   event type, grouped by day, plus a diff against the live version. Read it critically: does Friday stop
   early? Is the trip in London hours? Show the owner a compact summary.
8. `push_config` — `push_config(rules_js, "note")` stores and activates the version and returns the
   booking URL. Give the owner the link.

Later changes are the same loop: start from the live file (`list_versions`), edit, `preview`, `push_config`.
Keep the owner's own copy of rules.js on disk if they want it in git.

### Other tools

| Tool                                       | When                                                                                     |
| ------------------------------------------ | ---------------------------------------------------------------------------------------- |
| `list_versions`, `activate_version`        | History; roll back by activating an older id (no preview needed: it ran before)          |
| `list_bookings`, `cancel_booking`          | Owner's view of bookings; cancel only when the owner asks for that booking               |
| `explain`                                  | "Why is Tuesday 3pm not bookable?" — `explain(event_type, time)` walks the pipeline      |
| `get_errors`                               | rules.js failures with the event or day that triggered them; check when slots look wrong |
| `sync_now`                                 | Force a full calendar resync after the owner says an event is missing or stale           |
| `get_logs`                                 | Recent request log for the owner's page                                                  |
| `remove_connection`                        | Drop an account and its calendars; update rules.js first if it references it             |
| `revoke_agent`                             | Cut off another agent's access (lists clients with last use)                             |
| `migrate`                                  | `migrate("calendly")` or `migrate("calcom")` drafts rules.js from the old tool           |
| `upgrade`, `billing_status`, `cancel_plan` | Pro plan; state the price and renewal terms and let the owner approve                    |
| `delete_account`                           | Only on an explicit request; it needs a confirmation string and cannot be undone         |
| `get_skill`                                | Returns this file or a reference verbatim, for agents without file access                |

## Rules of thumb

- Call `get_types` before writing rules.js; `caloo.d.ts` is the contract. Use only what it exports.
- rules.js imports only from `"caloo"` and default-exports `defineConfig({...})`. No network, no
  `Date.now()`, no randomness; the clock is `ctx.now`.
- Prefer stdlib helpers: `weekdays(day, ...)`, `day.in(tz)`, `day.between(a, b)`, `every(...)`, `block(...)`,
  `dur("30m")`. A two-week trip is three lines, not a date library.
- Name calendars with `get_calendar_refs` output. A wrong name gives a ref with `missing: true` and a
  warning; it never throws, so read the warnings in `preview`.
- `isBusy` decides what blocks time. Omit it and every event on the selected calendars blocks. Common
  filters: declined invites, all-day events on program calendars, "Lunch"/"Focus" holds, free events.
- `hours` returns the owner's available ranges for one day. Return `day.off()` (or nothing) for a day off.
- Keep rules.js readable: short comments that say why ("Program calendar: all-day items are reminders").
- Read every warning in `preview` and `push_config` results and relay the ones the owner must act on
  (missing calendar, revoked account, rule errors).
- When the owner says a slot looks wrong, `explain` that exact time before changing anything, then
  `get_errors`.
- Times you show the owner are in their zone; times you show an invitee are in the invitee's zone. Always
  name the zone.
- Do not book, cancel, or delete on a guess. Bookings email real people.
