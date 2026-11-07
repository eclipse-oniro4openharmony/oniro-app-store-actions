# Oniro App Store actions

A reusable GitHub Actions workflow that builds an OpenHarmony app from its
source repository, attaches the package to a GitHub release, and publishes it
to the [Oniro App Store](https://store.iparo.app). Versions published this way
show the **Built from source** badge, linked to the commit and the build log.

No secrets are needed. GitHub's OIDC token proves to the store which
repository, commit, tag and workflow produced the package.

## Quick start

Add `.github/workflows/oniro-store.yml` to your app's repository:

```yaml
name: Publish to the Oniro App Store
on:
  push:
    tags: ['v*']
permissions:
  id-token: write   # proves to the store which repository and commit this is
  contents: write   # creates the GitHub release that hosts the package
jobs:
  publish:
    uses: eclipse-oniro4openharmony/oniro-app-store-actions/.github/workflows/publish.yml@v1
```

Then bump `versionCode` in `AppScope/app.json5`, commit, and push a tag:

```sh
git tag v1.2.0 && git push origin v1.2.0
```

The run builds the app, creates the `v1.2.0` release with the package
attached, and registers it with the store. The job summary links to the app's
store page.

### First release

The first release creates the app in the store, owned by the GitHub account
that pushed the tag. If that account is not a store publisher, the app goes to
the account that owns the repository. One of them must have signed in to the
store once. An app you already published by upload joins the repository on its
first release from it, provided you pushed the tag or own the repository.

From then on, GitHub Actions can publish the app only from this repository,
and a release can come from anyone who can push tags to it. Versions are
reviewed like uploads: live at once if the store approves automatically,
otherwise once an admin approves them. If the package has no icon, the listing
starts without one; add it on the store's publisher page.

### Build check on pull requests

The build job runs on any event, while publishing happens only for tags. To
check that pull requests still build the way the store builds them:

```yaml
on:
  pull_request:
  push:
    tags: ['v*']
```

The caller must still grant `id-token: write` and `contents: write`, because
GitHub checks a called workflow's permissions before it runs. On pull requests
the publish job is skipped, and the build job never gets more than
`contents: read`.

This works for pull requests from branches of the same repository. Pull
requests from forks, and Dependabot's, run with read-only permissions, so
GitHub may refuse to start the workflow at all. To check those, use a
separate build workflow.

## Inputs

| Input | Default | |
|---|---|---|
| `project-dir` | `.` | Directory of the OpenHarmony project, for monorepos. |
| `product` | `default` | hvigor product to build. |
| `release-name` | the tag | Title of the GitHub release, when the workflow creates it. |
| `app-name` | repository name | Store listing name. Used only for the first release. |
| `category` | `app` | `app` or `game`. Used only for the first release. |
| `store-url` | `https://store.iparo.app` | The store to publish to. |

The listing description of a new app is the repository's GitHub description.
Afterwards, edit the listing on the store.

## Outputs

| Output | |
|---|---|
| `package-name` | Bundle name of the published app. |
| `version-code` | `versionCode` of the published version. |
| `build-kind` | `ci_source` when the version got the *Built from source* badge. |
| `store-page` | The app's page on the store. |

## What the build does

1. Checks out the tagged commit (`persist-credentials: false`).
2. Reads `compileSdkVersion` for the product from `build-profile.json5`. If
   that file is not committed, `build-profile.example.json5` is used.
   HarmonyOS projects are refused: the store distributes OpenHarmony apps.
3. Installs the OpenHarmony command-line tools and the matching SDK with
   [`oniro-app`](https://www.npmjs.com/package/@oniroproject/oniro-app).
4. Signs with the OpenHarmony SDK's development key (`oniro-app sign`),
   replacing any `signingConfigs` in the project.
5. Runs `ohpm install --all` and
   `hvigorw assembleApp --mode project -p product=<product> -p buildMode=release`.
6. Packages the result:
   * a project with a single module ships as `<bundle>-<version>.hap`;
   * otherwise it ships as `<bundle>-<version>.app.zip`, a zip of the signed
     modules plus `pack.info` (the store lists it as an `.app`). hvigor's own
     `.app` contains unsigned modules, which devices refuse, and GitHub does
     not accept release assets named `*.app`.

The toolchain (about 4.5 GB of downloads) is installed fresh on every run and
never restored from a cache, so a cache written by an earlier build cannot
change this one.

## What the badge means

*Built from source* means this package was produced by this workflow, from the
named commit of the named public repository, on a GitHub-hosted runner. The
store checks GitHub's signed OIDC token for that, and it records the commit,
workflow and run. It is provenance, not a review. The build runs the
repository's own build scripts and dependencies, so the package is whatever
that commit builds.

The store grants the badge only when all of these hold:

* the token's `job_workflow_ref` is this workflow at a tag the store trusts
  (currently `@refs/tags/v1`). Calling the workflow by commit sha or by
  branch still publishes, but without the badge;
* the run is for the tag being published (`ref` is `refs/tags/<tag>`), from a
  `push`, `release` or `workflow_dispatch` event. Runs from any other event,
  pull requests included, cannot publish at all;
* the job ran on a GitHub-hosted runner (`runner_environment` is
  `github-hosted`). A self-hosted runner is a machine the repository's owner
  controls.

The two jobs keep the app's code away from the token:

* `build` runs the app's code with only `contents: read` and no `id-token`.
* `publish` holds `id-token: write` and `contents: write` but never checks out
  or runs anything from the app's repository. It hashes the artifact itself
  and sends the store the asset id, the size and the sha256.

The store then downloads the asset from GitHub, checks the hash, and parses
and verifies the package as it does any upload. It re-checks the release asset
daily. A version whose asset disappears is hidden until the asset is back.
One whose bytes change is withdrawn, and comes back only if a store admin's
re-check finds the original bytes again.

## Monorepos

Point `project-dir` at each app, and give each app its own tag pattern:

```yaml
on:
  push:
    tags: ['clock/v*', 'weather/v*']
jobs:
  clock:
    if: startsWith(github.ref_name, 'clock/')
    uses: eclipse-oniro4openharmony/oniro-app-store-actions/.github/workflows/publish.yml@v1
    with:
      project-dir: apps/clock
  weather:
    if: startsWith(github.ref_name, 'weather/')
    uses: eclipse-oniro4openharmony/oniro-app-store-actions/.github/workflows/publish.yml@v1
    with:
      project-dir: apps/weather
```

Several apps can also publish from one tag: each attaches its own package to
the shared release. Their publish jobs take turns. GitHub keeps only one job
waiting per tag, so with three or more apps, chain the calls with `needs:`.
This does not work with immutable releases (see below).

## Releases and re-runs

* With no release for the tag yet, the workflow creates one with the package
  attached. This also works when the repository makes releases immutable.
* If you create the release yourself first, the workflow adds the package to
  it, unless the release is already immutable.
* A draft release for the tag, such as one written by release-drafter, gets
  the package and is then published, keeping its notes. The store serves only
  public assets.
* Immutable releases and several apps on one tag do not combine. The first
  publish job publishes the release, which locks it for the next app. Give
  each app its own tag instead.
* Re-running the publish job registers the same package again, which is safe.
  Re-running the whole workflow builds a new package. If the release already
  has a different file of that name, delete it first.
* When the store refuses a package, the workflow deletes the asset it just
  uploaded, so a fixed re-run can attach a new one.

## Troubleshooting

| Error | Fix |
|---|---|
| `requesting 'id-token: write', but is only allowed 'id-token: none'` | Grant `id-token: write` and `contents: write` in the calling workflow. |
| `PUBLISHER_UNKNOWN` | First release: the account that pushed the tag, or the repository owner, must sign in to the store once. |
| `VERSION_NOT_MONOTONIC` | Bump `versionCode` in `AppScope/app.json5` and push a new tag. |
| `DUPLICATE_FILE` | This exact package is already published. |
| `… is owned by another publisher` (403) | Another publisher already published this bundle name. |
| `REPOSITORY_MISMATCH` | The app is bound to a different repository. A store admin can rebind it, for example after a transfer. |
| `No OpenHarmony SDK release for API N` | `compileSdkVersion` names an API level with no public OpenHarmony SDK. |
| `HarmonyOS projects are not supported` | Set `runtimeOS` to `OpenHarmony` for the product. |
| `expected one signed module for …` | Signing failed or the product name is wrong. Check the build step's log. |

## Other CI systems

Without GitHub Actions, create an API token at
`https://store.iparo.app/publisher/tokens` and upload with it. Token uploads
do not get the badge.

## Maintaining this workflow

* The store trusts the refs in its `TRUSTED_WORKFLOW_REFS` setting, currently
  `eclipse-oniro4openharmony/oniro-app-store-actions/.github/workflows/publish.yml@refs/tags/v1`.
  Releasing a compatible change means moving the `v1` tag. An incompatible one
  gets a new tag, such as `v2`, which the store adds to that setting.
* Everyone with write access to this repository can move `v1`, so they are
  all inside the badge's trust boundary. That is deliberate: see the store's
  design doc, `docs/design/github-ci-publishing.md`, section 7.
* Third-party actions are pinned by commit sha to releases at least two weeks
  old. `oniro-app` is pinned by version in `ONIRO_APP_VERSION`. Its own
  dependencies, the command-line tools and the SDK are not pinned by hash, so
  `job_workflow_sha` identifies the pipeline's steps, not every byte of the
  toolchain.
* Keep everything the publish job runs inside `publish.yml`. The store records
  `job_workflow_sha`, which then identifies the whole pipeline.
* `lint.yml` runs actionlint, with shellcheck, on every change.
