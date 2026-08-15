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

**Closet** — a wardrobe per person, laid out as a paper doll. Pick a day, tap items to
dress the figure, tap a worn item again to take it off. Five layers: hat, outer, top,
bottom, shoes. Tapping the doll jumps the wardrobe filter to that slot.

- *Standard graphics* — 19 garment shapes (t-shirt, button-down, hoodie, jeans, skirt,
  boots, beanie…) in 14 clothing colours. Eight one-tap starters get a wardrobe usable
  in seconds.
- *Real photos* — snap a garment from the tile and the photo is clipped into that
  garment's silhouette, on the doll and the tile. That's what ties the catalogue to the
  actual item in the drawer.
- *Out of season* — mark items packed away, or bulk-pack a whole season. Packed items
  stay in the catalogue but drop out of outfit planning and are removed from any day
  they were already planned for.
- *The week* — seven mini dolls across the top, one per day, so a week of outfits is
  visible at a glance. "Copy \<yesterday\>" and "Clear day" for fast planning.

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

Everything lives in this browser under `yoder-laundry-v1` (localStorage) plus a
`laundry-hq-photos` IndexedDB store. Nothing is uploaded and nothing leaves the device —
which also means **each device keeps its own board**. There is an *Export JSON* button
under ⚙ for backups.

Photos are deliberately **not** in localStorage: that quota is about 5MB total, and one
iPad photo is 3–5MB, so a couple of garments would break the whole app. Instead every
photo is downscaled to 440px at capture (a 122KB test shot lands around 19KB) and
written to IndexedDB, which has room for hundreds of items. The *Export JSON* backup
covers your loads, wardrobe and plans — **it does not include the photos.**

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
