# Configurable Loader System — Design Doc

Scope for v1: the **email-submit** step on lander/prelander pages. Built so it extends to
survey-complete, registration-submit, etc. later without a redesign.

---

## 1. What already exists (don't rebuild this, wrap it)

Two pieces of real, working infrastructure already do most of what's needed:

1. **A backend → JS config channel already exists and is heavily used.**
   `window.experiment_config` is server-rendered straight into the page (not fetched — it's
   already sitting in `window` by the time our JS runs) and is read everywhere with the
   `window.experiment_config?.someFlag || default` pattern — see
   [js/lander.js:23-45](../js/lander.js#L23-L45) for ~10 examples. This is the natural home for
   the new loader JSON. No new delivery mechanism is needed — just a new key on an object that
   already reaches the page.

2. **A DOM custom-event bus already exists and is already used for this exact moment.**
   `app.publishDomEvents(type, data)` ([js/app.js:2774-2777](../js/app.js#L2774-L2777)) does
   `document.body.dispatchEvent(new CustomEvent(type, data))`. `register.js` already fires
   `registerevents-started` / `registerevents-success` / `registerevents-failed` at exactly the
   moments a loader would need to hook into
   ([js/register.js:1154, 1159, 1163ish](../js/register.js#L1154)). And there's already a
   consumer: `test_fe/lander-pehdup-scripts.njk` listens for `registerevents-started` and toggles
   a `.ctaprocessing` class + injects a spinner div. **This is the existing, shipped version of
   the exact feature being asked for** — just hardcoded to one selector (`#entersweep`) and one
   effect (button spinner). The new system should be a generalization of this file, not a
   parallel mechanism.

3. **Selector conventions are already documented** in
   `test_fe/EMAIL_SUBMIT_LOADER_REFERENCE.md` — the short version: `#landerform` (whole form),
   `#entersweeps` (email input), `#entersweep` (submit button) are universal across every lander
   template, legacy and modern. `.form-field.pii-email` / `.form-field.submit-box` wrapper divs
   exist on modern templates only (~90%) and silently don't exist on ~110 legacy pages — so any
   config pointing at those must have a fallback, or the whole point of "configurable per
   experiment" breaks the moment it's pointed at a legacy template.

---

## 2. Design goals

- **No deploy for a new loader placement.** Backend sets JSON at experiment level; JS reads it
  generically. Adding "show loader on survey-complete" next quarter should not require touching
  `lander.js`/`register.js` again — only adding a config entry (assuming the trigger event
  already exists or is added once, generically).
- **Fail-safe by construction.** A bad/missing selector, a malformed config, or a missing script
  must never leave the page in a stuck state (loader on forever, page hidden forever, form
  unsubmittable). Every activation must have a corresponding, guaranteed deactivation path —
  this mirrors why every existing loader in this codebase (page loader, express-flow loader,
  linkout loader) has a fail-open/timeout-fallback path (see prior loader research — e.g.
  `hideLoaderAndDisplayPII`'s catch-all call on error, linkout's fail-open on fetch `.catch()`).
- **Config describes *what*, not *how*.** The JSON should say "on event X, target selector Y,
  apply effect Z, for at least N ms" — it should not contain arbitrary executable script.
- **One engine, reused across pages.** A single small JS module (not one script per page)
  interprets the config and attaches listeners. Adding it to a new template = one `<script>`
  include, same as `lander-pehdup-scripts.njk` today.

---

## 3. Proposed JSON structure

Your structure is a good starting skeleton. Changes made and why, below the JSON.

```json
{
  "loaderConfig": {
    "version": 1,
    "rules": [
      {
        "id": "emailSubmitButtonLoader",
        "enabled": true,
        "trigger": {
          "event": "registerevents-started",
          "source": "dom"
        },
        "target": {
          "selector": "#entersweep",
          "matchType": "id",
          "fallbackSelector": "#landerform",
          "scope": "button"
        },
        "effect": {
          "type": "buttonSpinner",
          "minDurationMs": 800,
          "disableTarget": true,
          "swapText": "Processing.."
        },
        "revert": {
          "on": ["registerevents-success", "registerevents-failed"],
          "timeoutMs": 8000
        }
      },
      {
        "id": "emailSubmitFullPageLoader",
        "enabled": false,
        "trigger": {
          "event": "registerevents-started",
          "source": "dom"
        },
        "target": {
          "selector": "#landerform",
          "matchType": "id",
          "fallbackSelector": ".enter-sweep-box",
          "scope": "container"
        },
        "effect": {
          "type": "overlay",
          "minDurationMs": 1500,
          "overlayHtml": "<div class=\"custom-page-loader\"><span>Hold on...</span></div>"
        },
        "revert": {
          "on": ["registerevents-success", "registerevents-failed"],
          "timeoutMs": 10000
        }
      }
    ]
  }
}
```

### What changed from your draft, and why

| Your field | Change | Why |
|---|---|---|
| top-level `loaderScript` array | renamed `loaderConfig.rules`, added `version` | `version` lets the JS engine reject/ignore a config shape it doesn't understand yet, instead of guessing — cheap insurance for a backend-controlled config that will evolve. |
| `"placement":"emailSubmit"` | replaced by `"id"` (free-text label) | `placement` implied a fixed enum of known spots. An `id` is just a label for logs/debugging; *where* it applies is fully described by `target`, not by the id string. |
| (missing) | added `"enabled"` per rule | Lets backend A/B test or kill-switch **one rule** without removing it from the JSON or touching others — same pattern as `experiment_config.enableX` flags used everywhere else in this repo. |
| `"event":"EmailSubmit"` | split into `trigger.event` + `trigger.source` | `EmailSubmit` in the existing codebase is actually an **analytics tracking event name** (`eventTracker.trackCustomEvent(...)`), not a DOM event — conflating the two is the most likely real bug in the original sketch. The real DOM hook available today is `registerevents-started` (fired via `publishDomEvents`, listened to via `document.body.addEventListener`). `source: "dom"` future-proofs for a later `source: "custom-tracker"` if you ever want to trigger off an analytics event instead. |
| `"whichPlace":"div/button/id/class"` | replaced by `target.selector` + `target.matchType` + `target.fallbackSelector` + `target.scope` | Your field conflated "how to query the DOM" (id vs class vs tag) with "what kind of thing it is" (div vs button). Split those: `matchType` = the *query strategy* (`id`/`class`/`selector` for a raw `querySelector` string), `selector` = the actual value, `scope` = a semantic hint the effect renderer uses to decide behavior (e.g. `button` scope disables+spinner-izes; `container` scope overlays). **`fallbackSelector` is the important addition** — per the reference doc, `.form-field.submit-box` doesn't exist on ~110 legacy templates; without a fallback, "fully configurable" quietly becomes "configurable only on templates newer than 2023." |
| `"script":"loader1.html"` | replaced by `effect.type` (enum) + effect-specific params, no HTML/script file reference | Loading an arbitrary `.html`/script file by name from a backend-controlled JSON is effectively remote code execution by config — a compromised or mistyped experiment config could inject arbitrary markup/script into every lander page simultaneously. Instead, ship a **fixed, reviewed set of effect renderers** in the loader engine (`buttonSpinner`, `overlay`, `hideAndShow`, ...) and let config pick one by name + pass safe parameters (text, min-duration, an escaped HTML snippet for overlay content if truly needed — never a script). This is a security/blast-radius decision as much as an architecture one — flagging it explicitly since it changes your original idea's shape the most. |
| `"time":"100ms"` | `effect.minDurationMs` (integer, ms, no unit suffix) | Match the numeric-ms convention already used everywhere in this repo (`loaderDelay = 3000`, `delayExpressUserFlow`, `data-loader-delay="2000"`) instead of a unit-suffixed string that needs parsing. Also renamed to make clear it's a *minimum* show duration (the pattern every existing loader in this codebase uses — see linkout's `data-loader-delay` polling, express-flow's per-step `loaderStepLimit`), not a fixed on-screen time unrelated to when the real work finishes. |
| (missing) | added `revert.on` (array of DOM events) + `revert.timeoutMs` | **This is the fail-safe.** Every existing loader in this codebase has a guaranteed way to turn itself back off (`hideLoaderAndDisplayPII`'s error-path call with no args, linkout's fail-open on `.catch()`, offerwall's `hideLoader` timeout). This config must declare its own hide-trigger(s) explicitly, *plus* a hard timeout backstop — so if `registerevents-success`/`registerevents-failed` never fires (network hang, JS error elsewhere, unexpected backend response shape), the loader still self-clears after `timeoutMs` instead of leaving the form permanently disabled/hidden. This is non-negotiable for a config-driven system since a bad config can't be caught by code review the way a bad PR would be. |

---

## 4. Effect types (v1 proposal — small, fixed set)

Only these ship initially; the set can grow later, but each one is a reviewed code path, not
arbitrary config content.

| `effect.type` | What it does | Maps to existing precedent |
|---|---|---|
| `buttonSpinner` | Injects `.ctaspinner` into the target button, adds a processing class to the nearest `form`, optionally swaps button text, optionally `disabled = true` on the target | Direct generalization of `test_fe/lander-pehdup-scripts.njk` — same DOM shape, same CSS classes, just selector-driven instead of hardcoded to `#entersweep`/`#landerform`. |
| `overlay` | Inserts a loader overlay element as a child of `target.selector` (or `fallbackSelector`), absolutely positioned to cover it, with a fixed small set of allowed inner content (text + a CSS-only spinner — no arbitrary HTML/script) | Same conceptual shape as `#loaderSpinner` full-page loader, scoped to an arbitrary container instead of a fixed id. |
| `hideAndShowLoader` | Hides `target.selector` (`display:none`, matches your original "hide that page/div and show loader on it") and shows a sibling loader element (or an injected one) in its place | This is literally your stated idea #1 ("based on div/classes/ids we want to hide that page and show loader on that") — implemented as its own named effect rather than folded into `overlay`, since fully hiding vs. overlaying have different failure modes (hiding the form and failing to revert is worse than an overlay failing to revert, since the overlay at least doesn't remove the user's typed input from view/DOM). |

All three share the same revert contract: whatever they showed/hid/disabled gets exactly
reversed by `revert`, and `revert` always fires — either from a listed DOM event or the timeout,
whichever comes first, `clearTimeout`-ing the other.

---

## 5. Engine sketch (where this lives in the repo)

- New file, e.g. `js/loader-engine.js`, loaded once per page (same include pattern as
  `lander-pehdup-scripts.njk` today, or folded into `app.js` if you'd rather not add a new
  script tag per template).
- On page init:
  1. Read `window.experiment_config?.loaderConfig` (mirrors every other flag read in this repo).
  2. Validate `version`; if absent/unsupported, log + no-op (fail-safe: absence of config must
     never break the page — it must look exactly like today's behavior).
  3. For each `rules[]` entry where `enabled !== false`:
     - Resolve `target.selector`, falling back to `target.fallbackSelector` if the primary isn't
       found in the DOM (`document.querySelector` returns null) — log which one was used, for
       debugging a template that's missing an expected selector.
     - If neither selector resolves, skip the rule (log a warning) rather than throwing —
       one bad rule must not stop other rules or break the page.
     - Attach `document.body.addEventListener(trigger.event, () => applyEffect(rule, target))`.
     - `applyEffect` looks up `effect.type` in the fixed renderer map; unknown type → skip + log.
     - After applying, start the `revert.timeoutMs` timer and attach one-time listeners for each
       `revert.on` event, whichever fires first wins and clears the other.
- This mirrors the existing "read config → conditionally wire a DOM listener" shape already used
  throughout `lander.js` (e.g. `this.isExpressReg` gating `registerevents-started`-adjacent
  logic), so it fits the codebase's existing idiom rather than introducing a new pattern.

---

## 6. Open questions to settle with backend before wiring this up

1. **Confirm the config actually arrives via `window.experiment_config`** (not a separate fetch)
   — this repo has zero precedent for fetching a JSON config client-side; everything is
   server-injected today. If backend intends a separate async fetch, the engine needs a
   loading-state of its own (chicken-and-egg: can't show a loader-config-driven loader before
   the config that describes it has loaded) — strongly recommend server-injection to avoid this.
2. **Per the reference doc's open item**: no `window.config.*` flag is read anywhere in this
   repo today (`window.config.hostApi` etc. are used for API paths, not feature flags) — confirm
   `loaderConfig` should live under `experiment_config`, not a new top-level global.
3. Should `rules` be allowed to target **the same trigger event with multiple rules** (e.g. one
   button-spinner rule + one full-page-overlay rule both firing on `registerevents-started`)?
   The design above allows it (array), but worth confirming that's an intended use case and not
   just redundant config.
4. Registration pages (no email field, per the reference doc) will need their own trigger
   event(s) later (`registerevents-success`, or a new `surveycomplete` DOM event) — confirm
   naming convention for future events now, so rule `id`s and event names stay consistent
   instead of ad hoc per page type.

---

## 7. Suggested next step

Build `js/loader-engine.js` + a `test_features/test_loader_configurable-engine.html` demo that:
loads a sample `experiment_config.loaderConfig` inline, simulates `registerevents-started` /
`registerevents-success` firing (mirroring `register.js:1154`/`1159`), and shows both a
`buttonSpinner` and a `hideAndShowLoader` rule running off the same config shape above — so you
can see the whole config → DOM-event → effect → revert loop working end-to-end before wiring it
into a real template.
