# Changelog

## v0.4.1 — 2026-06-10

### Fixed
- **jq 1.6 compatibility (default path):** the standard-anchor jq object
  literal carried a trailing comma after `force_new: $force_new` — fine
  on jq 1.7 (ubuntu-24.04) but a syntax error on jq 1.6 (ubuntu-22.04
  runners and common self-hosted boxes), which killed the anchor step
  under `set -euo pipefail`. The provenance path was unaffected.
  (Probe finding S2-F2, 2026-06-10.)

### Added
- CI guard `jq16-compat` in the smoke workflow: rejects any trailing
  comma before a closing brace in `action.yml` so the class of bug
  can't land again (local jq 1.7 silently accepts them).

## v0.4.0 — 2026-06-09

Canonical-vocabulary release (Satsignal vocabulary sunset, decision 0046).

### Changed
- **Wire request:** the anchor body now sends the canonical `folder_slug`
  key (was `matter_slug`) on both `/api/v1/anchors` and
  `/api/v1/provenance/anchor`. Current Satsignal servers accept legacy
  keys as silent aliases, but tooling now speaks canonical.
- **Wire response:** canonical `proof_id` / `proof_url` are read as the
  primary keys (current servers emit canonical keys only); the legacy
  `bundle_id` / `receipt_url` fallback is retained for self-hosted
  servers older than the sunset.
- Logs, step summary, and input/output descriptions use the canonical
  folder/proof vocabulary. `User-Agent` is now `satsignal-action/v0.4`.
- README examples and error-code references are canonical
  (`folder_not_found`).

### Unchanged (compatibility)
- **Inputs:** `folder:` remains the documented input; the legacy
  `matter:` input keeps working forever as a silent alias (the step
  fails if both are set to different non-empty values, mirroring the
  API's `conflicting_alias` behavior).
- **Outputs:** the legacy `bundle_id` / `receipt_url` outputs stay
  populated with the same values as `proof_id` / `proof_url` — outputs
  are this action's public contract; changes are additive only.
- Default slug when neither input is set is still `github-actions`.

### Compatibility note
Self-hosted Satsignal servers older than the 2026-06 sunset only accept
`matter_slug`; pin `Steleet/satsignal-action@v0.3.0` against those.

## v0.3.0 — 2026-05

- Provenance mode: `provenance: 'true'` wraps the file digest in a
  `satsignal.provenance.v1` manifest with typed-authority metadata from
  the GitHub Actions environment.

## v0.2.0 — 2026-05

- Additive folder/proof vocabulary aliases: new `folder:` input
  (aliasing `matter:`) and new `proof_id` / `proof_url` outputs
  (aliasing `bundle_id` / `receipt_url`), compat-preserving.
- `User-Agent: satsignal-action/v0` on the anchor POST.

## v0.1.0 — 2026-05

- Initial release: composite action that sha256s an artifact and anchors
  it via `POST /api/v1/anchors`, emitting receipt step outputs.
