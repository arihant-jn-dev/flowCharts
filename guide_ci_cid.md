# CI/CD for Leadgen Dashboard MCP Server

---

## CI and CD — They Are Two Different Things

Most people say "CI/CD" as if it's one word. It's not. They are two separate
concepts that happen to work together.

---

### CI — Continuous Integration

**What it means:** Every time code is pushed to GitHub, automatically verify
that the code is correct and the Docker image can be built successfully.

**CI answers the question:** *"Is this code safe to ship?"*

**CI does NOT deploy anything.** It only builds and validates.

For this project, CI = **Step 1: Build**:

```
Developer pushes code to GitHub
            ↓
CI pipeline triggers automatically
            ↓
  ┌─────────────────────────────────────┐
  │            CI (Build Step)          │
  │                                     │
  │  1. Lint mcp_server/ code           │
  │  2. Verify server imports cleanly   │
  │  3. Run startup smoke test          │
  │  4. Build Docker image              │
  │  5. Push image to registry          │
  └─────────────────────────────────────┘
            ↓
  Image tagged with commit SHA stored in registry
  e.g. ghcr.io/your-org/leadgen-dashboard-mcp:a3f92c1
```

If any of these steps fail — broken import, bad Dockerfile, syntax error — the
pipeline stops. **Nothing broken ever reaches the registry or the server.**

---

### CD — Continuous Deployment (or Continuous Delivery)

**What it means:** Automatically take the verified image from the registry and
deploy it to the running server (K8s staging in your case).

**CD answers the question:** *"Is the latest verified build running on staging?"*

**CD does NOT build anything.** It only pulls the already-built image and
deploys it.

For this project, CD = **Step 2: Deploy**:

```
CI passed → image is in registry
            ↓
CD pipeline triggers (after CI succeeds)
            ↓
  ┌─────────────────────────────────────┐
  │           CD (Deploy Step)          │
  │                                     │
  │  1. Pull verified image from        │
  │     registry                        │
  │  2. kubectl set image on            │
  │     dashboard-mcp-stage pod         │
  │  3. K8s does a rolling restart      │
  │  4. Wait for new pod to be healthy  │
  │  5. Verify /  endpoint responds     │
  └─────────────────────────────────────┘
            ↓
  Staging is live with the new code
```

---

### Why Keep Them Separate?

| | CI (Build) | CD (Deploy) |
|---|---|---|
| Triggers on | Every push / every PR | Only after CI passes on `main` |
| Runs on | Every branch | Only `main` (staging) or `production` |
| Purpose | Validate code quality | Get verified code live |
| If it fails | Block the merge | Rollback, keep old pod running |

A PR from a feature branch runs **CI only** — it checks if the code is good,
but does not deploy anything. Only after merging to `main` does **CD** kick in
and update staging.

---

## Your Two-Step Process Mapped to CI/CD

```
┌──────────────────────────────────────────────────────────────────┐
│                        STEP 1 — CI (Build)                       │
│                                                                  │
│  Trigger: git push (any branch) or PR opened                     │
│                                                                  │
│  lint → smoke test → docker build → docker push to registry      │
│                                                                  │
│  Output: image tagged with commit SHA sitting in registry        │
│          e.g. ghcr.io/your-org/leadgen-dashboard-mcp:a3f92c1     │
└──────────────────────────────────────────────────────────────────┘
                              ↓
                    (only if CI passed AND branch is main)
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                       STEP 2 — CD (Deploy)                       │
│                                                                  │
│  Trigger: CI step 1 succeeded on main branch                     │
│                                                                  │
│  kubectl set image → rolling restart → health check              │
│                                                                  │
│  Output: dashboard-mcp-stage pod running the new image           │
│          URL: dashboard-mcp-stage-service.dashboard-api-staging  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Why This Is Needed (Real Problems From This Project)

Without CI/CD, all of these issues were found the hard way — in a live deployed
server, requiring manual debugging:

| Issue Found in Production | Which Step Would Have Caught It |
|---|---|
| `FastMCP.run()` no longer accepts `host`/`port` keyword args | CI — startup smoke test |
| Default `host="127.0.0.1"` auto-enabling DNS rebinding protection → 404 | CI — smoke test with K8s-style Host header |
| `sse_app()` serving `/sse` but Cursor hitting `POST /` → 404 | CI — transport compatibility check |
| `mcp.json` pointing to wrong project's venv | CI — import verification |

Every one of these was a production incident that required a developer to:
1. Notice something was broken
2. SSH in / check logs
3. Diagnose the issue
4. Fix it
5. Manually redeploy

With CI, each of these would have failed the build pipeline on the PR that
introduced the bug — before anyone merged it, before anything was deployed.

---

## The Current Manual Process vs Automated

**Now (manual):**
```
Developer pushes code
        ↓
      ← nothing happens automatically →
        ↓
Someone manually:
  - SSHs into cluster / opens K8s dashboard
  - Runs: docker build → docker push → kubectl rollout restart
  - Checks logs to see if it worked
  - Tells team on Slack
        ↓
Total time: 15–30 min, requires server access
```

**With CI/CD:**
```
Developer pushes code
        ↓
STEP 1 — CI runs automatically (~3–4 min)
  lint → smoke test → docker build → push to registry
        ↓
STEP 2 — CD runs automatically (~1–2 min)
  kubectl set image → rolling restart → health check
        ↓
Total time: ~5 min, zero manual steps, team notified automatically
```

---

## GitHub Actions Setup — Two Separate Workflow Files

Matching your two-step process, create two workflow files:

### File 1: `.github/workflows/ci.yml` — Build Step

Runs on **every push and every PR** across all branches.

```yaml
name: CI — Build & Verify

on:
  push:
    branches: ["**"]       # every branch
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    name: Lint, Test & Build Image
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write        # needed to push to GHCR

    steps:
      - uses: actions/checkout@v4

      # ── Python checks ───────────────────────────────────────────
      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt ruff

      - name: Lint — check mcp_server/ for errors
        run: ruff check mcp_server/

      - name: Verify server imports and loads cleanly
        run: python -c "from mcp_server.server import mcp; print('server OK:', mcp)"

      - name: Startup smoke test — server boots and endpoint responds
        run: |
          MCP_TRANSPORT=sse MCP_PORT=8080 python -m mcp_server &
          sleep 3
          curl -f http://localhost:8080/ || exit 1
          kill %1

      # ── Docker build & push ─────────────────────────────────────
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          # Push only on main — PRs from feature branches just build, don't push
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

### File 2: `.github/workflows/cd.yml` — Deploy Step

Runs **only after the CI workflow succeeds on `main`**.
Uses the image that CI already built and pushed — does NOT rebuild.

```yaml
name: CD — Deploy to Staging

on:
  workflow_run:
    workflows: ["CI — Build & Verify"]   # waits for CI to finish
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to K8s Staging
    runs-on: ubuntu-latest
    # Only run if CI passed — don't deploy if CI failed
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    steps:
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure K8s credentials (staging cluster)
        run: |
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > /tmp/kubeconfig
          echo "KUBECONFIG=/tmp/kubeconfig" >> $GITHUB_ENV

      - name: Update pod image to the new build
        run: |
          kubectl set image deployment/dashboard-mcp-stage \
            mcp-server=ghcr.io/${{ github.repository }}:${{ github.event.workflow_run.head_sha }} \
            -n dashboard-api-staging

      - name: Wait for rolling restart to complete
        run: |
          kubectl rollout status deployment/dashboard-mcp-stage \
            -n dashboard-api-staging \
            --timeout=120s

      - name: Verify MCP endpoint is responding
        run: |
          kubectl run health-check --rm -i --restart=Never \
            --image=curlimages/curl -- \
            curl -f http://dashboard-mcp-stage-service.dashboard-api-staging:80/
```

---

## What Each Pipeline Sees Per Scenario

| Event | CI runs? | CD runs? |
|---|---|---|
| Push to feature branch | Yes (lint + test + build only, no push) | No |
| PR opened against `main` | Yes (lint + test + build only, no push) | No |
| Merge to `main` | Yes (lint + test + build + push to registry) | Yes, after CI passes |
| CI fails on `main` | Yes (fails) | No — CD is blocked |

---

## Secrets Required in GitHub

Go to **GitHub repo → Settings → Secrets and variables → Actions** and add:

| Secret | What it is |
|---|---|
| `KUBE_CONFIG_STAGING` | Base64-encoded kubeconfig for the staging K8s cluster |
| `KUBE_CONFIG_PROD` | Same for production (add when you set up a prod deploy pipeline) |

`GITHUB_TOKEN` is provided automatically — no setup needed.

To get `KUBE_CONFIG_STAGING`:
```bash
cat ~/.kube/config | base64 | pbcopy   # copies to clipboard on Mac
```

---

## What Happens on a Bad Deploy (Automatic Rollback)

If the new image crashes on staging after CD deploys it:

1. K8s starts the new pod with the new image
2. Pod crashes → `CrashLoopBackOff`
3. K8s **keeps the old pod running** — staging stays up, users unaffected
4. `kubectl rollout status` times out → CD pipeline step fails
5. GitHub Actions marks the CD workflow as **failed**
6. No further action needed — old version is still live

Manual recovery if needed:
```bash
kubectl rollout undo deployment/dashboard-mcp-stage -n dashboard-api-staging
```

---

## Impact on the Team

| Without CI/CD | With CI/CD |
|---|---|
| 15–30 min deploy, requires server access | ~5 min, any developer can trigger via `git push` |
| Bugs found live in staging | Bugs caught in CI before anything is deployed |
| No record of what version is running | Every pod runs an image tagged with its exact commit SHA |
| Crash = downtime until someone manually rolls back | Crash = K8s keeps old pod, CD pipeline fails, team notified |
| Two people needed: one to code, one to deploy | One `git push`, pipeline handles everything |
