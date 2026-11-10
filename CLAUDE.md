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
app stores. **As of Oct 24, 2026 the Android Google Play submission is well
underway — see "Google Play submission progress" below.**

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
   writing; keep that pattern. **See also the Oct 23 finding below: client
   writes also silently fail BEFORE a new account confirms its email, because
   the session isn't authenticated yet.**
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
8. **Do NOT add a www↔non-www redirect in `vercel.json`.** Vercel already
   canonicalizes the custom domain at the platform level, so a repo redirect
   fights Vercel's own primary-domain redirect → an infinite loop
   (`ERR_TOO_MANY_REDIRECTS`) that takes the live site down. This happened
   Nov 7, 2026 (added in PR #33, reverted in PR #34). To force a single canonical
   host, set the **primary domain in the Vercel dashboard**, never a `vercel.json`
   redirect. Also: a `permanent` (301) redirect is cached hard by browsers, so a
   bad one keeps looping even after removal until the cache is cleared — prefer
   `302`/temporary for any redirect you do add.

## Membership / tier mechanics (how Gold/Premium work)

- Tier is stored in `profiles.subscription_tier` — values `'free'`, `'premium'`,
  `'gold'` — plus `profiles.subscription_expires_at`.
- `getUserTier()` (index.html ~L3331) returns the tier, but downgrades to
  `'free'` only if `subscription_expires_at` exists AND is past (with a 24h
  grace). If `subscription_expires_at` is **NULL**, the tier is treated as
  permanent (never expires). So **tier=gold + expires=NULL = permanent Gold**.
- `isPremium()` = premium or gold; `isGold()` = gold. `getPhotoLimit()` = 12
  gold / 5 premium / 1 free.
- `window.setTestTier('gold'|'premium'|'free')` is a debug helper BUT has a bug:
  its DB write (`.update(...).eq('id', ...)`) is **not awaited**, and an
  un-awaited supabase-js query is never actually sent, so it flips the on-screen
  tier but doesn't persist. To set a tier reliably, either edit the row in the
  Supabase Table Editor, or run an awaited console update, e.g.
  `await _supabase.from('profiles').update({subscription_tier:'gold',subscription_expires_at:null}).eq('id',currentUser.supabaseId).select()`.
  (Fixing setTestTier to await is a queued minor fix.)

## Queued code fixes (status)

1. **Silent write failures (onboarding + listings data loss). ✅ FIXED.** Root
   cause: without a fully authenticated Supabase session, client writes are
   rejected while the UI updates optimistically, so data silently vanishes
   (profiles: 0 rows; listings INSERT: RLS error 42501).
   - *Onboarding/email-confirmation path* — fixed in **PR #27**: signup with
     "Confirm email" ON now shows a "Confirm your email" modal instead of
     dropping unconfirmed users into profile setup where writes fail.
   - *Listing-create path* — fixed in **PR #31 (Nov 7)**: Supabase client now
     initialized with `persistSession`/`autoRefreshToken`/`detectSessionInUrl`/
     `storage` (rule #3); `_createListingFromForm()` verifies a live session
     (`auth.getUser()` id == owner) and awaits the insert BEFORE celebrating,
     and on failure shows a clear toast and leaves no ghost listing. Found via a
     real tester (Brendan) whose posted PairdUp silently failed — Postgres 42501
     in the Supabase logs. (The `listings` RLS INSERT policy is correct; the bug
     was entirely client-side.)
2. **Edited-listing detail card not opening. ✅ FIXED (PR #28).** The edit path
   in `saveListing()` now uses the same `esc()` escaping as the display path,
   and `esc()` was hardened to escape line terminators — multi-line descriptions
   were breaking the inline onclick (`Uncaught SyntaxError`).
3. **Admin "edit user account" feature. ✅ DONE (PR #29).** `admin.html` Users
   table now has a membership-tier selector (free/premium/gold, permanent via
   `subscription_expires_at=NULL`) and an Edit modal for name/location/birthday/
   bio (email/password deliberately excluded to protect the email-reconcile gate).
4. **setTestTier await bug (minor, debug-only). Open but moot** — the admin UI
   (#3) now sets tiers reliably; the un-awaited debug helper is low priority.
5. **Session persistence + clean logout. ✅ DONE (PR #31, #33, #36).** With
   `autoRefreshToken` on, access tokens (3600s) refresh silently, so signed-in
   users stay logged in across visits — Supabase → Auth → Sessions confirmed at
   Time-box = never, Inactivity = never, single-session off (leave these).
   The listing-post path refreshes a lapsed token before giving up (PR #33).
   When a session genuinely dies (refresh token revoked, or signed out on
   another device), the `onAuthStateChange` listener catches `SIGNED_OUT` (when
   `isSignedIn` and not a user-initiated sign-out, guarded by `_signingOut`) and
   runs `logOut('Your session ended — please sign in again.')` to land the user
   on the welcome/login screen — no more half-logged-in state where writes fail.
   (Brendan's suggestion, Nov 9.)

## Google Play submission progress (updated Nov 7, 2026)

**Store assets — DONE (all created, delivered, saved on owner's PC):**
- App icon (512 store + 1024 launcher), feature graphic (1024×500), 8 final
  phone screenshots (numbered upload order), short + full descriptions, privacy
  policy (live at `pairdups.com/privacy.html`). The screenshot set leads with a
  diverse Discover feed (female + male listings) and balances dating vs.
  activity-partner framing.

**App created in Play Console — DONE:**
- App name: **PairdUps: Activity Partners**
- Package name (PERMANENT, registered to the account): **`com.pairdups.app`**
  (matches `appId` in `capacitor.config.ts`).
- App, Free, English (US).

**App content declarations — ALL DONE:**
- Content rating (IARC): submitted. Social category, "significant portion used
  for dating = Yes" → Mature 17+/PEGI 18/USK 18 across regions (fits 18+).
- Sign in details: reviewer demo account provided (see Test accounts).
- Target audience: 18 and over; "Restrict users Google determines to be minors"
  opt-in = ON.
- Data safety: submitted. No third-party sharing; encrypted in transit = Yes;
  deletion URLs = pairdups.com/privacy.html; collects Name/Email/User IDs/Other
  info (age), Approximate location, In-app messages, Photos, App interactions +
  Other UGC, Device/other IDs; purposes mostly App functionality (+ Account
  management on identity, Developer communications on email, Analytics on app
  interactions).
- Ads = No; Advertising ID = No; Government apps = No; Financial features =
  None; Health = None.
- Child Safety Standards: submitted. Published standards page live at
  `pairdups.com/child-safety.html` (**PR #30**); in-app Report User / Report
  listing / Block User satisfy the required reporting mechanism.

**Closed test — LIVE (app APPROVED Oct 29, 2026):**
- **Main store listing — DONE:** icon, feature graphic, 8 screenshots,
  descriptions; category = **Social**; contact details set.
- **Signing key + signed `.aab` — DONE:** upload key generated and backed up
  (Carbonite + a `.jks` entry in Dashlane Secure Notes). Losing it = no future
  updates.
- **Closed-testing release — LIVE & APPROVED (Oct 29):** release "Available to
  selected testers." Testers are managed by a **Google Group**
  (`pairdups-testers@googlegroups.com`); the track is set to Google-Groups mode
  (NOT an email list — the two are mutually exclusive, and switching to Groups
  drops the old email-list testers unless they're added to the group). Group is
  "anyone can join." Reviewer demo account (`fredhockey23+demo`) does NOT need to
  be in the group — it's a `+` alias of the owner account, which is already a
  member.
- **Now recruiting toward 12 opted-in testers for 14 continuous days**, then
  apply for production. CRITICAL distinction: *group join ≠ opt-in ≠ engagement.*
  Google counts **opt-ins + real app sessions** (Android Vitals / installs), not
  group membership. Monitor real engagement via Play Console Statistics (lags
  ~2–3 days) and the app's own Supabase `auth.users` last-sign-in + `listings`
  activity — NOT the group member count (browse-without-account testers are
  real but invisible in `auth.users`). Considering a paid top-up (e.g. TesterBee,
  ~$15 for 15) to reach 12 — buy engagement, not just opt-ins; never give any
  service Google/Play Console account access (only the public group + opt-in
  links). Reddit r/AndroidAppTesters + ACT Party used for free reciprocal swaps.

## Pre-launch checklist status (code)

- [x] Items 1–12 complete (see git history / prior notes). Highlights: offline
      behavior, push/OneSignal audit, console-error audit, Lighthouse (A11y/BP/
      SEO 100, Perf 84), image lazy-loading.
- [ ] Item 13 — "Rate the app" prompt (after Capacitor).
- [x] Email-change fix (original launch blocker).

## Recently completed (code; verify these stay intact)

- **"Need Ideas?" pill**, **Discover feed sort fix**, **first-load veil**,
  **image lazy loading**, **offline awareness** (banner + write-blocking at
  `_createListingFromForm()` / `sendMsg()` / `sendJoinRequest()`),
  **My PairdUps reconciliation fix**, **Discover filter badge + sync fixes**,
  and **Gold alert bell read/dismiss states**. (Full detail in prior git
  history; these all shipped from earlier sessions and are deployed.)

## Launch / Capacitor

**Strategy:** Android first — the owner is on Windows, so Android needs no
Mac. iOS is a post-launch fast-follow via a cloud Mac service.

**Decisions locked:** Google Play first; iOS later via a cloud Mac. **Package ID
= `com.pairdups.app` (FINALIZED & registered at app creation — cannot change).**

**v1.1 decision (Oct 8, 2026):** native push (Stage 3) deferred to post-launch
v1.1. v1.0 ships with web push + email + in-app alerts; only phone lock-screen
push banners are absent until v1.1.

### Stages

- **Stage 0 — Google Play Developer account. DONE (Sept 30, 2026):** Personal
  account, $25 paid, identity + phone verified.
- **Stage 1 — Windows toolchain. DONE.** Node 22 LTS, Android Studio, JDK.
- **Stage 2 — Capacitor added. DONE.** `capacitor.config.ts`, Android platform,
  `appId: com.pairdups.app`, pointed at `public/`.
- **Stage 3 — Native push. DEFERRED to v1.1.**
- **Stage 4 — Build & test in Android Studio. DONE (Oct 6, 2026):** builds and
  runs on the emulator (sign-in, Discover feed, nav, create listing confirmed).
  Needed Node 22, TypeScript dev dep, Gradle JVM 21.
- **Stage 5 — Store assets. DONE (Oct 23–24, 2026):** icon, feature graphic,
  8 screenshots, descriptions, privacy policy, content rating, data-safety — all
  complete. See "Google Play submission progress" above.
- **Stage 6 — Signing key + release build. DONE (Nov 2026):** upload key
  generated and backed up; signed release `.aab` built in Android Studio.
- **Stage 7 — Submit via Play Console. IN PROGRESS.** App APPROVED Oct 29;
  closed test LIVE. Remaining: reach 12 opted-in testers for 14 continuous days
  (recruiting via the Google Group), then apply for production.
- **Stage 8 — Post-launch.** iOS via cloud Mac; updates repeat Stages 4→7.

## Test accounts

- **`fredhockey23+demo@gmail.com` — "Alex" — GOOGLE PLAY REVIEWER DEMO ACCOUNT.
  Do NOT delete, change its password, or remove Gold.** Set to permanent Gold
  (`profiles.subscription_tier='gold'`, `subscription_expires_at=NULL`). Its
  credentials + instructions are entered in Play Console → App content → Sign in
  details. Email is confirmed. profiles.id = auth uid =
  `a7929441-1e30-46c9-a3e4-dafce1dfc5ef`.
- `fredhockey23@gmail.com` — owner's primary account. **Tier varies** — he pays
  to test a tier, then refunds via Stripe. Ask which tier it's on; don't
  "correct" it. Also the only `admin.html` login.
- Juan — `fredhockey23+juan@gmail.com` — main messaging/test account.
- Amber — `fredhockey23+amber@gmail.com` — female test account (Ann Arbor, MI);
  created for screenshot gender balance.
- `fredhockey23+test1@gmail.com`, `+empty1..+empty6@gmail.com` — throwaway
  signups (Gmail `+` aliases route to the main inbox).

## Edge functions

Source in the repo may be stale — pull current source from the Supabase Dashboard
before editing `send-pairdups-email`, `manage-subscription`, or `stripe-webhook`.

## Post-launch / backlog

- **`profile_visitors` insert returns 401 / RLS policy violation** on profile
  views — likely a Supabase-dashboard RLS insert-policy fix, not index.html.
- **Offline queue-and-sync** — offline actions are blocked with a toast, not
  queued for retry. Deferred for v1.
- **OneSignal MCP connector** — explore post-launch for push campaign management.
- **Web-push opt-in is low** — expected to be superseded by native push (v1.1).
- **Spam/abuse protection:** "Confirm email" is ON (done Oct 8). `blocked_users`,
  `reports`, `support_tickets` tables exist; block user, report user, and report
  listing are all wired up in-app (confirmed Oct 23 — this also satisfies
  Google's UGC report/block requirement). Content moderation on listings is
  still a possible add.
- **Must-do before real users:** (1) Upgrade Supabase Free → Pro. ✅ DONE
  (Nov 5, 2026) — now on Pro + bumped compute (baseline disk IO 5→11 MB/s, daily
  backups). NOTE: the "Disk IO Budget about to deplete" warning is a per-tier
  BURST-budget heads-up, not an outage; it still appears under sustained IO. If
  it recurs, don't just upsize compute — find the disk-heavy query (prime
  suspect: the `profile_visitors` 401/RLS loop below) via Reports → Query
  Performance. A separate "Unhealthy / connection timeout" outage on Nov 5 was
  cleared by a project restart. (2) Test the signup confirmation email
  end-to-end (web + native). (3) Verify welcome / new-message / join-request
  emails fire from the native app flow.
- **In-app Privacy Policy link UX** — Settings → Privacy Policy opens
  `pairdups.com/privacy.html` via `window.open` (leaves the app on mobile).
  Consider an in-app slide-up/webview post-launch. Google's requirement (an
  in-app link) is already satisfied — not a blocker.
- **Photo cropping on cards — mostly resolved:** on a real device (S21+), one
  tester's photo (Juan's) was cut off while others framed fine; fixed by
  re-uploading/repositioning that photo. Photos are Supabase DATA, so a re-upload
  reflects in the installed app immediately (no rebuild). Watch for recurrence
  across device sizes — if several photos clip, revisit the card `object-position`.

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
- Android package ID: `com.pairdups.app`

## Supabase config (verified)

- "Password changed" / "Email address changed" notifications ON (confirmed).
- "Change email address" template exists (confirmed).
- Redirect URLs `https://pairdups.com/*` + `https://www.pairdups.com/*`.
- **"Confirm email" toggle: ON** (Oct 8, 2026). New signups require email
  confirmation before first login. NOTE the onboarding-write UX gap this creates
  — see Queued code fixes #1.
