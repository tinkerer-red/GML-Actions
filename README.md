# GML-Actions

A collection of single-purpose GitHub Actions for GameMaker library and
project authors. Each action does exactly one thing; pick what you need and
compose them in your own workflows.

## Available actions

| Action | Path | Purpose |
|---|---|---|
| **GameMaker Package** | `actions/package` | Build a `.yymps` from a project subfolder, attach to release |
| **Release Notes - Header** | `actions/release-notes/header` | Append `# Release X:` + description placeholder + `---` to release body |
| **Release Notes - Issues** | `actions/release-notes/issues` | Append "Issues Closed" section listing closing-keyword references |
| **Release Notes - Commits** | `actions/release-notes/commits` | Append "Commits" section listing commits since previous tag |

`_internal/` actions are shared helpers called by the public actions above.
Do not reference them directly — their contracts are unstable.

## Quick start

We provide ready-made workflow files you can drop directly into your
`.github/workflows/` folder. Download or copy the ones you want:

| Workflow file | What it does |
|---|---|
| [release-package.yml](workflows/release-package.yml) | Builds and attaches the `.yymps` to the release |
| [release-notes.yml](workflows/release-notes.yml) | Generates release notes (header, closed issues, commits) |

Both workflows trigger on the same tag patterns so they run in parallel.
Each can be re-run or disabled independently. Drop either one you don't need.

**Minimum setup** — edit the two required inputs in `release-package.yml`:

```yaml
project-file: "MyGame.yyp"   # path to your .yyp
package-id:   "MyLibrary"    # your package id
include-folders: "MyLibrary" # resource folder(s) to bundle
```

Then tag and push:

```
git tag 1.0
git push origin 1.0
```

## Composing your own workflow

Each action is usable on its own. A custom workflow that runs unit tests
before packaging would look like:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... your test steps ...

  package:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v5
        with:
          node-version: "20"
      - uses: tinkerer-red/GML-Actions/actions/package@v2
        with:
          project-file: "MyGame.yyp"
          package-id: "MyLibrary"
          include-folders: "MyLibrary"
```

The `release-notes/*` actions must run in series (each one appends to the
release body; concurrent writes would race). The provided
[release-notes.yml](workflows/release-notes.yml) handles this with a
`needs:` chain. If you only want one section, just remove the other jobs.

## Tag formats

Both 2-part and 3-part tags are accepted, with or without a leading `v`.
Two-part tags are padded to three parts in the package metadata: `1.0`
becomes `1.0.0`. The filename uses the raw form.

Examples that work: `1.0`, `v2026.5`, `1.2.3`, `v40.50.0`.

## Versioning

This repo follows floating-major pinning. Use `@v2` to get all v2.x.y
releases automatically:

```yaml
- uses: tinkerer-red/GML-Actions/actions/package@v2
```

Pin to an exact version if you want immutability:

```yaml
- uses: tinkerer-red/GML-Actions/actions/package@v2.1.3
```

Breaking changes only ship in new major versions (`v3`, `v4`, etc).

## Repo layout

```
GML-Actions/
├── actions/
│   ├── package/                 # public: build & attach .yymps
│   ├── release-notes/
│   │   ├── header/              # public: H1 + description placeholder + ---
│   │   ├── issues/              # public: "Issues Closed" section
│   │   └── commits/             # public: "Commits" section
│   └── _internal/               # shared helpers - not for direct use
│       ├── setup-project-tool/
│       ├── parse-version/
│       └── git-range/
├── workflows/                   # ready-made consumer workflow files
│   ├── release-package.yml
│   └── release-notes.yml
└── .github/workflows/
    └── tag-major.yml            # maintains the floating v2 tag
```
