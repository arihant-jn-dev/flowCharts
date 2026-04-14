# OAuth Implementation Plan — MCP + Cloudflare Access

## Table of Contents

1. [Current State](#1-current-state)
2. [Goal](#2-goal)
3. [Architecture Overview](#3-architecture-overview)
4. [How Cloudflare Access OAuth Works](#4-how-cloudflare-access-oauth-works)
5. [Component Breakdown](#5-component-breakdown)
6. [Step-by-Step Implementation](#6-step-by-step-implementation)
7. [File Changes Summary](#7-file-changes-summary)
8. [Client Configuration (Cursor & Claude)](#8-client-configuration-cursor--claude)
9. [Deployment Changes](#9-deployment-changes)
10. [Testing Checklist](#10-testing-checklist)

---

## 1. Current State

```
Cursor / Claude
      │
      │  MCP (stdio or Streamable HTTP)
      ▼
┌──────────────┐         ┌──────────────────┐
│  MCP Server  │──HTTP──▶│  Laravel API      │
│  (FastMCP)   │         │  /api/admin/*     │
└──────────────┘         └──────────────────┘
```

**How auth works today:**

| Layer | What happens |
|-------|-------------|
| MCP → Laravel API | A hardcoded `API_TOKEN` (Laravel Passport JWT) is sent as `Bearer` header from `api.py:_headers()` |
| MCP Server itself | **No authentication at all** — anyone who can reach the HTTP port can call any tool |
| Laravel middleware | You are **bypassing** the CF Access middleware to allow the MCP server through |

**Problems:**

- The MCP server is an open door — no user identity, no RBAC
- The `API_TOKEN` is a single shared credential; you can't tell which user made a change
- Bypassing CF middleware defeats the security model

---

## 2. Goal

```
Cursor / Claude
      │
      │  1. Connect to MCP server URL
      │  2. MCP returns 401 → triggers OAuth flow
      │  3. User logs in via Cloudflare Access (SSO/Google/GitHub)
      │  4. Client receives JWT, stores it
      │  5. Every tool call includes Bearer <CF_JWT>
      │
      ▼
┌──────────────┐         ┌──────────────────┐
│  MCP Server  │──HTTP──▶│  Laravel API      │
│  (FastMCP)   │         │  /api/admin/*     │
│              │         │                  │
│  • Validates │         │  • Validates same │
│    CF JWT    │         │    CF JWT         │
│  • Forwards  │         │  • RBAC by user   │
│    JWT as    │         │    identity       │
│    Bearer    │         │                  │
└──────────────┘         └──────────────────┘
```

**After implementation:**

- Every MCP user authenticates via Cloudflare Access (same SSO your team already uses)
- The MCP server validates the CF JWT on every request
- The same JWT is forwarded to the Laravel API — no more hardcoded `API_TOKEN`
- Laravel's existing CF middleware works as-is (no more bypass)
- Per-user identity flows end-to-end (audit logs show who did what)

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     FULL AUTH FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Cursor / Claude Desktop                                    │
│       │                                                     │
│       │ ① POST / (tool call, no token)                      │
│       ▼                                                     │
│  ┌─────────────────────┐                                    │
│  │   MCP Server        │                                    │
│  │   (your FastMCP)    │                                    │
│  │                     │                                    │
│  │  Returns 401 +      │                                    │
│  │  WWW-Authenticate   │                                    │
│  │  header pointing to │                                    │
│  │  OAuth metadata     │                                    │
│  └────────┬────────────┘                                    │
│           │                                                 │
│       │ ② Client discovers /.well-known/oauth-authorization-│
│       │    server (RFC 8414) on MCP server                  │
│       ▼                                                     │
│  ┌─────────────────────┐                                    │
│  │  OAuth Shim         │  (Cloudflare Worker or             │
│  │                     │   built into MCP server)           │
│  │  /authorize         │──③──▶ Cloudflare Access login page │
│  │  /token             │◀─④── CF returns signed JWT         │
│  │  /register          │       (contains user email, groups)│
│  └─────────────────────┘                                    │
│           │                                                 │
│       │ ⑤ Client stores JWT, attaches as Bearer on every    │
│       │    subsequent MCP request                           │
│       ▼                                                     │
│  ┌─────────────────────┐                                    │
│  │   MCP Server        │                                    │
│  │                     │                                    │
│  │  ⑥ Validates JWT    │                                    │
│  │    against CF JWKS  │                                    │
│  │                     │                                    │
│  │  ⑦ Forwards same    │                                    │
│  │    JWT as Bearer    │──────▶ Laravel API                  │
│  │    to backend       │       (validates CF token natively) │
│  └─────────────────────┘                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. How Cloudflare Access OAuth Works

### 4.1 What You Already Have

Your team uses **Cloudflare Access** as the identity layer. Users sign in via an Identity Provider (IdP) — Google Workspace, GitHub, Okta, etc. — and CF Access issues a signed JWT.

### 4.2 Key CF Access Concepts

| Concept | What it is | Where to find it |
|---------|-----------|------------------|
| **Team Domain** | `<your-team>.cloudflareaccess.com` | CF Zero Trust dashboard → Settings |
| **Application** | A CF Access "application" protects a hostname/path. You'll create one for the MCP server. | CF Zero Trust → Access → Applications |
| **Access Policy** | Rules defining who can access (e.g., emails ending in `@yourcompany.com`) | Attached to the Application |
| **AUD tag** | The `aud` claim in the JWT — unique per CF Application. Used to validate tokens. | Application settings → Overview |
| **JWKS endpoint** | `https://<team-domain>/cdn-cgi/access/certs` — public keys to verify JWTs | Always at this URL |
| **CF_Authorization cookie / `Cf-Access-Jwt-Assertion` header** | How CF passes the JWT to your app after login | Automatic (CF injects it) |

### 4.3 The JWT You'll Receive

When a user logs in through CF Access, the JWT payload looks like:

```json
{
  "aud": ["<your-application-aud-tag>"],
  "email": "arihant@yourcompany.com",
  "exp": 1700000000,
  "iat": 1699990000,
  "iss": "https://<your-team>.cloudflareaccess.com",
  "sub": "user-uuid-from-cf",
  "identity_nonce": "...",
  "custom": {
    "groups": ["ops-team", "admin"]
  }
}
```

---

## 5. Component Breakdown

### 5.1 Approach A: CF Access Directly Protects MCP Server (Simpler)

If your MCP server is already behind Cloudflare (proxied via CF DNS), you can add a CF Access Application for the MCP server's hostname. CF will handle the login flow automatically — users see the CF login page before they can reach the MCP server.

**Pros:** No OAuth shim needed, simplest setup
**Cons:** Only works when MCP is accessed over HTTP (not stdio), requires the MCP domain to be CF-proxied

### 5.2 Approach B: OAuth Shim (MCP OAuth Spec-Compliant)

If you want MCP clients (Cursor, Claude) to handle the login via the MCP protocol's built-in OAuth support, you need an OAuth shim that translates between:

- MCP client's OAuth flow → Cloudflare Access login
- CF Access JWT → OAuth token response back to the MCP client

**Pros:** Works with any MCP client, follows the MCP spec
**Cons:** More code, need to host the shim (can be in the same server or a CF Worker)

### 5.3 Recommended: Approach A for Cursor, Approach B for Claude

| Client | Transport | Auth Approach |
|--------|-----------|---------------|
| **Cursor** (your team, internal) | Streamable HTTP behind CF | **Approach A** — CF Access protects the domain; Cursor sends `Cf-Access-Jwt-Assertion` header |
| **Claude Desktop** | Streamable HTTP (MCP remote) | **Approach B** — OAuth shim so Claude can do standard OAuth |
| **stdio** (local dev) | stdio pipe | No auth needed (localhost only) |

---

## 6. Step-by-Step Implementation

### Step 1: Set Up Cloudflare Access Application

**Where:** Cloudflare Zero Trust Dashboard

1. Go to **Zero Trust** → **Access** → **Applications** → **Add an Application**
2. Choose **Self-hosted**
3. Configure:
   - **Application name:** `Leadgen MCP Server`
   - **Session duration:** 24h (or as your team prefers)
   - **Application domain:** The hostname where your MCP server will be deployed (e.g., `mcp.yourdomain.com`)
4. Add an **Access Policy**:
   - **Policy name:** `Ops Team Access`
   - **Action:** Allow
   - **Include rule:** Emails ending in `@yourcompany.com` (or a CF Access Group)
5. After creating, note down:
   - **Application Audience (AUD) tag** — you'll need this in your MCP server config
   - **Team domain** — e.g., `yourteam.cloudflareaccess.com`

### Step 2: Add Auth Config to Settings

**File:** `mcp_server/config.py`

Add these new settings:

```python
class Settings(BaseSettings):
    # ... existing settings ...

    # ----- Cloudflare Access auth -----
    cf_team_domain: str = ""        # e.g. "yourteam.cloudflareaccess.com"
    cf_aud_tag: str = ""            # Application Audience (AUD) tag from CF dashboard
    auth_enabled: bool = False      # Set to True in production deployments
```

**Corresponding env vars to add in `.env`:**

```
CF_TEAM_DOMAIN=yourteam.cloudflareaccess.com
CF_AUD_TAG=abc123your-aud-tag-from-cf-dashboard
AUTH_ENABLED=true
```

### Step 3: Create JWT Validation Module

**New file:** `mcp_server/auth.py`

This module handles:
- Fetching CF's public keys (JWKS) to verify JWTs
- Validating the JWT signature, expiration, audience
- Extracting user identity (email, groups) from the token

```python
"""
Cloudflare Access JWT validation.

Validates JWTs issued by Cloudflare Access by:
  1. Fetching the JWKS (public keys) from CF's /cdn-cgi/access/certs endpoint
  2. Verifying the token signature (RS256)
  3. Checking aud, exp, iss claims
  4. Returning the user's email and identity
"""

from __future__ import annotations

import logging
import time
from dataclasses import dataclass
from typing import Any

import httpx
import jwt  # PyJWT

from .config import Settings

logger = logging.getLogger(__name__)

# Cache JWKS for 1 hour to avoid hitting CF on every request
_jwks_cache: dict[str, Any] = {}
_jwks_cache_ts: float = 0
JWKS_CACHE_TTL = 3600  # seconds


@dataclass
class CFIdentity:
    """Represents the authenticated Cloudflare Access user."""
    email: str
    sub: str           # CF user UUID
    groups: list[str]  # from custom.groups claim (if configured)
    raw_token: str     # original JWT string (forwarded to backend)


def _get_jwks(team_domain: str) -> dict[str, Any]:
    """Fetch and cache CF Access JWKS (public keys for JWT verification)."""
    global _jwks_cache, _jwks_cache_ts

    now = time.time()
    if _jwks_cache and (now - _jwks_cache_ts) < JWKS_CACHE_TTL:
        return _jwks_cache

    url = f"https://{team_domain}/cdn-cgi/access/certs"
    logger.debug("Fetching JWKS from %s", url)
    resp = httpx.get(url, timeout=10)
    resp.raise_for_status()
    _jwks_cache = resp.json()
    _jwks_cache_ts = now
    return _jwks_cache


def validate_cf_token(token: str, settings: Settings) -> CFIdentity:
    """
    Validate a Cloudflare Access JWT.

    Raises ValueError if the token is invalid, expired, or has wrong audience.
    """
    jwks = _get_jwks(settings.cf_team_domain)

    # Decode the header to find which key was used
    unverified_header = jwt.get_unverified_header(token)
    kid = unverified_header.get("kid")

    # Find the matching public key
    public_key = None
    for key_data in jwks.get("keys", []):
        if key_data["kid"] == kid:
            public_key = jwt.algorithms.RSAAlgorithm.from_jwk(key_data)
            break

    if public_key is None:
        raise ValueError(f"No matching key found for kid={kid}")

    # Verify and decode
    payload = jwt.decode(
        token,
        public_key,
        algorithms=["RS256"],
        audience=settings.cf_aud_tag,
        issuer=f"https://{settings.cf_team_domain}",
    )

    return CFIdentity(
        email=payload.get("email", ""),
        sub=payload.get("sub", ""),
        groups=payload.get("custom", {}).get("groups", []),
        raw_token=token,
    )
```

### Step 4: Add Auth Middleware to the MCP Server

**File:** `mcp_server/__main__.py`

Wrap the ASGI app with middleware that:
1. Checks for `Authorization: Bearer <token>` or `Cf-Access-Jwt-Assertion` header
2. Validates the JWT
3. Injects the user identity into the request state
4. Returns 401 if no valid token is found

```python
"""
Auth middleware — sits between the HTTP server and the MCP ASGI app.

Extracts the CF Access JWT from either:
  - Authorization: Bearer <token>  (MCP clients like Claude)
  - Cf-Access-Jwt-Assertion header (CF Access proxy — Cursor behind CF)

On failure, returns 401 with a WWW-Authenticate header so MCP clients
know to start the OAuth flow.
"""

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

from .auth import validate_cf_token, CFIdentity
from .config import settings

class CFAuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Skip auth for health checks, OAuth metadata, and well-known endpoints
        skip_paths = ["/health", "/.well-known/", "/oauth/"]
        if any(request.url.path.startswith(p) for p in skip_paths):
            return await call_next(request)

        token = None

        # Try Authorization header first (MCP OAuth flow)
        auth_header = request.headers.get("authorization", "")
        if auth_header.lower().startswith("bearer "):
            token = auth_header[7:]

        # Fall back to CF Access header (CF-proxied access)
        if not token:
            token = request.headers.get("cf-access-jwt-assertion")

        if not token:
            return JSONResponse(
                status_code=401,
                content={"error": "Authentication required"},
                headers={
                    "WWW-Authenticate": (
                        'Bearer resource_metadata='
                        f'"https://{request.url.hostname}/.well-known/oauth-authorization-server"'
                    )
                },
            )

        try:
            identity = validate_cf_token(token, settings)
        except Exception as exc:
            return JSONResponse(
                status_code=401,
                content={"error": f"Invalid token: {exc}"},
            )

        # Store identity on request state so MCP tools can access it
        request.state.cf_identity = identity
        return await call_next(request)
```

### Step 5: Forward User's JWT to Backend API

**File:** `mcp_server/api.py`

Change `_headers()` to accept an optional JWT token — the per-user CF token forwarded from the authenticated request instead of the hardcoded `API_TOKEN`.

```python
# BEFORE (current)
def _headers(settings: Settings) -> dict[str, str]:
    headers = {"Accept": "application/json", "Content-Type": "application/json"}
    if settings.api_token:
        headers["Authorization"] = f"Bearer {settings.api_token}"
    return headers


# AFTER (with per-user token forwarding)
def _headers(settings: Settings, user_token: str | None = None) -> dict[str, str]:
    headers = {"Accept": "application/json", "Content-Type": "application/json"}
    if user_token:
        # Forward the user's CF JWT to the backend (per-user identity)
        headers["Authorization"] = f"Bearer {user_token}"
        headers["Cf-Access-Jwt-Assertion"] = user_token
    elif settings.api_token:
        # Fallback for local dev / stdio mode with no auth
        headers["Authorization"] = f"Bearer {settings.api_token}"
    return headers
```

Then thread `user_token` through every API function:

```python
def get_offer(settings: Settings, offer_unique_id: str, user_token: str | None = None) -> dict:
    url = f"{settings.api_base_url}/api/admin/offer/get-offer/{offer_unique_id}"
    with httpx.Client(timeout=30) as client:
        resp = client.get(url, headers=_headers(settings, user_token))
        ...
```

### Step 6: Pass Identity from MCP Tools to API Layer

**File:** `mcp_server/server.py`

MCP tools need access to the current user's token. With FastMCP's Streamable HTTP transport, you can access the request context:

```python
from mcp.server.fastmcp import Context

@mcp.tool(name="list_offers", ...)
async def list_offers(
    ctx: Context,
    offer_type: str = "",
    ...
) -> dict[str, Any]:
    # Extract user token from the request context
    user_token = _get_user_token(ctx)
    return api.list_offers(settings, offer_type=..., user_token=user_token)
```

> **Note:** How exactly to extract the token from `ctx` depends on how FastMCP
> exposes the underlying HTTP request. You may need to store the identity in
> a contextvars-based store in the middleware and read it in the tool.
> See [Section 6.1](#61-contextvars-approach) below.

#### 6.1 Contextvars Approach

Since ASGI middleware and MCP tool handlers run in the same async context, you can use Python's `contextvars` to pass the identity without changing FastMCP internals.

**New file or add to `auth.py`:**

```python
import contextvars

# Holds the authenticated user's identity for the current request
current_identity: contextvars.ContextVar[CFIdentity | None] = contextvars.ContextVar(
    "current_identity", default=None
)
```

**In the middleware:** after validation, set the context var:

```python
from .auth import current_identity

# Inside CFAuthMiddleware.dispatch(), after validate_cf_token():
token_for_ctx = current_identity.set(identity)
response = await call_next(request)
current_identity.reset(token_for_ctx)
return response
```

**In `server.py`:** read the context var in tools:

```python
from .auth import current_identity

def _get_user_token() -> str | None:
    identity = current_identity.get()
    return identity.raw_token if identity else None

@mcp.tool(name="list_offers", ...)
async def list_offers(...) -> dict[str, Any]:
    return api.list_offers(settings, ..., user_token=_get_user_token())
```

### Step 7: OAuth Metadata & Shim Endpoints (for Claude / MCP OAuth spec)

For MCP clients that support the [MCP OAuth spec](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/authorization/), you need to serve OAuth discovery metadata and act as a thin OAuth shim that delegates to CF Access.

**Add to `__main__.py` (or a new `oauth.py` router):**

```python
from starlette.routing import Route
from starlette.responses import JSONResponse, RedirectResponse

# RFC 8414 — OAuth Authorization Server Metadata
async def oauth_metadata(request):
    base = str(request.base_url).rstrip("/")
    return JSONResponse({
        "issuer": base,
        "authorization_endpoint": f"{base}/oauth/authorize",
        "token_endpoint": f"{base}/oauth/token",
        "registration_endpoint": f"{base}/oauth/register",
        "response_types_supported": ["code"],
        "grant_types_supported": ["authorization_code"],
        "code_challenge_methods_supported": ["S256"],
    })

# Dynamic client registration (required by MCP spec)
async def oauth_register(request):
    body = await request.json()
    return JSONResponse({
        "client_id": "mcp-client",
        "client_secret": "",
        "redirect_uris": body.get("redirect_uris", []),
    })

# Authorization — redirect to CF Access login
async def oauth_authorize(request):
    redirect_uri = request.query_params.get("redirect_uri")
    state = request.query_params.get("state", "")
    # Redirect to CF Access; after login CF will redirect back
    cf_login_url = (
        f"https://{settings.cf_team_domain}/cdn-cgi/access/login/"
        f"{settings.cf_aud_tag}"
        f"?redirect_url={redirect_uri}%3Fstate%3D{state}"
    )
    return RedirectResponse(cf_login_url)

# Token exchange — CF Access token comes back; return it as OAuth token
async def oauth_token(request):
    form = await request.form()
    code = form.get("code", "")
    # In the CF Access flow, the "code" is the CF JWT itself
    return JSONResponse({
        "access_token": code,
        "token_type": "Bearer",
        "expires_in": 86400,
    })

oauth_routes = [
    Route("/.well-known/oauth-authorization-server", oauth_metadata),
    Route("/oauth/register", oauth_register, methods=["POST"]),
    Route("/oauth/authorize", oauth_authorize),
    Route("/oauth/token", oauth_token, methods=["POST"]),
]
```

> **Important:** The OAuth shim above is a simplified version. The exact redirect
> flow with Cloudflare Access may require a Cloudflare Worker to properly
> intercept the CF login callback and extract the JWT as the authorization code.
> See [Section 6.2](#62-cloudflare-worker-shim-alternative) if you want a
> production-grade version.

#### 6.2 Cloudflare Worker Shim (Alternative)

Instead of building the OAuth endpoints into the MCP server, you can deploy a Cloudflare Worker that:

1. Hosts the `/.well-known/oauth-authorization-server` metadata
2. Handles `/authorize` by redirecting to CF Access
3. Captures the CF JWT from the callback and returns it via `/token`

This is the approach from the diagram you shared. Cloudflare has an open-source template for this:
- [cloudflare/ai-connect-mcp-server-template](https://github.com/cloudflare/ai/tree/main/packages/workers-oauth-provider)

---

## 7. File Changes Summary

| File | Change Type | What Changes |
|------|------------|--------------|
| `mcp_server/config.py` | **Edit** | Add `cf_team_domain`, `cf_aud_tag`, `auth_enabled` settings |
| `mcp_server/auth.py` | **New** | JWT validation, JWKS fetching, `CFIdentity` dataclass, `current_identity` contextvar |
| `mcp_server/api.py` | **Edit** | `_headers()` accepts `user_token` param; all API functions accept optional `user_token` |
| `mcp_server/server.py` | **Edit** | Tools read `current_identity` contextvar and pass `user_token` to API calls |
| `mcp_server/__main__.py` | **Edit** | Add `CFAuthMiddleware` and OAuth routes when `AUTH_ENABLED=true` |
| `requirements.txt` | **Edit** | Add `PyJWT>=2.8.0` and `cryptography>=41.0.0` |
| `.env` | **Edit** | Add `CF_TEAM_DOMAIN`, `CF_AUD_TAG`, `AUTH_ENABLED` |
| `Dockerfile` | **Edit** | No changes needed (dependencies come via requirements.txt) |

---

## 8. Client Configuration (Cursor & Claude)

### 8.1 Cursor — Behind Cloudflare Access (Approach A)

When the MCP server is behind a CF-protected domain, the browser-based CF login happens once. After that, the CF cookie/token is available.

```json
// .cursor/mcp.json — each team member
{
  "mcpServers": {
    "leadgen-ops": {
      "url": "https://mcp.yourdomain.com/"
    }
  }
}
```

Cursor will hit the URL, get a 401, and prompt the user to authenticate. After CF Access login, Cursor stores the token and sends it on subsequent requests.

> **Note:** As of early 2025, Cursor's MCP OAuth support is still evolving.
> If Cursor doesn't handle the 401 → OAuth flow automatically, you can
> set the token manually via headers or use stdio mode for local dev.

### 8.2 Claude Desktop — MCP OAuth Flow (Approach B)

Claude Desktop supports the MCP OAuth spec natively. When it connects to your MCP server and gets a 401 with the `WWW-Authenticate` header, it will:

1. Discover the OAuth metadata at `/.well-known/oauth-authorization-server`
2. Register a client at `/oauth/register`
3. Open the browser for `/oauth/authorize` → user logs in via CF Access
4. Exchange the code at `/oauth/token`
5. Store the token and send `Authorization: Bearer <token>` on every tool call

```json
// Claude Desktop config
{
  "mcpServers": {
    "leadgen-ops": {
      "url": "https://mcp.yourdomain.com/"
    }
  }
}
```

### 8.3 stdio Mode — Local Dev (No Auth)

For local development, continue using stdio transport. No auth is needed because the process runs on your machine.

```json
// .cursor/mcp.json — local dev
{
  "mcpServers": {
    "leadgen-ops": {
      "command": "/path/to/.venv/bin/python",
      "args": ["-m", "mcp_server"],
      "env": {
        "API_BASE_URL": "http://127.0.0.1:8000",
        "API_TOKEN": "your-dev-token",
        "AUTH_ENABLED": "false"
      }
    }
  }
}
```

---

## 9. Deployment Changes

### 9.1 Environment Variables to Add in K8s / Docker

```yaml
# New env vars for production
CF_TEAM_DOMAIN: "yourteam.cloudflareaccess.com"
CF_AUD_TAG: "your-application-aud-tag"
AUTH_ENABLED: "true"

# Remove or leave empty — no longer needed with per-user tokens
API_TOKEN: ""
```

### 9.2 Cloudflare DNS & Access Setup

1. **DNS:** Point `mcp.yourdomain.com` → your K8s ingress/load balancer (orange cloud = proxied)
2. **CF Access Application:** Protect `mcp.yourdomain.com` with the Access Application from Step 1
3. **Access Policy:** Allow `@yourcompany.com` emails (or specific groups)
4. **Skip paths:** If using the OAuth shim endpoints, add a bypass rule for `/.well-known/*` and `/oauth/*` in the CF Access Application so unauthenticated clients can discover the OAuth metadata

### 9.3 Laravel Backend — Remove Bypass

Once the MCP server forwards the real CF JWT, you can **remove the middleware bypass** you added for the MCP server. The Laravel API will see valid CF tokens on every request, just as if a user was accessing the dashboard directly.

---

## 10. Testing Checklist

### Local Testing (before deploying)

- [ ] `AUTH_ENABLED=false` — verify existing stdio flow still works (no regression)
- [ ] `AUTH_ENABLED=true` with a manually obtained CF JWT — verify `auth.py` validates it correctly
- [ ] Verify 401 is returned when no token is sent to HTTP endpoint
- [ ] Verify `/.well-known/oauth-authorization-server` returns correct metadata
- [ ] Verify user's email is extractable from validated token

### Integration Testing

- [ ] Deploy to staging behind CF Access
- [ ] Open MCP URL in browser — CF login page should appear
- [ ] After login, MCP health check returns 200
- [ ] Connect Cursor to the CF-protected MCP URL — verify tools work
- [ ] Connect Claude Desktop — verify OAuth flow completes and tools work
- [ ] Verify Laravel API receives the CF JWT (check `Cf-Access-Jwt-Assertion` header in Laravel logs)
- [ ] Remove the middleware bypass on Laravel side and verify everything still works
- [ ] Verify activity logs in Laravel show the correct user email

### Security Checks

- [ ] Expired token → 401
- [ ] Token with wrong `aud` → 401
- [ ] Token signed by different key → 401
- [ ] Token for user not in Access Policy → blocked by CF before reaching MCP
- [ ] `API_TOKEN` no longer needed in production config
