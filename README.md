# ThePicpixel's Notebook

Hugo blog using the [toha](https://github.com/hugo-toha/toha) theme.

## Prerequisites

- [Hugo extended](https://gohugo.io/installation/) (this project uses `v0.165.0+extended`)
- [Node.js and npm](https://nodejs.org/) — required because the toha theme pulls some assets (Bootstrap, KaTeX, flag-icons, etc.) from `node_modules` via Hugo Modules mounts (see `hugo.yaml`)

On openSUSE Tumbleweed, Node.js/npm can be installed with:

```bash
sudo zypper install nodejs npm
```

## Setup

Install the npm dependencies listed in `package.json`:

```bash
npm install
```

This creates the `node_modules` directory that the theme needs at build time.

## Local development

```bash
hugo server
```

If `node_modules` is missing or out of date, `hugo server` will fail with an error like:

```
error calling RelPermalink: TOCSS: failed to transform "styles/application.scss" ...
File to import not found or unreadable: bootstrap/scss/bootstrap.
```

Running `npm install` resolves this.
