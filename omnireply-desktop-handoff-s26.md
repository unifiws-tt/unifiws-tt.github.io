# OmniReply — Desktop Handoff Brief (s26)

> Supersedes omnireply-desktop-handoff-s23.md. Ground-truth as of 2026-06-29, verified live against Supabase (not state .md alone).
> **Job:** Deploy the full wired v8.jsx tenant UI to GitHub Pages. This is the OmniReply ship-blocker. Desktop/Claude Code only (182KB — unreliable through chat tools).

---

## 0. WHY DESKTOP
- v8.jsx = 182KB. Passing through chat (Drive MCP base64 / SQL) is ~31% token/pass and corruption-prone. Pull direct from Drive on desktop.
- Backend is DONE and PROVEN. This job is purely: get the real UI live + wired + smoke-tested.

## 1. SOURCE
- File: **v8.jsx** — canonical wired tenant UI (14 screens, matches locked design)
- Drive ID: `1C0WKyknxxbNzaZBZ-g386BgvcY-JUIct` (182KB)
- Pull direct from Drive. Do NOT route through chat.

## 2. TARGET
- Repo: `unifiws-tt/unifiws-tt.github.io` (GitHub Pages)
- **DECIDED (rekemen s26):** deploy wired build as **`omnireply-app.html`**.
  - `omnireply.html` stays = v8 14-screen FRONTEND DEMO (mock data); left intact.
  - Wire landing CTA → `omnireply-app.html` after deploy.
- Landing already live: `omnireply-landing.html`. Wire its CTA → the new app URL after deploy.

## 3. BACKEND STATE (live, verified s26)
- Supabase project: `jmtcdegcsrxjenxjnxse` (ap-southeast-1)
- URL: `https://jmtcdegcsrxjenxjnxse.supabase.co`
- Client publishable key: `sb_publishable_ufUjg7d8sbcyFCZUv1JvCg_0hAuhzm3`
- 21 `omr_` tables. RLS read(s22) + write(s23) + **GRANTS(s25)** COMPLETE & verified live.
- Helpers: `omr_is_member()` (SECURITY DEFINER), `omr_has_role()`.
- Signup chicken/egg SOLVED: `omr-signup` edge fn (v1) + `omr_create_org_with_owner` RPC (service_role, atomic org+owner, 1-org/user dedupe).
- **E2E PROVEN** (real user JWT): login → REST POST omr_contacts → 201; omr-signup → first org (role=owner, plan=free).

### GRANT matrix (authenticated role) — verified via has_table_privilege
- Full CRUD: omr_assets, omr_auto_rules, omr_broadcasts, omr_contacts, omr_conversations, omr_csat_responses, omr_csat_surveys, omr_kb, omr_marketing, omr_messages, omr_templates, omr_members
- SELECT + UPDATE only: omr_orgs (insert/delete = service_role via edge fn)
- SELECT only: omr_analytics_daily, omr_analytics_events, omr_announcements, omr_subscriptions, omr_wa_accounts
- omr_wa_sessions: COLUMN-SCOPED select EXCLUDING `creds_encrypted` (verified: status=true, creds_encrypted=FALSE)
- omr_admin_audit, omr_wa_workers: NO authenticated grant (service_role only)

### Edge fns (11 omr, all ACTIVE)
omr-wa-webhook(5), omr-wa-send(5), omr-wa-session(5), omr-wa-connect(2), omr-wa-worker-registry(5), omr-auto-reply(4), omr-broadcast-runner(2), omr-analytics-rollup(2), omr-stripe-webhook(3), omr-admin(2), omr-signup(1).
- Stripe webhook `we_1Tn9xt` VERIFIED.

## 4. CRITICAL: HTML WRAPPER (3-bug chain — do NOT skip)
v8 deploy previously hit a 3-bug chain. The standalone HTML template MUST use the v9 pattern:
1. **Classic React runtime** — NOT automatic jsx-runtime import. Babel preset classic.
2. **Run on window load** — wrap JSX transform+exec in `window.addEventListener("load", run)`. Never run inline before Babel CDN finishes (caused "Can't find variable: Babel").
3. **Multi-CDN loader** — load each dep (react, react-dom, babel) with fallback chain jsdelivr → unpkg → cdnjs. If all fail, show on-page message: "open in Safari (… → Open in Safari)".
4. **Visible errors** — catch + render errors on page (not blank). In-app webviews (Claude app, X) block unpkg → Safari hint required.
- Simplest user fix if blank: open in real Safari, not in-app webview.

## 5. WIRING CHECKLIST (in v8.jsx → HTML)
- Supabase client: URL + publishable key above.
- Auth: email/password signup → call `omr-signup` (NOT direct insert to omr_orgs — RLS will correctly reject client org insert).
- After signup: session JWT → REST/RPC for tenant data (CRUD allowed per GRANT matrix).
- Respect column-scoping: never request `creds_encrypted` from omr_wa_sessions (will 403).

## 6. SMOKE TEST (after deploy, in real Safari)
1. Open app URL in Safari (not in-app webview).
2. Signup new email → expect first org created, role=owner, plan=free.
3. Inbox screen loads (member SELECT on tenant tables).
4. Create a contact → expect 201 (RLS WITH CHECK + grants pass).
5. Confirm no creds/admin_audit leakage in network tab.
6. After test: purge test footprint (order: omr_contacts(org) → omr_members(org) → omr_orgs(org) → auth.identities(user) → auth.users(user)). Live should return to 0/0/0.

## 7. PITFALLS / LEARNINGS
- State .md files lag reality — verify live infra (list_edge_functions, has_table_privilege as target role) before trusting.
- RLS policies are inert without GRANTs (the s25 lesson). Smoke as postgres masks this; test as authenticated.
- 182KB file: pull from Drive direct; don't base64 through chat.
- One active build chat per project; save state after every deploy or risk multi-head Drive conflict.

## 8. AFTER DEPLOY → SAVE STATE
- Bump `state/omnireply` to s27 (or next), append history line in `shipkit_store` (Supabase canonical).
- Update URL REGISTRY with the live app URL.

---
### URL REGISTRY (current)
- harness: https://unifiws-tt.github.io/omr-signup-harness.html
- signup fn: https://jmtcdegcsrxjenxjnxse.supabase.co/functions/v1/omr-signup
- god-mode: https://unifiws-tt.github.io/omnireply-admin.html
- app (demo, mock): https://unifiws-tt.github.io/omnireply.html
- landing: https://unifiws-tt.github.io/omnireply-landing.html
- keyvault UI (static, canonical): https://unifiws-tt.github.io/  (keyvault-ui edge fn deleted s26)

### REMAINING OMNIREPLY ITEMS (post-deploy)
- Secret rotation (own Opus session): WA_WEBHOOK_SECRET, KEYVAULT_SERVICE_TOKEN — coordinate consumers.
- Viraly Stripe reconciliation (separate session).
