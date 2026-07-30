# Vue 3 (Vite / any client-rendered Vue SPA — no Nuxt)

Confirm the install method (loader script vs npm package) from the docs
installation page (WebFetch) — don't assume a package name. The wiring is the
same either way: **init once at app startup; identify when the user is known;
reset on logout.** There is no SSR here, so none of Nuxt's `.client` gymnastics
apply — for Nuxt see [nuxt.md](nuxt.md).

```ts
// src/usertour.ts — one plain module. If Usertour is loaded via the CDN
// snippet, `usertour` is on window; on the npm path add the import (confirm the
// package name/export from the docs — currently a default export):
// import usertour from "usertour.js";
const ENV_TOKEN = import.meta.env.VITE_USERTOUR_TOKEN; // environment (public) token

export function bootUsertour(user?: { id: string; name?: string; email?: string }) {
  usertour.init(ENV_TOKEN);
  if (user) {
    usertour.identify(user.id, { name: user.name, email: user.email });
  }
}
```

```ts
// src/main.ts
import { createApp } from "vue";
import App from "./App.vue";
import { bootUsertour } from "./usertour";

createApp(App).mount("#app");
bootUsertour(currentUser); // not tied to Vue's lifecycle — before or after mount both work
```

Notes:
- The SDK is framework-agnostic — no Vue plugin / provide-inject needed; a plain
  module called from `main.ts` is the whole integration. Reactive auth state?
  Call `usertour.identify(...)` from a `watch` on your user store
  (Pinia / composable) and `usertour.reset()` on logout — the same watch pattern
  nuxt.md shows, minus the plugin wrapper.
- Put the **environment token** in a client env var (`VITE_…`). It's public, so
  that's fine. The API token must never reach the client.
- `user.id` must equal the `externalId` the content targets — see identify.md.
- **SPA routing (vue-router):** the SDK watches for URL changes; if path-tied
  content doesn't re-evaluate on a soft navigation, check the docs for the
  route-change hook / `usertour.start()` on the relevant screen.
- **Flow `navigate` actions:** so a flow step's navigate uses SPA soft-nav (not a
  full reload that drops the in-progress flow), wire
  `usertour.setCustomNavigate((url) => router.push(url))` — vue-router's `push`
  takes the URL string directly. See [troubleshooting.md](troubleshooting.md).
