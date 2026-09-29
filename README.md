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

⛔ **The version dimension is gone.** mike published one version, `0.1`, aliased
`latest`, with `/docs/` redirecting to it — so in practice the site had one
version with an extra path segment in front of it. Hugo has no mike, and
inventing one for a single version would be building a mechanism for a problem
nobody had. If go-fft ever needs docs per release, that is a deliberate piece of
work, not something to reconstruct by reflex.

The address is unchanged, so every link into the site still resolves.

BSD-3-Clause © the go-fft/docs authors.
