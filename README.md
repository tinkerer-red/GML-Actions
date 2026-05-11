# GML-Actions

A collection of single-purpose GitHub Actions for GameMaker library and
project authors. Each action does exactly one thing; pick what you need and
compose them in your own workflows.

## Available actions

| Action | Path | Purpose |
|---|---|---|
| **GameMaker Package** | [`actions/package`](actions/package/action.yml) | Build a `.yymps` from a project subfolder, attach to release |
| **Release Notes - Header** | [`actions/release-notes/header`](actions/release-notes/header/action.yml) | Append `# Release X:` + description placeholder + `---` to release body |
| **Release Notes - Issues** | [`actions/release-notes/issues`](actions/release-notes/issues/action.yml) | Append "Issues Closed" section listing closing-keyword references |
| **Release Notes - Commits** | [`actions/release-notes/commits`](actions/release-notes/commits/action.yml) | Append "Commits" section listing commits since previous tag |

`_internal/` actions are shared helpers called by the public actions above.
Do not reference them directly, their contracts are unstable.

## Quick start

Provided are ready-made workflow files you can drop directly into your
`.github/workflows/` folder. Download or copy the ones you want:

| Workflow file | What it does |
|---|---|
| [release-package.yml](workflows/release-package.yml) | Builds and attaches the `.yymps` to the release |
| [release-notes.yml](workflows/release-notes.yml) | Generates release notes (header, closed issues, commits) |
| [generate-docs.yml](workflows/generate-docs.yml) | Runs Tome and publishes a Docsify site to gh-pages |

Both release workflows trigger on the same tag patterns so they run in parallel.
Each can be re-run or disabled independently. Drop whichever ones you don't need.

**Minimum setup** : edit the required inputs in your imported workflow, as an example `release-package.yml`:

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
When testing changes on an unpublished branch, remember that public action references such as `tinkerer-red/GML-Actions/actions/_internal/gm-compile@v2` resolve from the published `v2` tag. Update the floating tag or temporarily point test workflows at the branch/SHA that contains the new internal actions.

## Generate Docs action

The `actions/generate-docs` action compiles a GameMaker project, runs it
against the [Tome](https://github.com/CataclysmicStudios/Tome) documentation
library, and publishes the generated Docsify site to your repo's `gh-pages`
branch. **Requires a `windows-latest` runner.**

### One-time GitHub Pages setup

The action pushes to `gh-pages` but GitHub Pages must be configured to serve
it:

> Settings → Pages → Build and deployment → Source: **Deploy from a branch** →
> Branch: **gh-pages** → Folder: **/**

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `project-file` | No | *(auto-detect)* | Path to the `.yyp`. If omitted the action searches the repo recursively and fails when zero or multiple are found. |
| `toolchain` | No | *(latest)* | GameMaker toolchain version, e.g. `GMS2@2024.14.3.260`. |
| `release-tag` | No | *(from gh-pages)* | Docs version label. Release events use the GitHub tag. Manual runs use this override, then fall back to the current version in `gh-pages/config.js`. |
| `output-path` | No | `C:/tome-output` | Absolute path where Tome writes the site. |
| `create-issue-on-failure` | No | `false` | Open a GitHub issue when Tome reports failure. Requires `issues: write`. |
| `verbose` | No | `false` | Pass `TOME_CI_VERBOSE=1` to the runner. |

### Environment variables passed to Tome

The action sets these in the GameMaker runner process before executing:

| Variable | Example value | Purpose |
|---|---|---|
| `TOME_CI_MODE` | `1` | Signals automated/CI execution to Tome and the project. |
| `TOME_OUTPUT_PATH` | `C:/tome-output` | Override for Tome's output directory (read natively by CI-aware Tome). |
| `TOME_RELEASE_TAG` | `1.4.3` | The docs version being generated. |
| `TOME_OLDER_VERSION` | `["1.4.2","1.4.1"]` | JSON array of previous version folders, newest first. |
| `TOME_CI_VERBOSE` | `1` | Set only when the `verbose` input is `true`. |

`TOME_RELEASE_TAG` and `TOME_OLDER_VERSION` are action conventions. Your
project reads them in `tomeSetup()` and forwards them to Tome's API.

### Project setup — CI-aware Tome

If the [Tome CI PR](https://github.com/CataclysmicStudios/Tome) is accepted,
Tome reads `TOME_CI_MODE` and `TOME_OUTPUT_PATH` natively and calls
`game_end()` automatically. Your `tomeSetup()` only needs site configuration:

```gml
function tomeSetup() {
    Tome.site.setName("My Library");
    Tome.site.setDescription("API documentation.");

    var _tag = environment_get_variable("TOME_RELEASE_TAG");
    if (_tag == "") _tag = "1.0.0";
    Tome.site.setLatestVersion(_tag);

    var _old = environment_get_variable("TOME_OLDER_VERSION");
    if (_old != "") Tome.site.setOlderVersions(json_parse(_old));

    Tome.site.setHomepage("myHomepage", true);
    Tome.site.add("myDocsPage");
}
```

### Project setup — legacy Tome

For Tome versions without native CI support, bridge the env vars manually
and add a `game_end()` call:

```gml
function tomeSetup() {
    Tome.site.setName("My Library");
    Tome.site.setDescription("API documentation.");

    var _tag = environment_get_variable("TOME_RELEASE_TAG");
    if (_tag == "") _tag = "1.0.0";
    Tome.site.setLatestVersion(_tag);

    var _old = environment_get_variable("TOME_OLDER_VERSION");
    if (_old != "") Tome.site.setOlderVersions(json_parse(_old));

    Tome.site.setHomepage("myHomepage", true);
    Tome.site.add("myDocsPage");

    if (environment_get_variable("TOME_CI_MODE") == "1") {
        game_end();
    }
}
```

> **Legacy output path:** `TOME_OUTPUT_PATH` is not read by legacy Tome. Set
> `TOME_LOCAL_REPO_PATH` to the same path as the action's `output-path` input:
> ```gml
> #macro TOME_LOCAL_REPO_PATH "C:/tome-output/"
> ```

### Success and failure detection

The action detects outcome from Tome's own log output:

- `Tome: All docs generated!` → success, docs are published
- `Tome: Doc generation failed` → failure, diagnostics artifact is uploaded

GameMaker runner exit codes are not used because they are unreliable.

---

│   ├── generate-docs/           # public: compile, run Tome, publish gh-pages
│       ├── gm-compile/          # Node setup, .yyp detection, cache, gm-cli compile
│       ├── gm-run/              # locate Runner.exe + .win, run, capture log
│       ├── gh-pages-publish/    # overlay output dir onto gh-pages branch
│       ├── resolve-docs-meta/   # resolve version tag + older-versions JSON
│   └── generate-docs.yml
