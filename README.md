# GML-Actions

A collection of single-purpose GitHub Actions for GameMaker library and
project authors. Each action does exactly one thing; pick what you need and
compose them in your own workflows.

## Available actions

| Action | Path | Purpose |
|---|---|---|
| **GameMaker Package** | [`actions/package`](actions/package/action.yml) | Build a `.yymps` from a project subfolder, attach to release |
| **GameMaker Build** | [`actions/build`](actions/build/action.yml) | Package a whole project into a distributable zip, upload it as a workflow artifact |
| **Steam Upload** | [`actions/steam-upload`](actions/steam-upload/action.yml) | Upload a build to Steam via SteamPipe, optionally set it live on a private beta branch |
| **Generate Docs** | [`actions/generate-docs`](actions/generate-docs/action.yml) | Compile a project, run [Tome](https://github.com/CataclysmicStudios/Tome), publish generated docs to `gh-pages` |
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
| [generate-docs.yml](workflows/generate-docs.yml) | Runs [Tome](https://github.com/CataclysmicStudios/Tome) and publishes generated docs to `gh-pages` |
| [build-nightly.yml](workflows/build-nightly.yml) | Daily cron, skipped when `main` has not moved: build, upload artifact, set live on the `nightly` Steam branch |
| [build-playtest.yml](workflows/build-playtest.yml) | Every version tag: build, publish a GitHub pre-release, set live on the `playtest` Steam branch |
| [build-stable.yml](workflows/build-stable.yml) | Manual trigger only: build and upload to Steam, promoted by hand |

The release workflows trigger on the same tag patterns so they run in parallel.
Each can be re-run or disabled independently. Drop whichever ones you don't need.

**Minimum setup** : edit the required inputs in your imported workflow, as an example `release-package.yml`:

```yaml
project-file: "MyLibrary.yyp"   # path to your .yyp
package-id:   "MyLibrary"    # your package id
include-folders: "MyLibrary" # resource folder(s) to bundle
````

Then publish a release.

## Generate Docs setup

The docs workflow uses [Tome](https://github.com/CataclysmicStudios/Tome). For repositories using GitHub Pages, configure:

```text
Settings -> Pages -> Build and deployment
Source: Deploy from a branch
Branch: gh-pages
Folder: /
```

The workflow passes these environment variables to the GameMaker runner:

```text
TOME_CI_MODE=1
TOME_OUTPUT_PATH=C:/tome-output
TOME_RELEASE_TAG=<resolved release/docs version>
TOME_OLDER_VERSION=<json array of older versions>
```

If you are using a Tome version with native CI support, Tome handles the output
path and exits automatically after generation.

If you are using an older Tome version, either update Tome or bridge the same
environment values in your project. At minimum, legacy Tome projects should set
their output path to match the workflow:

```gml
#macro TOME_ENABLED bool(environment_get_variable("TOME_CI_MODE") == "1")
#macro TOME_LOCAL_REPO_PATH "C:/tome-output/"
```

```gml
function tomeSetup() {
    //ENV TAG
    var _tag = environment_get_variable("TOME_RELEASE_TAG");
    if (_tag == "") _tag = "1.0.0";
    Tome.site.setLatestVersion(_tag);
	
	//ENV OLDER VERSIONS
    var _old = environment_get_variable("TOME_OLDER_VERSION");
    if (_old != "") Tome.site.setOlderVersions(json_parse(_old));
	
	//CLEANUP
    if (environment_get_variable("TOME_CI_MODE") == "1") {
        game_end();
    }
}
```

The provided [generate-docs.yml](workflows/generate-docs.yml) includes comments
for the available inputs and common options.

## Game builds and Steam setup

`actions/build` packages the whole project with `gm-cli package` and uploads the
zip as a workflow artifact. `actions/steam-upload` takes that zip (or a plain
directory) and pushes it through SteamPipe.

Split the two across runners: `build` needs `windows-latest`, `steam-upload`
runs on `ubuntu-latest`. The provided workflows already do this, passing the
build between jobs as an artifact.

### Secrets

| Secret | What it is |
|---|---|
| `STEAM_USERNAME` | Builder account username. Give this account only the app permissions it needs. |
| `STEAM_CONFIG_VDF` | Base64-encoded `config.vdf` for that account, after a successful Steam Guard login. |
The appid is not a secret - keep it in the workflow, or in a repo variable
(`vars.STEAM_APP_ID`). Same for the depot id (`vars.STEAM_PLAYTEST_DEPOT_ID`),
which can be left unset to use appid + 1.

To produce `STEAM_CONFIG_VDF`:

1. Create a dedicated builder account in Steamworks with only `Edit App
   Metadata` and `Publish App Changes To Steam`, scoped to the app you upload
   to. Do not reuse a personal account.
2. On a local machine, run `steamcmd +login <builder-account> +quit` and
   complete the Steam Guard prompt.
3. Run the same command again. It must log in with no prompt - that is the
   proof a reusable token was written.
4. Base64-encode `config/config.vdf` (next to `steamcmd.exe` on Windows, or
   `~/Steam/config/config.vdf` on Linux) and store the result as the secret:
   `cat config/config.vdf | base64 > config_base64.txt`.

The token is invalidated when Steam issues a new Guard challenge for the
account, so regenerate the secret when uploads start failing at login.

### Branches

Steam's private beta branches are the access control. Create them in
Steamworks under App Admin, give each one a password, and hand out the
password per audience. SteamPipe cannot create branches and cannot set the
`default` branch live - that promotion is a manual step in App Admin, by
design.

`steam-upload` refuses `default` and `public` unless `allow-public-branch` is
`"true"`. Leave `branch` empty to upload a build without setting it live
anywhere.

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
      - uses: tinkerer-red/GML-Actions/actions/package@v2
        with:
          project-file: "MyGame.yyp"
          package-id: "MyLibrary"
          include-folders: "MyLibrary"
```

The `release-notes/*` actions must run in series because each one appends to the
release body and concurrent writes would race. The provided
[release-notes.yml](workflows/release-notes.yml) handles this with a `needs:`
chain. If you only want one section, remove the other jobs.

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
