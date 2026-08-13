# VetDIY Waitlist — Go-Live runbook

> ## ⚠️ HISTORICAL — THIS GO-LIVE IS COMPLETE. DO NOT RUN THESE COMMANDS.
>
> This document describes a go-live that **finished on 2026-07-03**. It is kept for provenance
> only. The landing page is **not** in preview mode — it is live and wired to Supabase project
> `njdzwcoognjmzibihewl`, and the waitlist backend was **hardened on 2026-07-23**.
>
> **Running the commands below would damage production**, because they:
> - deploy from branch `feat/waitlist-backend`, which is frozen at 2026-07-12 and **predates the
>   hardening** — it would strip `escapeHtml()` from outbound email and the per-IP `rateLimit()`
>   from the public signup endpoint;
> - run a bare `supabase db push` against the live project, which the marketplace repo's own
>   runbook names as its rule #1 never to run;
> - omit `--no-verify-jwt`, flipping the functions to `verify_jwt=true` and breaking every raw
>   email link (confirm / unsubscribe) plus the landing page's publishable-key auth;
> - reset `WAITLIST_SITE_URL` to the old GitHub Pages URL, repointing every confirm, share and
>   unsubscribe link in live email away from `vetdiy.com`.
>
> **The authoritative source and the only sanctioned deploy procedure** live in the marketplace
> repo: `docs/WAITLIST-DEPLOY-RUNBOOK.md`, deploying from
> `All Code Projects/VetDIY/supabase/functions/waitlist-*`.

Backend code lives in the marketplace repo on branch **`feat/waitlist-backend`**:
- `supabase/migrations/0025_waitlist.sql` (additive — waitlist tables + counter + RLS)
- `supabase/functions/waitlist-submit` and `supabase/functions/waitlist-confirm`

All of it is verified end-to-end against **local** Supabase. The only things gated on your accounts are the
**hosted Supabase project** and the **Resend key** (email).

## 1. Create the hosted Supabase project (free, no card)
- supabase.com → **New project**. Save the **Project URL** (`https://<ref>.supabase.co`) and the **anon public key** (Settings → API).

## 2. Deploy the schema + functions
From the marketplace repo, on `feat/waitlist-backend`:
```bash
supabase link --project-ref <ref>
supabase db push                       # applies 0025_waitlist.sql (additive; touches nothing else)
supabase functions deploy waitlist-submit
supabase functions deploy waitlist-confirm
```

## 3. Email (Resend — free 3,000/mo, 100/day, no card)
- resend.com → **API Keys** → create a key.
- Add + verify your sending domain (recommended) — or use `onboarding@resend.dev` for a quick test.
- Set the function secrets:
```bash
supabase secrets set RESEND_API_KEY=re_xxxxx
supabase secrets set WAITLIST_FROM_EMAIL="VetDIY <waitlist@yourdomain.com>"
supabase secrets set WAITLIST_SITE_URL="https://marcusdavenport.github.io/second-day-repo"
```

## 4. Flip the page live
- In `index.html`, set `CONFIG.supabaseUrl` + `CONFIG.anonKey` (from step 1).
- `git commit -am "go-live: connect waitlist to Supabase" && git push`. Pages redeploys in ~1 min.

## 5. Verify (the real go-live gate — don't claim live until these pass)
1. Submit a test signup on the page → a row appears in **Supabase → Table editor → `waitlist_entries`**.
2. The confirmation email arrives (Resend) → click it → page shows your position, `real_signups` +1.
3. Open your invite link `?ref=CODE` in a private window, sign up + confirm → your points jump **+100**.

## Operating it
- **Invite targeting** (queryable city/profile filters):
  ```sql
  select email, phone, city, zip, trade, role, points
  from waitlist_entries where confirmed and city = 'Tampa' order by points desc;
  ```
- **Counter** (base / real / drift are separate, display never goes backward):
  ```sql
  select base_count, real_signups, display_drift, base_count+real_signups+display_drift as shown
  from waitlist_counter;
  ```
  Tune the organic rate in `waitlist_tick()` (`rate_per_hr := 180 + random()*240`).
- **Export**: Table editor → `waitlist_entries` → Export CSV.

## Anti-gaming already built in
One entry per email (dedupe), a bot **honeypot** (`company` field), referral points awarded **only after the
referee confirms their email**, and the FOMO counter is server-authoritative + monotonic.
