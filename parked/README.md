# Parked: the Closet / wardrobe feature

Removed from the live app on 2026-08-22 to keep Laundry HQ focused while sync
was landing. **Nothing here is lost** — this directory holds a complete,
runnable copy, and the full history is in git.

## What's here

`closet-reference.html` — the entire app exactly as it stood at build
`2026-08-22b`, the last version with the Closet in it. Open it in a browser and
the Closet works, wardrobe photos and all. It is a working reference, not a
fragment, so nothing can have been missed in the extraction.

Also recoverable from git:

```bash
git show 2026-08-22b-closet:index.html > closet-app.html   # if the tag exists
git log --oneline -- index.html                            # find the commit
git show <commit>:index.html > closet-app.html
```

The Closet commits, newest first:
`6de80a0` full-screen photo cropper · `fd2e9be` doll redraw ·
`cec071e` photo modal library access · `7e01044` polished templates + canvas
masking · `f0ece32` mix & match · `e329f5c` never mask uploads ·
`725e916` cut-out photos · `f9b40eb` original paper doll

## What was removed from index.html

- **JS**, one contiguous block (~1,140 lines): the `WARDROBE — model`,
  `WARDROBE — photos`, `PHOTO UPLOAD MODAL`, `WARDROBE — paper doll` and
  `WARDROBE — view` sections.
- **CSS**: the `wardrobe / paper doll`, `mix & match` and
  `full-screen dialog (photo cropping)` blocks.
- **Markup**: the Closet tab, and the offscreen `#measure` SVG used by
  `shapeBBox()` for `getBBox()`.
- **State**: `S.wardrobe` and `S.plans`, plus their migration guards in `load()`.
- Assorted wiring: the `render()` dispatch branch, `wireDoll()`, the `setView`
  tab list, and the fabbar's closet exception.

`.chores` / `.chore-row` CSS was **kept** — those style the Week view's
household rows (Towels, Rags, Sheets), which are laundry, not wardrobe.

## Database

The Supabase tables were **left in place** and are empty:

- `wardrobe_items` — garments; `photo_path` was to point at Supabase Storage
- `outfit_plans` — one row per filled outfit slot

Both have RLS enabled and use the same `private.auth_household_ids()` pattern as
everything else, and both key off `laundry_people`. They cost nothing sitting
there, and they mean the schema work doesn't have to be redone.

## If you bring it back

Three things were unfinished and should be settled before it ships again:

1. **Photos were never synced.** They lived in IndexedDB on the device. Making
   the Closet shared needs a Supabase Storage bucket, an upload path, and
   signed URLs — the reason the Closet stayed device-local when the Board and
   Week went shared.
2. **The roster changed underneath it.** The parked code reads `S.people` from
   localStorage. The app now loads people from `laundry_people` with prefixed
   ids (`p12`), so `wardrobe_items.person_id` needs the prefix stripped.
3. **The photo pipeline had two live paths** — a pre-masked PNG (`masked:true`,
   drawn over the full doll canvas) and a legacy unmasked one fitted to
   `PHOTO_BOX`. Any revival should collapse those into one.
