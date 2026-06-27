# VetDIY waitlist live backend + design preservation spec

## Operator request and success criteria

Marcus wants the live VetDIY waitlist at `https://marcusdavenport.github.io/second-day-repo/` connected to the waitlist backend without changing the active UI design. The active design source of truth is `/Users/mineallmine/Downloads/vetdiy-presignup-ui.html`, not the current deployed `second-day-repo` page.

Success means the deployment repo serves the downloaded visual design, uses repo-relative assets instead of local `file://` URLs, keeps homeowner/contractor mode behavior and point display, and submits/confirm/ticks through the existing Supabase waitlist contract when public config is supplied. Until real hosted Supabase credentials are present, it must remain an honest preview and not falsely claim persistence.

## Current verified state

- Deployment repo: `/Users/mineallmine/second-day-repo`
- Branch: `main`
- HEAD: `c2a7aa6 fix(waitlist): use the polished -live design (was the early prototype) + wire it`
- Remote: `git@github.com:MarcusDavenport/second-day-repo.git`
- Dirty state before spec: clean.
- Live URL: `https://marcusdavenport.github.io/second-day-repo/`
- Live fetched hash: `/tmp/vetdiy-live.html` sha256 `1f461ae3dcc09e02820d556b06ac442a489bb9da0a354a3bb2a86074ee255ae5`.
- Live page currently has title `VetDIY Waitlist — Live Prototype`, bad white/sticky-nav design, and placeholder `CONFIG.supabaseUrl` / `CONFIG.anonKey`.
- Correct design source: `/Users/mineallmine/Downloads/vetdiy-presignup-ui.html`, sha256 `05cefa1bf4459824a739f016983da149a03945122d8fedabbcd9c13e1f9b323b`, 505 lines.
- Correct design assets:
  - Contractor image `/Users/mineallmine/Downloads/ChatGPT Image Jun 26, 2026, 07_37_06 PM (1).png`, 1254x1254, sha256 `bd12d37a42becc2dff3eb4bed4222ba18f63f927bcf57714cacd7d04257ba731`.
  - Homeowner image `/Users/mineallmine/Downloads/ChatGPT Image Jun 26, 2026, 07_37_07 PM (2).png`, 1254x1254, sha256 `627ccd2029694cb55f4093531c2ca376f76be64e76d2a2aa7ee31c52854574d4`.
  - Logo `/Users/mineallmine/Downloads/vetdiy-logo-mark-clean.png`, 240x240, sha256 `97d3285f364e5b365a8f0cd8070bae0110198b1635052bc26e3ad79276ca1c2e`, already matches `/Users/mineallmine/second-day-repo/assets/logo.png`.
- Backend source: `/Users/mineallmine/vetdiy-waitlist`, branch `feat/waitlist-backend`, dirty only from untracked `mobile/node_modules`.
- Backend contract:
  - `POST /functions/v1/waitlist-submit`
  - `GET/POST /functions/v1/waitlist-confirm?token=...`
  - `POST /rest/v1/rpc/waitlist_tick`
  - Payload includes `email`, `role`, `phone`, `business_name`, `website`, `facebook`, `trade`, `zip`, `project_type`, `project_notes`, `sms_consent`, `ref`, `source`, `utm`, `hp`.
- Running local servers: no listener found on 8000, 8080, or 8090.

## Pre-build verification

The break point is reproduced: the live page and local `second-day-repo/index.html` serve the wrong visual design and contain placeholder Supabase config. The correct design exists only in the downloaded standalone HTML and uses local `file://` image URLs, so it cannot be deployed as-is.

Commands/evidence used:
- `curl -L --max-time 20 -s -D /tmp/vetdiy-live.headers https://marcusdavenport.github.io/second-day-repo/ -o /tmp/vetdiy-live.html`
- `rg -n "Get first access|Join the VetDIY|CONFIG|YOUR-PROJECT|Live Prototype|Private launch" /tmp/vetdiy-live.html`
- `shasum -a 256 /Users/mineallmine/Downloads/vetdiy-presignup-ui.html /tmp/vetdiy-live.html`
- `git -C /Users/mineallmine/second-day-repo status --short --branch`
- Backend source reads from `supabase/functions/waitlist-submit/index.ts`, `waitlist-confirm/index.ts`, `_shared/util.ts`, and `supabase/migrations/0025_waitlist.sql`.

## Proposed architecture and data flow

Use the downloaded HTML/CSS as the page shell. Change only deployability and behavior:

1. Move the correct background images into `second-day-repo/assets/`.
2. Point CSS variables to repo-relative image paths and keep the logo path repo-relative.
3. Replace the local demo submit behavior with the already verified static-page Supabase client behavior from the bad page, adapted to the correct design's DOM IDs.
4. Keep preview fallback when `CONFIG` is not supplied.
5. Preserve exact visible copy, layout classes, role toggles, progressive field reveal, points panel, share box, and success area styling.

Rejected alternatives:
- Keep the current bad page and restyle it: rejected because Marcus identified it as the wrong agent-built design.
- Require a build system: rejected because this repo is a static GitHub Pages deployment and a single-page patch is lower risk.
- Hard-code unknown hosted Supabase values: rejected because no hosted project URL or anon key is verified in the workspace, and fabricating config would create false-live behavior.

## Write set

- `/Users/mineallmine/second-day-repo/index.html`
- `/Users/mineallmine/second-day-repo/assets/bg-contractor.png`
- `/Users/mineallmine/second-day-repo/assets/bg-homeowner.png`
- `/Users/mineallmine/second-day-repo/docs/agent-specs/20260627-1556-vetdiy-waitlist-live-backend.md`
- `/Users/mineallmine/second-day-repo/output/playwright/vetdiy-desktop.png`
- `/Users/mineallmine/second-day-repo/output/playwright/vetdiy-mobile.png`

## Non-touch set

- Do not edit `/Users/mineallmine/All Code Projects/VetDIY` dirty backend-rebuild files.
- Do not edit `/Users/mineallmine/vetdiy-waitlist` backend files unless a backend contract defect is proven.
- Do not deploy Supabase schema/functions or set secrets without explicit account/credential authorization.
- Do not push to GitHub without an explicit push request.
- Do not change the active visual design beyond replacing `file://` paths with deployable asset paths and adding invisible integration-only fields/logic.

## Risk register

- Visual drift from adapting IDs/scripts. Detection: screenshot comparison of local static page against downloaded HTML at desktop and mobile widths.
- Backend mismatch. Detection: inspect payload against `waitlist-submit` expected fields and test preview fallback; live POST can only be verified after real config exists.
- False-live claim. Detection: page must show preview/demo status when `CONFIGURED` is false.
- GitHub Pages cache delay. Detection: after push, fetch live headers/hash and allow cache window.
- File URL asset failure. Detection: static local server screenshot must show backgrounds and logo from repo assets.

## Proof plan

- Static code checks:
  - `rg -n "file://|YOUR-PROJECT|waitlist-submit|waitlist-confirm|waitlist_tick|Get first access|Private launch" /Users/mineallmine/second-day-repo/index.html`
  - `git -C /Users/mineallmine/second-day-repo diff --check`
- Local runtime checks:
  - Start a local static server from `/Users/mineallmine/second-day-repo`.
  - Load page in a browser.
  - Verify homeowner default design, contractor toggle, email reveal, points, share link, preview submit, and confirm-token preview path.
  - Capture screenshots at desktop and mobile.
- Backend proof:
  - Without hosted Supabase config, verify no persistence claim.
  - With real hosted config later, submit a test signup and confirm a token against Supabase.

## Completion verification

Before saying the implementation is complete for this pass:
- The local `index.html` must render the downloaded visual design from repo assets.
- The page must no longer contain local `file://` URLs.
- The JS must call the backend endpoints when configured and fall back honestly when not configured.
- Local browser interaction checks must pass.
- Git diff must be limited to the write set.

## Red-team questions

- Is the downloaded HTML truly the active source of truth, or is there another even newer variant?
- Does the hosted Supabase project already exist under credentials not present locally?
- Will GitHub Pages serve PNG assets at acceptable load speed, or should they be optimized after functional proof?
- Does replacing the bad page remove any field the backend expects? It should not; optional fields are preserved or mapped.
- Does the confirmation email link use the correct deployed `WAITLIST_SITE_URL` once functions are hosted?

## Mobile follow-up: duplicate hero image

On 2026-06-27 Marcus reported that mobile/tablet widths show a bad "double look" where the same role image appears as both the page background and a translucent foreground panel. Evidence screenshot: `/Users/mineallmine/Desktop/Screenshot 2026-06-27 at 6.35.53 PM.png` at 822x870.

Mobile success criteria:
- At widths under the existing `900px` breakpoint, render one coherent hero image treatment, not a second `.bg::after` photo panel.
- Keep desktop `>900px` hero composition unchanged.
- Preserve the active design copy, role toggles, waitlist form, preview/backend wiring, and assets.
- Verify at 822x870 and phone width with no horizontal overflow, clean console, and no page errors.
