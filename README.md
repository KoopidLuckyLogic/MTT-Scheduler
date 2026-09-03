# Vegas Infinite — MTT Tournament Schedule Viewer

A player-facing tournament schedule, published as a single static page on GitHub Pages.
Every viewer sees the schedule in **Eastern Time (ET)** — there is no per-viewer local-time
conversion. See **Timezones** below for what that means when authoring events.

```
index.html          the entire application — data, styles and logic
assets/             brand logo, mark and favicon (local, not fetched from anywhere)
assets/fonts/       Capitana, the brand font, self-hosted
.nojekyll           stops GitHub Pages' Jekyll pass from touching the files
README.md           this file
```

There is no build step, no server and no external dependency. Edit `index.html`, commit, done.

---

## Updating the schedule

Open `index.html` in any text editor and find this banner near the middle of the file:

```
############   SCHEDULE DATA — EDIT BELOW THIS LINE   ############
```

Everything you need to change is between that banner and the matching
`END OF SCHEDULE DATA` banner. **Nothing below it needs editing.**

> ### ⚠️ This file is fully public
> Anyone can read the whole thing with View Source, comments included. Do **not** commit
> unannounced events, embargoed dates, or prizing you haven't confirmed publicly.
> Keep drafts in a local file you don't commit.

After any edit, bump `lastUpdated` so players can see the page is current:

```js
const CONFIG = {
  lastUpdated: "2026-09-03",   // shown in the footer
  horizonDays: 35,             // how far ahead weekly events are generated
  liveGraceMins: 20            // how long an event stays flagged "Running now"
};
```

### The data blocks

| Block | What it's for |
|---|---|
| `SPEEDS` | The speed labels players see (`turbo` → "Turbo"). Add your own keys freely. |
| `CLUBS` | Club membership tiers (currently `Club` / `Club+`). The key is what a `club` field refers to. |
| `WEEKLY` | The recurring MTT ring. Define each tournament **once**; occurrences generate automatically. |
| `CLUB_SCHEDULE` | The real Club / Club+ freeroll calendar — one-off dated entries, since day/time/prize pool vary month to month. |
| `SERIES` | Highlight events with a real schedule to drill into (currently MPT and the Spotlight Series). Each has a date range and its own event list. |
| `OVERRIDES` | One-off changes to a single week of a recurring `WEEKLY` event. |
| `HIGHLIGHT_ANNOUNCEMENTS` | A confirmed, dated highlight with nothing to drill into — a themed card with no schedule (see below). |
| `LOCKED_HIGHLIGHTS` | A teaser for something not announced yet — muted card, no real date (see below). |

### Adding a weekly tournament

Copy an existing entry and change the fields. Every field:

| Field | Notes |
|---|---|
| `id` | Unique string. Referenced by `OVERRIDES`, so keep it stable. |
| `name` | What players see. |
| `day` | `0`=Sunday, `1`=Monday … `6`=Saturday — or `"daily"` for a tournament that runs every night (see **Nightly tournaments** below) — or an array of those numbers (e.g. `[1,3,5]`) for a fixed subset of days. |
| `time` | `"HH:MM"`, 24-hour. |
| `tz` | About how the time was *authored*, not how it displays (every viewer sees Eastern Time regardless — see **Timezones** below). `"America/New_York"` when the source time is already in Eastern (almost everything). `null` when the source time is UTC/GMT instead. |
| `rollover` | `true` only for an event at `"00:00"` that's really the tail end of the *previous* evening (e.g. "Monday Midnight Million" ending Monday night). The real instant becomes the start of the next calendar day for correct sorting and countdowns; the event still displays and is still targeted by `OVERRIDES` as `day`. Omit for anything not scheduled exactly at midnight. |
| `speed` | A key from `SPEEDS`. |
| `stack` | Starting chip stack, a number. |
| `buyIn` | A number. `0` displays as **Freeroll**. |
| `currency` | `"Chips"`, `"Creds"`, etc. |
| `category` | `"weekly"` or `"club"`. |
| `club` | A key from `CLUBS` when `category` is `"club"`, otherwise `null`. |
| `prizes` | `{ participation, winner, extra }` — each a string or `null`. |
| `prizePool` | Optional. A number shown as this event's headline prize pool (e.g. a Freeroll's guaranteed prize). Omit or leave unset when not applicable. |
| `notes` | Optional line shown when a player expands the event. |

### Nightly tournaments (same time, every day)

A handful of tournaments — a late-night freeroll, an overnight turbo — run at the same
time every single day rather than on one weekday. Give that rule `day:"daily"` instead of
a number, and it generates one occurrence per day automatically:

```js
{ id:"nightly-eclipse", name:"Nightly Eclipse", day:"daily", time:"02:00",
  tz:"America/New_York", speed:"turbo", stack:1000, buyIn:500000,
  currency:"Chips", category:"weekly", club:null,
  prizes:{ participation:null, winner:null, extra:null }, notes:"" }
```

It shows up under every column on the Weekly tab, and every calendar day everywhere else
— exactly like any other recurring rule, just with seven times the occurrences.

**Highlight days stand down the regular ring.** Any date a `SERIES` has an event on, every
`WEEKLY` rule *other than* the `"daily"` ones is skipped automatically that day — the felts
are given over to the highlight series instead. You don't need an `OVERRIDES` entry to make
this happen; it's automatic. This is why adding a `SERIES` event on, say, a Friday will make
that Friday's usual tournaments disappear from the Weekly tab and every other view — that's
by design, not a bug.

### Prizing

Three independent slots, any of which can be `null`:

```js
prizes:{
  participation:"500 Chips for all entrants",   // shown as "All entrants"
  winner:"25,000 Chips",                        // shown as "Winner"
  extra:"Top 10 receive 1 Raffle Ticket"        // shown as "Also"
}
```

If **any** slot is filled, the event gets a **★ Prizes** badge in the list, and the full
breakdown appears when a player expands the card. If all three are `null`, no badge.

### Changing or cancelling one week only

Don't edit the `WEEKLY` entry — that would change every week. Use `OVERRIDES`:

```js
const OVERRIDES = [
  // Cancel one occurrence
  { weeklyId:"sun-million", date:"2026-10-11", cancelled:true },

  // Change fields for one occurrence only (gets a "Changed this week" badge)
  { weeklyId:"mon-mayhem", date:"2026-09-28", patch:{ name:"Monday Mayhem — Freeroll Special", buyIn:0 } }
];
```

`date` is the calendar date of the occurrence you're targeting. Old overrides are harmless
once the date has passed, but tidying them up occasionally keeps the file readable.

You don't need an override to cancel a `WEEKLY` occurrence for a highlight-series day —
that's automatic (see **Highlight days stand down the regular ring** above). `OVERRIDES`
is for exceptions on an otherwise-ordinary week, like the promo above.

### The Club Exclusive schedule

Unlike `WEEKLY`, Club/Club+ freerolls run once on their own date — day of week, time and
prize pool all vary month to month — so they're listed individually in `CLUB_SCHEDULE`
instead of as a recurring rule:

```js
const CLUB_SCHEDULE = [
  { date:"2026-09-05", time:"19:00", club:"member", prizePool:10000000 },
  { date:"2026-09-12", time:"13:00", club:"plus",   prizePool:10000000 }
];
```

All Club Schedule entries are 1,000-chip turbo freerolls (`buyIn:0`), so those fields aren't
repeated per row. `club` is a key from `CLUBS` (`member` → "Club", `plus` → "Club+"), always
announced and authored in **America/New_York**. The event's name (`"Club Freeroll"` /
`"Club+ Freeroll"`) and winner prize (`"Club Badge"` / `"Club+ Badge"`) are generated from
the tier automatically — you only ever need `date`, `time`, `club` and `prizePool`.

The Club tab's neon pink/violet theme is scoped to that tab only (`#p-clubs` overrides the
`--c-club` colour token) — Club events shown elsewhere (Next Up, All Events) keep the site's
normal gold/surface look.

### Adding a highlight series

```js
{
  id:"mpt-2026-09",              // unique
  name:"Metaverse Poker Tour - Macau 2050",
  short:"MPT",                   // the small badge on each event
  accent:"gold",                 // gold | orange | violet | teal | mpt
  start:"2026-09-24",
  end:"2026-09-28",
  blurb:"One or two sentences describing the series.",
  events:[ /* each needs an explicit `date`; other fields match WEEKLY */ ]
}
```

The series card shows **Starts in N days** → **Running now** → **Finished** automatically
from `start`/`end`. Series events always appear regardless of `horizonDays`, so you can
publish December's schedule in September.

Leave `events` as `[]` for a series whose dates are set but whose lineup isn't announced
yet — it automatically shows a "Schedule to be announced" empty state instead of an event
list (see the future MPT stops in `SERIES` for an example).

**`accent: "mpt"` and `accent: "gold"` get a gradient treatment** (the tour's own
cyan-to-blue mark, or the site's brand-gold gradient) instead of a flat colour, via
`[data-theme="mpt"|"gold"]` rules in the stylesheet keyed off the accent value — any future
series reusing one of those two accents gets the same look automatically, no extra markup
needed. `orange`, `violet` and `teal` stay flat accent colours.

### A highlight with no schedule to click into

Two lighter-weight alternatives to a full `SERIES` entry, both rendered as non-interactive
cards on the Highlights list (no button, no drill-down page):

```js
// A confirmed, dated event with nothing to drill into — themed like a real series card,
// just inert. Fold any date/time detail into `note` since there's no event row to show it.
const HIGHLIGHT_ANNOUNCEMENTS = [
  { name:"Main Stage Showdown", short:"SHOWDOWN", accent:"violet", when:"18 September 2026",
    note:"Invite only. 64 players face off in a single-elimination heads-up bracket. Begins at 5:00 PM ET." }
];

// Something not announced yet — no real date, so nothing to click into. Muted styling
// and a 🔒 icon instead of an accent colour.
const LOCKED_HIGHLIGHTS = [
  { name:"Spooky Stacks", when:"Oct TBD", note:"More information coming soon" }
];
```

Use `SERIES` (with `events:[]`) when dates are locked in but the lineup isn't; use
`HIGHLIGHT_ANNOUNCEMENTS` for a one-off with nothing resembling a schedule at all; use
`LOCKED_HIGHLIGHTS` when even the date isn't confirmed yet.

### How players get to a series' schedule

The **Highlights** tab is a browsable list of series cards (tag, name, date range, blurb,
status) — tapping one navigates into that series' own dedicated page: header, week picker
(if it has one), filter chips, and the full day-by-day schedule, with a "← All Highlights"
button back to the list. It behaves like a drill-down, not an accordion — the schedule
isn't squeezed into a collapsible strip on a crowded card, it gets the whole screen.

The **Next Up** tab also shows a short teaser row per upcoming/running series (name, status,
event count) — tapping one jumps straight to that series' page, same as clicking its card
on the Highlights tab. A finished series drops out of the Next Up teaser automatically but
stays visible in the Highlights list.

### Large series (festivals with 50+ events)

A short series like MPT just shows every event in one scrollable list. For something
Sunny-Stacks-scale — a multi-week festival with 100+ events — add `weeks` and (optionally)
`dayTags`, and the series gets a **week picker** inside it, so a player only ever looks at
one week at a time instead of scrolling past everything else:

```js
{
  id:"sunny-stacks-iv", name:"Sunny Stacks IV", short:"SUNNY IV", accent:"teal",
  start:"2027-08-05", end:"2027-08-15",
  blurb:"...",
  weeks:[
    { label:"Week 1", sub:"Opening", start:"2027-08-05", end:"2027-08-08" },
    { label:"Week 2", sub:"Finale",  start:"2027-08-09", end:"2027-08-15" }
  ],
  dayTags:{
    "2027-08-05":"Opening Day",
    "2027-08-06":"Opening Weekend (×2)"
    // any date without an entry here just shows no tag — totally optional
  },
  events:[ /* same shape as before — this is what dayTags/weeks group */ ]
}
```

`weeks` is only needed once a series has enough days that one continuous list would be
unwieldy — a 3-event, 3-day highlight doesn't need it. If you omit `weeks`, the series
renders exactly as before: one flat, day-grouped list, no week picker.

One event field worth knowing about: **`groupDate`**. A tournament that runs past midnight
is naturally "the next calendar day" the instant it ticks over — but it's still that
*evening's* event, and players expect to find it grouped with the night it started, not
filed under the next morning. Set the event's real `date`/`time` to the true instant
(e.g. `date:"2027-08-06", time:"00:00"` for a session that runs to midnight), and add
`groupDate:"2027-08-05"` to display it under the evening it belongs to. Countdown and
sort order use the real instant; only which day-card it appears in changes. Most events
never need this — only ones scheduled exactly at midnight.

### Filtering inside a series

Every series — large or small — gets filter chips on its own schedule page: **event
type** (Main Events, Satellites, Freerolls, High Roller) and **speed / prizing**. A chip
only appears if that series actually has an event matching it, so a series with no
freerolls just won't show a "Freerolls" chip. Filters are per-series (picking "Main
Events" in one series doesn't affect any other) and persist across a week switch, so a
player can filter to Main Events and then browse both weeks without re-selecting it.

Event type is detected automatically from the event's `name`:

| Name contains | Tagged |
|---|---|
| "satellite" | `satellite` |
| "main event" (and not "satellite") | `main` |
| — | `freeroll` whenever `buyIn` is `0` |
| "high roller" | `highroller` |

An event can carry more than one tag (a "High Roller Satellite" is both). If the
detection guesses wrong, or you want to mark something like a "Grand Final" that doesn't
literally say "Main Event", add an explicit `tags` array to that event — it's merged
with whatever was auto-detected:

```js
{ name:"MPT Grand Final", date:"2026-10-11", time:"19:00", tz:null, tags:["main"], ... }
```

Most events never need this field — it exists for the handful that the automatic
detection can't or shouldn't guess at.

---

## Timezones

**Every viewer sees the schedule in Eastern Time.** There is no per-viewer local-time
conversion — `VIEW_TZ` is hardcoded to `"America/New_York"` in `index.html`, not detected
from the visitor's device. This is a deliberate choice for an "ET schedule" operator; it's
not the historical default (see below if you ever need to revert it).

The `tz` field on an event is about **how the source time was authored**, not about display:

- **`tz: "America/New_York"`** — the time is already in Eastern, exactly as announced
  (e.g. a "Starting Time (ET)" column in a schedule spreadsheet). This is what almost every
  `WEEKLY` and `CLUB_SCHEDULE` entry uses.
- **`tz: null`** — the time is UTC/GMT (e.g. a "Starting Time (GMT)" column). Since GMT has
  no DST, this is also correct for a GMT-labeled schedule column.

Either way, the displayed clock time is computed correctly and converted to Eastern for
everyone. Use a full IANA name (`America/New_York`), never an abbreviation like `EDT`/`EST`
— the engine resolves the correct one automatically depending on the date.

**If you ever want per-viewer local time back:** replace the hardcoded
`let VIEW_TZ = "America/New_York";` near the top of the script with
`let VIEW_TZ = Intl.DateTimeFormat().resolvedOptions().timeZone || "UTC";` (optionally still
allowing the `?tz=` override below it). Nothing else in the file depends on the fixed value.

---

## Testing your changes

Open `index.html` by double-clicking it. It works straight from the filesystem — no server
needed. Add these to the URL when you want to check something:

| Parameter | What it does |
|---|---|
| `?debug=1` | Shows a data-check panel listing typos, unknown keys, bad dates and out-of-range series events. **Always run this after an edit.** |
| `?tz=Europe/London` | Preview the schedule in a different timezone than the fixed Eastern Time default, for testing. |
| `?now=2026-09-24T19:45:00Z` | Pretend it's another moment in time. Lets you verify what players will see during an event without waiting for the date. |

Combine them: `index.html?now=2026-09-24T19:45:00Z&tz=Europe/London&debug=1`

These are harmless if a player finds them — they only change that person's own view.

### If the page comes up blank or shows an error

Almost always a typo in the `SCHEDULE DATA` block — a missing comma, quote or brace. The
page shows a readable error panel rather than a blank screen, and names the problem. Your
browser's developer console (F12) gives the exact line number.

---

## Publishing

1. Commit and push to the repository's default branch.
2. **Settings → Pages** → Source: *Deploy from a branch* → your branch, folder `/ (root)`.
3. The page goes live at `https://<org>.github.io/<repo>/` within a minute or so.

Every later update is just a commit.

### Security — what actually matters

GitHub Pages serves **static, read-only** content. Nothing a visitor does in their browser
can change what other players see: each person's browser fetches its own copy, and any
client-side state stays on that one device. The only way to change the live page is a push
to the repo. So the repository is where the security lives, not the page:

- [ ] **Branch protection** on the published branch — require a pull request, block force-push and deletion
- [ ] **2FA** on every account with write access
- [ ] Keep the **collaborator list** to the people who actually publish
- [ ] **Settings → Actions → General → Workflow permissions** → read-only (nothing here needs Actions)
- [ ] Never commit unannounced events or unconfirmed prizing (see the warning above)

Forks and pull requests from strangers **cannot** reach the live page — someone with write
access has to merge them first.

The page also deliberately has **no external dependencies**: no CDN scripts, no webfonts, no
runtime API calls. Everything is inlined or served from `assets/` in this repo. That's not
just for speed — anything loaded from a third party would let whoever controls that third
party change the page for every player. Please keep it that way; if you need a library,
inline it.

---

## Rebranding

All colour lives in the `:root` block at the top of `index.html`. The golds were sampled
directly from the Vegas Infinite wordmark. Change values there and nothing else needs
touching.

The brand font is **Capitana**, self-hosted from `assets/fonts/` (two weights, 216 KB
total) and declared with `font-display:swap` so text paints immediately in a system
fallback and swaps when the font arrives — a slow headset connection never sees a blank
page. Only 400 and 700 are available, so the heavier weights in the CSS all resolve to
bold; that's intentional, not a bug.

If you want to cut the page weight, converting the two `.otf` files to `.woff2` typically
saves 50-60% with no visual change. Keep them in this repo either way — loading a font
from a third-party host would break the zero-external-dependency property.

Two rules if you edit the CSS:

- **Every `font-size` under `1rem` is wrapped in `max(16px, …)`.** The root font size scales
  fluidly with the viewport, so without that floor small labels drop to ~14px on a phone.
  16px is the floor for in-headset legibility. Keep the pattern.
- Controls are `56px` minimum (`48px` for filter chips) via the `--tap` tokens, and no
  information is hover-only — a headset browser is driven by a controller, not a mouse.
