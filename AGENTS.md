# Repository Guidelines

## Purpose and layout

This is the organization's standalone `.github` repository. Its primary content
is the public homelab overview, not the application or infrastructure configuration.

- `profile/README.md`: organization profile displayed on GitHub.
- `docs/diagrams/*.d2`: editable architecture and pipeline diagram sources.
- `docs/diagrams/icons/`: local icons and their source/attribution README.
- `docs/diagrams/`: generated SVG and light/dark PNG assets.
- `scripts/render-diagrams.sh`: diagram renderer.
- `scripts/check-readme-images.py`: local documentation image checks.
- `.github/workflows/diagrams.yml`: rendering and image-validation CI.

## Editing conventions

Keep descriptions consistent with the sibling `packer`, `terraform`, `ansible`,
and `apps` repositories. Verify workflow names, counts, architecture claims, and
pipeline behavior against current source rather than copying potentially stale
profile prose. Each sibling is a separate Git repository.

Edit D2 sources and regenerate their assets together. Preserve the profile's
`<picture>` elements and light/dark images. Image paths in `profile/README.md`
are relative to `profile/`, so diagram references use `../docs/diagrams/`.
Preserve icon attribution and avoid adding secrets or private access details to
the public profile.

## Validation

Run from the repository root:

```sh
python3 scripts/check-readme-images.py
```

For diagram changes, first run `bash scripts/render-diagrams.sh`. It requires
`rsvg-convert` plus D2 (use the version pinned in the script/CI) or Docker for the
pinned D2 image. Inspect the resulting light and dark PNGs and include the SVG
and PNG outputs with the source change. The renderer's `--check` option renders
again and checks Git cleanliness, so it will flag intended uncommitted diagram edits too.

There is no application build or test suite here. For prose-only changes, review
links, image references, and factual consistency. Report validation results and
any rendering dependencies that were unavailable.
