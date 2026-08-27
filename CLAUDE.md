# CLAUDE.md — PairdUps

> Persistent context for Claude Code. Loads at the start of every session.
> Place this file in the **repo root**. Keep it up to date as work progresses.

## What PairdUps is

PairdUps is a PWA (pairdups.com, hosted on Vercel) where users post short
activity listings ("PairdUps") and browse a Discover feed to send join
requests. It's an activity-partner matcher — think Bumble BFF + Meetup.
Features: messaging, three tiers (Free / Premium $4.99 / Gold $9.99 via Stripe),
gamification (XP + badges), and Gold-only perks (saved filters, up to 12 photos,
custom theme colors, pinned/featured listings, instant push, read receipts).

Goal: finish the pre-launch checklist, wrap with Capacitor, and submit to the
app stores.

## Stack

- **Frontend:** a single `public/index.html` (~12,880 lines, vanilla JS + inline
  styles). `admin.html` is a small admin panel. No build step, no framework.
- **Backend:** Supabase (Postgres, Auth, Storage, Edge Functions).
- **Payments:** Stripe subscriptions (`manage-subscription` + `stripe-webhook`).
- **Email:** `send-pairdups-email` edge function is the unified sender
  (`notifications@pairdups.com`); `send-welcome-email` sends the welcome email
  (`noreply@pairdups.com`), one per signup.
- **Push:** OneSignal web push (send by calling `send-pairdups-email` with
  `push_only: true`).
- **Hosting:** Vercel, auto-deploys on push to `main` of the repo.

## How to work in this repo (please follow)

1. **Start in Plan mode.** Propose the approach and wait for approval before
   editing. The owner is non-technical and reviews every diff — small, focused,
   clearly-explained changes only.
2. **One task → one PR.** Keep each change self-contained and easy to review and
   verify live. Don't batch unrelated changes.
3. **Verify after merge.** Merging to `main` triggers a Vercel deploy (~30s).
   Confirm a deploy landed by checking `view-source:https://pairdups.com/index.html`
   for a string unique to the new code (e.g. a new function name), then a
   hard-refresh + behavioral check.
4. **Quality over speed.** The owner explicitly prioritizes a clean, bug-free
   launch. When unsure, ask.

## Absolutely critical rules

1. **Filename case matters** on Linux/GitHub. The main file is `index.html`
   (lowercase i). Always state the exact path (`public/index.html`).
2. **Silent-RLS trap.** RLS-filtered UPDATEs return HTTP 200 with `data: []` /
   0 rows — that is NOT an error. Always log/verify the row count. The
   `profiles.email` reconcile and the theme save both read the value back after
   writing; keep that pattern.
3. **Supabase auth client** must be initialized with `persistSession: true`,
   `autoRefreshToken: true`, `detectSessionInUrl: true`,
   `storage: window.localStorage`, or sessions silently fail to persist → silent
   RLS write failures.
4. **Preserve the two email-reconcile paths.** Do not remove or break the code
   near the log strings `Detected confirmed email change` and
   `profiles.email reconciled to auth`. This was the original launch blocker.
5. **localStorage keys:** `pu_db` = USER_DB, `pu_ses` = session.
   (`pu_user_db` / `pu_session` do NOT exist.)
6. **Colorblind constraint (owner is red-green colorblind).** Never rely on
   color alone to convey meaning — distinguish by weight, size, shape, or
   position. Describe any proposed color in words + hex. This also improves
   accessibility for end users.
7. **Avoid spaces in any filename that becomes a URL** (OneSignal images,
   Supabase Storage uploads, repo assets) — use dashes (`pear-logo.png`).

## Recently completed (verify these stay intact)

All four shipped from a prior working session and are deployed unless noted:

- **"Need Ideas?" pill** — the Create-a-PairdUp title-field link is now a bolder,
  bordered pill with a gentle pulse (class `need-ideas-btn`, keyframe
  `ideaPulse`). Stands out by size/weight/shape, not color.
- **Discover feed sort fix (Polish A)** — `restoreUserListings()` now sorts the
  cached feed with `_discoverSortCmp` (mirrors the Supabase sort) so the cached
  first paint matches the final order — kills the load reorder flicker.
- **First-load veil** — a one-shot spinner (`_veilDiscoverFeed` /
  `_revealDiscoverFeed`, flag `_discoverInitialVeil`) hides the Discover feed
  until the sorted Supabase data has painted, so users only ever see the final
  Gold-sorted order. Has a 4s safety-timeout reveal and offline/cache fallback;
  resets on sign-out.
- **Image lazy loading (Item 12)** — `loading="lazy" decoding="async"` on the 6
  numerous/below-the-fold image sources (feed card photo + poster avatar, feed
  card carousel slides, gallery grid, gallery-viewer + detail carousels). Single
  detail-hero images left eager on purpose. NOTE: confirm this is deployed live
  (search view-source for `loading="lazy"`).

## Pre-launch checklist status

- [x] Item 1 — Alert debug logs cleaned (behind `window._DEBUG_ALERTS`)
- [x] Item 2 — Email template fix (renewal date)
- [x] Item 3 — Stripe key audit (clean)
- [x] Item 4 — Service-role key audit (clean)
- [x] Item 5 — Downgrade/cancellation end-to-end
- [x] Item 6 — Cross-device sync (alert prefs + user_settings + theme_color)
- [x] Item 7 — Empty states audit
- [ ] Item 8 — Slow network / offline behavior (not started)
- [ ] Item 9 — Push deep-link "second tab open" edge case; also verify no saved
      OneSignal template/scheduled push references the stale
      `p_Q9_Pear%20Pic.png` image (cosmetic, from an old test push)
- [x] Item 10 — Console error audit (clean; only benign non-app errors)
- [ ] Item 11 — Lighthouse audit — **IN PROGRESS.** Note: Lighthouse removed the
      PWA category (v12+); it now scores Performance, Accessibility, Best
      Practices, SEO. Check PWA installability separately via DevTools →
      Application → Manifest + the address-bar install icon. Run signed-in on
      mobile emulation. Watch for a "lazily-loaded LCP image" flag on the first
      feed card (targeted fix: mark the top card's image eager).
- [x] Item 12 — Image lazy loading (code done; confirm deployed)
- [ ] Item 13 — "Rate the app" prompt (after Capacitor)
- [x] Email-change fix (original launch blocker)

## After the checklist: Capacitor wrap + store submission

- **iOS** requires macOS + Xcode + an Apple Developer account ($99/yr). The owner
  is on **Windows** — plan for Mac access (a physical Mac or a cloud Mac service);
  no tooling removes this requirement.
- **Android** can be built on Windows with Android Studio; Google Play is a $25
  one-time fee.
- Open question carried forward: Stripe-vs-StoreKit / Google Play billing rules
  for in-app subscriptions (unresolved — research before wrapping).

## Environment values (stable)

- Supabase project ref: `phajurcdiotfvtcwvydk`
  (URL `https://phajurcdiotfvtcwvydk.supabase.co`)
- Vercel project: `pairdups`, org `teachersconnectteachers-projects`
- GitHub repo: `github.com/teachersconnectteachers/pairdups`
- Custom domain: `pairdups.com` (+ www + raw `pairdups.vercel.app`);
  Supabase Site URL = `https://pairdups.com`
- OneSignal app ID: `133a857e-2a5c-4afe-b726-031f74f27586`; default icon
  `pear-logo.png`
- Admin email (only `admin.html` access): `fredhockey23@gmail.com`

## Test accounts

- `fredhockey23@gmail.com` — owner's primary account. **Tier varies** — he pays
  to test a tier (e.g. Gold), then refunds via Stripe. Ask which tier it's on;
  don't "correct" it.
- Juan — currently `fredhockey23+empty6@gmail.com` — main messaging/test account.
- `fredhockey23+test1@gmail.com`, `+empty1..+empty6@gmail.com` — throwaway
  signups (Gmail `+` aliases route to the main inbox).

## Edge functions

Source in the repo may be stale — pull current source from the Supabase Dashboard
before editing `send-pairdups-email`, `manage-subscription`, or `stripe-webhook`.

## Supabase config to verify before real launch

- "Password changed" / "Email address changed" notifications ON (confirmed).
- "Change email address" template exists (confirmed).
- Redirect URLs `https://pairdups.com/*` + `https://www.pairdups.com/*`.
- **"Confirm email" toggle** (Sign In / Providers): OFF for testing, must be **ON**
  before real launch — confirm with the owner.
