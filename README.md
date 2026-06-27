# qualify-build-action

[![ci](https://github.com/Arda-cards/qualify-build-action/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/Arda-cards/qualify-build-action/actions/workflows/ci.yaml?query=branch%3Amain)
[CHANGELOG.md](CHANGELOG.md)

This action analyzes the GitHub event, the target ref and the project changelog to decide whether the workflow should run a test build or publish artifacts.

This action expects the project to have been checked out already in the `github.workspace` and will look for:

| file                         | required | description                                     |
|------------------------------|----------|-------------------------------------------------|
| `CHANGELOG.md`               | yes      | Contains the version extracted by `clq-action`. |
| `.github/clq/changemap.json` | yes      | Configures changelog version extraction.        |

## Build qualification

The action first determines the build target by running `gh ruleset check` against the pull request base ref or the triggering ref.
If the ruleset output contains the configured workflow name, the target is `release`; otherwise it is `feature`.

| target    | description                                                               |
|-----------|---------------------------------------------------------------------------|
| `feature` | Default target for refs that do not require release validation.           |
| `release` | Target for refs whose ruleset includes the configured release validation. |

It then combines the workflow event and target into a trigger classification.

| trigger                          | description                                                                              |
|----------------------------------|------------------------------------------------------------------------------------------|
| `push_to_feature_branch`         | A push to a ref that does not require release validation.                                |
| `push_to_release_branch`         | A push to a branch whose ruleset includes the `context:validate-release`.                |
| `pull_request_to_feature_branch` | A pull request targeting an unprotected branch.                                          |
| `pull_request_to_release_branch` | A pull request targeting a protected branch; the changelog must contain a release value. |

Only `push` and `pull_request` events are supported.

It then extracts the changelog version and decides the build kind.

| condition                                                      | kind      | version output                                      |
|----------------------------------------------------------------|-----------|-----------------------------------------------------|
| Push to a release branch with a released version               | `publish` | The changelog version.                              |
| Push to a feature branch with a feature version                | `publish` | The changelog version plus the GitHub run identity. |
| Pull request targeting a protected branch with a release value | `test`    | Not set.                                            |
| Any other valid feature branch workflow                        | `test`    | Not set.                                            |

Release targets must use a changelog version whose `clq-action` status is `released`.
Feature branch publish versions must match `major.minor.patch-user-issue`, with an optional suffix. Their published version is written as `version-run_id.run_number.run_attempt`.

## Inputs

| name                           | default                                         | description                                                           |
|--------------------------------|-------------------------------------------------|-----------------------------------------------------------------------|
| `feature_branch_version_regex` | `^[0-9]+(\.[0-9]+){2}(-[[:alnum:]]+){2}(-.+)?$` | A regular expression that identify feature branch versions.           |
| `workflow_name`                | `context:validate-release`                      | Name of the workflow context used to identify release-protected refs. |

## Outputs

| name      | description                                                                                                                                              |
|-----------|----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `kind`    | Build kind to run: `publish` when the workflow should produce published artifacts, otherwise `test`.                                                     |
| `tag`     | Version to tag, that is the version prefixed with a `v`. Set for publish builds; omitted for test builds.                                                |
| `target`  | Target of the build: `feature` or `release`.                                                                                                             |
| `trigger` | Build trigger classification: `push_to_feature_branch`, `push_to_release_branch`, `pull_request_to_release_branch`, or `pull_request_to_feature_branch`. |
| `version` | Version to publish. Set for publish builds; omitted for test builds.                                                                                     |

## Usage

```yaml
jobs:
  qualify-build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      kind: ${{ steps.qualify-build.outputs.kind }}
      version: ${{ steps.qualify-build.outputs.version }}
    steps:
      - uses: actions/checkout@v7
      - id: qualify-build
        uses: Arda-cards/qualify-build-action@v1
        env:
          GH_TOKEN: ${{ github.token }}

  build:
    runs-on: ubuntu-latest
    needs: qualify-build
    steps:
      - run: ./gradlew build
      - if: needs.qualify-build.outputs.kind == 'publish'
        run: ./gradlew publish -Pversion=${{ needs.qualify-build.outputs.version }}
```

## Permission Required

```yaml
permissions:
  contents: read
```
