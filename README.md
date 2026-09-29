# go-fft docs

Documentation site for [go-fft](https://github.com/go-fft/fft) — built with
[Hugo](https://gohugo.io) and
[hugo-theme-relearn](https://mcshelby.github.io/hugo-theme-relearn/), served at
<https://go-fft.github.io/docs/>.

## Local preview

```sh
hugo server
```

The theme is a **Hugo Module**, so the Go toolchain resolves it from the pin in
`go.mod` — there is no submodule to initialise. `hugo mod get -u` updates it.

## Publishing

Pushing to `main` builds the site and publishes it through the GitHub Pages
**Actions** deployment, which is what the rest of this organisation uses. A
pull request builds but does not publish.

## What changed from MkDocs

The site was MkDocs Material, versioned with [mike](https://github.com/jimporter/mike)
onto a `gh-pages` branch.

### Versioning: the theme has it, and it is not mike

⛔ An earlier version of this file said the version dimension was gone because
"Hugo has no mike". **That was wrong**, and it is corrected here rather than
quietly: hugo-theme-relearn ships versioning — a `versions` array in `params`, a
version switcher at the top of the sidebar, and a banner on any page that is not
the latest version.

What it does *not* ship is mike's automation. With mike, `mike deploy 0.1 latest`
builds and pushes a version onto `gh-pages` and manages the aliases. With
relearn, each version is a **separate source tree, a separate build and a
separate deploy** to its own `baseURL`; the shared `versions` array is what makes
the switcher find them.

It is not configured here, because this site has **one** version. mike published
exactly one too — `0.1`, aliased `latest`, with `/docs/` redirecting to it — so
turning it on today would add a switcher with a single entry. The day go-fft
wants documentation per release, the theme is ready and the recipe is in
[its docs](https://mcshelby.github.io/hugo-theme-relearn/configuration/sitemanagement/versioning/).

### Colours

The theme ships as `go-fft-light` / `go-fft-dark`, and every value in
`assets/css/theme-go-fft-*.css` is a token already used by
[go-fft.github.io](https://github.com/go-fft/go-fft.github.io) — so the
documentation and the landing page are one site to a reader who moves between
them. The two relearn defaults stay available in the variant switcher.

The sidebar logo is `static/images/logo.svg`, byte-identical to
[go-fft/brand](https://github.com/go-fft/brand)'s `svg/color/go-fft.svg`.

⛔ Printed pages keep the theme's own colours: relearn scopes every variant with
`:root:not([data-r-output-format='print'])`, so the print stylesheet is fixed by
design. Nothing to configure, and worth knowing before someone reports it.

The address is unchanged, so every link into the site still resolves.

BSD-3-Clause © the go-fft/docs authors.
