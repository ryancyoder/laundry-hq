# Laundry HQ — developer handoff

Everything you need to work on this app: what it is, how it is built, why it is built
that way, and the traps that will cost you time.

**Supersedes the earlier three-app merge brief.** That merge was cancelled — Laundry HQ
stays an independent app with its own repo and its own URL. It does, however, share a
Supabase project with a second app; see §5, which is the single most surprising thing
about this codebase.

---

## 1. What this is

A laundry tracker for a household of seven.

- **Repo:** https://github.com/ryancyoder/laundry-hq (public)
- **Live:** https://ryancyoder.github.io/laundry-hq/ (GitHub Pages, `main`, root)
- **One file:** `index.html` — ~2,280 lines, ~101 KB. No build step, no bundler, no
  framework. Inline `<style>`, one classic `<script>`, plus a small ES-module script
  that loads `supabase-js` from a CDN.
- Deploy = `git push`. Pages rebuilds in ~45–60 s.
- Primary device is an **iPad**, added to the Home Screen. Design and test touch first.

**Three tabs:** Board (the laundry pipeline), Week (the schedule), Stats.

**Why single-file, no build:** the user wanted something openable immediately on family
devices with no tooling. That constraint drove most of the design. Don't convert it to a
framework unless asked.

### Build stamp

`const BUILD = '…'` near the top, shown at the bottom of ⚙ Settings. There is no service
worker and no cache-busting, so **a phone can easily be running an old copy**. When a bug
is reported, ask what build they see before debugging — this wasted real time more than
once. Bump `BUILD` when you ship.

---

## 2. Architecture and its traps

### 2.1 Everything is global, and must stay that way

~100 top-level functions and ~60 bindings, all global. **Most interactivity is inline
`onclick="fn()"` written into HTML strings.** If you wrap the script in a module, an IIFE,
or any bundler that scopes top-level declarations, **every button silently stops working**
— no error, just dead clicks. Converting to delegated listeners is possible but must be
its own verified commit.

### 2.2 Full re-render on every change

`commit()` = `save()` + `render()`, and `render()` replaces `#app.innerHTML`. Two
workarounds exist because of this and must be preserved:

- **`tick()` runs every second** and surgically updates only timer text, progress bars and
  status classes via `data-*` selectors. It does **not** call `render()` unless a timer
  actually fires — a full re-render every second would destroy input focus and restart CSS
  animations.
- **Scroll position resets on render.** Anything scrollable must repaint narrowly rather
  than via `render()`.

### 2.3 One shared `<dialog id="dlg">`

Every modal writes into the same element via `innerHTML`. `showDlg(full)` sets the mode
**when opening**. Do not go back to clearing state on the dialog's `close` event — that
event is **not reliably delivered** (it never fires at all in at least one engine tested
here), which previously left Settings opening full-bleed after the photo cropper had run.

### 2.4 Other singletons

`#app`, `#toasts`, `#confetti`, `#aurora`, `#bubbles`, `#measure` (offscreen SVG used by
`shapeBBox()` — must be in the DOM and not `display:none`). A global
`setInterval(tick, 1000)`. `render()` writes `document.title`. Bare **`n`** is bound to
"new load" on `window`.

---

## 3. Data and sync

### Signed out — local only

`localStorage` key `yoder-laundry-v1` holds the whole `S` object. `blank()` is the schema
of record; read it first. `load()` does forward-compatible repair on every read (backfills
new fields, migrates old shapes) — keep that pattern, the user has live data and there is
no server-side migration path.

### Signed in — the database wins

`isSynced()` is the switch. When true:

- **`save()` becomes a no-op**, so a device's own local board is preserved untouched and
  returns intact on sign-out. Nothing is migrated or destroyed.
- **Reads are a full pull** (`pullAll()`). At household scale that is simpler and less
  error-prone than diffing, and the round trip is negligible.
- **Writes are per-row** (`pushLoad`, `pushPerson`, `pushChore`, `pushMachine`) so two
  phones editing at once don't clobber each other — the entire point of syncing.
- **Realtime** subscriptions on the four laundry tables trigger a debounced re-pull.

### Ids

In-memory ids are strings with a type prefix: `p12` person, `c3` chore, `m1` machine,
`l88` load. `dbId()` strips the prefix. This keeps the existing subject model working and
keeps person and chore ids distinct inside `subjectId`.

### The "subject" abstraction

A load does **not** have a type. It belongs to exactly one **subject** — a person *or* a
household item (Towels, Rags, Sheets). `subjects()` concatenates both; `subjectOf(id)`
resolves either. This replaced an earlier design with both an owner *and* a garment-type
dropdown, which overlapped confusingly. **Do not reintroduce a type field on loads.**

One status function, `subjectStatus()`, covers both:
- With target weekdays → "has anything finished since that day last came round?"
- Without them (Sheets, monthly) → fall back to the `everyDays` interval.
- **No finished loads at all → `none` / "no loads logged yet", deliberately not
  "behind".** A new board must not open by accusing the whole family. This was a real bug;
  don't regress it.

### Timers

A finished cycle is green **"ready"** for 15 minutes (`NAG_AFTER`), then escalates to amber
**"sitting Nm"**. `phaseOf(left)` returns `run | ready | nag`. This two-tier state is the
actual point of the feature — the wet-clothes-overnight problem.

---

## 4. Auth

Google sign-in via Supabase. **👤** in the header → Account dialog → sign in / sign out /
join with a code.

- **The welcome screen leads with Sign in.** The local-only setup path is hidden and
  reappears *only* if the Supabase client fails to load, so a dropped network can't lock
  someone out. There's also a 9-second timeout so a hanging CDN doesn't strand a new device
  on "Connecting…".
- The account button is the one header control that stays visible during first-run setup —
  a new device must be able to sign in without first inventing a local roster.

### Adding a family member

1. Add their Gmail as a **test user** in Google Cloud (the consent screen is in Testing
   mode, cap 100): `https://console.cloud.google.com/auth/audience?project=laundry-app-506314`
2. They open the app → **Sign in with Google** → enter the **household code**.

The code lives in ⚙ Settings, is rotatable, and is 8 chars from a 31-char alphabet with
`0/O/1/I/L` excluded so it can be read aloud. `join_household()` and `rotate_join_code()`
are `SECURITY DEFINER` (joining writes a membership row the caller can't write directly);
both guard internally and `EXECUTE` is granted to `authenticated` only, so anonymous
visitors can't brute-force codes.

### Google Cloud config (do not re-derive; this took several passes)

- Project `laundry-app-506314`, one Web application client.
- Authorized redirect URI: `https://ivjxtlznikqxyscyyxzk.supabase.co/auth/v1/callback`
- Consent screen audience must be **External**, not Internal (Internal → `403 org_internal`
  for personal Gmail).
- Supabase → URL Configuration must list **both** `https://ryancyoder.github.io/laundry-hq/`
  and `…/laundry-hq/**`. The docs don't confirm `**` matches an empty tail, and the app
  redirects to the bare path, so keep both.

Failures we hit, in order, each with a distinctive signature:
`redirect_uri_mismatch` (URI never saved — the "+ ADD URI" button was not clicked) →
`403 org_internal` (audience was Internal) → `invalid_client` (wrong client secret).
**The Supabase auth logs name the cause exactly**; read them before guessing.

---

## 5. ⚠️ The shared Supabase project

**This is the thing that will surprise you.** Laundry HQ shares a Supabase project with
**Larder**, a separate kitchen-inventory app (repo `ryancyoder/larder`, React PWA).

- Project ref `ivjxtlznikqxyscyyxzk`, publishable key is in the source (that's correct —
  RLS is the security boundary).
- **Shared:** `auth.users`, `households`, `household_members`, and the
  `private.auth_household_ids()` RLS helper. One sign-in covers both apps.
- **Not shared:** `public.people` belongs to **Larder** (meal planning: "Littles" is a
  group, "Family meal" is not a person). Laundry HQ has its own `laundry_people`. Do not
  merge them — an earlier plan to share the roster was abandoned for exactly this reason.

Laundry HQ's tables: `laundry_people`, `laundry_loads`, `laundry_machines`,
`laundry_chores`, plus `wardrobe_items` and `outfit_plans` (unused while the Closet is
parked — see §6). All follow the same policy shape:
`household_id in (select private.auth_household_ids())`.

### A trigger was disabled

`handle_new_user()` on `auth.users` used to create a fresh "Our kitchen" household for
every new sign-up. That gave each new family member their own private household instead of
the family's. **The trigger is dropped; the function is deliberately kept** so restoring is
one statement:

```sql
create trigger on_auth_user_created after insert on auth.users
  for each row execute function public.handle_new_user();
```

`syncStart()` is also defensive about it: it fetches **all** memberships and picks the
household that actually has a laundry roster, falling back to the join prompt. Keep that —
it's what makes a stray empty household harmless.

**Current state:** one household (`10 Yoder`), three users (ryan.c.yoder, selah.may.yoder,
bjoyyoder), zero triggers on `auth.users`.

---

## 6. The Closet is parked, not deleted

A wardrobe/outfit-planner feature (paper doll, garment SVGs, photo masking, mix-and-match
sliding bands) was removed from the live app on 2026-08-22 to keep it focused while sync
landed.

- `parked/closet-reference.html` is the **complete, runnable app** as of build
  `2026-08-22b` — the last version with the Closet in it. Open it in a browser and the
  Closet works, photos and all. It's a working reference, not a fragment.
- `parked/README.md` explains how to bring it back.
- `wardrobe_items` and `outfit_plans` still exist in the database, empty and harmless.

If it's revived, the hard-won details are: photos must go to IndexedDB (localStorage is
~5 MB, one iPad photo is 3–5 MB), a photo must **never** be masked by a generated shape,
and JPEG re-encoding destroys the alpha channel of a cut-out PNG.

---

## 7. Design decisions that look arbitrary and aren't

- **Dark theme only**, deliberately.
- **The categorical palette** (`--s1`…`--s8`) is validated for colour-blind separation and
  contrast against `--surface` (`#17171f`). The slot *ordering* is the CVD-safety
  mechanism, not decoration. Re-validate if you change surfaces or reorder.
- Charts direct-label every bar so colour never carries meaning alone.
- **"Dad" is graphite, not black** — pure black is invisible on this background. `ink()`
  computes black-or-white avatar text from luminance so yellow and graphite are both
  legible. `assignInitials()` makes avatar initials unique across the household, so two
  similar names don't share a badge.
- **Names are never in the source.** The repo is public; the roster is entered at runtime.

---

## 8. Known gaps

- **Notifications only fire while the app is open.** No service worker, no push. The timers'
  whole purpose is catching wet clothes before they sit, and a backgrounded iOS web app
  can't wake itself. Real background alerts need a service worker plus Web Push, which
  needs a server to send them. This is the biggest outstanding item.
- **Export JSON does not include photos** (moot while the Closet is parked, but the
  wardrobe's photos live in IndexedDB and were never in that backup).
- No test suite. Verification is by driving the live page and asserting on DOM/state.
- Safari clears a non-installed site's localStorage after ~7 days idle. Much less serious
  now the board syncs, but true for signed-out use.

---

## 9. Working practices that paid off

- **Always `git fetch` and check `origin/main` before committing.** More than one agent
  session has worked in this repo; a local clone was once three commits behind, and a
  stale `.git/HEAD.lock` had to be cleared. Rebase, never force-push over other work.
- **After pushing, poll the live URL for the new `BUILD` string** before telling the user
  it's deployed.
- **Preview panes serve stale copies.** Repeatedly during development, "errors" turned out
  to be from an older build. Before trusting a failure, confirm the loaded code is current
  — check that a symbol you just added exists — and prefer a fresh error trap over an
  accumulated console log.
- **Read the Supabase auth logs** for any sign-in problem. They named the exact cause every
  single time; the browser-side message never did.
