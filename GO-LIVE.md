# VetDIY Waitlist — Go-Live runbook

The landing page is **live on GitHub Pages** and runs in **preview mode** (animated counter + optimistic
success, nothing persisted) until you connect Supabase. Flipping it live is a ~15-minute, free, no-credit-card
process.

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
