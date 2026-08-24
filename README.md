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

The action first determines the build target by running `gh ruleset check` against the pull request base ref or the triggering ref name.
If the ruleset output contains the configured workflow name, the branch is *protected* and the target is `release`; otherwise it is `feature`.

| target    | description                                                               |
|-----------|---------------------------------------------------------------------------|
| `feature` | Default target for refs that do not require release validation.           |
| `release` | Target for refs whose ruleset includes the configured release validation. |

It then combines the workflow event and target into a trigger classification.

| trigger                          | description                                                                              |
|----------------------------------|------------------------------------------------------------------------------------------|
| `push_to_feature_branch`         | A push to a ref that does not require release validation.                                |
| `push_to_release_branch`         | A push to a protected branch.                                                            |
| `pull_request_to_feature_branch` | A pull request targeting an unprotected branch.                                          |
| `pull_request_to_release_branch` | A pull request targeting a protected branch; the changelog must contain a release value. |
| `merge_group_to_feature_branch`  | A merge queue entry whose destination branch does not require release validation.        |
| `merge_group_to_release_branch`  | A merge queue entry destined for a protected branch.                                     |

Only `push`, `pull_request` and `merge_group` events are supported.

It then extracts the changelog version and decides the build kind.

| condition                                                      | kind      | version output                                      |
|----------------------------------------------------------------|-----------|-----------------------------------------------------|
| Push to a release branch with a released version               | `publish` | The changelog version.                              |
| Push to a feature branch with a feature version                | `publish` | The changelog version plus the GitHub run identity. |
| Pull request targeting a protected branch with a release value | `test`    | Not set.                                            |
| Any other valid feature branch workflow                        | `test`    | Not set.                                            |
| A merge queue entry                                            | `test`    | Not set.                                            |

Release targets must use a changelog version whose `clq-action` status is `released`.
Feature branch publish versions must match `major.minor.patch-user-issue`, with an optional suffix. Their published version is written as `version-run_id.run_number.run_attempt`.

## Feature-build markers

A feature build publishes under a version carrying a marker that identifies the branch it came from.
Where the marker comes from depends on how the repository keeps its changelog.

| repository model                                                  | `feature_marker`       | `derive_feature_marker`            | how a feature build is signalled                                                                       |
|-------------------------------------------------------------------|------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| *Direct `CHANGELOG.md` editing*                                   | unset                  | `true` (default; never consulted)  | A feature version — `major.minor.patch-user-issue` — written into `CHANGELOG.md`.                      |
| *`CHANGELOG` in PR Body or `.changelog` directory, direct merge*  | unset                  | `true`                             | A `feature-build:` key in the frontmatter of the single `.changelog` entry.                             |
| *`CHANGELOG` in PR Body or `.changelog` directory with Merge Queue* | supplied by the caller | `false`                            | A `feature-build:` key in the entry belonging to the pull request being merged, resolved by the caller. |

The first row is the original mechanism and sets neither input: with no marker, the action matches the
changelog version itself against `feature_branch_version_regex`.

The other two exist because a composed changelog carries no version on the branch, leaving that regular
expression nothing to read, so the signal moves to the file the author is already writing. The third row supplies the
marker rather than deriving it because a merge queue stages the entries of every pull request in the batch,
and the derivation refuses a directory holding more than one.

The two inputs name one choice. Supplying a marker while `derive_feature_marker` is `true` fails the build
with an error naming both, because an empty marker means *this branch is not a feature build* and must not
also be read as *the caller said nothing*.

Markers are restricted to `[A-Za-z0-9._-]`.

## Inputs

| name                           | default                                         | description                                                                                     |
|--------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------|
| `changelog_dir`                | `.changelog`                                    | Directory holding per-pull-request changelog files, read when `derive_feature_marker` is `true`. |
| `derive_feature_marker`        | `true`                                          | Whether to read the marker from `changelog_dir` instead of taking it from `feature_marker`.     |
| `feature_branch_version_regex` | `^[0-9]+(\.[0-9]+){2}(-[[:alnum:]]+){2}(-.+)?$` | A regular expression that identifies feature branch versions.                                   |
| `feature_marker`               | *(empty)*                                       | The feature-build marker for this ref; requires `derive_feature_marker` to be `false`.          |
| `validate_against_base`        | `true`                                          | Whether a pull request must introduce exactly one new changelog version.                        |
| `workflow_name`                | `context:validate-release`                      | Name of the workflow context used to identify release-protected refs.                           |

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
        uses: Arda-cards/qualify-build-action@v2
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
