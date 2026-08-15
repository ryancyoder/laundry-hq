# Laundry HQ

A laundry tracker for a household of seven. One self-contained HTML file — no build,
no server, no account. Open it and it works.

```bash
open index.html
```

To use it on phones, put `index.html` somewhere shared (iCloud Drive, Dropbox) or host
it, then "Add to Home Screen" so it opens full-screen like an app.

## What it does

**Board** — five stages: Sorted → Washing → Drying → To fold → Put away. Move loads with
the buttons, or drag cards between columns on a desktop.

**Machines** — a live dial per washer and dryer. While a cycle runs the ring fills and
the drum spins. When it finishes you get a chime, a browser notification, and a green
"needs unloading" pulse. After 15 minutes that escalates to amber and starts counting
how long the clothes have been sitting.

**Week** — a grid of who washes on which day, plus:
- *Next in line* — who is up after today
- *The next seven days* — each day's loads; empty days are hatched and called out as
  open, so a gap is somewhere to catch up rather than something you notice too late
- *Household loads* — towels, rags and sheets, with their intervals
- *Falling behind* — anything whose turn came round with nothing finished since

**Stats** — loads in flight, folding backlog, weekly total vs. the week before, average
turnaround, a 14-day column chart, a bar chart per person and household item, and the
last 20 completed loads.

## What a load belongs to

There is no separate "type" on a load. A load belongs to exactly one **subject**, and a
subject is either a person or a household item:

| Subject | Interval | Weekday |
|---|---|---|
| Each person | weekly | one day each, set in the grid |
| Towels | every 4 days | twice a week — Wed & Sat by default |
| Rags | weekly | Saturday by default |
| Sheets | every 30 days | none — monthly doesn't sit on a weekly grid |

Adding a load is one tap on a subject, with an optional free-text note ("soccer kit,
muddy"). Anything overdue is marked in the picker so the right choice is usually the
first one you see.

**Rags was the one interval you didn't specify** — it's set to weekly, changeable in the
Week view like the others.

Status is worked out the same way for everyone. With target weekdays, the question is
whether anything has been finished since that day last came round; without them, it
falls back to the interval. A subject with no finished loads at all reads as "no loads
logged yet" rather than "behind" — a new board shouldn't open accusing everyone.

## The household

First run shows seven colour slots — orange, purple, red, green, black, yellow, blue.
Type a name beside each one you need; blank rows are skipped. **Names are never stored
in this repo**, only in the browser you enter them on, so nothing personal is published.
Everything is editable later under ⚙.

Two notes on the colours. Pure black is invisible on a dark background, so the "black"
slot is a graphite that reads as black next to the others while staying visible. Avatar
text flips between black and white automatically so every chip is legible, and initials
are made unique across the household — two people whose names start alike get different
badges rather than both showing the same two letters.

## Data

Everything lives in this browser's `localStorage` under `yoder-laundry-v1`. Nothing is
uploaded and nothing leaves the device — which also means **each device keeps its own
board**. There is an *Export JSON* button under ⚙ for backups.

If you later want all seven phones looking at the same live board, that needs a real
backend; the state is a single `S` object read and written through `load()`/`save()`,
so swapping in a sync layer is a contained change rather than a rewrite.

## Notes

- Chart colours come from a palette validated for colour-blind separation and contrast
  against this background. Every bar is directly labelled, so colour never carries
  meaning on its own. Household items are deliberately neutral grey — they aren't
  people, and that keeps the six family colours unambiguous.
- Notifications are off until you enable them under ⚙ (the browser will ask once).
- Press `n` anywhere to add a load.
- "Clear board" empties the Put away column without losing those loads from Stats.
