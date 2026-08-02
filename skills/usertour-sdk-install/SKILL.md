---
name: usertour-sdk-install
description: Install and wire the Usertour Web SDK into a host web app so published content actually renders — the loader snippet, init() with the environment token, identify(), SPA routing, and verification. Use when the user asks to install / set up / embed / integrate Usertour in their app, add the loader snippet, wire identify, or asks "why isn't my flow / checklist showing up" in the running app. NOT for authoring content (use usertour-content-authoring) or server/API-token work. Also handles self-hosted / local instances (pointing the SDK at your own server). Prefers the live usertour.js docs and the environment token from the Usertour MCP over pre-trained snippets.
---

# Install the Usertour SDK

Goal: **published Usertour content renders for the right users in the host app.**
Authoring + publishing only stores content; it appears only once the app loads
the SDK and identifies the user. This skill wires that up.

Your baked-in knowledge of the exact loader snippet may be stale. **Prefer
retrieval**: read the live docs for the snippet, and get the token from the MCP.

## Retrieval sources (consult first)

| Source | How | Use for |
|--------|-----|---------|
| usertour.js installation | WebFetch https://docs.usertour.io/developers/usertourjs-reference/installation | The loader snippet + CDN URL, `npm install usertour.js`, and `init` / `identify` usage |
| usertour.js method signatures | Read the package's TypeScript types: `node_modules/usertour.js/dist/types/usertour.d.ts` | Exact signatures for host-integration hooks (`setCustomNavigate` SPA nav, `setCustomScrollIntoView`, `registerCustomInput`, `setBaseZIndex`, `setUrlFilter`, `setLinkUrlDecorator`) — the shipped types are authoritative when the docs' method list is incomplete. ⚠️ The types also list legacy methods — do NOT call `setServerEndpoint()` (see self-hosted.md). |
| `get_authoring_guide` (MCP tool) | call it with `{ "section": ["sdk", "host-dependencies"] }` | The SDK wiring + host-hook gotchas, without the rest of the authoring guide |
| `list_environments` (MCP tool) | call it | The **environment token** for `init()` (the `token` field) |
| Self-hosted SDK config | WebFetch https://docs.usertour.io/open-source/usertourjs | `USERTOURJS_ENV_VARS` keys when the instance isn't Usertour Cloud |
| Identity verification guide | WebFetch https://docs.usertour.io/developers/identity-verification | Backend-signed identify JWTs, when the environment enforces them |

If the `usertour` MCP isn't connected, you can still install from the docs, but
ask the user for the environment token (Settings → Environments) — it is the
**public SDK token**, not the API token.

## Workflow

1. **Detect the app** — read `package.json` / file layout to classify it: plain
   HTML (`plain-html.md`), React (Vite/CRA — `react.md`), Next.js (app vs pages
   router — `nextjs.md`), Vue SPA (`vue.md`), or Nuxt (`nuxt.md`). Load only the
   matching `references/<framework>.md`.
2. **Get the snippet + API** — WebFetch the usertour.js reference above; also call
   `get_authoring_guide` with `{ "section": ["sdk", "host-dependencies"] }`.
   Don't reconstruct the loader from memory — read it.
3. **Get the environment token** — call `list_environments` and use the primary
   environment's `token`. This is the public SDK token for `init()`.
   ⛔ **Never** put the API token (`utp_…`, used for this MCP) in client code — it
   grants full project write access. Before finishing, grep the app for `utp_` to
   be sure none leaked in.
4. **Install the loader** — add the snippet for the detected framework and call
   `usertour.init("<environment token>")` once, early in app startup. If the
   deployment is **self-hosted or local** (not Usertour Cloud), point the SDK at
   it first — see [references/self-hosted.md](references/self-hosted.md).
   When installing the npm package, install **`usertour.js@latest`** and confirm
   the version that actually landed matches `npm view usertour.js version` —
   lockfiles, caches, or a minimum-release-age policy can silently pin an old
   one. Versions **≤ 0.0.22 have a known footgun**: calling certain removed
   legacy methods before `init()` hangs the page (0.0.23+ warns and ignores
   them). If a project policy blocks the latest version, tell the user an
   explicit exception is worth it here.
5. **Wire `identify()`** — see [references/identify.md](references/identify.md).
   Call `usertour.identify(userId, attributes)` when the app knows the user.
   **`userId` must equal the `externalId`** the content's segments / start-rules
   target, or published content silently never shows. Call `usertour.reset()` on
   logout. If the environment **requires identity verification**, `identify()` /
   `group()` must also carry a backend-signed JWT (the `{ token }` option) or the
   identity is rejected at connect — near-silently: the only signal is the
   `identify()` promise rejecting with a misleading network-ish message — see
   [references/identity-verification.md](references/identity-verification.md).
6. **SPA routing** — for single-page apps, confirm content re-evaluates on route
   change (see the framework reference).
7. **Verify** — [references/verify.md](references/verify.md). Load the app,
   confirm the SDK is ready and a published flow appears for an identified test
   user. Don't claim "done" before this.

## Non-negotiables

- **Token type (security):** environment token for `init()`; the API token
  (`utp_…`) and the identity-verification **signing secret** (`utv_…`) never
  touch client code — only the backend-signed JWT does. Grep the app for both
  prefixes before finishing.
- **Identify linkage:** `identify()` id == the `externalId` used in
  segments / start-rules, or nothing renders (the #1 "it's not showing" cause).
- **Enforced identity verification:** when the target environment requires it,
  the install is not done until `identify()`/`group()` carry a valid signed
  token — unsigned calls are rejected near-silently (the promise rejects with a
  misleading "check your network" message; looks exactly like "identify
  ignored"). See
  [references/identity-verification.md](references/identity-verification.md).
- **Defer the API to the docs:** this skill is the wiring + gotchas, not a copy of
  the snippet. Read the reference; it's the source of truth.
- **Self-host target:** if the instance isn't Usertour Cloud, point the SDK at it
  (`USERTOURJS_ENV_VARS`) or it keeps talking to Cloud and nothing loads — see
  [references/self-hosted.md](references/self-hosted.md).
- **SSR / meta-frameworks:** the SDK is browser-only — load it and call `init()`
  **client-side only**, never during server render (guard `window`; init after
  mount). Next → `"use client"` / `next/script`; **Nuxt → a `.client` plugin**
  ([references/nuxt.md](references/nuxt.md)); SvelteKit → `onMount` / `browser`
  guard. Running it during SSR is a common "nothing loads."

## When it's not working

Published content that doesn't render, or a flow/condition that silently
misbehaves, is usually a host integration hook that wasn't wired. See
[references/troubleshooting.md](references/troubleshooting.md) — a symptom → hook
index (full reload mid-flow → `setCustomNavigate`, custom-input condition never
matches → `registerCustomInput`, UI behind a modal → `setBaseZIndex`, …). It maps
the symptom to the hook; the exact API is in the usertour.js advanced docs.

## SDK methods (from the reference)

`usertour.init(token)`, `usertour.identify(userId, attributes?)`,
`usertour.group(groupId, attributes?)`, `usertour.track(name, props?)`,
`usertour.start(contentId, opts?)` (launch content programmatically),
`usertour.reset()`. Confirm exact signatures in the package types
(`usertour.d.ts`). Stick to these documented methods — the types also expose
legacy ones; in particular do **not** call `setServerEndpoint()` to point at a
self-hosted backend (use `WS_URI` instead — see
[references/self-hosted.md](references/self-hosted.md)). Calling it alongside
`WS_URI` hangs the SDK. And `init` takes ONLY the token — `usertour.init(token)`;
the package README may show a 2-arg `init(id, { userInfo })` form — ignore it and
identify the user separately with `identify(userId, attrs)`, or identify silently
never fires.

## Related

Authoring the content these render is a different job — see the
`usertour-content-authoring` skill. The two halves connect through the
`externalId` you target in segments and pass to `identify()`.
