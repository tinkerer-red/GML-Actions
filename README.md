# release-actions

A GitHub Action for GameMaker library authors. On a tag push, it builds a
`.yymps` from a project subfolder, attaches it to the GitHub release, and
fills in a templated release body (closed issues + commits since the
previous tag).

## Quick start

In each library repo, add `.github/workflows/release.yml`:

```yaml
name: release

on:
  push:
    tags:
      - "[0-9]*.[0-9]*"
      - "v[0-9]*.[0-9]*"

permissions:
  contents: write
  issues: read
  pull-requests: read

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          fetch-depth: 0   # needed for git log / git describe

      - name: Set up Node.js
        uses: actions/setup-node@v5
        with:
          node-version: "20"

      - uses: tinkerer-red/release-actions@v1
        with:
          project-file: "MyGame.yyp"
          package-id: "MyLibrary"
          package-name: "My Library"
          package-publisher: "Tinkerer_Red"
          include-folders: "MyLibrary"
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

Tag and push to release:

```
git tag 1.0
git push origin 1.0
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `project-file` | yes | - | Path to the source `.yyp` inside the consumer repo |
| `package-id` | yes | - | Internal package id - drives the output filename and the package's `.yyp` basename |
| `include-folders` | yes | - | Comma-separated resource-tree folder names to include |
| `github-token` | yes | - | Pass `${{ secrets.GITHUB_TOKEN }}` |
| `package-name` | no | (= `package-id`) | Display name shown in the IDE's package importer |
| `package-publisher` | no | `""` | Publisher name shown next to the package |
| `append-version-to-filename` | no | `"true"` | When `"true"`, produced filename is `<package-id>-<version>.yymps` |
| `auto-generate-description` | no | `"true"` | When `"false"`, no release body is written |
| `include-issues-in-description` | no | `"true"` | Include "Issues Closed" section |
| `include-commits-in-description` | no | `"true"` | Include "Commits" section |
| `project-tool-version` | no | `"2024.14.160"` | Pinned GMPM `project-tool` version |
| `gmpm-registry` | no | `"https://gmpm.gamemaker.io"` | GMPM registry URL |

### `include-folders` examples

- `"MyLibrary"` - root folder, includes everything inside it
- `"MyLibrary/Core"` - just one sub-folder
- `"MyLibrary/Core,MyLibrary/Extras"` - multiple sub-folders

The `folders/` prefix and `.yy` suffix are stripped if present, so pasting raw `folderPath` values from the `.yyp` also works.

## Outputs

| Output | Description |
|---|---|
| `filename` | Filename of the produced `.yymps` |
| `version` | Normalized 3-part version stamped into the package (e.g. `1.0.0`) |
| `raw-version` | Version as the user tagged it (e.g. `1.0`) |

## Versioning

This action follows floating-major pinning. Use `@v1` to get all v1.x.y
releases automatically:

```yaml
- uses: tinkerer-red/release-actions@v1
```

Pin to an exact version if you want immutability:

```yaml
- uses: tinkerer-red/release-actions@v1.2.3
```

Breaking changes will only ship in new major versions (`v2`, `v3`, etc).

## Tag formats accepted by the consumer workflow

The consumer's `release.yml` accepts both 2-part and 3-part tags, with or
without a leading `v`. Two-part tags are padded to three parts in the
package metadata: `1.0` becomes `1.0.0`. The filename uses the raw form.

Examples that work: `1.0`, `v2026.5`, `1.2.3`, `v40.50.0`.
