# Local Setup — MCP Server + CF Worker

How to run the entire OAuth authentication chain locally so you can verify
that Cursor is redirected to the CF Access login page before connecting to
the MCP server.

---

## What We Are Testing

```
Cursor → MCP Server (localhost:8090)
           ↓ 401 + WWW-Authenticate
         Cursor fetches /.well-known
           ↓ authorization_endpoint = http://localhost:8787/authorize
         Cursor opens browser →
           ↓ Worker (localhost:8787) 302-redirects to
         https://custac.cloudflareaccess.com  ← real CF Access login page
```

The goal is to confirm every hop of this chain works before touching
the deployed infrastructure.

---

## Architecture: Local vs Production

| What | Local (wrangler dev) | Production |
|---|---|---|
| MCP server | `localhost:8090` (SSE mode) | `mcp-stage.customer-acquisition.co` |
| CF Worker | `localhost:8787` (wrangler dev) | `leadgen-mcp-oauth.tech-2e7.workers.dev` |
| CF Access login | Browser redirected manually by worker code | CF Access intercepts `/authorize` BEFORE the worker runs |
| OAuth round-trip | One-way only (CF Access won't redirect back to `localhost`) | Full round-trip — Cursor gets a token |
| Token validation | Works if CF_AUD_TAG matches | Works |

> **Key difference:** In production, CF Access sits in front of the worker and intercepts
> `/authorize` before the worker code ever runs. Locally there is no CF Access layer,
> so the worker handles the redirect itself (see code change below).

---

## Code Changes Made for Local Testing

### 1. `cloudflare-worker/src/access-handler.ts`

**Problem:** When there is no `CF_Authorization` cookie, the worker was returning a
plain 401. Locally this is always the case because CF Access is not in front of
`wrangler dev`.

**Fix:** When no cookie is found, redirect the browser to the real CF Access login URL
instead of returning 401. In production this code path is never reached (CF Access
forces login before the worker runs), so it has zero effect on production.

```typescript
if (!cfJwt) {
  // In production, CF Access intercepts /authorize BEFORE this Worker code runs.
  // Locally (wrangler dev), no CF Access is in front — redirect manually.
  const cfTeamDomain = env.CF_TEAM_DOMAIN || "custac.cloudflareaccess.com";
  const loginUrl =
    `https://${cfTeamDomain}/cdn-cgi/access/login/${new URL(request.url).hostname}` +
    `?redirect_url=${encodeURIComponent(request.url)}`;
  return Response.redirect(loginUrl, 302);
}
```

### 2. `cloudflare-worker/src/index.ts` — `Env` interface

Added `CF_TEAM_DOMAIN` so TypeScript knows about the new variable:

```typescript
export interface Env {
  OAUTH_KV: KVNamespace;
  COOKIE_ENCRYPTION_KEY: string;
  CF_TEAM_DOMAIN: string;   // ← added (local dev only)
}
```

### 3. `cloudflare-worker/wrangler.toml` — plain-text variable

Added `CF_TEAM_DOMAIN` as a `[vars]` entry so `wrangler dev` picks it up automatically:

```toml
[vars]
CF_TEAM_DOMAIN = "custac.cloudflareaccess.com"
```

This is not a secret — it is the public team domain shown in the Cloudflare
Zero Trust dashboard under Settings → General.

### 4. `.cursor/mcp.json` — switched from `stdio` to `url` (SSE) mode

**Old (stdio):** Cursor spawns a Python subprocess. No HTTP, no 401, no OAuth.

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

**New (SSE/URL mode):** Cursor connects over HTTP — the middleware sees unauthenticated
requests and returns 401, triggering the full OAuth flow.

```json
{
  "mcpServers": {
    "leadgen-dashboard-mcp-local-sse": {
      "url": "http://localhost:8090"
    }
  }
}
```

> The name `leadgen-dashboard-mcp-local-sse` makes it clear this is the
> local SSE instance, not the production server or a stdio subprocess.

---

## Running Everything Locally

### Prerequisites

```bash
# Python venv with dependencies
cd /Users/arihant.ja/Desktop/leadgen-ops/leadgen-dashboard-mcp
source .venv/bin/activate

# Node deps for the worker (already installed)
cd cloudflare-worker && npm install
```

### Terminal 1 — MCP Server (SSE mode)

```bash
cd /path/to/leadgen-dashboard-mcp
source .venv/bin/activate

MCP_TRANSPORT=sse \
AUTH_ENABLED=true \
MCP_PORT=8090 \
MCP_PUBLIC_URL=http://localhost:8090 \
OAUTH_WORKER_URL=http://localhost:8787 \
CF_JWKS_URL=https://custac.cloudflareaccess.com/cdn-cgi/access/certs \
CF_AUD_TAG=53ab9a7d3d50521090b681db51507ab1679e63a19628ed4689f35c427c63caaa \
python -m mcp_server
```

Expected output:
```
INFO:     Uvicorn running on http://0.0.0.0:8090 (Press CTRL+C to quit)
```

**Key env vars explained:**

| Variable | Value | Why |
|---|---|---|
| `MCP_TRANSPORT` | `sse` | Runs as HTTP server (not stdio subprocess) |
| `AUTH_ENABLED` | `true` | Activates `CFAuthMiddleware` — returns 401 for unauthenticated requests |
| `MCP_PUBLIC_URL` | `http://localhost:8090` | Used in the `WWW-Authenticate` header and `/.well-known` response |
| `OAUTH_WORKER_URL` | `http://localhost:8787` | Tells `/.well-known` to point Cursor at the local worker |
| `CF_JWKS_URL` | `https://custac.cloudflareaccess.com/cdn-cgi/access/certs` | Where to fetch Cloudflare's public keys for JWT validation |
| `CF_AUD_TAG` | `53ab9a7d...` | Expected audience in the CF JWT (must match the CF Access app that protects the worker) |

### Terminal 2 — Cloudflare Worker (wrangler dev)

```bash
cd /path/to/leadgen-dashboard-mcp/cloudflare-worker

COOKIE_ENCRYPTION_KEY=$(openssl rand -hex 32) \
npx wrangler dev --port 8787
```

Expected output:
```
⛅️ wrangler 4.x.x
Your Worker has access to the following bindings:
  env.OAUTH_KV              KV Namespace  local
  env.CF_TEAM_DOMAIN        Environment Variable  local

⎔ Starting local server...
[wrangler:info] Ready on http://localhost:8787
```

> `COOKIE_ENCRYPTION_KEY` is a random 32-byte hex string used to encrypt
> OAuth state cookies. Generate a new one each time you start locally —
> it does not need to persist between restarts.

---

## Verifying the Chain with curl

Run these three commands to confirm each hop works before opening Cursor.

### Step 1 — MCP server must return 401

```bash
curl -si http://localhost:8090/ -X POST \
  -H "Content-Type: application/json" -d '{}'
```

Expected:
```
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer realm="leadgen-ops",
  resource_metadata="http://localhost:8090/.well-known/oauth-authorization-server"

{"error":"Authentication required. Provide a valid Cloudflare Access JWT."}
```

### Step 2 — OAuth discovery must list the local worker

```bash
curl -s http://localhost:8090/.well-known/oauth-authorization-server | python3 -m json.tool
```

Expected:
```json
{
    "issuer": "http://localhost:8090",
    "authorization_endpoint": "http://localhost:8787/authorize",
    "token_endpoint": "http://localhost:8787/token",
    "registration_endpoint": "http://localhost:8787/register",
    "response_types_supported": ["code"],
    "grant_types_supported": ["authorization_code"],
    "code_challenge_methods_supported": ["S256"]
}
```

### Step 3 — Worker must redirect to CF Access login

```bash
curl -si "http://localhost:8787/authorize?response_type=code&client_id=test&redirect_uri=http://localhost:8090/callback&state=teststate&code_challenge=abc123&code_challenge_method=S256"
```

Expected:
```
HTTP/1.1 302 Found
Location: https://custac.cloudflareaccess.com/cdn-cgi/access/login/localhost?redirect_url=...
```

If you see that `302` pointing to `custac.cloudflareaccess.com` — the chain is working.

---

## Enabling in Cursor

In `.cursor/mcp.json` (already updated):

```json
{
  "mcpServers": {
    "leadgen-dashboard-mcp-local-sse": {
      "url": "http://localhost:8090"
    }
  }
}
```

1. Make sure both terminals above are running.
2. Open Cursor → Settings → MCP Servers (or reload the window).
3. Cursor will try to connect to `http://localhost:8090`.
4. It receives a 401 → fetches `/.well-known` → discovers the local worker endpoints.
5. Cursor opens your browser to `http://localhost:8787/authorize?...`.
6. The local worker issues a `302` → your browser lands on the real CF Access login page.

---

## Known Limitation of Local Testing

CF Access will not redirect back to `http://localhost:8787` after login.
Cloudflare only allows redirects to registered HTTPS domains.

This means:
- You **will** see the real CF Access login page.
- The full OAuth token exchange **will not** complete locally.
- To complete the full round-trip (login → get token → use MCP tools), point Cursor
  at the deployed worker instead:

  ```json
  {
    "mcpServers": {
      "leadgen-dashboard-mcp-local-sse": {
        "url": "http://localhost:8090"
      }
    }
  }
  ```
  
  And restart the MCP server with `OAUTH_WORKER_URL=https://leadgen-mcp-oauth.tech-2e7.workers.dev`
  (which is already the default in `.env`). With the deployed worker + CF Access in front,
  the full flow completes.

---

## How Cursor Routes Through the OAuth Flow

A common point of confusion: **Cursor always talks to the Worker — it never talks
directly to CF Access.** CF Access is invisible to Cursor; it only appears to the
user's browser.

```
Cursor (MCP client)
   │
   │  1. POST http://localhost:8090/   ← hits MCP server
   │     ← 401 + WWW-Authenticate: Bearer resource_metadata=".../.well-known"
   │
   │  2. GET http://localhost:8090/.well-known/oauth-authorization-server
   │     ← { "registration_endpoint": "http://localhost:8787/register",
   │          "authorization_endpoint": "http://localhost:8787/authorize" }
   │
   │  3. POST http://localhost:8787/register   ← Worker first, NOT CF Access
   │     ← { "client_id": "abc123" }
   │
   │  4. Opens browser → http://localhost:8787/authorize?client_id=abc123&...
   │                              ↓
   │              ┌───────────────────────────────────────┐
   │              │         PRODUCTION only               │
   │              │  CF Access intercepts /authorize      │
   │              │  at the network edge BEFORE Worker    │
   │              │  code runs → shows login page         │
   │              │  User logs in → CF Access sets        │
   │              │  CF_Authorization cookie → redirects  │
   │              │  back to /authorize → Worker runs ✓   │
   │              └───────────────────────────────────────┘
   │                              ↓
   │              ┌───────────────────────────────────────┐
   │              │         LOCAL only                    │
   │              │  No CF Access in front → Worker code  │
   │              │  runs immediately → no cookie found   │
   │              │  → Worker issues 302 to CF Access     │
   │              │  login page manually                  │
   │              └───────────────────────────────────────┘
   │
   │  5. POST http://localhost:8787/token   ← Cursor exchanges code for JWT
   │     ← { "access_token": "<CF JWT>" }
   │
   │  6. POST http://localhost:8090/   ← MCP tool calls from now on
   │     Authorization: Bearer <CF JWT>
   │     ← 200 OK (JWT validated, tools respond)
```

**CF Access is invisible to Cursor.** Cursor only knows the Worker URL.
CF Access is a gatekeeper that the user's browser encounters at step 4.

---

## How Sign-in Works in Cursor (What You Actually See)

Cursor does **not** automatically redirect you to the login page. The flow from
your perspective as a user:

### First time ever

```
1. Both terminals running (MCP + Worker)
2. Cursor Settings → MCP Servers
   → "leadgen-dashboard-mcp-local-sse" appears with a "Sign in" button
3. You click "Sign in"
   → Browser opens → Worker 302 → CF Access login page
4. You log in with your company Google/email
   → CF Access sets CF_Authorization cookie
   → Cursor receives the JWT and stores it
5. MCP panel shows the server as connected ✓
   → You never touch MCP settings again
```

### Every subsequent Cursor session (token still valid)

```
Cursor starts → reads stored token → sends Bearer to MCP → works immediately
No action needed from you
```

### When the token expires (CF JWTs last ~24 hours)

**Scenario A — Cursor auto-refreshes silently (ideal)**
```
Token expires → Cursor detects 401 → auto-refreshes using stored OAuth state
You notice nothing
```

**Scenario B — Cursor prompts you (fallback)**
```
Token expires → Cursor shows "Reconnect" in MCP panel
You click it → Browser opens → CF Access login page
If your browser CF session is still alive → logs in silently, no password needed
If your browser CF session also expired → you type credentials once
```

In practice, CF Access browser sessions last much longer than the JWT (days vs
hours), so a "reconnect" usually completes in one click without typing anything.

**You will only manually intervene:**
- First-time setup
- You explicitly logged out of CF Access in the browser
- Browser cookies were cleared and the CF session died

Day-to-day usage requires **zero manual steps**.

---

## Quick Reference

```
localhost:8090  →  MCP server (Python / uvicorn, SSE mode, AUTH_ENABLED)
localhost:8787  →  CF Worker (wrangler dev, local KV, CF_TEAM_DOMAIN set)

/.well-known/oauth-authorization-server  →  served by MCP server (no auth)
/register                                →  served by CF Worker (bypass policy)
/authorize                               →  served by CF Worker → 302 to CF Access login
/token                                   →  served by CF Worker (bypass policy)
```
