# OmniReply — Tenant-App Wiring Handoff (state s23)

> For Claude Code desktop. SHIPKIT canonical state = Supabase `shipkit_store` key `state/omnireply` (phase2/s23). Verify live before trusting any narrative.

## Objective
Get a WIRED tenant app live on GitHub Pages (`unifiws-tt.github.io`). The currently deployed `omnireply.html` (187KB) is an UNWIRED UI demo (sample data, no auth/fetch). Backend + RLS are DONE.

## Step 1 — DECIDE deploy-v8 vs wire-demo
v8.jsx = Drive `1C0WKyknxxbNzaZBZ-g386BgvcY-JUIct` (186KB, in OmniReply folder). Pull it, then grep:
- `@supabase/supabase-js` / `createClient`
- `supabase.auth`
- `.from('omr_`
If present => v8 is wired => **deploy-v8 path** (build → single HTML → push to repo). Else => wire the demo from scratch.
Tie-break: demo has the polished 12-screen design-locked UI; if v8 UI is weaker, port wiring into demo or restyle v8.
RECOMMENDED default: deploy-v8 (verify+deploy cheaper than rebuild).

## Step 2 — Wire auth + reads
- Client uses ANON/publishable key + user JWT only. NEVER service_role in browser.
- Supabase URL: `https://jmtcdegcsrxjenxjnxse.supabase.co`
- RLS already enforces tenant isolation. Just query e.g. `.from('omr_conversations').select()` — auto-scoped to caller's org via membership.

## Step 3 — Signup = service_role EDGE FN (NOT client)
Chicken/egg: the first member has no role yet, so client RLS can't create the org. Build an edge fn (service_role) that, on signup:
1. insert into `omr_orgs (name, plan, status)` → org_id
2. insert into `omr_members (org_id, user_id, role)` with role='owner'
Then client takes over with normal RLS-scoped calls.

## Step 4 — Announcement banner
Query `.from('omr_announcements').select()`. Policy already filters: `active=true AND starts_at<=now() AND (ends_at IS NULL OR ends_at>now())`. Just render whatever rows come back.

## Locked RLS map (applied s22+s23, verified)
- Helpers: `omr_is_member(org uuid)`, `omr_has_role(org uuid, roles text[])` — both SECURITY DEFINER, execute→authenticated.
- Member full CRUD: conversations, messages, contacts, templates, auto_rules, kb, csat_responses, csat_surveys, broadcasts, assets, marketing.
- omr_members: SELECT=member, writes=owner/admin. omr_orgs: SELECT=member, UPDATE=owner/admin.
- Read-only for client (writes via webhook/edge fn): subscriptions, wa_accounts, analytics_events.
- service_role only (deny-all to client): wa_sessions, admin_audit, wa_workers.

## Edge base
`https://jmtcdegcsrxjenxjnxse.supabase.co/functions/v1/` — omr-admin, omr-stripe-webhook, omr-wa-connect (bridge; do NOT remove), omr-wa-send, omr-wa-session, omr-auto-reply, omr-broadcast-runner, omr-analytics-rollup, omr-wa-webhook, omr-wa-worker-registry.

## Security TODO (next session)
- AUDIT `omr_wa_sessions` SELECT policy: table holds `creds_encrypted` — must NOT be readable by authenticated org members. Likely tighten to service_role only.
- Rotation debt: WA_WEBHOOK_SECRET, KEYVAULT_SERVICE_TOKEN.

## Deploy mechanics
GitHub Pages serves single HTML directly. `push_file` to `omnireply.html` (overwrite demo) or `omnireply-app.html` (new path). GitHub MCP write to unifiws-tt org confirmed working.
