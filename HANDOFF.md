# Laundry HQ — handoff for the three-app merge

You are being asked to merge this app with two other household apps. This document is
everything you need to know about *this* one: what it is, how it is built, why it is
built that way, and specifically what will break when you merge it.

Read the **Merge hazards** section before you write any code. It is the part that will
cost you time if you skip it.

---

## 1. What this is

A laundry tracker for a household of seven, plus a per-person wardrobe/outfit planner.

- **Repo:** https://github.com/ryancyoder/laundry-hq (public)
- **Live:** https://ryancyoder.github.io/laundry-hq/ (GitHub Pages, `main` branch, root)
- **The whole app is one file:** `index.html` — ~3,070 lines, ~136 KB, no build step,
  no dependencies, no server, no framework. Inline `<style>` and one inline `<script>`.
- Deploy = `git push`. Pages rebuilds in about a minute.
- There is a **build stamp**: `const BUILD = '…'` near the top, surfaced at the bottom of
  ⚙ Settings. There is no service worker and no cache-busting, so a phone can easily be
  running an old copy. When a user reports a bug, **ask what build they see** before
  debugging — more than once during development a "bug" turned out to be a stale page.
  Bump `BUILD` when you ship.
- Primary device is an **iPad** (Safari, added to Home Screen). Design and test for
  touch first; desktop is secondary.

**Why one file with no build:** the user wanted something they could open immediately
and put on family devices without tooling, hosting, or accounts. That constraint drove
almost every other decision. Do not "modernise" it into a bundler/framework setup
unless the user asks — but see §9, because the merge may force part of that.

Four tabs: **Board** (laundry pipeline), **Week** (schedule), **Closet** (wardrobe),
**Stats**.

---

## 2. Merge hazards — read this first

This app was written as a standalone single-file app. Three such apps merged naively
will collide badly. Concretely:

### 2.1 Everything is in one global scope

There are roughly **100 top-level functions** and **60 top-level `const`/`let`
bindings**, all global. The names that are near-certain to collide with another
household app:

```
S  render  save  load  commit  toast  view  setView  esc  uid  now  db  tx
blank  clock  ago  dur  initials  ink  dlg  closeDlg  step  parts  cvs  cx
CATS  DAYS  DAYS_LONG  STAGE  STAGES  HEX  KEY  BUILD  PHOTOS  active  column
ARM  BODY  SHAPES  GARMENTS  FAMILY  SEASONS  CLOTH  STARTERS
```

`S` is the entire application state object. `render()` rebuilds `#app`. `KEY` is the
localStorage key. If another app also has `S` / `render` / `save`, whichever script
loads last silently wins and you get extremely confusing behaviour rather than an
error.

### 2.2 Inline `onclick="..."` handlers everywhere

Most interactivity is `onclick="moveTo('id','wash')"` written into HTML strings. **This
means every handler function must remain a global.** If you wrap this file's script in
an IIFE, an ES module, or a bundler that scopes top-level declarations, *every button
in the app silently stops working* — no console error, just dead clicks.

This is the single biggest constraint on modularising. You must either:
- keep the functions global (e.g. explicitly assign `window.moveTo = moveTo`), or
- convert all inline handlers to delegated event listeners first, as a separate,
  verified step before merging.

Do the conversion as its own commit and test it in isolation. Do not do it at the same
time as the merge.

### 2.3 CSS is global and uses generic class names

~80 unscoped class selectors. The ones most likely to collide:

```
.card  .panel  .chip  .chips  .btn  .col  .empty  .toast  .tabs  .field  .slot
.slots  .strip  .band  .rows  .up  .st  .who  .av  .thumb  .wrap  .dlg  .switch
.toggle  .setup  .hit  .bar  .acts  .kpi  .kpis  .board  .machines
```

Plus **30+ CSS custom properties defined on `:root`**: `--surface`, `--ink`, `--plane`,
`--border`, `--muted`, `--good`, `--warn`, `--crit`, `--s1`…`--s8`, `--st-queue`…
`--st-done`, `--r`, `--shadow`, `--font`.

If the other apps also theme via `:root`, last-loaded wins and you get a visually
scrambled app. Options: prefix everything (`.lh-card`), scope under a root class
(`.app-laundry .card`), or use Shadow DOM / iframes.

### 2.4 Storage keys

| Store | Key | Contents |
|---|---|---|
| localStorage | `yoder-laundry-v1` | The whole `S` object as JSON. Currently ~2–15 KB. |
| IndexedDB | db `laundry-hq-photos`, object store `p` | Wardrobe photos. Key = wardrobe item id, value = a data-URL **string**. |

Namespace these if the merged app has one settings/export surface. **If you change
`KEY`, write a migration** — the user has real data on their iPad and there is no
server copy.

### 2.5 Singletons that assume they own the page

- `#app` — the only view container; `render()` replaces its entire `innerHTML`.
- `#dlg` — **one shared `<dialog>`**. Every modal (new load, settings, new item, photo)
  writes into the same element via `innerHTML`. Two apps both using `#dlg` will fight.
- `#toasts`, `#confetti`, `#aurora`, `#bubbles`, `#measure` — fixed-position overlays
  appended to `<body>`. `#confetti` is a full-viewport canvas (`pointer-events:none`).
- `#measure` — an offscreen SVG used only by `shapeBBox()` for `getBBox()`. It must be
  in the DOM and must not be `display:none`, or measuring silently returns a fallback.
- A global `setInterval(tick, 1000)` runs for the lifetime of the page.
- `render()` writes `document.title` and `#tagline`.
- A `keydown` listener on `window` binds bare **`n`** to "new load" — will hijack typing
  in any other app's inputs that aren't `<input>`/`<textarea>`.

### 2.6 Full-page re-render on every state change

`commit()` = `save()` + `render()`, and `render()` blows away `#app.innerHTML`. This is
fine at this scale but has two consequences already worked around in the code, which you
must preserve:

- **Scroll position resets.** The Mix & Match bands would jump to the start on every
  interaction. Fixed by `repaintMixDoll()`, which updates only the doll element rather
  than calling `render()`.
- **A 1-second full re-render would destroy input focus and restart CSS animations.**
  `tick()` therefore surgically updates only timer text, progress bars and status
  classes via `data-*` selectors — it does *not* call `render()` unless a timer actually
  fires.

If you move to a component framework, both workarounds become unnecessary — but until
then, do not "simplify" them away.

---

## 3. Data model

All of it lives in one object, `S`, persisted as JSON. `blank()` is the schema of
record; read it first.

```js
S = {
  v: 1,
  setup: false,            // false → first-run name entry screen
  people:  [ { id, name, hex, initials, days:[0..6] } ],
  chores:  [ { id, name, emoji, everyDays, days:[0..6], hex } ],
  machines:[ { id, name, kind:'washer'|'dryer', mins } ],
  loads:   [ { id, subjectId, note, stage, createdAt, stageAt,
               machineId, timer:{startedAt,endsAt,mins}|null,
               fired, archived, finishedAt, washMs, dryMs } ],
  wardrobe:[ { id, personId, kind, name, hex, season, stored,
               photo:bool, alpha:bool, masked:bool } ],
  plans:   { [personId]: { [dayOfWeek]: { hat, top, outer, bottom, shoes } } },
  settings:{ sound, notify },
}
```

### The "subject" abstraction (important)

A laundry load does **not** have a type/category. It belongs to exactly one **subject**,
and a subject is either a **person** or a **household item** (Towels, Rags, Sheets).
`subjects()` concatenates both; `subjectOf(id)` resolves either.

This was a deliberate simplification requested by the user after an earlier design had
both an owner *and* a separate garment-type dropdown — the two overlapped confusingly.
**Do not reintroduce a separate "type" field on loads.**

One status function, `subjectStatus()`, covers people and household items alike:
- If the subject has target weekdays → "has anything been finished since that day last
  came round?"
- If it has none (Sheets, monthly) → fall back to the `everyDays` interval.
- If it has **no finished loads at all** → state is `none` / "no loads logged yet",
  deliberately **not** "behind". A brand-new board should not open by accusing the whole
  family of being behind. This was a real bug that got fixed; don't regress it.

### Migration is done in `load()`

`load()` does forward-compatible repair on every read: backfills `wardrobe`, `plans`,
`chores`, `p.days`; rebuilds `chores` if they predate the `{emoji, days, hex}` shape;
and converts old loads from `{ownerId, kind}` to `{subjectId}`. Keep this pattern — the
user has live data and there is no server-side migration path.

---

## 4. Storage: why photos are not in localStorage

This is the most important engineering constraint in the app.

localStorage is **~5 MB per origin, total**. A single iPad photo is **3–5 MB**. Putting
one garment photo in `S` would have broken the entire app — laundry loads included, not
just the wardrobe.

So:
- **Metadata** (`S`) → localStorage. Measured at 2 KB with a full wardrobe.
- **Photos** → IndexedDB (`laundry-hq-photos` / store `p`), which has orders of
  magnitude more room.
- Photos are **downscaled before storage**. A 122 KB test image stored at 19 KB.
- On boot, `photoAll()` loads every photo into the in-memory `PHOTOS` map, *then*
  re-renders. This is why `render()` can stay synchronous — `garment()` reads
  `PHOTOS[id]` directly. If IndexedDB is blocked (private browsing), it fails silently
  and garments fall back to flat colour.

**Consequence to preserve in any merged export feature:** the *Export JSON* button
exports `S` only. **It does not include photos.** If you build a unified backup, either
include the IndexedDB contents or say clearly that photos aren't covered.

---

## 5. The photo pipeline — read this before touching the Closet

This is the most confusing part of the codebase because it went through three designs.
There are currently **two live rendering paths plus a dead code path.**

### History (so you understand why it looks like this)

1. **v1** — the uploaded photo was clipped to the app's generated SVG garment silhouette
   at render time. Rejected: it distorted the user's image.
2. **v2** — masking removed entirely; the photo was drawn as-is, fitted into a per-slot
   box (`PHOTO_BOX`). Also in v2: a real bug was found where `downscale()` re-encoded
   *everything* to JPEG, and **JPEG has no alpha channel**, so a transparent PNG was
   flattened to an opaque rectangle before it was ever saved. Fixed by detecting alpha
   and encoding to WebP/PNG instead.
3. **v3 (current)** — a **photo modal** (`openPhotoModal`). The user picks a photo, sees
   it live inside a dashed outline of the garment shape, adjusts a zoom slider, and
   presses Save. `pmSave()` composites photo + shape mask **on a canvas at 2×**
   (400×760) using `globalCompositeOperation = 'destination-in'`, and stores the result
   as a pre-masked transparent PNG. This brings masking back but under user control,
   with live preview — which is why it is acceptable where v1 was not.

   The modal has **two** file inputs — "Choose photo" (library) and "Take photo"
   (`capture="environment"`). They were one button carrying `capture`, which on iOS
   forces the camera and makes existing library photos unreachable. Both route through
   `pmPick()`, which ignores a cancelled picker. If you touch this, keep the two-input
   split; a single input with `capture` is the bug.

   Modal internals: `_pmItemId`, `_pmImg`, `_pmScale`, `_pmDrawSeq`, `_pmMaskCache`,
   `_pmGetMask()`, `_pmDrawPlaceholder()`, `pmLoad()`, `pmPick()`, `pmDraw()`,
   `pmSave()`. `_pmDrawSeq` is a async-race guard — the zoom slider fires faster than
   mask images load, so stale draws must be discarded.

### Current rendering paths in `garment()`

| Condition | Behaviour |
|---|---|
| `item.masked === true` | Photo is a pre-masked 400×760 PNG aligned to the doll canvas. Drawn as `<image x=0 y=0 width=200 height=380>` — **no scaling logic, no clip**. |
| photo exists, `masked` falsy | **Legacy.** Drawn into `PHOTO_BOX[category]` with `preserveAspectRatio="xMidYMid meet"`. |
| no photo | The generated SVG shape, filled with `item.hex`. |

`itemThumb()` mirrors the same three cases.

**Any item photographed before v3 is on the legacy path.** Don't delete `PHOTO_BOX` or
the legacy branch without a migration, or those items will render wrong.

### Dead code — safe to delete

`attachPhoto()` is **never called** (the tile now calls `openPhotoModal`). It is the
only caller of `downscale()`, which is the only caller of `detectAlpha()`. That's a dead
chain of ~50 lines. Verified by grep. Either delete it during the merge or leave it, but
know it isn't live — don't "fix" it thinking it's the upload path.

### Geometry

- The doll is a **200 × 380 SVG viewBox**. Every garment shape and body part is authored
  in those coordinates. Don't change the viewBox casually — `SHAPES`, `BODY`, `ARM`,
  `PHOTO_BOX`, the canvas mask in `pmSave()`, and every hit zone all assume it.
- `SHAPES[kind]` = raw SVG elements with **no fill** — deliberately, so the same markup
  can be painted flat, used as a canvas mask, or measured.
- Garments and the body are **bezier paths**, not stacked rectangles (both were redrawn
  from rects; the body was redrawn again afterwards so its edges tuck under the garment
  paths). Edits here are fiddly — change one, then look at the figure in several
  outfits, because body edges are positioned to hide beneath specific garment lines.
- `shapeBBox(kind)` measures real bounds via `getBBox()` on the offscreen `#measure` SVG,
  memoised in `_bbox`. Used for thumbnail crops and photo placement.
- Body draws first, then garments in `SLOT_ORDER = ['bottom','top','outer','shoes','hat']`
  (back → front).

---

## 6. Design decisions worth preserving

These look arbitrary and are not. Please don't "clean them up".

**Colour**
- Dark theme only, deliberately — not an oversight, no light mode was ever built.
- The categorical palette (`--s1`…`--s8`) is **validated for colour-blind separation and
  contrast** against this specific surface (`#17171f`) — lightness band, chroma floor,
  adjacent-pair CVD ΔE, normal-vision ΔE, and 3:1 contrast all pass. If you change
  surface colours or reorder these slots in the merged app, **re-validate**; the slot
  *ordering* is the CVD-safety mechanism, not decoration.
- Charts: every bar is direct-labelled so colour never carries meaning alone.
- Household items (Towels/Rags/Sheets) are deliberately **neutral grey** in charts —
  they aren't people, and it keeps the family colours unambiguous.

**People**
- The user specified family colours: orange, purple, red, green, **black**, yellow.
  Pure black is invisible on this background, so the "black" slot is graphite
  (`#4a4a57`) with a hairline ring on avatars.
- **Names are deliberately NOT in the source.** The repo is public. Setup shows seven
  blank colour slots; names live only in the user's browser. Do not hardcode family
  names into the merged app.
- `ink()` computes black-or-white avatar text from the chip colour's relative luminance,
  so yellow and graphite chips are both legible.
- `assignInitials()` makes avatar initials **unique across the household** — two names
  starting with the same letters get different badges, otherwise the chips lie.

**Laundry timers**
- A finished cycle is green **"ready"** for 15 minutes (`NAG_AFTER`), then escalates to
  amber **"sitting Nm"**. This two-tier state is the actual point of the feature — the
  wet-clothes-left-overnight problem. `phaseOf(left)` returns `run | ready | nag`.

**Wardrobe**
- The mannequin is a **light tone** (`#a9a09a`) so bare arms and legs read as skin.
  An earlier dark mannequin made the figure look like a blocky robot and dark garments
  dissolved into the background. Garment outlines are **light** (`rgba(255,255,255,.30)`)
  for the same reason.
- Mix & Match is **draft-until-saved** (`mixDraft`), so hammering Shuffle can't destroy
  a plan the user already made.
- Shuffle is **season-aware** with a fallback (if filtering by season would strand a
  slot, it uses everything), and treats hats/outerwear as optional so you don't get a
  hat every time.
- **Band locks** exist because a random generator without them is a toy you tap six
  times and abandon. "Lock the jeans, shuffle the tops" is the real use case.

---

## 7. Known rough edges

- **No sync.** Every device has its own board. The user knows and accepted this; they
  have flagged shared state as a likely future step. This is the most likely reason the
  merged app would need a backend.
- Safari clears localStorage after ~7 days of not visiting a site. Home Screen install
  mitigates; *Export JSON* is the real backup (and doesn't cover photos — see §4).
- Dead photo-upload chain (§5).
- No tests. Verification was done by driving the live page in a browser and asserting on
  DOM/state — see §10.
- `subjChip()`, `PHOTO_BOX`, and the legacy photo branch are all lightly used; check
  references before removing.

---

## 8. What each tab does (quick map)

| Tab | Entry point | Notes |
|---|---|---|
| Board | `boardView()` | 5-stage pipeline + machine dials. Drag-and-drop is desktop-only sugar; buttons are the primary path. `wireDnD()`. |
| Week | `weekView()` | Day-of-week grid per subject, next-in-line, 7-day lookahead with gaps, household cadence rows. |
| Closet | `closetView()` | Two modes: `grid` (paper doll + wardrobe tiles) and `mix` (`mixView()`, five sliding bands + shuffle). `wireDoll()` / `wireBands()`. |
| Stats | `statsView()` | KPI tiles, 14-day column chart, per-subject bars, history table. Charts are hand-rolled SVG. |

---

## 9. Merge strategy — options and a recommendation

You do not yet know what the other two apps look like. Pick after you've read them, but
here is the honest trade space:

**A. Iframe shell.** A parent page with nav; each app in its own iframe, untouched.
- *Pros:* zero collision risk (JS, CSS, and DOM are all isolated), fastest by far,
  each app keeps working exactly as it does today.
- *Cons:* no shared state (the household roster stays duplicated three times), nav feels
  like three apps in a trenchcoat, awkward on mobile, three separate storage islands.
- Reasonable if the goal is mainly "one home screen icon".

**B. Namespace and merge into one page.** Wrap each app in an object/module, prefix CSS,
unify the shell (tabs, toasts, dialog, storage helpers, theme).
- *Pros:* genuinely one app; shared roster, shared theme, one export/backup.
- *Cons:* **requires converting all inline `onclick` handlers first** (§2.2) — that is
  the bulk of the work and is where silent breakage lives. Also requires resolving CSS
  collisions across three files.
- **This is what I'd recommend** if the user wants a real single app, with the handler
  conversion done and verified as a separate commit *before* any merging.

**C. Rewrite into a framework.** Honest option if all three are this size, but it is a
rewrite, not a merge — scope it explicitly with the user rather than drifting into it.

### The thing most worth unifying

Whatever route you take: **the household roster (people, names, colours) is almost
certainly duplicated across all three apps.** That is the highest-value thing to make
shared, and the most annoying thing for the user to maintain in triplicate. Design that
first and let the rest follow. Secondary candidates: theme tokens, the toast system, the
dialog, and the localStorage read/write helpers.

---

## 10. Before you start — ask the user

1. What are the other two apps, and are they also single-file no-build? Read all three
   before proposing a structure.
2. **One app or one shell?** (Option A vs B above.) This changes everything downstream.
3. Should the three keep **separate storage keys** or migrate to one namespaced store?
   If migrating: they have **real data on an iPad** with no server copy — a migration
   path and a backup step are mandatory, not optional.
4. Is the merged app still going to be a **public** repo? If so, keep the no-names-in-
   source rule (§6) and check the other two apps for personal data before pushing.
5. Does this merge finally need **cross-device sync**? If the answer is yes, that's a
   backend decision and should be made before the merge, not after.

## 11. How to verify your work

There is no test suite. What worked well:

- Open the page and drive it with JavaScript, then assert on state and DOM. Example
  pattern used throughout development: install `window.addEventListener('error', …)`,
  exercise every view / dialog / empty state in a loop, and return the collected errors.
- Check `document.body.scrollWidth > innerWidth` for horizontal overflow at 375 px
  (phone), 768 px (iPad portrait) and desktop.
- **Gotcha that wasted real time:** a preview pane can keep serving a *stale* copy of the
  file after an edit, and the console buffer can persist across navigations. Repeatedly
  during development, "errors" turned out to be from an older build. Before trusting a
  failure, confirm the loaded code is current — e.g. check that a symbol you just added
  exists — and prefer a fresh error trap over reading an accumulated console log. The
  `BUILD` stamp exists for the same reason on real devices.
- After pushing, poll the live URL for a string unique to the new build before telling
  the user it's deployed; Pages takes ~45–60 s.

## 12. Repo housekeeping you may hit

- More than one Claude session has worked on this repo. **Always `git fetch` and check
  `origin/main` before committing** — a local clone was found three commits behind, and
  a stale `.git/HEAD.lock` from an interrupted process had to be cleared. Rebase onto
  `origin/main`; do not force-push over other sessions' work.
- There is at least one extra remote branch (`claude/photo-button-modal-issue-*`).
  Check `git branch -r` before assuming `main` is the whole story.
