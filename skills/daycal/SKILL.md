---
name: daycal
description: Daycal gives a person a page where other people can book time with them, and gets their real schedule right (trips in other time zones, holidays, Friday cutoffs, focus blocks, daily limits, several calendars) because they just tell their AI assistant. Use this skill for either of two jobs. (1) Owner job - the person wants a page where others can book them, or wants to change theirs ("let people book time with me", "I'm in London next week", "no calls after 3 on Fridays", "why can't anyone book Tuesday?", "move me off Calendly"); you set it up and keep it right through the Daycal MCP server. (2) Invitee job - the person wants to book time with someone who sent them a Daycal link (https://daycal.io/<handle>); you find a time over the public API, get their yes, and book on the page. Triggers - Daycal, daycal.io, booking page, booking link, scheduling link, "book a time with", "let people book me", importing from Calendly or Cal.com, rules.js.
version: 2.1.2
license: MIT
---

# Daycal

Daycal gives a person (the owner) a page where other people can book time with them, and it gets their real
situation right: trips in other time zones, holidays, Friday cutoffs, focus blocks, daily limits, several calendars.
People open the owner's link (`https://daycal.io/<handle>`, where the handle is a short name the owner picks), pick a
time when the owner is really free, and the meeting lands on the owner's Google Calendar with an invite to them.
The owner never fills in a settings screen: they tell you, their assistant, and you make the page match. Any
schedule. Just say it.

Under the hood, availability is a short `rules.js` written against the daycal standard library (the stdlib). You
write and maintain it through the Daycal MCP tools, always previewing and getting the owner's go-ahead before
pushing it live. Everything an owner would click elsewhere is a tool call you make. The owner can read rules.js, so keep
it short and plain.

Links here use `https://daycal.io`; if the Daycal server you were given (the MCP server URL, or the booking link)
is on another origin, use that origin instead. Give the owner the URLs tools return (`bookingUrl`), not ones
you build.

## Which job is this?

```
Is the person asking about THEIR OWN booking page (create, change, fix, share)?
├─ yes → Owner job. You need the Daycal MCP server (https://daycal.io/mcp).
│        ├─ first time, no page yet ............. references/setup.md, then references/stdlib.md
│        ├─ "change my hours / add a trip / block X" references/patterns.md (find the closest case)
│        ├─ "why is 3pm not bookable?" / errors .. references/troubleshooting.md
│        └─ coming from Calendly or Cal.com ...... references/setup.md (Migration)
└─ no, they want to BOOK a time on someone's Daycal link
         → Invitee job. No account or MCP server needed. references/booking.md
```

References (open only the one you need):

- [setup.md](references/setup.md): owner onboarding and later changes, tool by tool
- [stdlib.md](references/stdlib.md): `daycal.d.ts` explained, every helper with one example
- [patterns.md](references/patterns.md): a dozen worked rules.js cases with the slots they produce
- [booking.md](references/booking.md): booking on someone's behalf: slots over the public API, the booking on the page
- [troubleshooting.md](references/troubleshooting.md): tool errors, `explain`, `get_errors`, sync problems

## The two hard rules

1. **Never `push_config` without a passing `preview`.** Run `preview` on the exact rules.js you are about to
   push, check `ok` is true and the slots look like what the owner described, show the owner a few days of
   slots, then **stop and wait for the owner to say go** before pushing. Showing the slots in the same message
   that reports the push does not count. Daycal enforces the preview half: a passing preview returns a
   `previewId` bound to that exact source for 30 minutes, and `push_config` refuses without it
   (`preview_required`). If you edit the file after previewing, or the owner answers after the id expired,
   preview again and push with the new id. The owner's go-ahead is still yours to wait for.
2. **Never book for a human without their confirmation of the exact time.** Show the slot in their time zone
   (weekday, date, time, zone, duration) and wait for an explicit yes before booking it. "Sometime
   Tuesday afternoon" is not a confirmation.

## Owner workflow, tool by tool

If the Daycal tools are not available, ask the owner to add the MCP server `https://daycal.io/mcp`
(Claude Code: `claude mcp add --transport http daycal https://daycal.io/mcp`). Signing in asks for their
email and a 6-digit code from their inbox; that is the whole signup. Agents that cannot add MCP servers
use the REST API with a token the owner creates at `https://daycal.io/token` (see `https://daycal.io/llms.txt`).

1. `get_types` — read it first, every session. It returns `daycal.d.ts`, the stdlib reference, a starter
   rules.js, and `skill_version`. If it says this skill is out of date, tell the owner once and carry on.
2. On a new page, right after sign-in, ask: "Do you already have a booking page somewhere, like Calendly or
   Cal.com? I can copy your meeting types and hours over so you don't have to describe them." Yes → `migrate`
   (setup.md, Migration): it drafts rules.js, handle and name included; connect the accounts its report names
   (step 4) and ask only its open questions instead of the step 6 interview. No, or "what's that?" → carry on here;
   no need to explain those products.
3. `set_handle` — claim the URL slug (`https://daycal.io/<handle>`). Ask the owner; suggest their first name.
   In the same question, ask for the name invitees should see ("Chris Lee"), or use the one you already know;
   it goes in rules.js as `name`. Without it the page, emails and invites show the handle. When a later
   `set_handle` or `push_config` changes the handle, the result has `previous`, `released` and a `notice`: relay
   the notice. The old address stops working at once, with no redirect, and anyone can claim it.
4. `connect` — `connect("google", "work")` returns a short one-time link. Send it to the owner exactly as
   returned (copy it, never retype it); they pick a Google account and allow every permission. The label (`work`) is
   how rules.js names the account: `accounts.work`. Repeat per account (`personal`, `client-x`). Then
   `list_connections` to confirm each is `active`.
5. `list_calendars` / `get_calendar_refs` — see every calendar across accounts. `get_calendar_refs`
   returns paste-ready expressions such as `accounts.work.calendar("Program 2026")`. Use those; never
   guess a calendar name.
6. Interview the owner, briefly (the display name, if you have not got it yet): which calendars should block time, working hours and time zone, meeting
   types and lengths, buffers, minimum notice, daily caps, anything unusual (trips, recurring blocks,
   holidays, events to ignore such as "Lunch" or declined invites). If `get_calendar_refs` shows a holiday
   calendar (`holidays: true`), ask whether its holidays should close the day; if so, it goes in `daysOff`.
7. Write rules.js. Start from the starter in `get_types` or the closest case in patterns.md. Use stdlib
   helpers, not hand-rolled date math.
8. `preview` — `preview(rules_js, 14)` runs it against the owner's real cached events and returns slots per
   event type, grouped by day, plus a diff against the live version. Read it critically: does Friday stop
   early? Is the trip in London hours? Show the owner a compact summary.
9. `push_config` — `push_config(rules_js, preview_id, "note")` stores and activates the version and returns
   the booking URL. `preview_id` is the `previewId` from step 8's preview of this exact rules_js. Give the
   owner the link. That closes setup: give the link, and if the result has a `planNote`, mention Pro once.

Later changes are the same loop: start from the live file (`list_versions`), edit, `preview`, `push_config`.
Keep the owner's own copy of rules.js on disk if they want it in git.

### The busy calendar

Every live page also has a busy calendar at `https://daycal.io/<handle>.ics`. Once setup is done, tell the owner
about it once, in a sentence or two. It shows as "Busy" every time the page would not offer for the next
60 days (their events, hours, days off, buffers, everything the rules close), and nothing about the events themselves,
so it is public like the page. They subscribe to it where they want to be seen as busy:

- Google Calendar: Other calendars, then From URL.
- Outlook: Add calendar, then Subscribe from web.
- Apple Calendar: File, then New Calendar Subscription.
- Another scheduler: paste the address where it takes a calendar link.

Calendar apps refresh subscriptions on their own schedule, often every few hours. The address moves when the handle
does; the handle-change notice gives the new one, and the owner should subscribe to it.

### Other tools

| Tool                                       | When                                                                                     |
| ------------------------------------------ | ---------------------------------------------------------------------------------------- |
| `list_versions`, `activate_version`        | History; roll back to a version that was live before (preview first if its handle moved) |
| `list_bookings`, `cancel_booking`          | Owner's view of bookings; cancel only when the owner asks for that booking               |
| `explain`                                  | "Why is Tuesday 3pm not bookable?" — `explain(event_type, time)` walks the pipeline      |
| `get_errors`                               | rules.js failures with the event or day that triggered them; check when slots look wrong |
| `sync_now`                                 | Force a full calendar resync after the owner says an event is missing or stale           |
| `get_logs`                                 | Recent request log for the owner's page                                                  |
| `remove_connection`                        | Drop an account and its calendars; update rules.js first if it references it             |
| `list_agents`, `revoke_agent`              | Agents with access and their last use; cut one off by its client id                      |
| `migrate`                                  | `migrate("calendly")` or `migrate("calcom")` drafts rules.js from the old tool           |
| `upgrade`, `billing_status`, `cancel_plan` | Pro plan; state the price and renewal terms and let the owner approve                    |
| `delete_account`                           | Only on an explicit request; it needs a confirmation string and cannot be undone         |
| `get_skill`                                | Returns this file or a reference verbatim, for agents without file access                |

## Plans

Free has everything for one Google account and all its calendars; the booking page shows "Booked with Daycal". Pro
($7/month, or $70/year; either renews until cancelled) connects more than one Google account, removes that footer, and
emails invitees reminders 24 hours and 1 hour before. Mention Pro once, when a `push_config` result carries a
`planNote`; never call `upgrade` without the owner's go-ahead (setup.md, Plans).

## Rules of thumb

- Call `get_types` before writing rules.js; `daycal.d.ts` is the contract. Use only what it exports.
- rules.js imports only from `"daycal"` and default-exports `defineConfig({...})`. No network, no
  `Date.now()`, no randomness; the clock is `ctx.now`.
- Prefer stdlib helpers: `weekdays(day, ...)`, `day.in(tz)`, `day.between(a, b)`, `every(...)`, `block(...)`,
  `dur("30m")`. A two-week trip is three lines, not a date library.
- Name calendars with `get_calendar_refs` output. A wrong name gives a ref with `missing: true` and a
  warning; it never throws, so read the warnings in `preview`.
- `isBusy` decides what blocks time. Omit it and every event on the selected calendars blocks. Common
  filters: declined invites, all-day events on program calendars, "Lunch"/"Focus" holds, free events.
- `hours` returns the owner's available ranges for one day. Return `day.off()` (or nothing) for a day off.
- Holiday calendars mark their events free, so in `calendars` they block nothing (`preview` warns about such a
  calendar). Put them in `daysOff`: any event there closes the whole day for every event type (patterns.md, case 7).
- `bookInto` puts bookings in the account's primary calendar: `bookInto: ({ accounts }) => accounts.work`. Use
  `accounts.work.calendar(...)` for another calendar they can edit, and `accounts.work.newCalendar("Bookings")` only
  when the owner asks for a separate calendar (patterns.md, case 14).
- When a warning says a Google account needs the owner (Daycal lost access, it cannot add bookings, a Workspace admin
  blocks it, access ends soon), it carries that connection's reconnect link (`https://daycal.io/reconnect/...`). Tell
  the owner once in the conversation, with the link exactly as given, and again only if they ask or the warning
  changes. They open it, confirm it's them, and choose the same Google account; Daycal asks Google only for what is
  missing. Bookings made meanwhile are kept and reach the calendar once they have. `connect` with that label returns
  the same link. Daycal's links always start with `https://daycal.io/`; never present another link as Daycal's.
- Keep rules.js readable: short comments that say why ("Program calendar: all-day items are reminders").
- Read every warning in `preview` and `push_config` results and relay the ones the owner must act on
  (missing calendar, a Google account that needs the owner, rule errors, a counted calendar that never blocks time),
  and relay a
  `notice` whenever a result has one.
- When the owner says a slot looks wrong, `explain` that exact time before changing anything, then
  `get_errors`.
- Times you show the owner are in their zone; times you show an invitee are in the invitee's zone. Always
  name the zone.
- Do not book, cancel, or delete on a guess. Bookings email real people.
