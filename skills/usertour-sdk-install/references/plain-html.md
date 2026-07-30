# Plain HTML / server-rendered site

Get the exact loader snippet from the docs installation page (WebFetch
https://docs.usertour.io/developers/usertourjs-reference/installation); it is
served from the Usertour CDN. **Paste it on EVERY page (or in the shared
layout/template every page includes) — "once" means once per page, not one page
of the site.** A snippet only on index.html renders content only on index.html,
and the symptom (works on the home page, nothing anywhere else) reads like a
targeting/URL-rule problem and sends debugging the wrong way. Then init +
identify. Shape:

```html
<!-- In <head> or before </body>: paste the loader snippet from the docs. -->
<!-- After the loader, with YOUR environment token (list_environments → token): -->
<script>
  usertour.init("ENV_TOKEN");

  // Once you know the user (e.g. server-injected into the page):
  usertour.identify("USER_EXTERNAL_ID", {
    name: "Ada Lovelace",
    email: "ada@example.com",
    plan: "pro"
  });
</script>
```

Notes:
- `ENV_TOKEN` is the **environment** token (public). Never the API token.
- `USER_EXTERNAL_ID` must match the `externalId` the content targets — see
  identify.md.
- On a multi-page site each page reload re-runs init/identify, which is fine.
- Pasting a **fetched** minified loader? Check its integrity before shipping it:
  extract the JS and run `node --check` on it — retrieval/summarization can
  subtly corrupt minified code, and a corrupted loader fails silently. Docs
  excerpts may also return the bare JS without `<script>` tags — wrap it
  yourself.
- For an anonymous marketing page you can `init` without `identify`; content that
  targets identified users won't show until you identify.
