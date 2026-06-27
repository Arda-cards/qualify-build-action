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
