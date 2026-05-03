# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.3.0] - 2026-05-01

### Added

- Two-step model picker (`/vision-proxy pick`): provider first, then model. Replaces the single flat list of 400+ models.
- Current provider is shown first with a ★ marker and pre-selected — picker opens directly on the model list, no need to re-select the same provider every time.
- `← Change provider` option inside the model list to switch providers without restarting the picker.
- `🔍 Type to filter models…` option for providers with more than 8 models. Uses fuzzy character-order matching (e.g. `cs4` matches `Claude Sonnet 4.5`). Single matches are auto-selected.
- `fuzzyMatches()` helper exported from `internal.ts` with full test coverage.

### Changed

- Duplicated picker code between `/vision-proxy pick` and the interactive `Model:` row consolidated into a single `pickVisionModel()` function.

## [1.2.0] - 2026-05-01

### Added

- `/vision-proxy pick` sub-command. Lists vision-capable models from the registry with friendly names and provider tags via `ctx.ui.select`. Avoids typing canonical ids like `accounts/fireworks/models/kimi-k2p6`.
- Interactive `Model:` row in `/vision-proxy` config now opens the same vision-only picker (was raw text input).
- `friendlyModelLabel(config, registry)` helper. Status line and notifies now display `Kimi K2.6 [fireworks]` instead of `fireworks/accounts/fireworks/models/kimi-k2p6` when the registry knows the model.

### Changed

- "Model not found" error now points to `/vision-proxy pick` instead of `/vision-proxy model`.

## [1.1.0] - 2026-05-01

### Changed

- Settings (mode, model, context) now persist across sessions to `~/.pi/agent/vision-proxy.json`. Previously settings were stored only in session entries and lost when starting a new session. Config precedence (highest → lowest): environment variables → session entries → persistent file → defaults.

### Added

- `readPersistentFile()` / `writePersistentFile()` helpers for file-based config storage.
- `fileConfig` parameter on `resolveConfig()` to layer persisted file config between defaults and session entries.
- Tests for persistent file round-trip and layered config resolution.
