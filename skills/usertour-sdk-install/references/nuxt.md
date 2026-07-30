# Nuxt (Vue 3 / Nitro SSR)

The SDK is a **client** concern — it must run in the browser, never during SSR.
Confirm the loader/package from the docs installation page (WebFetch). The Nuxt-specific
move is a **client-only plugin**: the `.client` suffix makes Nuxt run the file
only in the browser, so the SDK never executes during server render (the usual
cause of "I added it but nothing loads" on Nuxt).

- The environment token goes in **public** runtime config — `runtimeConfig.public`
  (e.g. `NUXT_PUBLIC_USERTOUR_TOKEN` → `useRuntimeConfig().public.usertourToken`).
  The API token must never be exposed — keep it out of `public` config and any
  client code.

## The client plugin

**Where the file goes depends on the Nuxt major** — get it wrong and the plugin
is **silently ignored**: no error, no warning, the SDK just never starts (with
an `app/` directory present, a root-level `plugins/` file isn't even registered
in `.nuxt/types/plugins.d.ts`).

- **Nuxt 4** (project has an `app/` directory — today's scaffold default):
  `app/plugins/usertour.client.ts`
- **Nuxt 3** (no `app/` directory): root-level `plugins/usertour.client.ts`
- Not sure? Put it in a `plugins/` folder next to wherever `app.vue` lives.

The `.client` suffix = browser-only:

```ts
export default defineNuxtPlugin(() => {
  const token = useRuntimeConfig().public.usertourToken as string;
  usertour.init(token);

  // identify once the user is known (see identify.md). Example with a reactive
  // auth state — swap in your app's user composable/store:
  const user = useAuthUser();
  watch(
    user,
    (u) => {
      if (u) usertour.identify(u.id, { name: u.name, email: u.email });
      else usertour.reset();
    },
    { immediate: true },
  );
});
```

Notes:
- Load the SDK loader/package the way the docs installation page shows (WebFetch it); call
  `init()` once, inside this plugin. Don't reconstruct the loader from memory.
- `u.id` must equal the `externalId` the content targets — see identify.md.
- **SPA routing:** Nuxt navigations are client-side. If path-tied content doesn't
  re-evaluate on route change, hook `useRouter().afterEach(...)` or trigger with
  `usertour.start()` (see the docs for the route/navigate hook).
- Call `usertour.reset()` on logout.
- A plain Vue SPA (no Nuxt/SSR) doesn't need the plugin — init in `main.ts` after
  `createApp(...).mount(...)`; the SSR caveat above doesn't apply.
