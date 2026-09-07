# The Becoming Studio

The portfolio site for [thebecoming.studio](https://thebecoming.studio) — a single-page
gallery of works (apps, stories, campaigns) that reveal on scroll and expand into a
full-screen detail overlay.

## What this is

Not a static site and not a React project. It is a *Design Component* (`x-dc`) document:
`index.html` holds the template, the logic class and the content, and `support.js` compiles
it to React at load time. There is no build step and no `package.json`.

```
index.html    the entire app — template, logic, and the DATA array of works
support.js    generated dc-runtime bundle — do not edit by hand
wrangler.toml Cloudflare Worker config (assets-only)
```

## Running it locally

```sh
python3 -m http.server 8000    # then http://localhost:8000/
```

Boot needs the network: the runtime pulls React 18.3.1 from a CDN.

There are no tests. Verification is visual — load the page and check the reveal sequence,
the hover dimming, and the detail overlay opening and closing.

## Deploying

Cloudflare Workers Builds deploys `main` to an assets-only Worker named `thebecomingstudio`,
which owns the `thebecoming.studio` custom domain. `.assetsignore` controls what is
published; everything not listed there is served at the site root.

`www` is handled by a Cloudflare Redirect Rule to the apex, not by a route here.
