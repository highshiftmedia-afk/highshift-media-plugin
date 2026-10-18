# HSM MCP server — OAuth 2.1 implementation guide

**Audience:** the developer who maintains the HSM MCP server at `https://highshiftmedia.com/mcp`.
**Goal:** let any user of the public "Highshift Media" plugin (Cursor Marketplace, Grok Bot, Claude,
VS Code, MCP Inspector, …) click **Connect**, sign in to HSM in the browser, approve access, and get
a per-user, scoped, revocable token — no manual `hsm_mcp_…` copy/paste.

This guide is framework-agnostic. It follows the MCP Authorization spec
(revision **2025-11-25**, https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization),
which builds on OAuth 2.1 (draft-ietf-oauth-v2-1-13), RFC 9728 (Protected Resource Metadata),
RFC 8414 (Authorization Server Metadata), RFC 7591 (Dynamic Client Registration), RFC 8707
(Resource Indicators), RFC 7636 (PKCE), RFC 7009 (Revocation) and RFC 9207 (`iss` in responses).

---

## 0. Current state (probed 2026-10-18) and what must change

| Check | Today | Required |
| --- | --- | --- |
| `POST /mcp` without token | `401`, JSON-RPC `-32001`, **no `WWW-Authenticate` header** | `401` **with** `WWW-Authenticate: Bearer resource_metadata="…"` |
| `GET /.well-known/oauth-protected-resource` | `200 text/html` (the website's SPA fallback page) | `200 application/json` metadata (§2) |
| `GET /.well-known/oauth-protected-resource/mcp` | `200 text/html` (SPA fallback) | `200 application/json` metadata (§2) |
| `GET /.well-known/oauth-authorization-server` | `200 text/html` (SPA fallback) | `200 application/json` metadata (§3) |
| `GET /.well-known/openid-configuration` | `200 text/html` (SPA fallback) | `404` (or valid OIDC JSON if you add OIDC) |

> ⚠️ The SPA catch-all is the most important thing to fix. MCP clients probe these URLs and expect
> JSON; an HTML `200` makes discovery fail with a JSON parse error. Make sure the backend routes
> for `/.well-known/*`, `/oauth/*` and `/mcp` are matched **before** the static-site/SPA fallback,
> and that Cloudflare does not cache them, challenge them (Bot Fight / "Under attack" / Turnstile),
> or rewrite them. Unknown `/.well-known/*` paths must return `404`, not the SPA.

---

## 1. Identifiers (use these exact strings everywhere)

| Name | Value |
| --- | --- |
| Resource (canonical MCP server URI, RFC 8707) | `https://highshiftmedia.com/mcp` |
| Authorization server issuer | `https://highshiftmedia.com` (no trailing slash, no path) |
| Protected resource metadata URL | `https://highshiftmedia.com/.well-known/oauth-protected-resource/mcp` |
| AS metadata URL | `https://highshiftmedia.com/.well-known/oauth-authorization-server` |
| Authorization endpoint | `https://highshiftmedia.com/oauth/authorize` |
| Token endpoint | `https://highshiftmedia.com/oauth/token` |
| Registration endpoint (DCR) | `https://highshiftmedia.com/oauth/register` |
| Revocation endpoint | `https://highshiftmedia.com/oauth/revoke` |

The endpoints live under `/oauth/…` on purpose: plain `/register` or `/authorize` very likely collide
with existing website routes (e.g. the HSM sign-up page). Any path works as long as the AS metadata
points to it.

---

## 2. Protected Resource Metadata (RFC 9728) — REQUIRED

Serve the **same** JSON at both URLs (clients try the path-specific one first, then the root):

- `GET /.well-known/oauth-protected-resource/mcp`
- `GET /.well-known/oauth-protected-resource`

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: public, max-age=3600
Access-Control-Allow-Origin: *
```

```json
{
  "resource": "https://highshiftmedia.com/mcp",
  "authorization_servers": ["https://highshiftmedia.com"],
  "scopes_supported": [
    "account.read",
    "billing.read",
    "crm.read",
    "crm.write",
    "email.read",
    "social.publish"
  ],
  "bearer_methods_supported": ["header"],
  "resource_name": "Highshift Media",
  "resource_documentation": "https://highshiftmedia.com",
  "resource_logo_uri": "https://highshiftmedia.com/apple-touch-icon.png"
}
```

`resource` MUST exactly equal the MCP URL clients connect to. Do **not** list `offline_access` here
(per spec, refresh is not a resource requirement).

---

## 3. Authorization Server Metadata (RFC 8414) — REQUIRED

`GET /.well-known/oauth-authorization-server` → `200 application/json`, CORS `*`:

```json
{
  "issuer": "https://highshiftmedia.com",
  "authorization_endpoint": "https://highshiftmedia.com/oauth/authorize",
  "token_endpoint": "https://highshiftmedia.com/oauth/token",
  "registration_endpoint": "https://highshiftmedia.com/oauth/register",
  "revocation_endpoint": "https://highshiftmedia.com/oauth/revoke",
  "scopes_supported": [
    "account.read",
    "billing.read",
    "crm.read",
    "crm.write",
    "email.read",
    "social.publish",
    "offline_access"
  ],
  "response_types_supported": ["code"],
  "response_modes_supported": ["query"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "token_endpoint_auth_methods_supported": ["none", "client_secret_post", "client_secret_basic"],
  "revocation_endpoint_auth_methods_supported": ["none", "client_secret_post", "client_secret_basic"],
  "code_challenge_methods_supported": ["S256"],
  "authorization_response_iss_parameter_supported": true,
  "client_id_metadata_document_supported": true,
  "service_documentation": "https://highshiftmedia.com"
}
```

Notes:
- `code_challenge_methods_supported` is **mandatory in practice**: MCP clients MUST refuse to proceed
  if it is absent.
- `issuer` must be byte-identical to the issuer string in §1 and to the `iss` you return in §6.
- Only advertise `client_id_metadata_document_supported: true` if you implement §5.2. If you start
  with DCR only, remove that line.

---

## 4. The 401 challenge on `/mcp` — REQUIRED

Every request to `/mcp` without a valid token (missing, malformed, expired, revoked, wrong audience):

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json
WWW-Authenticate: Bearer resource_metadata="https://highshiftmedia.com/.well-known/oauth-protected-resource/mcp", scope="account.read crm.read email.read billing.read"
Access-Control-Expose-Headers: WWW-Authenticate

{"jsonrpc":"2.0","id":null,"error":{"code":-32001,"message":"unauthorized: a valid MCP token is required"}}
```

- For an invalid/expired token add `error="invalid_token"` to the header (RFC 6750 §3.1).
- Keep the JSON body you have today — it is fine; the header is what is missing.
- The `scope` hint is what clients request on first connect. Suggested default: the read scopes.
  Write scopes are then obtained by step-up (§9). If you prefer a single consent screen with
  everything, put all six scopes in the hint.

---

## 5. Client registration

Cursor's docs say it uses **Dynamic Client Registration** by default (with optional static
`CLIENT_ID` in `mcp.json`). Implement DCR first; CIMD is recommended by the 2025-11-25 spec and is
cheap to add.

### 5.1 Dynamic Client Registration (RFC 7591) — implement this

`POST /oauth/register`, `Content-Type: application/json`, **no auth required** (open registration).

Request (example from an MCP client):
```json
{
  "client_name": "Cursor",
  "redirect_uris": ["http://localhost:8787/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "account.read crm.read"
}
```

Response `201 Created`:
```json
{
  "client_id": "hsmc_3f9a1c2e7b8d4e60",
  "client_id_issued_at": 1792339200,
  "client_name": "Cursor",
  "redirect_uris": ["http://localhost:8787/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```
If the client asked for `client_secret_post`/`client_secret_basic`, also generate and return
`client_secret` (store only its hash) and `"client_secret_expires_at": 0`.

Validation rules:
- `redirect_uris` required, non-empty. Each must be either `https://…` or a loopback URI
  (`http://localhost[:port]/…`, `http://127.0.0.1[:port]/…`, `http://[::1][:port]/…`).
  Reject `http://` non-loopback, fragments (`#`), and wildcards. Custom schemes (e.g.
  `cursor://`, `vscode://`) MAY be allowed for native apps.
- Default `token_endpoint_auth_method` to `none` if omitted? RFC 7591 default is
  `client_secret_basic`; accept both, and issue a secret only when one was requested/defaulted.
- Errors: `400 {"error":"invalid_redirect_uri" | "invalid_client_metadata", "error_description":"…"}`.
- Rate-limit by IP (e.g. 20/hour) and garbage-collect clients that never completed an authorization
  within 30 days.

### 5.2 Client ID Metadata Documents (optional, recommended)

If `client_id` passed to `/oauth/authorize` is an `https://` URL with a path:
1. Fetch it server-side (timeout 5 s, max 64 KB, **block private/loopback IP ranges — SSRF**, no
   redirects to non-https).
2. Require valid JSON with `client_id` **exactly equal** to the URL, plus `client_name` and
   `redirect_uris`.
3. Require the request's `redirect_uri` to be in that list. Cache per HTTP cache headers (cap 24 h).
4. On the consent screen show the client_id **hostname** prominently, and warn if the only redirect
   URIs are localhost.

### 5.3 Known redirect URIs

| Client | Redirect URI | Source |
| --- | --- | --- |
| Cursor — web & Cursor Agents | `https://www.cursor.com/agents/mcp/oauth/callback` | cursor.com/docs/context/mcp ("Static redirect URL") |
| Cursor — desktop app | `http://localhost:8787/callback` | same |
| MCP Inspector (local) | `http://localhost:6274/oauth/callback` (port may vary) | Inspector default; verify |
| Others (Claude, VS Code, Grok Bot app…) | registered dynamically | not verified here |

With DCR you don't hard-code these — each client registers its own. They matter if you also create
**pre-registered** static clients (§5.4) or if you want an allowlist.

### 5.4 Pre-registered client (optional fallback)

Create one static public client, e.g. `client_id = "cursor-hsm"`, with exactly the two Cursor
redirect URIs above. Users or the plugin could then use Cursor's static OAuth (`"auth": {"CLIENT_ID": "cursor-hsm"}`
in `mcp.json`). Not needed if DCR works; keep the published plugin's `mcp.json` URL-only.

---

## 6. `GET /oauth/authorize` — HSM login + consent

Query parameters (all from the client):

| Param | Rule |
| --- | --- |
| `response_type` | must be `code` |
| `client_id` | must exist (DCR) or be a valid CIMD URL |
| `redirect_uri` | must **exactly** match one registered for this client (string compare) |
| `code_challenge` | required, 43–128 chars base64url |
| `code_challenge_method` | required, must be `S256` (reject `plain` or missing) |
| `state` | optional but echo it back unchanged if present |
| `scope` | space-separated; drop unknown scopes; if empty use the default read set |
| `resource` | should be `https://highshiftmedia.com/mcp`; if present and different → `invalid_target`. Compare after lower-casing scheme+host and stripping one trailing `/`. |

Flow:
1. **Validate `client_id` and `redirect_uri` first.** If either is invalid, show an HSM error page
   and do **not** redirect (open-redirect protection).
2. Other parameter errors → redirect to `redirect_uri?error=invalid_request&error_description=…&state=…&iss=https%3A%2F%2Fhighshiftmedia.com`.
3. If the user isn't logged in to HSM → send them to the normal HSM login (with 2FA if you have it),
   then return to this same authorize URL. Persist the request server-side (e.g. signed
   `auth_request_id`), not in a long query string.
4. Show a **consent screen** (`X-Frame-Options: DENY` / `frame-ancestors 'none'` — already set):
   - "**{client_name}** wants to access your Highshift Media account **{company}** as **{email}**"
   - Redirect host, e.g. "You'll be sent back to **www.cursor.com**" / "**localhost**" (warn for localhost)
   - Checklist of requested scopes in plain language (see §8), with destructive/public ones flagged
   - Buttons **Allow** / **Deny**, protected by a CSRF token (POST back to `/oauth/authorize/decision`)
   - Optional: remember consent per (user, client_id, scopes) and skip the screen next time.
5. **Deny** → redirect `redirect_uri?error=access_denied&state=…&iss=…`
6. **Allow** → create an authorization code:
   - 32+ random bytes, base64url; store **hash**; single use; TTL **60 s–10 min** (use 5 min)
   - bind: `client_id`, `redirect_uri`, `code_challenge`, `user_id`, `company_id`, granted `scope`,
     `resource`
   - redirect `302` to `redirect_uri?code=…&state=…&iss=https%3A%2F%2Fhighshiftmedia.com`

`iss` (RFC 9207) must be exactly the issuer from §3.

---

## 7. `POST /oauth/token`

`Content-Type: application/x-www-form-urlencoded`. CORS: allow `*` (browser-based clients like
MCP Inspector call it directly). Response always `Cache-Control: no-store`.

Client authentication: public clients (`none`) send `client_id` in the body; confidential clients use
`client_secret_post` or HTTP Basic. Enforce the method the client registered with.

### 7.1 `grant_type=authorization_code`

Body: `code`, `redirect_uri`, `client_id`, `code_verifier`, `resource` (should be present).

Checks, in order — any failure → `400 {"error":"invalid_grant"}` (unless noted):
1. Code exists, not expired, not used. **If reused**: reject and revoke every token issued from it.
2. `client_id` matches the code; `redirect_uri` equals the one on the code.
3. PKCE: `BASE64URL(SHA256(ASCII(code_verifier))) == code_challenge` (no padding, constant-time compare).
   Verifier must be 43–128 chars of `[A-Za-z0-9-._~]`.
4. `resource` (if sent) equals the code's resource → else `invalid_target`.
5. Mark code used (atomically).

Success `200`:
```json
{
  "access_token": "hsm_oat_Yp3…",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "hsm_ort_Q8k…",
  "scope": "account.read crm.read email.read"
}
```

### 7.2 `grant_type=refresh_token`

Body: `refresh_token`, `client_id` (+ secret if confidential), optional `scope` (must be ⊆ original),
optional `resource`.

- **Rotate** on every use (MUST for public clients): issue a new refresh token, invalidate the old one.
- **Reuse detection:** if an already-rotated refresh token is presented, revoke the whole token family
  (all access + refresh tokens from that grant) and return `invalid_grant`.
- Suggested lifetimes: access token **1 h**; refresh token **30 days sliding**, **90 days absolute**.
- Re-check that the user still exists, is active, still belongs to the company, and that the
  company subscription allows API/agent access.

### 7.3 Error format
```json
{ "error": "invalid_grant", "error_description": "authorization code expired" }
```
Codes: `invalid_request`, `invalid_client` (401), `invalid_grant`, `unauthorized_client`,
`unsupported_grant_type`, `invalid_scope`, `invalid_target`.

---

## 8. Scopes

| Scope | Tools | Consent-screen wording |
| --- | --- | --- |
| `account.read` | `whoami`, `services_list` | See your name, email, company and HSM services |
| `billing.read` | `billing_status` | See your subscription, plan and trial status |
| `crm.read` | `crm_list_contacts`, `crm_get_contact`, `crm_list_conversations` | Read your CRM contacts and conversations |
| `crm.write` | `crm_create_contact`, `crm_delete_contact` | Create and **delete** CRM contacts |
| `email.read` | `email_list_accounts`, `email_list_messages`, `email_read_message` | Read emails in inboxes connected to HSM |
| `social.publish` | `reel_publish`, `social_publish_status` | **Publish** Reels to your connected Facebook Pages |
| `offline_access` | — (AS only) | Stay connected (refresh tokens) |

Rules:
- `crm.write` implies `crm.read` (scope hierarchy — the spec requires servers to honor it).
- Effective permission = granted scopes **∩** the user's own HSM role permissions. A scope never
  grants more than the user could do in the dashboard.
- Optionally keep `tools/list` complete but enforce on `tools/call` (simpler for clients), or filter
  `tools/list` by scope — pick one and be consistent.

---

## 9. Validating tokens on `/mcp` (resource server)

For **every** HTTP request to `/mcp` (POST, GET for SSE stream, DELETE for session end):

1. Read `Authorization: Bearer <token>`. Reject tokens in the query string.
2. Dispatch by prefix:
   - `hsm_oat_…` → OAuth access token (below)
   - `hsm_mcp_…` → legacy personal token (§10)
   - anything else → 401 (§4) with `error="invalid_token"`
3. OAuth access token checks:
   - Look up `SHA-256(token)` in `oauth_access_tokens` (opaque tokens, recommended) — or verify a JWT
     signature if you choose JWTs.
   - Not expired, not revoked, user active, company active.
   - **Audience:** token's `resource`/`aud` == `https://highshiftmedia.com/mcp`. Reject otherwise
     (RFC 8707 / MCP "token audience binding"). Never accept tokens from other issuers (e.g. Facebook,
     Google) and never pass the client's token through to upstream APIs.
4. Attach `{user_id, company_id, scopes, client_id, token_id}` to the request; audit-log every
   tool call with `client_id` and `token_id` (you already audit-log).
5. On `tools/call`, check the tool's required scope (§8). If missing:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope", scope="account.read crm.read crm.write", resource_metadata="https://highshiftmedia.com/.well-known/oauth-protected-resource/mcp", error_description="crm.write is required to create contacts"
```
   Include previously granted scopes plus the new one in `scope` so clients don't lose access on
   re-consent.

If you prefer JWT access tokens: sign with RS256/ES256, publish `jwks_uri` in §3, claims
`iss=https://highshiftmedia.com`, `aud=https://highshiftmedia.com/mcp`, `sub=<user id>`,
`scope`, `client_id`, `exp`, `iat`, `jti`; check revocation via a `jti` denylist. Opaque tokens are
simpler here because the AS and RS are the same service.

---

## 10. Keep existing `hsm_mcp_…` tokens working

- Continue to accept `Authorization: Bearer hsm_mcp_…` exactly as today (same lookup, same
  "Settings → Connect an AI agent" UI, same revoke button).
- Treat a legacy token as having **all scopes** (or the scopes chosen when it was created, if you add
  that option).
- Legacy tokens must still get the new 401 `WWW-Authenticate` header when invalid.
- In "Settings → Connect an AI agent", also list **OAuth connections** (client name, scopes, created,
  last used) with a **Revoke** button that revokes the whole grant (all access + refresh tokens).

---

## 11. `POST /oauth/revoke` (RFC 7009)

Form body: `token`, optional `token_type_hint` (`access_token` | `refresh_token`), `client_id`
(+ secret if confidential).

- Always respond `200` with empty body (even for unknown tokens).
- Only revoke if the token belongs to the authenticated `client_id`.
- Revoking a refresh token revokes its whole family; revoking an access token revokes only it
  (or the family — your choice, document it).

---

## 12. Data model (minimal)

```text
oauth_clients        (client_id PK, client_secret_hash NULL, client_name, redirect_uris JSON,
                      grant_types JSON, token_endpoint_auth_method, is_cimd BOOL, created_at, last_used_at)
oauth_auth_codes     (code_hash PK, client_id, user_id, company_id, redirect_uri, code_challenge,
                      scope, resource, expires_at, used_at NULL)
oauth_grants         (grant_id PK, user_id, company_id, client_id, scope, created_at, revoked_at NULL)
oauth_access_tokens  (token_hash PK, grant_id, scope, resource, expires_at, revoked_at NULL, last_used_at)
oauth_refresh_tokens (token_hash PK, grant_id, parent_hash NULL, expires_at, rotated_at NULL, revoked_at NULL)
oauth_consents       (user_id, client_id, scope, created_at)      -- optional "remember consent"
```
Store **hashes** (SHA-256) of codes and tokens, never the raw values. Generate secrets with a CSPRNG
(≥ 32 bytes).

---

## 13. Security checklist

- [ ] HTTPS only for every endpoint (already behind Cloudflare).
- [ ] PKCE S256 required for every client, public or confidential; `plain` rejected.
- [ ] Exact `redirect_uri` match; error page (no redirect) for unknown client/redirect.
- [ ] `state` echoed back; `iss` returned on every authorization response (success and error).
- [ ] Codes single-use, ≤ 10 min; code replay revokes derived tokens.
- [ ] Refresh-token rotation + reuse detection.
- [ ] Audience (`resource`) bound into every token and checked on `/mcp`.
- [ ] Consent screen shows client name, redirect host and scopes; CSRF-protected; not frameable.
- [ ] Rate limits on `/oauth/register`, `/oauth/token`, login.
- [ ] SSRF guard on CIMD fetches.
- [ ] Never log raw tokens, codes, verifiers or secrets.
- [ ] Cloudflare: no cache and no bot challenge on `/.well-known/*`, `/oauth/token`,
      `/oauth/register`, `/oauth/revoke`, `/mcp` (machine clients can't solve challenges).
- [ ] CORS: `Access-Control-Allow-Origin: *` on metadata, `/oauth/token`, `/oauth/register`,
      `/oauth/revoke`; expose `WWW-Authenticate` on `/mcp`; handle `OPTIONS` preflight.
      (Do **not** add CORS to `/oauth/authorize` — it's a browser navigation.)

---

## 14. Test checklist

### 14.1 curl

```bash
BASE=https://highshiftmedia.com

# 1) 401 + header
curl -si -X POST $BASE/mcp -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}' \
  | grep -i -E '^HTTP|^www-authenticate'
# expect: HTTP/2 401  and  www-authenticate: Bearer resource_metadata="https://highshiftmedia.com/.well-known/oauth-protected-resource/mcp", ...

# 2) Metadata is JSON (not the website HTML)
curl -s $BASE/.well-known/oauth-protected-resource/mcp | jq .
curl -s $BASE/.well-known/oauth-protected-resource     | jq .resource   # "https://highshiftmedia.com/mcp"
curl -s $BASE/.well-known/oauth-authorization-server   | jq '.issuer, .code_challenge_methods_supported'
curl -s -o /dev/null -w '%{http_code}\n' $BASE/.well-known/does-not-exist   # expect 404

# 3) Dynamic client registration
curl -s -X POST $BASE/oauth/register -H 'Content-Type: application/json' -d '{
  "client_name":"curl test","redirect_uris":["http://localhost:8787/callback"],
  "grant_types":["authorization_code","refresh_token"],"response_types":["code"],
  "token_endpoint_auth_method":"none"}' | tee /tmp/client.json | jq .
CLIENT_ID=$(jq -r .client_id /tmp/client.json)

# 4) PKCE pair
VERIFIER=$(openssl rand -base64 48 | tr -d '=+/\n' | cut -c1-64)
CHALLENGE=$(printf %s "$VERIFIER" | openssl dgst -sha256 -binary | openssl base64 | tr '+/' '-_' | tr -d '=\n')

# 5) Open in a browser, log in, Allow; copy ?code=… from the (failing) localhost redirect
echo "$BASE/oauth/authorize?response_type=code&client_id=$CLIENT_ID&redirect_uri=http%3A%2F%2Flocalhost%3A8787%2Fcallback&code_challenge=$CHALLENGE&code_challenge_method=S256&scope=account.read%20crm.read&state=xyz&resource=https%3A%2F%2Fhighshiftmedia.com%2Fmcp"
# check the redirect contains code=…, state=xyz and iss=https%3A%2F%2Fhighshiftmedia.com

# 6) Exchange
CODE=paste_code_here
curl -s -X POST $BASE/oauth/token -d grant_type=authorization_code -d code=$CODE \
  -d redirect_uri=http://localhost:8787/callback -d client_id=$CLIENT_ID \
  -d code_verifier=$VERIFIER -d resource=https://highshiftmedia.com/mcp | tee /tmp/tok.json | jq .
AT=$(jq -r .access_token /tmp/tok.json); RT=$(jq -r .refresh_token /tmp/tok.json)

# 7) Use it
curl -s -X POST $BASE/mcp -H "Authorization: Bearer $AT" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}'

# 8) Refresh + rotation + reuse detection
curl -s -X POST $BASE/oauth/token -d grant_type=refresh_token -d refresh_token=$RT -d client_id=$CLIENT_ID | jq .
curl -s -X POST $BASE/oauth/token -d grant_type=refresh_token -d refresh_token=$RT -d client_id=$CLIENT_ID | jq .  # expect invalid_grant

# 9) Revoke
curl -s -o /dev/null -w '%{http_code}\n' -X POST $BASE/oauth/revoke -d token=$AT -d client_id=$CLIENT_ID  # 200
```

### 14.2 Negative tests (all must fail)
- [ ] `code_challenge_method=plain` or missing → error
- [ ] wrong `code_verifier` → `invalid_grant`
- [ ] same code twice → `invalid_grant` (and first tokens revoked)
- [ ] `redirect_uri` differing by one character → error page, no redirect
- [ ] `resource=https://evil.example/mcp` → `invalid_target`
- [ ] expired / revoked access token on `/mcp` → `401` + `WWW-Authenticate … error="invalid_token"`
- [ ] token with only `crm.read` calling `crm_create_contact` → `403 insufficient_scope`
- [ ] legacy `hsm_mcp_…` token still works on `/mcp`

### 14.3 MCP Inspector
```bash
npx @modelcontextprotocol/inspector
```
Transport **Streamable HTTP**, URL `https://highshiftmedia.com/mcp`, open **Auth / OAuth settings** and
run the guided "Quick OAuth flow". Every step (metadata discovery, DCR, authorize, token) should go
green, then **Connect** → **List tools** shows all 13 tools.

### 14.4 Real clients
- Cursor desktop: add `{"mcpServers":{"highshift-media":{"url":"https://highshiftmedia.com/mcp"}}}`
  to `~/.cursor/mcp.json` (no headers). Cursor should show **Needs login** → browser → HSM consent →
  connected. Check *Output → MCP Logs* on failure.
- Cursor web / Agents: same, verifies `https://www.cursor.com/agents/mcp/oauth/callback`.
- Load this plugin locally (see `docs/PUBLISHING.md`) and repeat.

---

## 15. Illustrative sketch (Node.js / Express) — not production code

```js
import express from "express";
import crypto from "node:crypto";

const ISSUER = "https://highshiftmedia.com";
const RESOURCE = "https://highshiftmedia.com/mcp";
const PRM_URL = `${ISSUER}/.well-known/oauth-protected-resource/mcp`;
const SCOPES = ["account.read","billing.read","crm.read","crm.write","email.read","social.publish"];
const sha256 = (s) => crypto.createHash("sha256").update(s).digest();
const b64url = (buf) => buf.toString("base64url");
const rand = (n = 32) => b64url(crypto.randomBytes(n));

const app = express();
app.use(express.urlencoded({ extended: false }));
app.use(express.json());

// ---- discovery (mount BEFORE the SPA fallback) ----
const prm = { resource: RESOURCE, authorization_servers: [ISSUER], scopes_supported: SCOPES,
              bearer_methods_supported: ["header"], resource_name: "Highshift Media" };
const cors = (_req, res, next) => { res.set("Access-Control-Allow-Origin", "*");
  res.set("Access-Control-Allow-Headers", "Authorization, Content-Type, Mcp-Session-Id, MCP-Protocol-Version");
  res.set("Access-Control-Expose-Headers", "WWW-Authenticate, Mcp-Session-Id"); next(); };
app.get(["/.well-known/oauth-protected-resource", "/.well-known/oauth-protected-resource/mcp"], cors,
  (_req, res) => res.json(prm));
app.get("/.well-known/oauth-authorization-server", cors, (_req, res) => res.json({
  issuer: ISSUER,
  authorization_endpoint: `${ISSUER}/oauth/authorize`,
  token_endpoint: `${ISSUER}/oauth/token`,
  registration_endpoint: `${ISSUER}/oauth/register`,
  revocation_endpoint: `${ISSUER}/oauth/revoke`,
  scopes_supported: [...SCOPES, "offline_access"],
  response_types_supported: ["code"],
  grant_types_supported: ["authorization_code", "refresh_token"],
  token_endpoint_auth_methods_supported: ["none", "client_secret_post", "client_secret_basic"],
  code_challenge_methods_supported: ["S256"],
  authorization_response_iss_parameter_supported: true,
}));
app.all("/.well-known/*", (_req, res) => res.status(404).end());

// ---- DCR ----
app.post("/oauth/register", cors, async (req, res) => {
  const { redirect_uris, client_name = "MCP client", token_endpoint_auth_method = "none" } = req.body || {};
  const ok = Array.isArray(redirect_uris) && redirect_uris.length && redirect_uris.every((u) => {
    try { const x = new URL(u); if (x.hash) return false;
      return x.protocol === "https:" ||
        (x.protocol === "http:" && ["localhost", "127.0.0.1", "[::1]"].includes(x.hostname));
    } catch { return false; } });
  if (!ok) return res.status(400).json({ error: "invalid_redirect_uri" });
  const client = { client_id: "hsmc_" + rand(12), client_name, redirect_uris,
    grant_types: ["authorization_code", "refresh_token"], response_types: ["code"],
    token_endpoint_auth_method, client_id_issued_at: Math.floor(Date.now() / 1000) };
  let secret;
  if (token_endpoint_auth_method !== "none") { secret = rand(); client.client_secret_expires_at = 0; }
  await db.clients.insert({ ...client, client_secret_hash: secret && sha256(secret) });
  res.status(201).json(secret ? { ...client, client_secret: secret } : client);
});

// ---- authorize: validate, require HSM session, render consent ----
app.get("/oauth/authorize", async (req, res) => {
  const q = req.query;
  const client = await db.clients.get(q.client_id);
  if (!client || !client.redirect_uris.includes(q.redirect_uri))
    return res.status(400).send("Unknown client or redirect URI");          // no redirect!
  const fail = (error) => { const u = new URL(q.redirect_uri);
    u.searchParams.set("error", error); if (q.state) u.searchParams.set("state", q.state);
    u.searchParams.set("iss", ISSUER); return res.redirect(u.toString()); };
  if (q.response_type !== "code") return fail("unsupported_response_type");
  if (!q.code_challenge || q.code_challenge_method !== "S256") return fail("invalid_request");
  if (q.resource && q.resource.replace(/\/$/, "").toLowerCase() !== RESOURCE) return fail("invalid_target");
  if (!req.session?.userId) return res.redirect(`/login?next=${encodeURIComponent(req.originalUrl)}`);
  const scope = (q.scope || "account.read crm.read email.read billing.read")
    .split(" ").filter((s) => SCOPES.includes(s) || s === "offline_access").join(" ");
  const reqId = rand(16);
  await db.authRequests.put(reqId, { ...q, scope, userId: req.session.userId }, 600);
  res.render("consent", { reqId, client, scope, csrf: req.csrfToken(),
    redirectHost: new URL(q.redirect_uri).host });
});

app.post("/oauth/authorize/decision", async (req, res) => {          // CSRF middleware assumed
  const r = await db.authRequests.take(req.body.reqId);
  if (!r || r.userId !== req.session?.userId) return res.status(400).send("Expired request");
  const u = new URL(r.redirect_uri);
  if (r.state) u.searchParams.set("state", r.state);
  u.searchParams.set("iss", ISSUER);
  if (req.body.decision !== "allow") { u.searchParams.set("error", "access_denied"); return res.redirect(u.toString()); }
  const code = rand();
  await db.codes.insert({ code_hash: sha256(code), client_id: r.client_id, user_id: r.userId,
    redirect_uri: r.redirect_uri, code_challenge: r.code_challenge, scope: r.scope,
    resource: RESOURCE, expires_at: Date.now() + 5 * 60_000 });
  u.searchParams.set("code", code);
  res.redirect(u.toString());
});

// ---- token ----
async function issueTokens(grantId, scope) {
  const at = "hsm_oat_" + rand(), rt = "hsm_ort_" + rand();
  await db.accessTokens.insert({ token_hash: sha256(at), grant_id: grantId, scope, resource: RESOURCE,
    expires_at: Date.now() + 3600_000 });
  await db.refreshTokens.insert({ token_hash: sha256(rt), grant_id: grantId,
    expires_at: Date.now() + 30 * 864e5 });
  return { access_token: at, token_type: "Bearer", expires_in: 3600, refresh_token: rt, scope };
}
app.post("/oauth/token", cors, async (req, res) => {
  res.set("Cache-Control", "no-store");
  const b = req.body; const err = (e, s = 400) => res.status(s).json({ error: e });
  const client = await authenticateClient(req);                       // none / post / basic
  if (!client) return err("invalid_client", 401);
  if (b.grant_type === "authorization_code") {
    const c = await db.codes.takeOnce(sha256(b.code || ""));          // atomic; on replay revoke grant
    if (!c || c.expires_at < Date.now() || c.client_id !== client.client_id ||
        c.redirect_uri !== b.redirect_uri) return err("invalid_grant");
    const expected = b64url(sha256(b.code_verifier || ""));
    if (!/^[A-Za-z0-9\-._~]{43,128}$/.test(b.code_verifier || "") || expected !== c.code_challenge)
      return err("invalid_grant");
    if (b.resource && b.resource.replace(/\/$/, "").toLowerCase() !== RESOURCE) return err("invalid_target");
    const grantId = await db.grants.create({ user_id: c.user_id, client_id: c.client_id, scope: c.scope });
    return res.json(await issueTokens(grantId, c.scope));
  }
  if (b.grant_type === "refresh_token") {
    const r = await db.refreshTokens.get(sha256(b.refresh_token || ""));
    if (!r) return err("invalid_grant");
    const g = await db.grants.get(r.grant_id);
    if (g.client_id !== client.client_id || g.revoked_at) return err("invalid_grant");
    if (r.rotated_at || r.revoked_at) { await db.grants.revoke(g.grant_id); return err("invalid_grant"); } // reuse
    if (r.expires_at < Date.now()) return err("invalid_grant");
    await db.refreshTokens.markRotated(r.token_hash);
    return res.json(await issueTokens(g.grant_id, g.scope));
  }
  return err("unsupported_grant_type");
});

// ---- resource server guard for /mcp ----
const TOOL_SCOPE = { whoami: "account.read", services_list: "account.read", billing_status: "billing.read",
  crm_list_contacts: "crm.read", crm_get_contact: "crm.read", crm_list_conversations: "crm.read",
  crm_create_contact: "crm.write", crm_delete_contact: "crm.write",
  email_list_accounts: "email.read", email_list_messages: "email.read", email_read_message: "email.read",
  reel_publish: "social.publish", social_publish_status: "social.publish" };
const has = (scopes, need) => scopes.includes(need) || (need === "crm.read" && scopes.includes("crm.write"));

app.use("/mcp", cors, async (req, res, next) => {
  if (req.method === "OPTIONS") return res.status(204).end();
  const challenge = (extra = "") => res.status(401)
    .set("WWW-Authenticate", `Bearer resource_metadata="${PRM_URL}", scope="account.read crm.read email.read billing.read"${extra}`)
    .json({ jsonrpc: "2.0", id: null, error: { code: -32001, message: "unauthorized: a valid MCP token is required" } });
  const m = /^Bearer (.+)$/.exec(req.get("authorization") || "");
  if (!m) return challenge();
  const tok = m[1];
  let ctx;
  if (tok.startsWith("hsm_mcp_")) ctx = await legacyTokenLookup(tok);  // existing code; all scopes
  else if (tok.startsWith("hsm_oat_")) {
    const t = await db.accessTokens.get(sha256(tok));
    if (t && !t.revoked_at && t.expires_at > Date.now() && t.resource === RESOURCE) {
      const g = await db.grants.get(t.grant_id);
      if (!g.revoked_at) ctx = { userId: g.user_id, scopes: t.scope.split(" "), clientId: g.client_id };
    }
  }
  if (!ctx) return challenge(`, error="invalid_token"`);
  const need = req.body?.method === "tools/call" && TOOL_SCOPE[req.body?.params?.name];
  if (need && !has(ctx.scopes, need)) {
    const want = [...new Set([...ctx.scopes, need])].join(" ");
    return res.status(403).set("WWW-Authenticate",
      `Bearer error="insufficient_scope", scope="${want}", resource_metadata="${PRM_URL}"`).end();
  }
  req.hsm = ctx; next();                                              // → existing MCP handler
});
```

(Python/FastAPI is structurally identical: same routes, same checks. Libraries that can do most of
this for you: `oidc-provider` / `@node-oauth/oauth2-server` (Node), `Authlib` (Python),
`league/oauth2-server` (PHP), or an external IdP such as Auth0, WorkOS, Stytch, Clerk, Descope or
Keycloak that supports DCR + PKCE + resource indicators — in that case point `authorization_servers`
in §2 at the IdP issuer and validate its JWTs on `/mcp`.)
