# Agents & Contributors Guide

Guidelines for all contributors — human and AI — working on Grafophone.

## Changelog

The project maintains a `CHANGELOG.md` following the [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format.

### Rules

1. **Every commit** that adds, changes, fixes, or removes user-facing behavior **must** include a corresponding entry under `## [Unreleased]` in `CHANGELOG.md`.
2. Group entries under the standard section headings: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.
3. Write entries from the user's perspective — describe *what changed*, not implementation details.
4. Do **not** create entries for internal-only changes (refactors, test-only changes, CI tweaks) unless they affect the developer experience (e.g., new linter rules, changed build commands).
5. On release, the `[Unreleased]` section is promoted to a versioned heading (e.g., `## [0.3.0] - 2026-03-15`) by GoReleaser. Never manually edit released sections.

### Example Entry

```markdown
## [Unreleased]

### Added

- Prometheus data source adapter with PromQL query support
- Ring buffer with configurable depth per track

### Fixed

- MIDI note-off messages now send velocity 0 instead of 64
```

## Commit Messages

The project uses [Conventional Commits](https://www.conventionalcommits.org/) to drive automated changelog generation and semantic version bumps.

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types

| Prefix | Purpose | Changelog |
|---|---|---|
| `feat:` | New feature | **Features** |
| `fix:` | Bug fix | **Bug Fixes** |
| `perf:` | Performance improvement | **Performance** |
| `docs:` | Documentation only | excluded |
| `refactor:` | Code restructuring (no behavior change) | excluded |
| `test:` | Adding or updating tests | excluded |
| `ci:` | CI/CD pipeline changes | excluded |
| `chore:` | Maintenance, dependency bumps | excluded |

### Breaking Changes

Append `!` after the type or include `BREAKING CHANGE:` in the commit footer. This bumps the **MAJOR** version (once the project reaches `v1.0.0`).

```
feat!: replace YAML config with TOML

BREAKING CHANGE: Configuration files must be migrated from .yml to .toml format.
```

### Scopes

Use scopes to identify the subsystem. Examples:

- `feat(sequencer):` — sequencer engine changes
- `fix(cv):` — CV output fixes
- `feat(datasource):` — new or changed data source adapter
- `fix(midi):` — MIDI output fixes
- `ci(lint):` — linter configuration changes

## Semantic Versioning

The project follows [Semantic Versioning 2.0.0](https://semver.org/):

- **MAJOR** (`v1.0.0` → `v2.0.0`): Incompatible API or config changes
- **MINOR** (`v0.2.0` → `v0.3.0`): New features, backwards-compatible
- **PATCH** (`v0.3.0` → `v0.3.1`): Bug fixes, backwards-compatible

Pre-release versions: `v0.1.0-alpha.1`, `v0.2.0-beta.1`, `v1.0.0-rc.1`.

Backwards-compatibility guarantees begin at `v1.0.0`. Before that, minor versions may include breaking changes.

### Release Checklist

1. Ensure `CHANGELOG.md` has entries under `## [Unreleased]` for all included changes.
2. Ensure all CI checks pass on `main`.
3. Tag the release:
   ```bash
   git tag -a v0.3.0 -m "Release v0.3.0: <summary>"
   git push origin v0.3.0
   ```
4. GoReleaser (via GitHub Actions) handles building, packaging, and publishing automatically.

## Code Review

- All changes to `main` go through pull requests.
- PRs must pass CI (lint, test, build) before merge.
- Squash-merge is preferred to keep `main` history clean; the squashed commit message must follow Conventional Commits format.

## References

- [DESIGN.md](DESIGN.md) — Architecture and feature design
- [SPECS.md](SPECS.md) — Detailed technical specifications (including full CI/CD pipeline config)
