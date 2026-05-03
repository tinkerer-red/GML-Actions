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
          # ----- packaging (required) -----------------------------------
          # Path to the source .yyp inside this repo.
          project-file: "MyGame.yyp"

          # Internal package id - drives the output filename and the
          # package's .yyp basename.
          package-id: "MyLibrary"

          # Comma-separated resource-tree folder names to include.
          # Sub-folders are picked up automatically. Examples:
          #   "MyLibrary"                       - root folder, everything inside
          #   "MyLibrary/Core"                  - just one sub-folder
          #   "MyLibrary/Core,MyLibrary/Extras" - multiple sub-folders
          # The "folders/" prefix and ".yy" suffix are stripped if present.
          include-folders: "MyLibrary"

          # ----- packaging (optional) -----------------------------------
          # Display name shown in the IDE's package importer.
          # Defaults to package-id when blank.
          package-name: ""

          # Publisher name shown next to the package in the IDE.
          package-publisher: ""

          # When "true", filename is "<package-id>-<version>.yymps"
          # (e.g. MyLibrary-1.0.yymps). When "false", just "<package-id>.yymps".
          append-version-to-filename: "true"

          # ----- release notes ------------------------------------------
          # Master switch. When "false", the action does not touch the
          # release body at all. The two flags below have no effect when
          # this is "false".
          auto-generate-description: "true"

          # When "true", append an "Issues Closed" section listing issues
          # closed by closing-keyword references (close/fix/resolve + #N)
          # in commits since the previous tag.
          include-issues-in-description: "true"

          # When "true", append a "Commits" section listing every commit
          # since the previous tag.
          include-commits-in-description: "true"

          # ----- tooling ------------------------------------------------
          # Pinned version of @gm-tools/project-tool from the GMPM registry.
          project-tool-version: "2024.14.160"

          # GMPM registry URL. Override only if you self-host the registry.
          gmpm-registry: "https://gmpm.gamemaker.io"
```

Tag and push to release:

```
git tag 1.0
git push origin 1.0
```

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
