# The daycal stdlib

`get_types` returns `daycal.d.ts`, the exact contract. This page explains it with one example per helper.
rules.js is an ES module that imports only from `"daycal"` and default-exports `defineConfig({...})`. It has
no network, no clock other than `ctx.now`, and no randomness. Plain modern JavaScript (ES2023) works.

## How a run works

For each request Daycal runs rules.js in a sandbox and asks it, in order:

1. `calendars(ctx)`: which calendars count for conflicts.
2. `hours(day, ctx)`: once per owner-local day in range, per event type.
3. `isBusy(event, ctx)`: once per event on those calendars, after cancelled events are dropped.
4. `extraBusy(ctx)`: once per event type, for blocks your calendars do not know about.
5. `eventTypes(ctx)` and `bookInto(ctx)`.

The slot engine then takes the hours, subtracts busy events (plus buffers) and existing bookings, applies
minimum notice, the daily cap and the booking horizon, and offers what is left. If rules.js throws, the
page does not go blank: every event counts as busy and hours come from the last good version, and the
error shows up in `get_errors`.

## defineConfig(cfg)

Validates the shape and returns it. Required: `handle`, `timezone`, `calendars`, `hours`, `eventTypes`,
`bookInto`. Optional: `name`, `daysOff`, `isBusy`, `extraBusy`, `booking`, `webhooks`. Fields typed `Fn<T>` take a value or a
function of `ctx`.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
  booking: { confirmation: "Talk soon!", note: true, reminders: true, ownerEmails: true },
  webhooks: { onBooking: "https://hooks.example.com/daycal" },
});
```

- `name`: the owner's name as invitees see it on the page, in emails and in invites ("Chris Lee"); 1-80
  characters once trimmed, on one line (no control characters, line breaks or hidden characters such as bidi
  overrides and zero-width spaces). Omit it and they see the handle.
- `booking.confirmation`: text on the confirmation screen and email (the email quotes it on one line).
  `booking.note`: collect an optional note (default true).
  `booking.reminders`: on Pro, Daycal emails each invitee a reminder 24 hours and 1 hour before the meeting (default
  true; the free plan sends none either way). `false` turns them off for every booking, those already made included;
  `true` again resumes the ones still ahead, never ones whose time passed while off (patterns.md, case 15).
  `booking.ownerEmails`: Daycal emails the owner's account address about each new booking, and each move or
  cancellation made through its reschedule-or-cancel link, with the time and the calendar it went on (default true,
  every plan). `false` stops them from the next change on.
- `webhooks.onBooking`, `webhooks.onCancel`: URLs that receive a JSON POST per booking or cancellation.

## ctx

Every function gets `ctx` (as the only argument, or the second one for `hours` and `isBusy`).

| Member          | Meaning                                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------------- |
| `ctx.now`       | Current time, ISO instant. The only clock.                                                                          |
| `ctx.timezone`  | The owner's zone from `timezone`.                                                                                   |
| `ctx.range`     | The days being computed: `range.from`, `range.to` (owner-local dates, inclusive), `range.startUtc`, `range.endUtc`. |
| `ctx.accounts`  | Connected accounts by label.                                                                                        |
| `ctx.eventType` | The event type being computed (inside `hours`, `isBusy`, `extraBusy`).                                              |

```js
extraBusy: ({ range }) => every("day", range, "12:00-13:00", { reason: "lunch" }),
```

## Accounts and calendars

`ctx.accounts` maps the label given to `connect` to an account. Each account has `account.label`,
`account.email`, `account.provider`, `account.primary` (its primary calendar), `account.calendars` (every
calendar in its list, shared ones included), `account.calendar(nameOrId)`, and `account.newCalendar(name)` (only
for `bookInto`).

```js
calendars: ({ accounts }) => [
  accounts.work.primary,
  accounts.work.calendar("Program 2026"),     // the one calendar with this id, or this name ignoring case
  ...accounts.personal.calendars.filter((c) => c.accessRole === "owner"),
],
```

A calendar ref has `id`, `name`, `account` (label), `accessRole` (`owner`, `writer`, `reader`,
`freeBusyReader`), `missing` and `holidays` (true for a public holiday calendar, a Google id ending
`#holiday@group.v.calendar.google.com`). A name that matches nothing returns a ref with `missing: true` and a
warning in `preview`. A string more than one calendar of the account answers to, by name or by id (anyone can share
a calendar into it under any name), fails the run instead of picking one: pass the calendar's id, which
`get_calendar_refs` then gives. Once the rules are live, each such string keeps the calendar it picked then; if it
later picks another (say a calendar shared in under the name of one the owner renamed), the page keeps its last good
calendars and `get_errors` shows the rule error until you preview and push the rules again. Compare refs with `is`,
never `===`:

```js
if (event.calendar.is(accounts.work.calendar("Program 2026"))) return false;
```

`calendar.is()` also accepts a calendar id or name string, read exactly as `account.calendar()` reads it (a name more than one calendar has fails the run): `event.calendar.is("Program 2026")`.

### daysOff

Calendars whose events close whole days (value or function, like `calendars`). Any non-cancelled event on them
closes every owner-local day it falls on, for every event type, whatever its `transparency` or `isBusy` says.
Holiday events are marked free, so a holiday calendar in `calendars` blocks nothing (and `preview` warns that it
never blocks time); put it here. `explain` shows such a day as a "day off" step naming the event.

```js
daysOff: ({ accounts }) => accounts.work.calendars.filter((c) => c.holidays),
```

## day (inside hours)

`hours(day, ctx)` is called once per owner-local day in range and returns that day's available ranges, as
`Hours` from a helper, an array of `"HH:MM-HH:MM"` strings, or `null`/`undefined` for a day off.

| Member                    | Example                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------- |
| `day.date`                | `"2026-10-27"`                                                                      |
| `day.weekday`             | `"tue"` (`mon` … `sun`)                                                             |
| `day.timezone`            | The zone hours are read in; the owner's unless `day.in()` changed it.               |
| `day.is(...)`             | `day.is("fri")`, `day.is("sat", "sun")`, `day.is("2026-12-25", "2026-12-26")`       |
| `day.between(from, to)`   | `day.between("2026-10-27", "2026-11-14")`, inclusive on both ends                   |
| `day.in(tz)`              | `day.in("Europe/London").hours("10:00-18:00")`: hours read in London, this day only |
| `day.hours(...ranges)`    | `day.hours("09:00-12:00", "13:00-17:00")`                                           |
| `day.weekdays(...ranges)` | `day.weekdays("09:00-17:00")`: those ranges Mon–Fri, off at weekends                |
| `day.weekends(...ranges)` | `day.weekends("10:00-14:00")`: those ranges Sat–Sun, off on weekdays                |
| `day.off()`               | Not available                                                                       |

Time ranges are `"HH:MM-HH:MM"` on a 24-hour clock. `"00:00-24:00"` is the whole day; an end at or before
the start runs past midnight (`"22:00-02:00"`).

```js
hours: (day) => {
  if (day.is("2026-12-24", "2026-12-25")) return day.off();
  if (day.is("fri")) return day.hours("09:00-15:00");
  return day.weekdays("09:00-12:00", "13:00-17:00");
},
```

## weekdays(day, ...ranges), weekends(day, ...ranges)

The common week in one call; the same as `day.weekdays(...)` and `day.weekends(...)`.

```js
hours: (day) => weekdays(day, "09:00-17:00"),
```

```js
hours: (day) => weekends(day, "10:00-14:00"),   // a weekend-only coaching page
```

## event (inside isBusy)

`isBusy(event, ctx)` returns true when the event blocks time. Omit `isBusy` and every event on the
selected calendars blocks. Cancelled events never reach it.

| Member                                               | Meaning                                                                                            |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `event.id`                                           | Provider event id                                                                                  |
| `event.calendar`                                     | The calendar ref (use `event.calendar.is(...)`)                                                    |
| `event.account`                                      | Account label                                                                                      |
| `event.title`, `event.description`, `event.location` | Empty strings on free/busy-only calendars                                                          |
| `event.start`, `event.end`                           | ISO instants (UTC)                                                                                 |
| `event.allDay`                                       | All-day event                                                                                      |
| `event.transparency`                                 | `"free"` when marked free / transparent, else `"busy"`; tentative and out-of-office count as busy  |
| `event.rawStatus`                                    | The provider's own status: `opaque`, `transparent`, `tentative`, `oof`, `workingElsewhere`, …      |
| `event.response`                                     | The owner's response: `accepted`, `declined`, `tentative`, `needsAction`, `none` (not an attendee) |
| `event.attendeeCount`                                | Number of attendees                                                                                |
| `event.organizerIsMe`                                | The owner organized it                                                                             |
| `event.recurringEventId`                             | Series id, or `null`                                                                               |

```js
isBusy: (event) => event.response !== "declined" && event.transparency !== "free",
```

## eventTypes

An array (or a function of `ctx` returning one) of event types. Each has a `slug` (the URL segment),
`title`, `duration`, and optional fields:

| Field                         | Meaning                                                                                                   |
| ----------------------------- | --------------------------------------------------------------------------------------------------------- |
| `description`                 | Shown on the booking page                                                                                 |
| `slotStep`                    | Minutes between slot starts, counted from the start of each block of hours. Defaults to min(duration, 30) |
| `bufferBefore`, `bufferAfter` | Free time required around the meeting; buffers must not overlap busy time                                 |
| `minNotice`                   | Earliest bookable time, relative to now (default 0)                                                       |
| `maxPerDay`                   | Bookings of this type per owner-local day (default unlimited)                                             |
| `maxDaysAhead`                | Booking horizon in days (default 60)                                                                      |
| `conferencing`                | `"google-meet"` (default), `"none"`, or `{ kind: "link", url }`                                           |
| `location`                    | Free-form location for the invite (an address, a phone number)                                            |

```js
eventTypes: [
  { slug: "intro", title: "Intro call", duration: dur("20m"), minNotice: dur("1d"), maxPerDay: 3 },
  { slug: "coffee", title: "Coffee", duration: dur("45m"), conferencing: "none", location: "Blue Bottle, Hayes St" },
],
```

Inside `hours`, `isBusy` and `extraBusy`, `ctx.eventType` is the type being computed, so rules can differ
per type: `ctx.eventType.slug === "sales"`.

## bookInto

Where bookings go. Each booking becomes an event with the invitee as attendee, so Google sends the invitation. A
value or a function of `ctx`:

```js
bookInto: ({ accounts }) => accounts.work,                              // the primary calendar (usual)
bookInto: ({ accounts }) => accounts.work.calendar("Team bookings"),    // a calendar the account can edit
bookInto: ({ accounts }) => accounts.work.newCalendar("Bookings"),      // a calendar Daycal creates, on request
```

- An account means its primary calendar.
- A calendar ref must have `accessRole` `owner` or `writer`; a read-only, free/busy or missing one fails
  validation with an error naming it.
- `newCalendar(name)` (1 to 60 characters, no line breaks) is for owners who ask to keep bookings apart. Daycal
  creates the calendar in that account with the first booking and reuses it for every later booking under the same
  name. It does not need to be in `calendars`: Daycal counts its own bookings as busy. Use a fixed name, never one
  built from `ctx`: Daycal creates at most 10 calendars per account, and a booking for an 11th name is not written.
- Changing `bookInto` moves only future bookings; booked events stay where they were written.

## extraBusy(ctx)

Busy blocks the calendars do not know about, as an array of `{ start, end, reason? }` with ISO instants.
Build them with `every` and `block` rather than by hand.

## every(weekday, range, timeRange, opts?)

A recurring block expanded over the range. `weekday` is one name, an array, or `"day"` for every day.
`opts.timezone` reads the time in another zone.

```js
extraBusy: ({ range }) => every(["mon", "wed"], range, "14:00-15:00", { reason: "school run" }),
```

## block(date, timeRange, opts?)

A one-off block on an owner-local date.

```js
extraBusy: () => block("2026-11-03", "09:00-12:00", { reason: "dentist" }),
```

Combine with spread: `[...every("day", range, "12:00-13:00"), ...block("2026-11-03", "09:00-12:00")]`.

## dur(spec)

Durations in minutes: `dur("30m")` is 30, `dur("1h30m")` is 90, `dur("4h")` is 240, `dur("2d")` is 2880.
A bare number is minutes.

```js
{ slug: "deep-dive", title: "Deep dive", duration: dur("1h30m"), bufferAfter: dur("15m") }
```

## date

Owner-local calendar arithmetic, so you never need `Date`.

| Helper                  | Example                                           |
| ----------------------- | ------------------------------------------------- |
| `date.addDays(d, n)`    | `date.addDays("2026-10-27", 7)` is `"2026-11-03"` |
| `date.weekday(d)`       | `date.weekday("2026-10-27")` is `"tue"`           |
| `date.of(instant, tz?)` | `date.of(ctx.now)` is today's owner-local date    |

```js
import { defineConfig, weekdays, date } from "daycal";
// Nothing bookable today: open from tomorrow on.
hours: (day, { now }) => (day.date === date.of(now) ? day.off() : weekdays(day, "09:00-17:00")),
```

## Things that are not in the stdlib

- No holiday list: holidays on a calendar close through `daysOff`; for ones on no calendar, list the dates and use
  `day.is(...dates)`.
- No network or imports other than `"daycal"`; the module loader rejects them.
- No `Date.now()`: use `ctx.now`. `new Date(iso)` for parsing is fine, but prefer `date.*` and `day.*` for
  anything zone-related.
