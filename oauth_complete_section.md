# OAuth Implementation Plan — MCP + Cloudflare Access (OAuth Shim Approach)

> **Approach chosen:** 5.2 — OAuth Shim that sits between any MCP client and Cloudflare Access.
> This lets Cursor, Claude Desktop, and any future MCP client connect via the standard MCP OAuth flow.

## Table of Contents

1. [Current State & Problems](#1-current-state--problems)
2. [Target Architecture](#2-target-architecture)
3. [How the OAuth Flow Works End-to-End](#3-how-the-oauth-flow-works-end-to-end)
4. [Cloudflare Zero Trust Setup](#4-cloudflare-zero-trust-setup)
5. [OAuth Shim — Cloudflare Worker](#5-oauth-shim--cloudflare-worker)
6. [MCP Server Changes](#6-mcp-server-changes)
7. [Laravel Backend Changes](#7-laravel-backend-changes)
8. [Client Configuration](#8-client-configuration)
9. [File Changes Summary](#9-file-changes-summary)
10. [Deployment Checklist](#10-deployment-checklist)
11. [Testing Plan](#11-testing-plan)
12. [FAQ](#12-faq)

---

## 1. Current State & Problems

### Current Architecture

```
Cursor (stdio)          Cursor/Claude (HTTP)
      │                        │
      │ JSON-RPC over stdin    │ POST / (Streamable HTTP)
      ▼                        ▼
┌─────────────────────────────────────────┐
│            MCP Server (FastMCP)         │
│                                         │
│  • No authentication on incoming reqs   │
│  • Uses hardcoded API_TOKEN for all     │
│    calls to Laravel                     │
│  • Single shared identity              │
└────────────────┬────────────────────────┘
                 │
                 │ Authorization: Bearer <hardcoded API_TOKEN>
                 ▼
┌─────────────────────────────────────────┐
│          Laravel API                    │
│  /api/admin/offer/*                     │
│                                         │
│  • CF Access middleware BYPASSED        │
│    for MCP server requests              │
│  • Cannot distinguish which user        │
│    made a change via MCP                │
└─────────────────────────────────────────┘
```

### Problems

| # | Problem | Impact |
|---|---------|--------|
| 1 | MCP HTTP endpoint has **zero authentication** | Anyone who can reach the port can call any tool |
| 2 | **Hardcoded `API_TOKEN`** shared by all users | Activity logs show one identity for all MCP changes |
| 3 | CF Access middleware is **bypassed** | Security model is broken for MCP-originated requests |
| 4 | No way to enforce **RBAC per user** | Every MCP user has identical permissions |

---

## 2. Target Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│   Cursor / Claude / Any MCP Client                                │
│       │                                                           │
│       │ ① Connects to MCP Server URL                              │
│       │                                                           │
│       ▼                                                           │
│   ┌──────────────────────────────────────────┐                    │
│   │  OAuth Shim (Cloudflare Worker)          │                    │
│   │                                          │                    │
│   │  /.well-known/oauth-authorization-server │ ◀── ② Client      │
│   │  /authorize                              │     discovers      │
│   │  /token                                  │     OAuth          │
│   │  /register                               │     endpoints      │
│   │  /callback                               │                    │
│   │                                          │                    │
│   │  ③ /authorize → redirect to CF Access    │                    │
│   │  ④ User logs in (SSO / Google / OTP)     │                    │
│   │  ⑤ CF returns signed JWT via /callback   │                    │
│   │  ⑥ Shim issues MCP token to client       │                    │
│   │                                          │                    │
│   │  /mcp  ──proxies──▶  MCP Server          │                    │
│   │  (with Authorization header attached)    │                    │
│   └──────────────┬───────────────────────────┘                    │
│                  │                                                 │
│                  │ ⑦ Every tool call carries user's JWT            │
│                  ▼                                                 │
│   ┌──────────────────────────────────────────┐                    │
│   │  MCP Server (your FastMCP, Python)       │                    │
│   │                                          │                    │
│   │  ⑧ Validates JWT (CF JWKS)               │                    │
│   │  ⑨ Extracts user identity (email, sub)   │                    │
│   │  ⑩ Forwards JWT to Laravel API           │                    │
│   └──────────────┬───────────────────────────┘                    │
│                  │                                                 │
│                  │ Authorization: Bearer <user's CF JWT>           │
│                  │ Cf-Access-Jwt-Assertion: <user's CF JWT>        │
│                  ▼                                                 │
│   ┌──────────────────────────────────────────┐                    │
│   │  Laravel API (existing)                  │                    │
│   │                                          │                    │
│   │  ⑪ CF Access middleware validates JWT    │                    │
│   │     (no more bypass!)                    │                    │
│   │  ⑫ RBAC based on user identity in JWT    │                    │
│   └──────────────────────────────────────────┘                    │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### What changes vs. today

| Component | Before | After |
|-----------|--------|-------|
| MCP Server | No auth, hardcoded token | Validates CF JWT, forwards per-user token |
| OAuth Shim | Does not exist | New Cloudflare Worker handles OAuth ↔ CF Access |
| Laravel API | CF middleware bypassed for MCP | CF middleware active — receives real user JWT |
| Clients | Point to raw MCP URL | Point to Worker URL (which proxies to MCP) |

---

## 3. How the OAuth Flow Works End-to-End

This follows the [MCP Authorization spec (2025-03-26)](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/authorization/) and Cloudflare's [Access for SaaS OIDC flow](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/saas-mcp/).

### Sequence Diagram

```
MCP Client            OAuth Shim             CF Access            Your IdP
(Cursor/Claude)       (CF Worker)            (SaaS OIDC app)      (Google/Okta)
     │                     │                      │                    │
     │─── POST /mcp ──────▶│                      │                    │
     │◀── 401 Unauthorized─│                      │                    │
     │    WWW-Authenticate: Bearer                 │                    │
     │    resource_metadata=".../.well-known/..."  │                    │
     │                     │                      │                    │
     │─── GET /.well-known/│                      │                    │
     │    oauth-authz-     │                      │                    │
     │    server ─────────▶│                      │                    │
     │◀── {endpoints} ────│                      │                    │
     │                     │                      │                    │
     │─── POST /register ─▶│                      │                    │
     │◀── {client_id} ────│                      │                    │
     │                     │                      │                    │
     │─── Browser opens ──▶│                      │                    │
     │    /authorize        │                      │                    │
     │    ?redirect_uri=... │                      │                    │
     │    &code_challenge=..│                      │                    │
     │                     │──── 302 redirect ───▶│                    │
     │                     │    to CF Access       │                    │
     │                     │    /authorization     │                    │
     │                     │                      │─── redirect ──────▶│
     │                     │                      │                    │
     │                     │                      │    User logs in    │
     │                     │                      │    (SSO/Google/OTP)│
     │                     │                      │                    │
     │                     │                      │◀── auth code ─────│
     │                     │◀── redirect to ─────│                    │
     │                     │    /callback?code=...│                    │
     │                     │                      │                    │
     │                     │──── POST to CF ─────▶│                    │
     │                     │    /token             │                    │
     │                     │    (exchange code     │                    │
     │                     │     for JWT)          │                    │
     │                     │◀── {access_token, ──│                    │
     │                     │     id_token (JWT)}   │                    │
     │                     │                      │                    │
     │                     │ Verify id_token       │                    │
     │                     │ Extract user identity │                    │
     │                     │ Issue MCP token       │                    │
     │                     │                      │                    │
     │◀── redirect to ────│                      │                    │
     │    client callback   │                      │                    │
     │    ?code=<mcp_code>  │                      │                    │
     │                     │                      │                    │
     │─── POST /token ────▶│                      │                    │
     │    (exchange code    │                      │                    │
     │     for MCP token)   │                      │                    │
     │◀── {access_token} ─│                      │                    │
     │                     │                      │                    │
     │ Store token locally  │                      │                    │
     │                     │                      │                    │
     │─── POST /mcp ──────▶│                      │                    │
     │    Authorization:    │                      │                    │
     │    Bearer <token>    │                      │                    │
     │                     │──── proxy to ────────────────────────────▶│
     │                     │    MCP Server                             │
     │                     │    (with user's JWT)                      │
     │◀── tool result ────│◀────────────────────────────────────────│
     │                     │                      │                    │
```

### Key Points

- The **OAuth Shim** is the intermediary. The MCP client never talks directly to CF Access.
- The shim issues its own MCP token that embeds/references the CF Access JWT.
- On every subsequent tool call, the shim forwards the user's JWT to the MCP server.
- The MCP server validates the JWT and forwards it to the Laravel API.
- Laravel's existing CF Access middleware validates the same JWT — **no bypass needed**.

---

## 4. Cloudflare Zero Trust Setup

### 4.1 Create an "Access for SaaS" Application (OIDC)

This is different from a standard "Self-hosted" Access Application. You need a **SaaS Application** with **OIDC** protocol so that CF acts as an OAuth/OIDC provider that the Worker can exchange tokens with.

**Steps (in the CF Zero Trust Dashboard):**

1. Go to [Cloudflare One](https://one.dash.cloudflare.com/) → **Access** → **Applications**
2. Click **Add an application** → Select **SaaS**
3. Configure:
   - **Application name:** `Leadgen MCP Server`
   - In the **Application** field, type a custom app name and select the textbox that appears
   - **Authentication protocol:** OIDC
4. Click **Add application**
5. Under **Redirect URLs**, add:
   ```
   https://<your-worker-domain>/callback
   ```
   Example: `https://leadgen-mcp-oauth.yourdomain.workers.dev/callback`
6. **Copy and save these values** (you'll need them for the Worker):

   | Value | Where it goes |
   |-------|--------------|
   | **Client ID** | Worker secret `ACCESS_CLIENT_ID` |
   | **Client secret** | Worker secret `ACCESS_CLIENT_SECRET` |
   | **Token endpoint** | Worker secret `ACCESS_TOKEN_URL` |
   | **Authorization endpoint** | Worker secret `ACCESS_AUTHORIZATION_URL` |
   | **Key endpoint (JWKS)** | Worker secret `ACCESS_JWKS_URL` |

   The endpoint URLs follow this pattern:
   ```
   Token:         https://<team-name>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/token
   Authorization: https://<team-name>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/authorization
   JWKS:          https://<team-name>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/jwks
   ```

7. (Optional) Under **Advanced settings**, enable **Refresh tokens** with a 90-day lifetime to reduce re-login frequency
8. Add an **Access Policy**:
   - **Policy name:** `Ops Team`
   - **Action:** Allow
   - **Include rule:** Emails ending in `@yourcompany.com` (or a specific CF Access Group)
9. Under **Authentication**, select the Identity Providers you want (Google Workspace, Okta, One-time PIN, etc.)
10. Save the application

### 4.2 Values You'll Need

After setup, you should have these 5+1 values:

```
ACCESS_CLIENT_ID=<from SaaS app>
ACCESS_CLIENT_SECRET=<from SaaS app>
ACCESS_TOKEN_URL=https://<team>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/token
ACCESS_AUTHORIZATION_URL=https://<team>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/authorization
ACCESS_JWKS_URL=https://<team>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/jwks
COOKIE_ENCRYPTION_KEY=<generate with: openssl rand -hex 32>
```

---

## 5. OAuth Shim — Cloudflare Worker

The OAuth Shim is a **Cloudflare Worker** that implements the MCP OAuth spec and delegates authentication to Cloudflare Access. It is based on Cloudflare's official [`remote-mcp-cf-access`](https://github.com/cloudflare/ai/tree/main/demos/remote-mcp-cf-access) demo.

### 5.1 What the Worker Does

| Endpoint | Purpose |
|----------|---------|
| `GET /.well-known/oauth-authorization-server` | Returns OAuth metadata (RFC 8414) so MCP clients discover the auth endpoints |
| `POST /register` | Dynamic client registration — MCP clients register themselves and get a `client_id` |
| `GET /authorize` | Shows an approval dialog, then redirects user to CF Access login |
| `POST /authorize` | Handles the approval form submission |
| `GET /callback` | Receives the auth code from CF Access, exchanges it for a JWT, verifies the JWT, and issues an MCP token back to the client |
| `POST /token` | Exchanges MCP auth codes for MCP access tokens |
| `POST /mcp` | **Proxies to your MCP server**, attaching the user's JWT as `Authorization` header |

### 5.2 Worker Project Structure

Create a new Worker project (separate from your MCP server):

```
leadgen-mcp-oauth-worker/
├── src/
│   ├── index.ts              # Entry point — OAuthProvider setup + MCP proxy
│   ├── access-handler.ts     # Handles /authorize, /callback (CF Access integration)
│   └── workers-oauth-utils.ts # Utility functions (CSRF, state mgmt, approval UI)
├── wrangler.jsonc             # Worker config (KV, secrets, durable objects)
├── package.json
└── tsconfig.json
```

### 5.3 How to Set Up the Worker

**Option A: Use Cloudflare's template directly (recommended to start)**

```bash
npm create cloudflare@latest -- leadgen-mcp-oauth \
  --template=cloudflare/ai/demos/remote-mcp-cf-access
```

**Option B: Deploy via the Cloudflare Dashboard**

Go to [deploy.workers.cloudflare.com](https://deploy.workers.cloudflare.com/?url=https://github.com/cloudflare/ai/tree/main/demos/remote-mcp-cf-access) and follow the guided setup.

### 5.4 Customize the Worker for Your MCP Server

The template's `index.ts` includes a demo MCP server (`MyMCP`) running inside the Worker. You need to **replace** that with a proxy to your existing Python MCP server.

#### `index.ts` — Replace the built-in MCP with a proxy

```typescript
import OAuthProvider from "@cloudflare/workers-oauth-provider";
import { handleAccessRequest } from "./access-handler";

export interface Env {
  // CF Access OIDC credentials
  ACCESS_CLIENT_ID: string;
  ACCESS_CLIENT_SECRET: string;
  ACCESS_TOKEN_URL: string;
  ACCESS_AUTHORIZATION_URL: string;
  ACCESS_JWKS_URL: string;
  COOKIE_ENCRYPTION_KEY: string;

  // Your MCP server's internal URL (not exposed publicly)
  MCP_UPSTREAM_URL: string; // e.g. "https://mcp-internal.yourdomain.com"

  // KV namespace for OAuth state
  OAUTH_KV: KVNamespace;

  // Durable Object (required by workers-oauth-provider)
  MCP_OBJECT: DurableObjectNamespace;
}

// Proxy handler — forwards authenticated /mcp requests to your Python MCP server
async function mcpProxyHandler(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);

  // The OAuthProvider injects user props into the request.
  // Extract the CF Access token that was stored during /callback.
  const props = (request as any).__props as { accessToken: string; email: string } | undefined;

  if (!props?.accessToken) {
    return new Response("Unauthorized", { status: 401 });
  }

  // Build the upstream URL (your Python MCP server)
  const upstreamUrl = new URL(url.pathname.replace(/^\/mcp/, "/"), env.MCP_UPSTREAM_URL);
  upstreamUrl.search = url.search;

  // Forward the request with the user's CF Access token
  const headers = new Headers(request.headers);
  headers.set("Authorization", `Bearer ${props.accessToken}`);
  headers.set("Cf-Access-Jwt-Assertion", props.accessToken);
  headers.set("X-User-Email", props.email);

  return fetch(upstreamUrl.toString(), {
    method: request.method,
    headers,
    body: request.body,
  });
}

export default new OAuthProvider({
  apiRoute: "/mcp",
  apiHandler: { fetch: mcpProxyHandler as any },
  defaultHandler: { fetch: handleAccessRequest as any },
  authorizeEndpoint: "/authorize",
  tokenEndpoint: "/token",
  clientRegistrationEndpoint: "/register",
});
```

#### `access-handler.ts` — Use as-is from the template

The [`access-handler.ts`](https://github.com/cloudflare/ai/blob/main/demos/remote-mcp-cf-access/src/access-handler.ts) from Cloudflare's demo handles:
- `GET /authorize` — shows approval dialog, redirects to CF Access
- `POST /authorize` — processes approval, redirects with PKCE
- `GET /callback` — exchanges code for JWT, verifies JWT, issues MCP token

Use it as-is. The only customization you might want is changing the server name/logo in the approval dialog.

#### `workers-oauth-utils.ts` — Use as-is from the template

The [`workers-oauth-utils.ts`](https://github.com/cloudflare/ai/blob/main/demos/remote-mcp-cf-access/src/workers-oauth-utils.ts) provides CSRF protection, state management, approval UI rendering, and token exchange utilities. Use as-is.

### 5.5 `wrangler.jsonc` Configuration

```jsonc
{
  "name": "leadgen-mcp-oauth",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-10",
  "compatibility_flags": ["nodejs_compat"],
  "kv_namespaces": [
    {
      "binding": "OAUTH_KV",
      "id": "<YOUR_KV_NAMESPACE_ID>"  // create via: npx wrangler kv namespace create "OAUTH_KV"
    }
  ],
  // Required by the OAuthProvider library for state management
  "durable_objects": {
    "bindings": [
      {
        "class_name": "OAuthStateObject",
        "name": "MCP_OBJECT"
      }
    ]
  },
  "migrations": [
    { "new_sqlite_classes": ["OAuthStateObject"], "tag": "v1" }
  ],
  // Variables (non-secret)
  "vars": {
    "MCP_UPSTREAM_URL": "https://mcp-internal.yourdomain.com"
  },
  "secrets": {
    "required": [
      "ACCESS_CLIENT_ID",
      "ACCESS_CLIENT_SECRET",
      "ACCESS_TOKEN_URL",
      "ACCESS_AUTHORIZATION_URL",
      "ACCESS_JWKS_URL",
      "COOKIE_ENCRYPTION_KEY"
    ]
  }
}
```

### 5.6 Deploy the Worker & Set Secrets

```bash
# Create KV namespace
npx wrangler kv namespace create "OAUTH_KV"
# → Copy the ID into wrangler.jsonc

# Set secrets (you'll be prompted for values)
npx wrangler secret put ACCESS_CLIENT_ID
npx wrangler secret put ACCESS_CLIENT_SECRET
npx wrangler secret put ACCESS_TOKEN_URL
npx wrangler secret put ACCESS_AUTHORIZATION_URL
npx wrangler secret put ACCESS_JWKS_URL

# Generate and set cookie encryption key
openssl rand -hex 32
npx wrangler secret put COOKIE_ENCRYPTION_KEY

# Deploy
npx wrangler deploy
```

Your Worker will be live at: `https://leadgen-mcp-oauth.<your-subdomain>.workers.dev`

---

## 6. MCP Server Changes

Your Python MCP server (`mcp_server/`) needs changes to:
1. Validate incoming JWTs (from the Worker proxy)
2. Extract user identity
3. Forward the JWT to the Laravel API (instead of the hardcoded `API_TOKEN`)

### 6.1 New Dependencies

Add to `requirements.txt`:

```
PyJWT>=2.8.0
cryptography>=41.0.0
```

### 6.2 New Config Settings

**File: `mcp_server/config.py`**

Add these fields to the `Settings` class:

```python
class Settings(BaseSettings):
    # ... existing fields ...

    # ----- Cloudflare Access auth -----
    # Team domain from CF Zero Trust (e.g. "yourteam.cloudflareaccess.com")
    cf_team_domain: str = ""
    # JWKS URL for token verification
    # (from the Access for SaaS app)
    cf_jwks_url: str = ""
    # Application Audience (AUD) tag — unique per CF Access Application
    cf_aud_tag: str = ""
    # Toggle auth on/off (off for local stdio dev, on for deployed HTTP)
    auth_enabled: bool = False
```

**Corresponding env vars to add in `.env`:**

```env
CF_TEAM_DOMAIN=yourteam.cloudflareaccess.com
CF_JWKS_URL=https://yourteam.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/jwks
CF_AUD_TAG=<aud-tag-from-cf-saas-app>
AUTH_ENABLED=true
```

### 6.3 New File: `mcp_server/auth.py`

Handles JWT validation and user identity extraction.

```python
"""
Cloudflare Access JWT validation.

Fetches CF's JWKS (public keys), verifies the JWT signature (RS256),
checks aud/exp/iss claims, and returns the user's identity.

The JWKS is cached in memory for 1 hour.
"""

from __future__ import annotations

import contextvars
import logging
import time
from dataclasses import dataclass, field
from typing import Any

import httpx
import jwt as pyjwt  # PyJWT

from .config import Settings

logger = logging.getLogger(__name__)

_jwks_cache: dict[str, Any] = {}
_jwks_cache_ts: float = 0
_JWKS_CACHE_TTL = 3600


@dataclass
class CFIdentity:
    email: str
    sub: str
    groups: list[str] = field(default_factory=list)
    raw_token: str = ""


# Contextvar to pass identity from middleware to MCP tool handlers
# without changing FastMCP's internal signatures
current_identity: contextvars.ContextVar[CFIdentity | None] = contextvars.ContextVar(
    "current_identity", default=None
)


def _get_jwks(jwks_url: str) -> dict[str, Any]:
    global _jwks_cache, _jwks_cache_ts
    now = time.time()
    if _jwks_cache and (now - _jwks_cache_ts) < _JWKS_CACHE_TTL:
        return _jwks_cache
    logger.debug("Fetching JWKS from %s", jwks_url)
    resp = httpx.get(jwks_url, timeout=10)
    resp.raise_for_status()
    _jwks_cache = resp.json()
    _jwks_cache_ts = now
    return _jwks_cache


def validate_cf_token(token: str, settings: Settings) -> CFIdentity:
    """
    Validate a CF Access JWT. Raises ValueError on failure.
    """
    jwks_data = _get_jwks(settings.cf_jwks_url)

    unverified_header = pyjwt.get_unverified_header(token)
    kid = unverified_header.get("kid")

    public_key = None
    for key_data in jwks_data.get("keys", []):
        if key_data.get("kid") == kid:
            public_key = pyjwt.algorithms.RSAAlgorithm.from_jwk(key_data)
            break

    if public_key is None:
        raise ValueError(f"No matching public key for kid={kid}")

    payload = pyjwt.decode(
        token,
        public_key,
        algorithms=["RS256"],
        audience=settings.cf_aud_tag,
        options={"verify_iss": False},  # CF SaaS apps may use different issuers
    )

    return CFIdentity(
        email=payload.get("email", ""),
        sub=payload.get("sub", ""),
        groups=payload.get("custom", {}).get("groups", []),
        raw_token=token,
    )
```

### 6.4 New File: `mcp_server/middleware.py`

ASGI middleware that intercepts every HTTP request, validates the JWT, and sets the `current_identity` contextvar so tool handlers can read the user's identity.

```python
"""
Auth middleware for the MCP HTTP server.

Extracts the CF Access JWT from:
  - Authorization: Bearer <token>  (forwarded by the OAuth shim Worker)
  - Cf-Access-Jwt-Assertion header (alternative CF header)

Skips auth for health checks. When AUTH_ENABLED=false (local stdio dev),
no auth is enforced and the middleware is not even mounted.
"""

from __future__ import annotations

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

from .auth import current_identity, validate_cf_token
from .config import Settings


class CFAuthMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, settings: Settings):
        super().__init__(app)
        self.settings = settings

    async def dispatch(self, request: Request, call_next):
        # Skip auth for health-check probes
        if request.url.path in ("/health", "/healthz", "/ready"):
            return await call_next(request)

        # Extract token — try Authorization header first, then CF-specific header
        token = None
        auth_header = request.headers.get("authorization", "")
        if auth_header.lower().startswith("bearer "):
            token = auth_header[7:]
        if not token:
            token = request.headers.get("cf-access-jwt-assertion")

        if not token:
            return JSONResponse(
                status_code=401,
                content={"error": "Authentication required"},
            )

        try:
            identity = validate_cf_token(token, self.settings)
        except Exception as exc:
            return JSONResponse(
                status_code=401,
                content={"error": f"Invalid token: {exc}"},
            )

        # Set identity in contextvar for the duration of this request
        # so MCP tool handlers can call current_identity.get() to read it
        ctx_token = current_identity.set(identity)
        try:
            return await call_next(request)
        finally:
            current_identity.reset(ctx_token)
```

### 6.5 Changes to `mcp_server/__main__.py`

The key change here: when running in HTTP (SSE) mode, `mcp.streamable_http_app()` returns a standard ASGI app. We wrap it with `CFAuthMiddleware` before handing it to uvicorn. stdio mode is **completely unaffected** — it has no HTTP requests, so there's nothing to intercept.

```python
import os
import uvicorn
from .server import mcp
from .config import settings

transport = os.environ.get("MCP_TRANSPORT", "stdio").lower()
host = os.environ.get("MCP_HOST", "0.0.0.0")
port = int(os.environ.get("MCP_PORT", "8080"))

if transport == "sse":
    app = mcp.streamable_http_app()

    # Wrap with auth middleware only when auth is enabled (production HTTP mode).
    # stdio mode never reaches this branch — no HTTP, no middleware needed.
    if settings.auth_enabled:
        from .middleware import CFAuthMiddleware
        app = CFAuthMiddleware(app, settings)

    uvicorn.run(app, host=host, port=port)
else:
    mcp.run(transport="stdio")
```

### 6.6 Changes to `mcp_server/api.py`

Thread the user's per-request JWT through all API calls instead of the hardcoded `API_TOKEN`.

**Change `_headers()` to accept an optional user token:**

```python
def _headers(settings: Settings, user_token: str | None = None) -> dict[str, str]:
    headers: dict[str, str] = {
        "Accept": "application/json",
        "Content-Type": "application/json",
    }
    if user_token:
        # Per-user token from CF Access — forwarded to Laravel so it sees
        # the real user identity and can apply RBAC
        headers["Authorization"] = f"Bearer {user_token}"
        headers["Cf-Access-Jwt-Assertion"] = user_token
    elif settings.api_token:
        # Fallback for local dev / stdio mode with AUTH_ENABLED=false
        headers["Authorization"] = f"Bearer {settings.api_token}"
    return headers
```

**Add `user_token` parameter to every API function:**

```python
def get_offer(settings: Settings, offer_unique_id: str, user_token: str | None = None) -> dict[str, Any]:
    url = f"{settings.api_base_url}/api/admin/offer/get-offer/{offer_unique_id}"
    ...
    resp = client.get(url, headers=_headers(settings, user_token))
    ...

def list_offers(settings: Settings, ..., user_token: str | None = None) -> dict[str, Any]:
    ...
    resp = client.get(url, headers=_headers(settings, user_token), params=params)
    ...

def save_draft(settings: Settings, body: dict[str, Any], user_token: str | None = None) -> dict[str, Any]:
    ...
    resp = client.post(url, headers=_headers(settings, user_token), json=body)
    ...

def update_offer_direct(settings: Settings, offer_type: str, offer_data: dict[str, Any], user_token: str | None = None) -> dict[str, Any]:
    ...
    resp = client.put(url, headers=_headers(settings, user_token), json=payload)
    ...

def promote_offer_draft(settings: Settings, offer_type: str, offer_unique_id: str, user_token: str | None = None) -> dict[str, Any]:
    ...
    resp = client.put(url, headers=_headers(settings, user_token), json={"offerUniqueId": offer_unique_id})
    ...
```

### 6.7 Changes to `mcp_server/server.py`

Read the authenticated user's token from the contextvar and pass it to every API call.

**Add a helper at the top of the file:**

```python
from .auth import current_identity

def _get_user_token() -> str | None:
    """Get the authenticated user's CF JWT for the current request."""
    identity = current_identity.get()
    return identity.raw_token if identity else None
```

**Update every tool to pass the token:**

```python
@mcp.tool(name="list_offers", ...)
async def list_offers(...) -> dict[str, Any]:
    return api.list_offers(
        settings,
        offer_type=offer_type or None,
        sub_type=sub_type or None,
        include_child_offers=include_child_offers,
        include_offer_id=include_offer_id,
        user_token=_get_user_token(),
    )

@mcp.tool(name="get_offer", ...)
async def get_offer(offer_unique_id: str, ...) -> dict[str, Any]:
    raw = api.get_offer(settings, offer_unique_id, user_token=_get_user_token())
    ...

@mcp.tool(name="update_offer", ...)
async def update_offer(offer_unique_id: str, ...) -> dict[str, Any]:
    user_token = _get_user_token()
    raw = api.get_offer(settings, offer_unique_id, user_token=user_token)
    ...
    result = api.update_offer_direct(settings, offer_type, data, user_token=user_token)
    ...
    draft_result = api.save_draft(settings, {"data": data}, user_token=user_token)
    ...

@mcp.tool(name="confirm_offer_update", ...)
async def confirm_offer_update(offer_unique_id: str, ...) -> dict[str, Any]:
    user_token = _get_user_token()
    ...
    raw = api.get_offer(settings, offer_unique_id, user_token=user_token)
    ...
    result = api.promote_offer_draft(settings, offer_type, offer_unique_id, user_token=user_token)
    ...
```

---

## 7. Laravel Backend Changes

### 7.1 Remove the MCP Middleware Bypass

Since the MCP server now forwards real user CF JWTs, the Laravel API will receive valid CF tokens on every request. Remove whatever bypass you added for the MCP server.

The CF Access middleware should now validate the `Cf-Access-Jwt-Assertion` header (or `Authorization: Bearer`) on all `/api/admin/*` routes, including those called by the MCP server.

### 7.2 Verify the Token Claims

The JWT forwarded by the MCP server contains the same claims that CF Access sends for browser users:
- `email` — the user's email
- `sub` — CF user UUID
- `aud` — the Application AUD tag

Your existing middleware should handle this without changes, since the JWT is issued by the same CF Access SaaS application.

### 7.3 Audit Logs

Activity logs will now show the actual user's email instead of a shared service account. The `activityLog` entries created by the MCP server's `update_offer` tool will be associated with the authenticated user's identity.

---

## 8. Client Configuration

### 8.1 Cursor — Remote MCP via Worker

```json
// .cursor/mcp.json
{
  "mcpServers": {
    "leadgen-ops": {
      "url": "https://leadgen-mcp-oauth.<subdomain>.workers.dev/mcp"
    }
  }
}
```

When you connect:
1. Cursor sends `POST /mcp` → gets `401`
2. Cursor discovers OAuth endpoints from `/.well-known/oauth-authorization-server`
3. Browser opens → CF Access login page
4. You log in with your SSO
5. Cursor receives the token and stores it
6. All subsequent tool calls include `Authorization: Bearer <token>`

> **Note on Cursor support:** Cursor's OAuth support for remote MCP servers is still maturing. If it doesn't handle the 401 → OAuth flow automatically, you can temporarily set the token manually via headers or use stdio mode for local dev. Check [Cursor docs](https://docs.cursor.com) for the latest MCP auth support.

### 8.2 Claude Desktop — Remote MCP via Worker

```json
// Claude Desktop settings
{
  "mcpServers": {
    "leadgen-ops": {
      "url": "https://leadgen-mcp-oauth.<subdomain>.workers.dev/mcp"
    }
  }
}
```

Claude Desktop supports the MCP OAuth spec natively. The flow is identical to Cursor.

### 8.3 stdio Mode — Local Development (No Auth)

For local development, continue using stdio transport. No auth is enforced.

```json
// .cursor/mcp.json — local dev
{
  "mcpServers": {
    "leadgen-ops": {
      "command": "/path/to/.venv/bin/python",
      "args": ["-m", "mcp_server"],
      "env": {
        "API_BASE_URL": "http://127.0.0.1:8000",
        "API_TOKEN": "your-local-dev-token",
        "AUTH_ENABLED": "false"
      }
    }
  }
}
```

### 8.4 Other MCP Clients

Any MCP client that implements the [MCP authorization spec](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/authorization/) will work. The OAuth shim is client-agnostic.

---

## 9. File Changes Summary

### New Files

| File | Description |
|------|------------|
| `mcp_server/auth.py` | CF JWT validation, JWKS fetching, `CFIdentity` dataclass, `current_identity` contextvar |
| `mcp_server/middleware.py` | ASGI auth middleware — extracts and validates JWT from headers |
| `leadgen-mcp-oauth-worker/` (separate repo) | Cloudflare Worker — OAuth shim between MCP clients and your MCP server |

### Modified Files

| File | What Changes |
|------|-------------|
| `mcp_server/config.py` | Add `cf_team_domain`, `cf_jwks_url`, `cf_aud_tag`, `auth_enabled` settings |
| `mcp_server/api.py` | `_headers()` accepts `user_token`; all API functions accept optional `user_token` |
| `mcp_server/server.py` | Import `current_identity`; add `_get_user_token()` helper; pass token to all API calls |
| `mcp_server/__main__.py` | Wrap ASGI app with `CFAuthMiddleware` when `auth_enabled=True` (HTTP mode only) |
| `requirements.txt` | Add `PyJWT>=2.8.0` and `cryptography>=41.0.0` |
| `.env` | Add `CF_TEAM_DOMAIN`, `CF_JWKS_URL`, `CF_AUD_TAG`, `AUTH_ENABLED` |

### No Changes Needed

| File | Why |
|------|-----|
| `Dockerfile` | New dependencies come via `requirements.txt` automatically |
| `mcp_server/offer_filter.py` | Pure logic, no HTTP/auth concerns |
| `client.py` | CLI agent — not used in production |

---

## 10. Deployment Checklist

### Phase 1: Deploy the OAuth Shim Worker

- [ ] Create the "Access for SaaS" application in CF Zero Trust (Section 4.1)
- [ ] Note down: Client ID, Client Secret, Token URL, Authorization URL, JWKS URL
- [ ] Create the Worker project from the CF template
- [ ] Customize `index.ts` to proxy to your MCP server (Section 5.4)
- [ ] Create KV namespace (`OAUTH_KV`)
- [ ] Set all 6 Worker secrets
- [ ] Deploy with `npx wrangler deploy`
- [ ] Verify the Worker is accessible at `https://leadgen-mcp-oauth.<subdomain>.workers.dev`
- [ ] Test `GET /.well-known/oauth-authorization-server` returns valid JSON

### Phase 2: Update the MCP Server

- [ ] Add `PyJWT` and `cryptography` to `requirements.txt`
- [ ] Create `mcp_server/auth.py` (Section 6.3)
- [ ] Create `mcp_server/middleware.py` (Section 6.4)
- [ ] Update `mcp_server/config.py` (Section 6.2)
- [ ] Update `mcp_server/__main__.py` — wrap app with middleware (Section 6.5)
- [ ] Update `mcp_server/api.py` — add `user_token` to all functions (Section 6.6)
- [ ] Update `mcp_server/server.py` — read `current_identity`, pass token (Section 6.7)
- [ ] Set env vars: `CF_TEAM_DOMAIN`, `CF_JWKS_URL`, `CF_AUD_TAG`, `AUTH_ENABLED=true`
- [ ] Rebuild and deploy Docker image

### Phase 3: Update Laravel Backend

- [ ] Remove the CF Access middleware bypass for MCP routes
- [ ] Verify Laravel's CF middleware validates the `Cf-Access-Jwt-Assertion` header
- [ ] Deploy Laravel changes

### Phase 4: Client Migration

- [ ] Update `.cursor/mcp.json` to point to the Worker URL (`/mcp`)
- [ ] Test with Cursor — verify OAuth flow, tool calls, and user identity
- [ ] Test with Claude Desktop — verify same
- [ ] Verify local stdio mode still works with `AUTH_ENABLED=false`

---

## 11. Testing Plan

### 11.1 Unit Tests — `auth.py`

```python
# Verify valid token → CFIdentity returned
# Verify expired token → ValueError
# Verify wrong audience → ValueError
# Verify unknown kid → ValueError
# Verify JWKS caching (no repeated HTTP calls within TTL)
```

### 11.2 Integration Tests — Worker

| Test | Expected Result |
|------|----------------|
| `GET /.well-known/oauth-authorization-server` | 200 with valid OAuth metadata JSON |
| `POST /mcp` (no token) | 401 with `WWW-Authenticate` header |
| `GET /authorize` | 302 redirect to CF Access login |
| `GET /callback?code=valid` | Exchanges code, issues MCP token, redirects to client |
| `POST /mcp` (with valid token) | Proxied to MCP server, tool response returned |

### 11.3 End-to-End Tests

| Scenario | Steps | Expected |
|----------|-------|----------|
| First-time connect (Cursor) | Add Worker URL to `.cursor/mcp.json`, connect | Browser opens CF login, after login tools appear |
| Tool call after auth | Call `list_offers` | Returns offer list; Laravel logs show your email |
| Token expiry | Wait for token to expire, call a tool | Re-auth triggered automatically |
| Unauthorized user | User not in CF Access policy tries to connect | Blocked at CF Access login (never reaches MCP) |
| stdio mode | Run locally with `AUTH_ENABLED=false` | Works exactly as before, no auth prompts |

---

## 12. FAQ

### Q: Why a separate Cloudflare Worker instead of building OAuth into the Python MCP server?

The MCP OAuth spec requires endpoints (`/authorize`, `/token`, `/register`) that handle browser redirects, PKCE, CSRF protection, and state management. Cloudflare provides a battle-tested [`workers-oauth-provider`](https://github.com/cloudflare/workers-oauth-provider) library that implements all of this. Building it from scratch in Python would be significant work and harder to maintain.

The Worker also gives you:
- Edge-level DDoS protection
- CF's global network (low latency)
- Serverless scaling
- Clean separation of concerns (auth vs business logic)

### Q: Do I need to change anything on the MCP protocol level?

No. The OAuth shim handles all the MCP OAuth protocol details. Your MCP server just receives authenticated HTTP requests with a JWT in the `Authorization` header — no MCP protocol changes.

### Q: What happens when the token expires?

MCP clients that support OAuth (Claude Desktop, Cursor) will automatically detect the 401 and re-trigger the OAuth flow. If you enabled refresh tokens in the CF Access SaaS app, the Worker can refresh tokens without requiring the user to log in again.

### Q: Can I restrict which tools are available to which users?

Yes. In `server.py`, you can read `current_identity.get()` to check the user's email or groups, and return permission errors. Example:

```python
from .auth import current_identity

@mcp.tool(name="update_offer", ...)
async def update_offer(...) -> dict[str, Any]:
    identity = current_identity.get()
    if identity and identity.email not in ALLOWED_WRITE_USERS:
        return {"success": False, "error": "You don't have write access"}
    ...
```

### Q: What about the `API_TOKEN` in `.env` and `config.py`?

Keep it for **local development only** (stdio mode with `AUTH_ENABLED=false`). In production, set `AUTH_ENABLED=true` and leave `API_TOKEN` empty — the user's CF JWT replaces it.

### Q: Will this work if my MCP server is not on Cloudflare's network?

Yes. The Worker proxies requests to your MCP server's internal URL (set via `MCP_UPSTREAM_URL`). Your MCP server can be on any infrastructure (K8s, EC2, on-prem) as long as the Worker can reach it over HTTPS.

---

## References

- [Cloudflare: Secure MCP servers with Access for SaaS](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/saas-mcp/)
- [Cloudflare: MCP Authorization](https://developers.cloudflare.com/agents/model-context-protocol/authorization/)
- [Cloudflare: `remote-mcp-cf-access` demo](https://github.com/cloudflare/ai/tree/main/demos/remote-mcp-cf-access)
- [Cloudflare: Workers OAuth Provider library](https://github.com/cloudflare/workers-oauth-provider)
- [MCP Spec: Authorization (2025-03-26)](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/authorization/)
