# Changelog

## 1.0.0 – 2026-11-13

First public release.

- `webgauge <url>` checks a website and reports a score from 0 to 100 with a grade (Excellent, Good, Needs improvement, Poor) and a remediation hint for every warning and failure.
- Text output (default) and JSON output (`--format json`, `schemaVersion` 1), `--min-score` for CI (exit code 1 below the threshold) and `--pages` to also analyse up to 49 pages from the sitemap.
- 56 checks in four categories:
  - **Availability:** `http-status`, `response-time`, `https-redirect`, `soft-404`, `redirect-chain`, `www-consistency`.
  - **Security:** `https`, `strict-transport-security`, `content-security-policy`, `x-content-type-options`, `referrer-policy`, `permissions-policy`, `tls-certificate`, `clickjacking`, `cookie-flags`, `mixed-content`, `subresource-integrity`, `form-action-https`, `info-disclosure`, `cross-origin-isolation`, `security-txt`.
  - **SEO:** `title`, `meta-description`, `h1`, `canonical-url`, `viewport-meta`, `indexable`, `open-graph`, `structured-data`, `hreflang`, `charset`, `doctype`, `duplicate-titles`, `duplicate-descriptions`, `robots-txt`, `sitemap-xml`.
  - **Accessibility** (static, with WCAG 2.2 references): `html-lang`, `html-lang-valid`, `image-alt`, `heading-structure`, `empty-heading`, `skip-link`, `list-structure`, `duplicate-id`, `link-name`, `button-name`, `form-label`, `iframe-title`, `landmarks`, `aria-valid`, `zoom-allowed`, `tabindex`, `meta-refresh`, `table-headers`, `autocomplete`, `media-autoplay`.
- Sites that block automated visitors (401/403/429, bot-protection challenge or `robots.txt` Disallow for `WebGauge`/`*`) are not scored: WebGauge prints `Access not allowed: <reason>` and exits with code 3. It identifies itself as `WebGauge/<version>` and never tries to bypass the protection.

Developed under the working name SiteCheck; nothing was published under that name.
