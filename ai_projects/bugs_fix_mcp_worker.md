# Bugs Found & Fixes Applied — MCP OAuth Worker

This document records every bug discovered during local testing of the
`leadgen-mcp-oauth` Cloudflare Worker and what was done to fix each one.
Ordered by discovery sequence.

---

## Overview of the Full OAuth Flow

Before diving into bugs, here is the correct end-to-end flow this Worker
is supposed to implement:

```
Cursor (MCP client)
   │
   │  1. POST http://localhost:8090/          ← hits MCP server
   │     ← 401 Unauthorized
   │        WWW-Authenticate: Bearer resource_metadata=".../.well-known"
   │
   │  2. GET http://localhost:8090/.well-known/oauth-authorization-server
   │     ← { registration_endpoint, authorization_endpoint, token_endpoint }
   │
   │  3. POST http://localhost:8787/register  ← Worker: store client in KV
   │     Body: { redirect_uris, grant_types }
   │     ← 201 { client_id, ... }
   │
   │  4. Opens browser → GET http://localhost:8787/authorize?client_id=...
   │                              │
   │              PRODUCTION      │  CF Access intercepts at network edge
   │              ─────────────   │  before Worker code runs → shows login page
   │                              │  User logs in → CF Access sets CF_Authorization
   │                              │  cookie → redirects back to /authorize
   │                              │  Worker code now runs WITH the cookie ✓
   │              LOCAL           │
   │              ─────────────   │  No CF Access in front → Worker code runs
   │                              │  immediately → no cookie → Worker manually
   │                              │  redirects to CF Access login page
   │
   │  5. POST http://localhost:8787/token     ← Cursor exchanges code for JWT
   │     ← { access_token: "<CF JWT>" }
   │
   │  6. POST http://localhost:8090/          ← all MCP tool calls from now on
   │     Authorization: Bearer <CF JWT>
   │     ← 200 (tools respond)
```

---

## Bug 1 — `oauthReqInfo` guard always returns 404

**File:** `cloudflare-worker/src/access-handler.ts`

**Discovered:** During local curl testing — `/authorize` always returned 404
even with a valid registered `client_id` and correct PKCE parameters.

### Root Cause

A guard was added at the top of `handleAccessRequest`:

```typescript
// ❌ BROKEN — oauthReqInfo is NEVER passed as a 4th argument
export async function handleAccessRequest(
  request: Request,
  env: Env,
  _ctx: ExecutionContext,
  oauthReqInfo: OAuthRequestInfo   // ← always undefined
): Promise<Response> {

  if (!oauthReqInfo) {
    return new Response("Not Found", { status: 404 });  // ← always hit
  }
  ...
}
```

The assumption was that `OAuthProvider` passes OAuth request data as a
4th argument to `defaultHandler.fetch`. **It does not.**

Reading the library source (`oauth-provider.js` line 160):

```javascript
// Library source — only 3 args are ever passed:
return this.typedDefaultHandler.handler.fetch(
  request,
  env,
  ctx       // ← no 4th argument, ever
);
```

The library injects a helper object into `env.OAUTH_PROVIDER` BEFORE
calling the handler. The OAuth params must be read from there, not from
a phantom 4th parameter.

### Fix

Removed the 4th parameter and the broken guard. Instead:

1. Check `url.pathname` to filter out noise (e.g. `/favicon.ico`) that
   the library also routes to `defaultHandler`.
2. Call `env.OAUTH_PROVIDER.parseAuthRequest(request)` to get the parsed
   OAuth params directly from the request URL.

```typescript
// ✅ FIXED
export async function handleAccessRequest(
  request: Request,
  env: Env,
  _ctx: ExecutionContext,           // no 4th parameter
): Promise<Response> {
  const url = new URL(request.url);

  // Filter non-authorize paths (favicon.ico, etc.)
  if (url.pathname !== "/authorize") {
    return new Response("Not Found", { status: 404 });
  }

  // Parse OAuth params via the library helper injected into env
  const oauthReq = await env.OAUTH_PROVIDER.parseAuthRequest(request);
  ...
}
```

---

## Bug 2 — `completeAuthorization is not defined` (production crash)

**File:** `cloudflare-worker/src/access-handler.ts`

**Discovered:** In Cloudflare production logs:

```json
{
  "error": "completeAuthorization is not defined",
  "trigger": "GET /authorize",
  "service": "leadgen-mcp-oauth"
}
```

### Root Cause

The original code used a TypeScript `declare function` at the bottom of
the file to satisfy the TypeScript compiler:

```typescript
// ❌ BROKEN — TypeScript-only declaration, emits zero JavaScript
declare function completeAuthorization(
  request: Request,
  env: Env,
  params: OAuthRequestInfo & { userId: string; metadata: { cfJwt: string } }
): Promise<Response>;
```

`declare function` tells TypeScript "trust me, this exists at runtime"
but emits **nothing** to the compiled JavaScript. At runtime, calling
`completeAuthorization(...)` throws `ReferenceError: completeAuthorization is not defined`.

**Why it was never caught locally:** In local testing (and before the
user logged in), the code always hit the `if (!cfJwt) { return redirect }` 
branch first — `completeAuthorization` was never reached. The crash only
happens after a user successfully logs in (CF Access sets the cookie)
which only happens in production.

**Why the return type was also wrong:** The real function returns
`{ redirectTo: string }` (a plain object with a redirect URL), not a
`Response`. The original code treated it as a `Response` directly:

```typescript
// ❌ BROKEN — wrong: completeAuthorization does NOT return a Response
const resp = await completeAuthorization(request, env, { ... });
return resp;  // resp would be { redirectTo: "..." }, not a Response
```

### The Real API

By reading `oauth-provider.js` source and its TypeScript declarations:

```typescript
// env.OAUTH_PROVIDER is an OAuthHelpers instance injected by the library
interface OAuthHelpers {
  parseAuthRequest(request: Request): Promise<AuthRequest>;
  completeAuthorization(options: CompleteAuthorizationOptions): Promise<{ redirectTo: string }>;
  lookupClient(clientId: string): Promise<ClientInfo | null>;
}

interface CompleteAuthorizationOptions {
  request:  AuthRequest;   // ← parsed result from parseAuthRequest()
  userId:   string;
  metadata: any;
  scope:    string[];
  props:    any;
}
```

### Fix

Deleted the `declare function` and the hand-written `OAuthRequestInfo`
interface. Replaced with the correct library call:

```typescript
// ✅ FIXED — use the library helper, handle its actual return type
const { redirectTo } = await env.OAUTH_PROVIDER.completeAuthorization({
  request:  oauthReq,       // AuthRequest from parseAuthRequest()
  userId:   email,
  metadata: { cfJwt },
  scope:    oauthReq.scope,
  props:    {},
});

return Response.redirect(redirectTo, 302);  // build Response ourselves
```

Added `OAUTH_PROVIDER: OAuthHelpers` to the `Env` interface in `index.ts`
so TypeScript knows about it:

```typescript
// index.ts
import OAuthProvider, { type OAuthHelpers } from "@cloudflare/workers-oauth-provider";

export interface Env {
  OAUTH_KV: KVNamespace;
  COOKIE_ENCRYPTION_KEY: string;
  CF_TEAM_DOMAIN: string;
  OAUTH_PROVIDER: OAuthHelpers;   // ← added: injected by the library at runtime
}
```

---

## Bug 3 — Local testing: no redirect to login page

**File:** `cloudflare-worker/src/access-handler.ts`

**Discovered:** When running `wrangler dev` locally, hitting `/authorize`
returned a 401 with the message "No CF_Authorization cookie found" instead
of redirecting to the CF Access login page.

### Root Cause

In production, CF Access sits at the Cloudflare edge and intercepts the
request to `/authorize` BEFORE the Worker code ever runs. If the user is
not authenticated, CF Access redirects them to the login page. The Worker
only sees the request AFTER the user has logged in and the
`CF_Authorization` cookie is set.

Locally (`wrangler dev`), there is no CF Access layer. The Worker code
runs immediately for every request. The original code returned 401 when
no cookie was found — which was correct for production but useless locally
since it showed no login page.

### Fix

When no `CF_Authorization` cookie is present (i.e. running locally),
manually redirect to the CF Access login URL instead of returning 401.
This simulates what CF Access does at the edge in production.

```typescript
// ✅ FIXED — redirect to CF Access login when no cookie (local dev only)
if (!cfJwt) {
  const cfTeamDomain = env.CF_TEAM_DOMAIN || "custac.cloudflareaccess.com";
  const loginUrl =
    `https://${cfTeamDomain}/cdn-cgi/access/login/${url.hostname}` +
    `?redirect_url=${encodeURIComponent(request.url)}`;

  return Response.redirect(loginUrl, 302);
}
```

Supporting changes:

- Added `CF_TEAM_DOMAIN = "custac.cloudflareaccess.com"` to `wrangler.toml`
  `[vars]` so `wrangler dev` picks it up automatically.
- Added `CF_TEAM_DOMAIN: string` to the `Env` interface in `index.ts`.

**Known limitation of this fix:** CF Access will not redirect back to
`http://localhost:8787` after login. Cloudflare only allows redirects to
registered HTTPS domains. You WILL see the real login page but the OAuth
round-trip will not complete locally. For a full round-trip, point Cursor
at the deployed worker.

---

## Bug 4 — `mcp.json` in `stdio` mode bypasses all OAuth

**File:** `.cursor/mcp.json`

**Discovered:** Cursor was connecting to the MCP server without any
authentication prompt — no 401, no login page, no OAuth flow.

### Root Cause

The `mcp.json` was configured in `stdio` (subprocess) mode:

```json
{
  "mcpServers": {
    "leadgen-dashboard-mcp": {
      "command": "/path/to/.venv/bin/python",
      "args": ["-m", "mcp_server"],
      "cwd": "..."
    }
  }
}
```

In `stdio` mode, Cursor spawns the MCP server as a local subprocess and
communicates over stdin/stdout. There is no HTTP — no requests, no 401,
no `WWW-Authenticate` header, no OAuth flow. The `CFAuthMiddleware` is
never entered because there are no HTTP requests to intercept.

### Fix

Switched to `url` (SSE / streamable HTTP) mode so Cursor connects over
HTTP, receives the 401, and starts the OAuth flow:

```json
{
  "mcpServers": {
    "leadgen-dashboard-mcp-local-sse": {
      "url": "http://localhost:8090"
    }
  }
}
```

The name `leadgen-dashboard-mcp-local-sse` makes it explicit this is
the local HTTP instance (not production, not stdio).

---

## Logging Added

### CF Worker (`cloudflare-worker/src/index.ts`)

Every request and response is now fully logged in `wrangler dev` output:

```
[Worker] ────────────────────────────────────────────────────────────
[Worker] ▶ REQUEST  POST http://localhost:8787/register
[Worker]   Headers: { "content-type": "application/json", ... }
[Worker]   Body: {"redirect_uris":["cursor://anysphere.cursor-mcp/oauth/callback"],...}
[Worker] ◀ RESPONSE POST http://localhost:8787/register → 201 Created
[Worker]   Response Headers: { "content-type": "application/json" }
[Worker]   Response Body: {"client_id":"5qguABKg5FULQGQb",...}
[Worker] ────────────────────────────────────────────────────────────
```

- Full URL (protocol + host + path + query string) on every request
- All request headers (`Authorization` masked to first 12 + last 6 chars,
  `Cookie` shows cookie names only — never values)
- Request body for POST/PUT (truncated at 2000 chars)
- All response headers
- Response body (truncated at 2000 chars)

Inside `/authorize` specifically (`cloudflare-worker/src/access-handler.ts`):

```
[Worker/authorize] client_id=5qguABKg5FULQGQb  redirect_uri=cursor://...  state=teststat…
[Worker/authorize] ✗ No CF_Authorization cookie — redirecting to CF Access login: https://...
[Worker/authorize] ✓ CF_Authorization cookie present — user=arihant@company.com  jwt=eyJhbGc......abc123
[Worker/authorize] ✓ Authorization complete — redirecting to cursor://anysphere.cursor-mcp/oauth/callback?code=...
```

### MCP Server (`mcp_server/middleware.py`)

Every request through `CFAuthMiddleware` is logged:

```
[MCP] ▶ POST /   headers={content-type: application/json, authorization: Bearer eyJhbGc......abc123}
[MCP] 🔑 Token found via [Authorization: Bearer]  preview=eyJhbGc......abc123
[MCP] ✓ Authenticated  user=arihant@company.com  sub=abc-uuid  groups=[]
[MCP] ◀ POST / → 200  user=arihant@company.com  (12ms)
```

Or when no token:

```
[MCP] ▶ POST /   headers={content-type: application/json}
[MCP] ✗ POST / — no Bearer token in request (checked Authorization header + cf-access-jwt-assertion)
[MCP] ↩ 401  WWW-Authenticate points Cursor to: http://localhost:8090/.well-known/oauth-authorization-server
```

---

## Verification

Run these three curl commands in order to confirm the full chain works:

```bash
# 1. MCP server returns 401 with WWW-Authenticate pointing to /.well-known
curl -si http://localhost:8090/ -X POST -H "Content-Type: application/json" -d '{}'

# 2. /.well-known lists the local worker endpoints
curl -s http://localhost:8090/.well-known/oauth-authorization-server | python3 -m json.tool

# 3. Register a client, then authorize — should get 302 to CF Access login
CLIENT_ID=$(curl -s http://localhost:8787/register \
  -X POST -H "Content-Type: application/json" \
  -d '{"redirect_uris":["cursor://anysphere.cursor-mcp/oauth/callback"],"grant_types":["authorization_code"],"token_endpoint_auth_method":"none"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['client_id'])")

CODE_CHALLENGE=$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '=' | \
  xargs -I{} sh -c 'echo -n "{}" | openssl dgst -sha256 -binary | openssl base64 | tr "+/" "-_" | tr -d "="')

curl -si "http://localhost:8787/authorize?response_type=code&client_id=${CLIENT_ID}&redirect_uri=cursor%3A%2F%2Fanysphere.cursor-mcp%2Foauth%2Fcallback&code_challenge=${CODE_CHALLENGE}&code_challenge_method=S256&state=teststate"
```

Expected final output:
```
HTTP/1.1 302 Found
Location: https://custac.cloudflareaccess.com/cdn-cgi/access/login/localhost?redirect_url=...
```

---

## File Change Summary

| File | What Changed |
|---|---|
| `cloudflare-worker/src/access-handler.ts` | Removed phantom 4th param + `declare function`; use `env.OAUTH_PROVIDER.parseAuthRequest()` and `env.OAUTH_PROVIDER.completeAuthorization()`; added local CF Access login redirect; added per-route logging |
| `cloudflare-worker/src/index.ts` | Added `OAUTH_PROVIDER: OAuthHelpers` to `Env` interface; imported `OAuthHelpers` type; added `CF_TEAM_DOMAIN`; wrapped OAuthProvider in a logging layer (full URL, headers, body, response) |
| `cloudflare-worker/wrangler.toml` | Added `[vars] CF_TEAM_DOMAIN = "custac.cloudflareaccess.com"` |
| `mcp_server/middleware.py` | Added structured per-request logging with token masking, source tracking, timing, and identity info |
| `.cursor/mcp.json` | Switched from `command` (stdio) to `url` (SSE) mode; renamed server to `leadgen-dashboard-mcp-local-sse` |
