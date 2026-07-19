# Changelog

Maps each release of this repo to the `ghcr.io/jamedina09/personal-elm-fates-image`
tag it's meant to be used with. The image itself is built from
[ELM-FATES-PERSONAL-CONTAINER](https://github.com/jamedina09/ELM-FATES-PERSONAL-CONTAINER)
— see that repo's own `CHANGELOG.md` for the underlying FATES/host-model/CIME versions.

## v1.1.1 — 2026-07-19

- The `Author identity unknown` warning documented in v1.1.0 is fixed as of
  the current `sci.1.92.3_api.44.1.0`/`latest` image (same version tags,
  updated image content — `podman pull` to get it). No repo files besides
  this changelog and the README Troubleshooting wording changed.

## v1.1.0 — 2026-07-19

- `latest` now points to image tag `sci.1.92.3_api.44.1.0` (was
  `sci.1.76.4_api.35.1.0` — that version remains pullable by its explicit tag,
  set `IMAGE_TAG=sci.1.76.4_api.35.1.0` in `.env` to stay on it).
- No changes needed to `compose.yaml`, `.env.example`, or
  `sample_script/e3sm_fates_test.sh` for this bump — verified via smoke test
  in the build repo using the exact same sample script this repo ships.
- Documented a new cosmetic (non-blocking) `Author identity unknown` warning
  from CIME 6.1.173's provenance auto-commit — see README Troubleshooting.

## v1.0.0 — 2026-07-18

- Works with image tag `sci.1.76.4_api.35.1.0` (and `latest`, at the time).
- Initial release: `compose.yaml`, `.env.example`, and `sample_script/e3sm_fates_test.sh`.
