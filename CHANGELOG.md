# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Comprehensive design document (`DESIGN.md`) covering architecture, sensor input layer,
  sequencer engine, output layer, hardware platforms, and software architecture
- Technical specifications (`SPECS.md`) with CV/Gate voltage standards, DAC selection,
  output stage circuits, gate/trigger circuits, sensor hardware specs, data source
  specifications, protection circuits, MIDI electrical specs, and calibration procedures
- Go language choice with module structure and HAL interface definitions
- CI/CD pipeline specification using GitHub Actions, golangci-lint, fuzz testing,
  Docker Compose integration tests, and acceptance test framework
- Build artifact specifications: cross-compiled binaries, `.deb` packages, OCI containers
- Release process using GoReleaser with semantic versioning (`v0.1.0` start)
- Conventional Commits convention for changelog generation
- Security review: credential management via environment variables, non-root containers,
  distroless base images, log redaction, read-only database users
- AGENTS.md with contributor guidelines for changelog maintenance and commit conventions
