# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic Versioning](http://semver.org/spec/v2.0.0.html).

## [v1.0.4] - 2025-01-02

### Added
* add `-q` CLI option: quiet mode, suppresses all CLI output

## [v1.0.3] - 2024-09-30

### Fixed
* fails to resume upgrade process after reboot ([#1])
* incorrect service name in README ([#2])

## [v1.0.2] - 2024-03-31

### Added
* check diskspace on / and /usr

### Changed
* minor documentation updates

## [v1.0.1] - 2023-11-29

### Fixed
* freebsd-update may get stuck when it restarts services

## v1.0.0 - 2023-11-28
Initial release

[Unreleased]: https://github.com/fraenki/f-upgrade/compare/v1.0.4...HEAD
[v1.0.4]: https://github.com/fraenki/f-upgrade/compare/v1.0.3...v1.0.4
[v1.0.3]: https://github.com/fraenki/f-upgrade/compare/v1.0.2...v1.0.3
[v1.0.2]: https://github.com/fraenki/f-upgrade/compare/v1.0.1...v1.0.2
[v1.0.1]: https://github.com/fraenki/f-upgrade/compare/v1.0.0...v1.0.1
[#2]: https://github.com/fraenki/f-upgrade/pull/2
[#1]: https://github.com/fraenki/f-upgrade/pull/1
