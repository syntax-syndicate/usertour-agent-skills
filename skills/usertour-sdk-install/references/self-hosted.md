# Self-hosted / local instances

By default the SDK talks to **Usertour Cloud** — `init(token)` is all you need.
If the deployment is **self-hosted** (or a **local** instance you're testing
against), the SDK must be pointed at that instance, or it keeps talking to Cloud
and your published content never loads (it'll look like "valid + published but
nothing shows").

**Canonical keys live in the docs — fetch, don't hardcode:**
WebFetch `https://docs.usertour.io/open-source/usertourjs`. The values below are
illustrative; the doc is the source of truth.

## Decide: same-origin or cross-origin?

**Same-origin** — the host app is served from the *same* deployment as Usertour
(same scheme/host/port). Nothing to configure: the SDK's built-in defaults are
relative (websocket on the same origin, assets under `/sdk`), so they resolve to
the deployment automatically. Just `init(token)`.

**Cross-origin** — the host app runs on a *different* origin than the Usertour
deployment (e.g. your app on `app.example.com`, Usertour on
`usertour.example.com`; or a local app on `:5173` vs Usertour on `:8011`). Set
`window.USERTOURJS_ENV_VARS` **before** the loader/`init()`:

```html
<script>
  window.USERTOURJS_ENV_VARS = {
    WS_URI: "https://usertour.example.com/",                 // realtime + content
    ASSETS_URI: "https://usertour.example.com/sdk",          // CSS / assets
    USERTOURJS_ES2020_URL: "https://usertour.example.com/sdk/es2020/usertour.js",
    USERTOURJS_LEGACY_URL: "https://usertour.example.com/sdk/legacy/usertour.iife.js"
  };
</script>
<!-- then the loader + usertour.init("<environment token>") -->
```

That is the COMPLETE key set the loader/SDK reads, plus one more:
`USERTOURJS_BROWSER_TARGET` (`"es2020"` | `"legacy"`) force-picks a bundle
instead of the loader's user-agent sniff — only needed when the sniff guesses
wrong (exotic webviews).

**Serving a locally BUILT bundle (apps/sdk/dist)?** The dist layout carries a
version layer — `/<version>/es2020/usertour.js` — and the bundle's baked-in CSS
path expects `ASSETS_URI` to be the serve ROOT (it appends
`/<version>/<target>/css/index.css` itself). Pointing `ASSETS_URI` at the
target folder double-nests the path → CSS 404s → widgets render 0×0 with no
error (this exact mistake has burned two debugging sessions).

Confirm the exact key names + which are required against the docs above.

## Local dev vs full self-host — don't over-configure

These are two different situations. Conflating them is the most common mistake:
copying the all-four-keys block for a *local backend* points `ASSETS_URI` /
`USERTOURJS_ES2020_URL` at `localhost/sdk`, which a bare dev server doesn't serve
→ the bundle 404s and nothing renders.

**Not sure which you are?** Probe whether the backend serves the SDK bundle:
`curl <backend>/sdk/es2020/usertour.js` — **200** ⇒ full self-host (set all keys);
**404** ⇒ local dev (set only `WS_URI`, bundle stays on Cloud).

**Local dev — your app talks to a local backend, bundle stays on Cloud (the
common case).** Set **only `WS_URI`**; leave the asset/bundle keys unset so the
SDK still loads its bundle + CSS from Usertour Cloud. Your local server does
**not** need to serve `/sdk`.

```js
// before the loader / init():
window.USERTOURJS_ENV_VARS = { WS_URI: "http://localhost:3001" }  // data/realtime only
```

**Full self-host — you serve everything yourself.** The all-in-one deployment
serves the SDK bundle + assets at `/sdk` (via nginx) and proxies the websocket.
Only then set all the keys (the cross-origin block above), every URL pointing at
your instance.

> The npm `usertour.js` package is a thin loader that lazy-loads the real bundle
> (from Cloud unless redirected), so `USERTOURJS_ENV_VARS` must be set **before**
> `init()` even on the npm path — it's not only for the HTML snippet.

> ⚠️ Point the endpoint **only** via `USERTOURJS_ENV_VARS.WS_URI`. Do **not** also
> call the SDK's `setServerEndpoint()` (a legacy method still present in the
> package types): setting both hangs the SDK (the page main thread freezes, nothing
> renders). Use `WS_URI` alone.

## Verify it took

In the browser console, confirm the SDK's network calls (websocket + `/sdk/…`)
go to **your instance**, not `*.usertour.io`. If they still hit Cloud,
`USERTOURJS_ENV_VARS` wasn't set before `init()`.
