# changelog

[![Keep a Changelog](https://img.shields.io/badge/Keep%20a%20Changelog-1.0.0-informational)](https://keepachangelog.com/en/1.0.0/)
[![Semantic Versioning](https://img.shields.io/badge/Semantic%20Versioning-2.0.0-informational)](https://semver.org/spec/v2.0.0.html)
![clq validated](https://img.shields.io/badge/clq-validated-success)

Keep the newest entry at top, format date according to ISO 8601: `YYYY-MM-DD`.

Categories, defined in [changemap.json](.github/clq/changemap.json):

- *major* release trigger:
  - `Changed` for changes in existing functionality.
  - `Removed` for now removed features.
- *minor* release trigger:
  - `Added` for new features.
  - `Deprecated` for soon-to-be removed features.
- *bugfix* release trigger:
  - `Fixed` for any bugfixes.
  - `Security` in case of vulnerabilities.

## [2.2.1] - 2026-08-25

### Fixed

- The feature-build marker step read `CHANGELOG_DIR`, an environment variable no longer
  set, so under `set -u` every build using this action failed before it could classify
  anything. The check now reads the input it validates.

## [2.2.0] - 2026-08-24

### Added

- `feature_marker`, so a caller can state the feature-build marker for the ref rather than
  have it derived from `changelog_dir`. The derivation cannot see a marker written in a
  pull-request body, and it refuses a changelog directory holding more than one entry —
  which is what a merge queue stages whenever it batches two pull requests.
- `derive_feature_marker`, selecting between the two sources and defaulting to `true` so
  existing callers are unchanged. Supplying a marker while it is `true` is now refused
  with an error naming both inputs: an empty marker means the branch is not a feature
  build, and must not also read as the caller having said nothing.

### Security

- The feature-build marker is restricted to `[A-Za-z0-9._-]` and passed through the
  environment rather than interpolated into the script, closing a path by which a marker
  taken from pull-request data could inject workflow commands or step outputs.

## [2.1.0] - 2026-08-06

### Added

- Classify a build running from a merge queue rather than rejecting it. `merge_group` was not among the
  recognised events, so a queued entry failed outright and the queue dropped it, which made a merge queue
  unadoptable in any repository using this action. Such a build reports the trigger
  `merge_group_to_release_branch`, or `merge_group_to_feature_branch` where the destination branch's
  ruleset does not require the configured `workflow_name` check. Either way it qualifies as a test build,
  like a pull request: the queue gates a merge, it does not publish.
- New optional input `validate_against_base`, defaulting to `true`, passed through to `clq-action`. That
  check requires a pull request to introduce exactly one new changelog version, which is right where the
  author edits `CHANGELOG.md` and wrong where the changelog is composed after merge — there the pull
  request introduces none, and every pull request would be rejected. Repositories on the composed model
  set it to `false`; the guarantee is not lost, only relocated to a gate that refuses any edit to
  `CHANGELOG.md` at all.
- New optional input `changelog_dir`, defaulting to `.changelog`. A `feature-build:` key in the
  frontmatter of the single file there marks the branch as a feature build, and the published version
  becomes `<changelog version>-<marker>-<run>`. Where a repository composes its changelog after merge,
  the version in `CHANGELOG.md` carries no feature suffix for the old mechanism to read; the marker
  moves the signal to the file the author is already writing. A branch with no marker behaves exactly
  as before.
- Read the branch a build targets from the merge-queue payload when it is present. A queued entry runs on
  a temporary `gh-readonly-queue/...` ref, so neither `github.base_ref` nor `github.ref_name` names the
  branch being merged into, and the ruleset probe would conclude the target is unprotected.

## [2.0.0] - 2026-06-27

### Changed

- Renamed the output `mode` to `trigger`.

### Added

- New `trigger` for pull requests targeting feature branches: `pull_request_to_feature_branch`.
- New output `target` to indicate whether the build targets a `feature` or a `release` branch.
- New optional input `feature_branch_version_regex` to set a custom pattern for feature branch versions.
- New optional input `workflow_name` for a workflow that protected branches should be validated against.

## [1.0.1] - 2026-06-23

### Fixed

- To use GitHub CLI in a GitHub Actions workflow, set the `GH_TOKEN` environment variable.

## [1.0.0] - 2026-06-22

### Added

- Extracted logic from other projects.
