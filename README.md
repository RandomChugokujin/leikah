# [LEIKAH](https://leikah.haoyingcao.xyz/en/)

Stands for **Literally Everything I Know About Hacking**.

LEIKAH is an ethical hacking technical documentation website built with the [Hugo](https://gohugo.io/) static site generator and the [Docsy](https://github.com/docsy/docsy) theme.

## Goals

- Document the techniques, tools, commands, and their corresponding sample outputs used in ethical hacking.
- Provide contextual knowledge for readers to understand under what circumstances to best use each technique and how to perform troubleshooting.

## Who is this website for?

This website is for you if:

- You are a cybersecurity industry professional.
- You are a student studying ethical hacking (OSCP, CPTS, CRTO, etc.).

## Running the website locally

Prerequisites: Node.js 26 with npm, and Go 1.26.3 or later.

The project pins Docsy **0.18.1**, Hugo **0.170.0**, and Dart Sass **1.106.1**. Hugo and Sass are installed locally by npm; no global Sass installation is required when using the npm commands below.

From the repository root:

```bash
npm ci
npm run serve
```

Open <http://localhost:1313/en/>. Hugo watches the source files and refreshes the browser when they change. To include draft pages, use `npm run serve -- -D`.

For a production build:

```bash
npm run build:production
```

The output is written to `public/`.

## Updating Docsy

The theme uses the Hugo module path `github.com/docsy/docsy/theme`. After deliberately changing its pinned version, synchronize the generated npm workspace and lockfile:

```bash
npm run hugo -- mod get github.com/docsy/docsy/theme@v0.18.1
npm run hugo -- mod tidy
npm run hugo -- mod npm pack
npm install
npm run build:production
```

Commit `go.mod`, `go.sum`, `package.json`, `package-lock.json`, and `packages/hugoautogen/package.json` when they change. Review the [Docsy upgrade guide](https://www.docsy.dev/blog/2026/0.18.1/) before upgrading to another release.

## Troubleshooting

- **Dart Sass / `TOCSS-DART` error:** use `npm run serve`, not a bare `hugo server`. npm adds the project's Hugo and Sass executables to `PATH`. For a one-off Hugo command, use `npm run hugo -- <arguments>`. Run `npm ci` if dependencies are missing.
- **Wrong title, missing pages, or ignored settings:** the main site configuration is `hugo.yaml`. Do not add a competing `hugo.toml` or `hugo.json` at the repository root; Hugo selects one main configuration file rather than merging them automatically.
- **Missing Go executable:** install Go and make sure `go version` works. Hugo uses Go to resolve the Docsy module.
- **Dependency workspace out of sync:** run `npm run hugo -- mod npm pack`, then `npm install`.
