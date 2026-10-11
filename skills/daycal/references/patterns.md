# rules.js patterns

Worked cases. Find the closest one, copy it, adapt names and times, then `preview`. Every case ends with a
fixture: one day of calendar data and the slot start times the rules produce for one event type, in the
owner's zone. Daycal's test suite runs each fixture through the real engine, so these numbers are exact.

Fixture fields: `now`, `accounts` and `calendars` (as `connect` and `list_calendars` see them), `events`
(provider fields: `selfResponse` is what rules.js sees as `event.response`; omitted fields default to a
confirmed, busy, accepted, timed event), optional `bookings`, then `eventType`, `day` and `expect`.

All cases are for Chris Lee (`name`, what invitees see; handle `chris`) in `America/Los_Angeles` (UTC-7 in late
October 2026) with a `work` Google account and, where it matters, a `personal` one.

## 1. The full example: two accounts, program calendar, lunch, declined, standup overflow

Weekdays 9–12 and 1–5, Fridays 9–3, three weeks in London. All-day items on the program calendar are
reminders, "Lunch"/"Focus" holds and declined invites do not block, and Tuesday 12–1 is reserved. The
30-minute type needs 10 minutes after each meeting, 4 hours' notice, and caps at 4 a day.

On Tuesday Oct 20: 9:30 and 10:00 collide with the 10:00 review, 11:30's buffer runs into the reserved
12:00 block, and 15:30 and 16:00 collide with the 16:00 errand. Lunch, the declined pitch, and the
all-day program item do not block.

```js
import { defineConfig, weekdays, every, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",

  calendars: ({ accounts }) => [
    accounts.work.primary,
    accounts.work.calendar("Program 2026"),
    accounts.personal.primary,
  ],

  hours: (day) => {
    if (day.between("2026-10-27", "2026-11-14")) {
      // Three weeks in London: later start, local hours, weekdays only.
      return day.in("Europe/London").weekdays("10:00-13:00", "14:00-18:00");
    }
    if (day.is("fri")) return day.hours("09:00-15:00");
    return weekdays(day, "09:00-12:00", "13:00-17:00");
  },

  isBusy: (event, { accounts }) => {
    if (event.calendar.is(accounts.work.calendar("Program 2026")) && event.allDay) return false;
    if (/^(lunch|focus)/i.test(event.title)) return false;
    if (event.response === "declined") return false;
    return event.transparency !== "free";
  },

  extraBusy: ({ range }) => every("tue", range, "12:00-13:00", { reason: "standup overflow" }),

  eventTypes: [
    {
      slug: "30min",
      title: "30 minute chat",
      duration: dur("30m"),
      bufferAfter: dur("10m"),
      minNotice: dur("4h"),
      maxPerDay: 4,
      maxDaysAhead: 30,
      conferencing: "google-meet",
    },
  ],

  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [
    { "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" },
    { "connectionId": "c-personal", "label": "personal", "email": "chris@gmail.com", "provider": "google" }
  ],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    },
    {
      "connectionId": "c-work",
      "id": "program@group.calendar.google.com",
      "name": "Program 2026",
      "accessRole": "reader",
      "primary": false
    },
    {
      "connectionId": "c-personal",
      "id": "chris@gmail.com",
      "name": "chris@gmail.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "program@group.calendar.google.com",
      "id": "p1",
      "title": "Week 6: demos",
      "start": "2026-10-20T00:00:00-07:00",
      "end": "2026-10-21T00:00:00-07:00",
      "allDay": true
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "Design review",
      "start": "2026-10-20T10:00:00-07:00",
      "end": "2026-10-20T10:30:00-07:00"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w2",
      "title": "Lunch",
      "start": "2026-10-20T13:00:00-07:00",
      "end": "2026-10-20T14:00:00-07:00"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w3",
      "title": "Vendor pitch",
      "start": "2026-10-20T15:00:00-07:00",
      "end": "2026-10-20T16:00:00-07:00",
      "selfResponse": "declined"
    },
    {
      "calendarId": "chris@gmail.com",
      "id": "g1",
      "title": "Pick up car",
      "start": "2026-10-20T16:00:00-07:00",
      "end": "2026-10-20T16:30:00-07:00"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-20",
  "expect": ["09:00", "10:30", "11:00", "13:00", "13:30", "14:00", "14:30", "15:00", "16:30"]
}
```

## 2. A trip in another time zone

`day.in(tz)` reads that day's hours in another zone without changing the owner's `timezone`. In London
(UTC+0 after Oct 25) 10:00–13:00 and 14:00–18:00 are 03:00–06:00 and 07:00–11:00 in Los Angeles, which is
how the page and the fixture show them. The 11:00 London workshop removes 04:00 and 04:30.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => {
    if (day.between("2026-10-27", "2026-11-14")) {
      return day.in("Europe/London").weekdays("10:00-13:00", "14:00-18:00");
    }
    return weekdays(day, "09:00-17:00");
  },
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-26T09:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "Client workshop",
      "start": "2026-10-28T11:00:00Z",
      "end": "2026-10-28T12:00:00Z"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-28",
  "expect": ["03:00", "03:30", "05:00", "05:30", "07:00", "07:30", "08:00", "08:30", "09:00", "09:30", "10:00", "10:30"]
}
```

## 3. Recurring and one-off blocks

Time the calendars do not know about goes in `extraBusy`. `every` repeats a block on weekdays (or `"day"`);
`block` adds a one-off. Spread them into one array.

```js
import { defineConfig, weekdays, every, block, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  extraBusy: ({ range }) => [
    ...every("day", range, "12:00-13:00", { reason: "lunch" }),
    ...every(["mon", "wed"], range, "14:00-15:00", { reason: "school run" }),
    ...block("2026-10-21", "09:00-10:00", { reason: "dentist" }),
  ],
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [],
  "eventType": "30min",
  "day": "2026-10-21",
  "expect": ["10:00", "10:30", "11:00", "11:30", "13:00", "13:30", "15:00", "15:30", "16:00", "16:30"]
}
```

## 4. Ignore all-day events on a program calendar

A shared program or team calendar full of all-day markers ("Week 6", "Demo week") would block whole days.
Keep its timed events, drop its all-day ones.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary, accounts.work.calendar("Program 2026")],
  hours: (day) => weekdays(day, "09:00-17:00"),
  isBusy: (event, { accounts }) => !(event.allDay && event.calendar.is(accounts.work.calendar("Program 2026"))),
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    },
    {
      "connectionId": "c-work",
      "id": "program@group.calendar.google.com",
      "name": "Program 2026",
      "accessRole": "reader",
      "primary": false
    }
  ],
  "events": [
    {
      "calendarId": "program@group.calendar.google.com",
      "id": "p1",
      "title": "Office hours week",
      "start": "2026-10-20T00:00:00-07:00",
      "end": "2026-10-21T00:00:00-07:00",
      "allDay": true
    },
    {
      "calendarId": "program@group.calendar.google.com",
      "id": "p2",
      "title": "Demo day rehearsal",
      "start": "2026-10-20T15:00:00-07:00",
      "end": "2026-10-20T16:00:00-07:00"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-20",
  "expect": [
    "09:00",
    "09:30",
    "10:00",
    "10:30",
    "11:00",
    "11:30",
    "12:00",
    "12:30",
    "13:00",
    "13:30",
    "14:00",
    "14:30",
    "16:00",
    "16:30"
  ]
}
```

## 5. Different hours per event type

`hours` gets `ctx.eventType`, so "sales calls only Tuesday and Thursday mornings, office hours any
weekday" is one branch.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day, { eventType }) => {
    if (eventType.slug === "sales") {
      return day.is("tue", "thu") ? day.hours("10:00-12:00") : day.off();
    }
    return weekdays(day, "09:00-17:00");
  },
  eventTypes: [
    { slug: "sales", title: "Sales call", duration: dur("30m") },
    { slug: "office-hours", title: "Office hours", duration: dur("30m") },
  ],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "Pipeline review",
      "start": "2026-10-20T11:00:00-07:00",
      "end": "2026-10-20T11:30:00-07:00"
    }
  ],
  "eventType": "sales",
  "day": "2026-10-20",
  "expect": ["10:00", "10:30", "11:30"]
}
```

## 6. Friday short day

Check the special day first, then fall through to the normal week.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => {
    if (day.is("fri")) return day.hours("09:00-13:00");
    return weekdays(day, "09:00-17:00");
  },
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [],
  "eventType": "30min",
  "day": "2026-10-23",
  "expect": ["09:00", "09:30", "10:00", "10:30", "11:00", "11:30", "12:00", "12:30"]
}
```

## 7. Holidays off

Google holiday calendars ("Holidays in United States", `holidays: true` in `get_calendar_refs`) mark every event
free, so listing one in `calendars` blocks nothing: a natural `isBusy` ("busy unless free") lets the holiday through,
and `preview` warns that the calendar never blocks time. List it in `daysOff` instead: any event on it closes the
whole owner-local day, for every event type, whatever `isBusy` says. Ask the owner which holiday calendars count
(someone subscribed to another country's holidays may still work those days).

Days the calendar does not know (a bridge day, a vacation) go in `hours`: `day.is(...dates)` or
`day.between(from, to)` returning `day.off()`.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  // Every event on the US holiday calendar closes its day.
  daysOff: ({ accounts }) => accounts.work.calendars.filter((c) => c.holidays),
  hours: (day) => {
    if (day.is("2026-11-27")) return day.off(); // the day after Thanksgiving
    return weekdays(day, "09:00-17:00");
  },
  isBusy: (event) => event.transparency !== "free",
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-11-20T08:00:00-08:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    },
    {
      "connectionId": "c-work",
      "id": "en.usa#holiday@group.v.calendar.google.com",
      "name": "Holidays in United States",
      "accessRole": "reader",
      "primary": false
    }
  ],
  "events": [
    {
      "calendarId": "en.usa#holiday@group.v.calendar.google.com",
      "id": "thanksgiving",
      "title": "Thanksgiving Day",
      "start": "2026-11-26T00:00:00-08:00",
      "end": "2026-11-27T00:00:00-08:00",
      "allDay": true,
      "transparency": "free",
      "rawStatus": "transparent",
      "attendeeCount": 0,
      "selfResponse": "none"
    }
  ],
  "eventType": "30min",
  "day": "2026-11-26",
  "expect": []
}
```

## 8. Lunch and focus holds do not block

People park "Lunch" and "Focus time" on their calendar but will move them for a meeting. The pattern
matches at the start of the title, so "Team lunch" still blocks but "Lunch with investors" does not; use
an exact match such as `/^(lunch|focus time)$/i` when that matters.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  isBusy: (event) => !/^(lunch|focus)\b/i.test(event.title),
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "1:1 with Sam",
      "start": "2026-10-22T10:00:00-07:00",
      "end": "2026-10-22T10:30:00-07:00"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w2",
      "title": "Lunch",
      "start": "2026-10-22T12:00:00-07:00",
      "end": "2026-10-22T13:00:00-07:00"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w3",
      "title": "Focus time",
      "start": "2026-10-22T14:00:00-07:00",
      "end": "2026-10-22T16:00:00-07:00"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-22",
  "expect": [
    "09:00",
    "09:30",
    "10:30",
    "11:00",
    "11:30",
    "12:00",
    "12:30",
    "13:00",
    "13:30",
    "14:00",
    "14:30",
    "15:00",
    "15:30",
    "16:00",
    "16:30"
  ]
}
```

## 9. Declined invites do not block

`event.response` is the owner's own answer. Declined frees the time; unanswered (`needsAction`) still
blocks, which is usually what owners want.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  isBusy: (event) => event.response !== "declined",
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "All-hands",
      "start": "2026-10-22T10:00:00-07:00",
      "end": "2026-10-22T11:00:00-07:00",
      "selfResponse": "declined"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w2",
      "title": "Board prep",
      "start": "2026-10-22T14:00:00-07:00",
      "end": "2026-10-22T15:00:00-07:00"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w3",
      "title": "Offsite planning",
      "start": "2026-10-22T16:00:00-07:00",
      "end": "2026-10-22T17:00:00-07:00",
      "selfResponse": "needsAction"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-22",
  "expect": ["09:00", "09:30", "10:00", "10:30", "11:00", "11:30", "12:00", "12:30", "13:00", "13:30", "15:00", "15:30"]
}
```

## 10. Tentative counts as free; free events never block

Daycal folds tentative into busy by default (`transparency` is `"busy"`). `event.rawStatus` and
`event.response` keep the original, so an owner who treats "maybe" as free can say so. Events marked
"Show as: free" have `transparency: "free"`.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  isBusy: (event) => {
    if (event.rawStatus === "tentative" || event.response === "tentative") return false;
    return event.transparency !== "free";
  },
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "Maybe: conference talk",
      "start": "2026-10-22T10:00:00-07:00",
      "end": "2026-10-22T12:00:00-07:00",
      "status": "tentative",
      "rawStatus": "tentative",
      "selfResponse": "tentative"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w2",
      "title": "Customer call",
      "start": "2026-10-22T13:00:00-07:00",
      "end": "2026-10-22T14:00:00-07:00"
    },
    {
      "calendarId": "chris@acme.com",
      "id": "w3",
      "title": "Coffee hold",
      "start": "2026-10-22T15:00:00-07:00",
      "end": "2026-10-22T16:00:00-07:00",
      "transparency": "free",
      "rawStatus": "transparent"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-22",
  "expect": [
    "09:00",
    "09:30",
    "10:00",
    "10:30",
    "11:00",
    "11:30",
    "12:00",
    "12:30",
    "14:00",
    "14:30",
    "15:00",
    "15:30",
    "16:00",
    "16:30"
  ]
}
```

## 11. A daily cap and buffers

`bufferBefore`/`bufferAfter` need free time around each meeting; they apply against calendar events and
existing bookings of any type. `maxPerDay` counts only bookings of the same type. Here one `intro` and one
`deep-dive` are already booked, so a second `intro` is still allowed, but the buffers remove every slot
within 15 minutes of the 10:00 intro, the 14:00 sync, and the 16:00 deep dive.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  eventTypes: [
    {
      slug: "intro",
      title: "Intro call",
      duration: dur("30m"),
      bufferBefore: dur("15m"),
      bufferAfter: dur("15m"),
      maxPerDay: 2,
    },
    { slug: "deep-dive", title: "Deep dive", duration: dur("1h"), slotStep: dur("30m") },
  ],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "Team sync",
      "start": "2026-10-20T14:00:00-07:00",
      "end": "2026-10-20T15:00:00-07:00"
    }
  ],
  "bookings": [
    { "id": "b1", "eventType": "intro", "start": "2026-10-20T10:00:00-07:00", "end": "2026-10-20T10:30:00-07:00" },
    { "id": "b2", "eventType": "deep-dive", "start": "2026-10-20T16:00:00-07:00", "end": "2026-10-20T17:00:00-07:00" }
  ],
  "eventType": "intro",
  "day": "2026-10-20",
  "expect": ["09:00", "11:00", "11:30", "12:00", "12:30", "13:00"]
}
```

## 12. Several accounts and a shared family calendar

List exactly the calendars that should block. The work account's "Team (shared)" calendar is connected but
not listed, so its events are ignored; the personal account's primary and its "Family" calendar both
block. Bookings go into the work account.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary, accounts.personal.primary, accounts.personal.calendar("Family")],
  hours: (day) => weekdays(day, "09:00-17:00"),
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [
    { "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" },
    { "connectionId": "c-personal", "label": "personal", "email": "chris@gmail.com", "provider": "google" }
  ],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    },
    {
      "connectionId": "c-work",
      "id": "team@group.calendar.google.com",
      "name": "Team (shared)",
      "accessRole": "reader",
      "primary": false
    },
    {
      "connectionId": "c-personal",
      "id": "chris@gmail.com",
      "name": "chris@gmail.com",
      "accessRole": "owner",
      "primary": true
    },
    {
      "connectionId": "c-personal",
      "id": "family@group.calendar.google.com",
      "name": "Family",
      "accessRole": "writer",
      "primary": false
    }
  ],
  "events": [
    {
      "calendarId": "team@group.calendar.google.com",
      "id": "t1",
      "title": "Team offsite planning",
      "start": "2026-10-22T10:00:00-07:00",
      "end": "2026-10-22T11:00:00-07:00"
    },
    {
      "calendarId": "chris@gmail.com",
      "id": "g1",
      "title": "Physio",
      "start": "2026-10-22T12:00:00-07:00",
      "end": "2026-10-22T13:00:00-07:00"
    },
    {
      "calendarId": "family@group.calendar.google.com",
      "id": "f1",
      "title": "School pickup",
      "start": "2026-10-22T15:00:00-07:00",
      "end": "2026-10-22T15:30:00-07:00"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-22",
  "expect": [
    "09:00",
    "09:30",
    "10:00",
    "10:30",
    "11:00",
    "11:30",
    "13:00",
    "13:30",
    "14:00",
    "14:30",
    "15:30",
    "16:00",
    "16:30"
  ]
}
```

## 13. A fixed meeting link, a phone call, and minimum notice

`conferencing` defaults to Google Meet. A personal Zoom room or any URL is `{ kind: "link", url }`;
in-person or phone is `"none"` with a `location`. Here it is 10:05 on the day and the Zoom type needs an
hour's notice, so the first slot is 11:30.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  eventTypes: [
    {
      slug: "zoom",
      title: "Zoom call",
      duration: dur("30m"),
      minNotice: dur("1h"),
      conferencing: { kind: "link", url: "https://zoom.us/j/5551234567" },
    },
    {
      slug: "phone",
      title: "Phone call",
      duration: dur("15m"),
      slotStep: dur("15m"),
      conferencing: "none",
      location: "Chris will call you at the number you leave in the note",
    },
  ],
  bookInto: ({ accounts }) => accounts.work,
});
```

```json
{
  "now": "2026-10-22T10:05:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [],
  "eventType": "zoom",
  "day": "2026-10-22",
  "expect": ["11:30", "12:00", "12:30", "13:00", "13:30", "14:00", "14:30", "15:00", "15:30", "16:00", "16:30"]
}
```

## 14. Bookings in a separate "Bookings" calendar

By default bookings go into the account's primary calendar (`bookInto: ({ accounts }) => accounts.work`), next to
everything else. Only when the owner asks to keep them apart, use `newCalendar`: Daycal creates a calendar with that
name in the account when the first booking arrives and reuses it after. Leave it out of `calendars`: Daycal already
counts its own bookings as busy. Bookings made before the change stay where they were written.

```js
import { defineConfig, weekdays, dur } from "daycal";

export default defineConfig({
  handle: "chris",
  name: "Chris Lee",
  timezone: "America/Los_Angeles",
  calendars: ({ accounts }) => [accounts.work.primary],
  hours: (day) => weekdays(day, "09:00-17:00"),
  eventTypes: [{ slug: "30min", title: "30 minute chat", duration: dur("30m") }],
  // The owner asked for bookings in their own calendar, apart from the rest of work.
  bookInto: ({ accounts }) => accounts.work.newCalendar("Bookings"),
});
```

```json
{
  "now": "2026-10-19T08:00:00-07:00",
  "accounts": [{ "connectionId": "c-work", "label": "work", "email": "chris@acme.com", "provider": "google" }],
  "calendars": [
    {
      "connectionId": "c-work",
      "id": "chris@acme.com",
      "name": "chris@acme.com",
      "accessRole": "owner",
      "primary": true
    }
  ],
  "events": [
    {
      "calendarId": "chris@acme.com",
      "id": "w1",
      "title": "Planning",
      "start": "2026-10-21T09:00:00-07:00",
      "end": "2026-10-21T12:00:00-07:00"
    }
  ],
  "eventType": "30min",
  "day": "2026-10-21",
  "expect": [
    "12:00",
    "12:30",
    "13:00",
    "13:30",
    "14:00",
    "14:30",
    "15:00",
    "15:30",
    "16:00",
    "16:30"
  ]
}
```
