# React (Vite / CRA / any client-rendered SPA)

Confirm the install method (loader script vs npm package) from the docs
installation page (WebFetch) — don't assume a package name. The wiring is the same either way:
**init once at app mount; identify when the user is known; reset on logout.**

```tsx
// A small effect near the app root. If Usertour is loaded via the CDN snippet,
// `usertour` is on window; if via an npm package, import it per the docs.
import { useEffect } from "react";

const ENV_TOKEN = import.meta.env.VITE_USERTOUR_TOKEN; // environment (public) token

export function UsertourBootstrap({ user }: { user?: { id: string; name?: string; email?: string; plan?: string } }) {
  // init once
  useEffect(() => {
    usertour.init(ENV_TOKEN);
  }, []);

  // identify whenever the user changes; reset on logout
  useEffect(() => {
    if (user) {
      usertour.identify(user.id, { name: user.name, email: user.email, plan: user.plan });
    } else {
      usertour.reset();
    }
  }, [user]);

  return null;
}
```

Notes:
- Put the **environment token** in a client env var (`VITE_…` / `REACT_APP_…`).
  It's public, so that's fine. The API token must never reach the client.
- `user.id` must equal the `externalId` the content targets — see identify.md.
- **SPA routing:** the SDK watches for URL changes; if your router does soft
  navigations and content tied to a path doesn't re-evaluate, check the docs for
  the route-change hook / `usertour.start()` on the relevant screen.
- **Flow `navigate` actions:** so a flow step's navigate uses SPA soft-nav (not a
  full reload that drops the in-progress flow), wire `usertour.setCustomNavigate()`.
  The callback gets a URL **string**; React Router takes it directly. **TanStack
  Router**'s `to` is strictly typed against your route tree, so a raw string fails
  the type check — and content URLs can be external. Handle both:

  ```ts
  usertour.setCustomNavigate((url) => {
    if (/^https?:\/\//.test(url)) {
      window.location.assign(url); // external link — leave the SPA
      return;
    }
    router.navigate({ to: url as string as any }); // runtime-known path, not in the typed tree
  });
  ```

  See [troubleshooting.md](troubleshooting.md).
- **Sticky/fixed headers:** tooltip auto-scroll can land the target UNDER a sticky
  header (nearly every admin layout has one). Wire
  `usertour.setCustomScrollIntoView((el) => el.scrollIntoView({ block: "center" }))`
  — centering keeps the target clear of both sticky headers and bottom bars. Also
  needed for custom scroll containers the default scroll never reaches.
- Mount `<UsertourBootstrap user={currentUser} />` once high in the tree.
