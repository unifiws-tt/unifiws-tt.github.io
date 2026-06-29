# OmniReply — Desktop Handoff (s25) · Part 3: wire + deploy `v8.jsx` tenant UI

> Resume target for a Claude Code / desktop session. In-chat can't carry the 182KB `v8.jsx`
> (~31% token budget per pass via Drive base64). Everything below is verified live as of 2026-06-29.

## 1. Where we are (proven, not narrative)
- Phase 2 / state **s25**. Backend, infra, RLS (read+write+**grants**), signup, god-mode, Stripe — all green.
- **Signup chicken/egg SOLVED** (s24): edge fn `omr-signup` (verify_jwt=true) → calls SECURITY DEFINER RPC
  `omr_create_org_with_owner(p_user_id, p_name)` → atomic org + owner member, dedupe 1-org/user.
- **GRANT layer fixed** (s25): authenticated previously had 0 privileges on all 21 `omr_` tables
  (s23 wrote RLS policies but no GRANTs → everything denied). Now granted per policy intent.
- **e2e PROVEN with real JWT**: login → REST insert `omr_contacts` → 201; RLS WITH CHECK + grants pass.
- Thin zero-dep harness validates the whole path: https://unifiws-tt.github.io/omr-signup-harness.html
- Live DB pristine: orgs/members/contacts = 0/0/0 (test footprint purged).

## 2. The Part 3 task
Wire `v8.jsx` (Drive ID `1C0WKyknxxbNzaZBZ-g386BgvcY-JUIct`, 182KB, OmniReply folder) to the live
backend and deploy to GitHub Pages repo `unifiws-tt/unifiws-tt.github.io`.

### ⚠️ CRITICAL design constraint — patch v12 / rule Q
The tenant app is opened via shared links / in-app webviews. Rule Q: such screens MUST be
**zero-dependency** (no CDN, no React-via-CDN, no in-browser Babel/JSX transform) or they blank-screen
in webviews (observed on omnireply-admin). `v8.jsx` is JSX → you MUST either:
  (a) run a real build step (bundle to plain compiled JS, self-hosted — NOT in-browser Babel), or
  (b) guarantee it only ever loads in real Safari / installed PWA.
Default to (a). Do NOT ship React + CDN Babel for a shareable tenant URL.

## 3. Integration points the UI must use
- SUPABASE_URL: `https://jmtcdegcsrxjenxjnxse.supabase.co`
- Publishable (anon) key (safe in client): `sb_publishable_ufUjg7d8sbcyFCZUv1JvCg_0hAuhzm3`
- Onboarding (first org): POST `/functions/v1/omr-signup` with Bearer <user JWT>, body `{name}`
  → returns `{org_id, role:owner, plan:free}`; errors: `already_member`(409), `invalid_name`(400).
- Auth: standard Supabase Auth (email confirm currently ON — see gotchas).
- All tenant reads/writes go through PostgREST `/rest/v1/...` with Bearer JWT; RLS scopes rows by org.

## 4. Grant matrix (what authenticated CAN/CANNOT do) — don't fight it
- Full CRUD: omr_contacts, omr_conversations, omr_messages, omr_broadcasts, omr_templates,
  omr_auto_rules, omr_kb, omr_assets, omr_csat_surveys, omr_csat_responses, omr_marketing, omr_members*
  (*member writes RLS-gated to owner/admin where applicable)
- omr_orgs: SELECT + UPDATE only (INSERT/DELETE are service_role via edge fn)
- SELECT only: omr_analytics_daily, omr_analytics_events, omr_announcements, omr_subscriptions, omr_wa_accounts
- omr_wa_sessions: SELECT but **creds_encrypted column is NOT granted** (read status/qr/phone only)
- NO access (service_role only): omr_admin_audit, omr_wa_workers
- Helpers: `omr_is_member(org_id)` (SECURITY DEFINER), `omr_has_role(org_id, role)`

## 5. Gotchas (learned this session)
- **postgres-context masks gaps**: `execute_sql` runs as postgres (bypasses grants+RLS). ALWAYS verify
  real access with `has_table_privilege('authenticated', ...)` / a real JWT, never postgres smoke alone.
- **Email confirm ON** + no SMTP → signup returns "Error sending confirmation email". For testing either
  toggle confirm off (Auth settings) or seed a pre-confirmed user. NOTE when seeding auth.users manually:
  GoTrue 500 "Database error querying schema" if token-string cols are NULL → set
  confirmation_token/recovery_token/email_change*/phone_change*/reauthentication_token = ''.
- Edge fns deploy via Supabase MCP (`deploy_edge_function`); no MCP delete/set_secret (dashboard only).

## 6. Suggested Part 3 sequence
1. Pull `v8.jsx`, identify data layer (mock vs live), strip any in-browser Babel/CDN per rule Q.
2. Point it at SUPABASE_URL + publishable key; route onboarding through `omr-signup`.
3. Build/bundle → single self-contained deploy → push to `unifiws-tt.github.io` (e.g. `omnireply-app.html`).
4. e2e: signup → org → conversations/contacts CRUD under RLS; confirm no creds_encrypted leakage.
5. Checkpoint s26; update state/omnireply.

## 7. Key IDs
- Supabase project: `jmtcdegcsrxjenxjnxse` (ap-southeast-1)
- Pages repo: `unifiws-tt/unifiws-tt.github.io`
- v8.jsx Drive ID: `1C0WKyknxxbNzaZBZ-g386BgvcY-JUIct`
- OmniReply Drive folder: `1IMi_HiCw3w73t8i5AdSbu9i5NFqXT4eN`
- omr-signup edge fn id: `4d65f779-67d7-4976-9343-87a6cb4ee260`
