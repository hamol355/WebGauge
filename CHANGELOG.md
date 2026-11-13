# Changelog

## Unreleased

### Added

- Security: `tls-certificate`, `clickjacking`, `cookie-flags`, `mixed-content`, `subresource-integrity`, `form-action-https`, `info-disclosure`, `cross-origin-isolation`, `security-txt`.
- SEO: `indexable`, `open-graph`, `structured-data`, `hreflang`, `charset`, `doctype`, `duplicate-titles`, `duplicate-descriptions`; `canonical-url` now warns when the canonical points to another origin.
- Availability: `soft-404` (one extra request to a random path), `redirect-chain`, `www-consistency` (one extra request).
- Accessibility: `html-lang-valid`, `empty-heading`, `skip-link`, `list-structure`, `duplicate-id`; `image-alt` also covers `input[type=image]`, `<area href>` and `svg[role=img]`.

### Changed

- **Renamed to WebGauge.** The npm package and command are now `webgauge`, the User-Agent is `WebGauge/<version>`, robots.txt rules are matched against the `WebGauge` token, and the JSON field `meta.sitecheckVersion` is now `meta.webgaugeVersion`.
- **Scores change.** The new checks raise the maximum points of their categories. Category weights are unchanged and the category score is normalised against them, so the same site can get a somewhat different score than before.
