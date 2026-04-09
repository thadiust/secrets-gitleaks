# Changelog

All notable changes to **secrets-gitleaks** are documented in this file.

## [Unreleased]

## [1.0.3] — 2026-04-09

### Changed

- Stricter validation of **`working_directory`**, **`config_path`**, **`baseline_path`**, and **`sarif_filename`** (reject absolute paths and `..` segments).

## [1.0.2] — 2026-04-06

### Added

- **`write_sarif`** / **`sarif_filename`**: optional SARIF report for **GitHub Code Scanning**; output **`sarif_path`**.

### Changed

- **Gitleaks** Linux x64 tarball verified against upstream **`checksums.txt`** before extract.

## [1.0.1] — (earlier)

## [1.0.0] — (initial)

See git tags and commit history for details.
