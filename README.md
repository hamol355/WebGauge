# WebGauge

WebGauge is a command-line tool that checks a website for availability, security, SEO and accessibility and reports a score from 0 to 100, with advice on how to fix each problem.

## Installation

Requires Node.js 22 or later.

```bash
npx webgauge https://example.com     # run without installing
npm install -g webgauge              # or install the webgauge command globally
```

From source:

```bash
npm install
npm run build
node dist/cli/index.js https://example.com
```

During development, run from source with `npm run dev -- https://example.com`.

## Usage

```text
webgauge <url> [--format text|json] [--min-score <0-100>] [--pages <1-50>] [--help] [--version]
```

- If the URL has no scheme, `https://` is assumed.
- `--format text` (default) prints a human-readable report; `--format json` prints the JSON report described below.
- `--min-score <n>` makes the command exit with code 1 when the score is below `n` (useful in CI). The report is still printed.
- `--pages <n>` (default 1) also analyses up to `n - 1` more pages from the site's sitemap (max 50). Only pages on the same origin as the start page are fetched, `robots.txt` is respected, and blocked pages are skipped with a reason instead of stopping the run.
- Colors are used only when stdout is a TTY and `NO_COLOR` is not set.
- Set `DEBUG=webgauge` to print stack traces for internal errors.

The report goes to **stdout**; errors and diagnostics go to **stderr**, so `--format json` can always be piped, e.g. `webgauge example.com --format json | jq .score`.

### Example

```text
$ webgauge example.com
WebGauge
────────────────────────────────────────

URL: https://example.com/

Availability (11/15)
  ✓ Responded with HTTP 200
  ✓ Response time 91 ms
  ✗ http://example.com/ is served over HTTP without redirecting to HTTPS
    → Redirect all http:// requests to https:// with a 301 response (e.g. "return 301 https://$host$request_uri;" in nginx).
  ✓ Missing pages respond with HTTP 404
  ✓ No redirects to the final URL
  ⚠ www.example.com also serves the site without redirecting to example.com (duplicate content)
    → Pick one of www and non-www and 301-redirect the other to it.

Security (17/35)
  ✓ Page is served over HTTPS
  ✗ Missing Strict-Transport-Security header
    → Add header: Strict-Transport-Security: max-age=15552000; includeSubDomains
  ✗ Missing Content-Security-Policy header
    → Add header: Content-Security-Policy: default-src 'self' (and avoid 'unsafe-inline' and 'unsafe-eval')
  ✓ Certificate is valid for 42 more days, TLSv1.3
  …

SEO (11/20)
  ✓ Title is 14 characters
  ✗ Missing meta description
    → Add <meta name="description" content="..."> to the <head>.
  – Skipped duplicate-titles: needs --pages 2 or more
  …

Accessibility (26/30)
  ✓ Document language is "en" (WCAG 3.1.1)
  ✗ Page has no main landmark (WCAG 1.3.1, 2.4.1)
    → Wrap the primary content in a <main> element so assistive technology can jump to it.
  …

────────────────────────────────────────
Score: 65/100 (Needs improvement)
```

(Shortened; the real report lists every check.) `✓` pass, `⚠` warn, `✗` fail, `–` skipped; `→` is the remediation.

### Exit codes

| Code | Meaning                                                     |
| ---- | ----------------------------------------------------------- |
| 0    | OK                                                          |
| 1    | Score is below `--min-score`                                |
| 2    | Invalid input (bad URL, unknown option, bad option value)   |
| 3    | The site could not be reached (including timeouts and redirect loops), **blocks automated visitors** (`Access not allowed: <reason>` on stderr, no report), or an internal error occurred |

## Checks

| Category     | Checks                                                                                                           |
| ------------ | ---------------------------------------------------------------------------------------------------------------- |
| Availability | `http-status`, `response-time`, `https-redirect`, `soft-404`, `redirect-chain`, `www-consistency`                                                     |
| Security     | `https`, `strict-transport-security`, `content-security-policy`, `x-content-type-options`, `referrer-policy`, `permissions-policy`, `tls-certificate`, `clickjacking`, `cookie-flags`, `mixed-content`, `subresource-integrity`, `form-action-https`, `info-disclosure`, `cross-origin-isolation`, `security-txt` |
| SEO          | `title`, `meta-description`, `h1`, `canonical-url`, `viewport-meta`, `indexable`, `open-graph`, `structured-data`, `hreflang`, `charset`, `doctype`, `duplicate-titles`, `duplicate-descriptions`, `robots-txt`, `sitemap-xml`   |
| Accessibility | `html-lang`, `html-lang-valid`, `image-alt`, `heading-structure`, `empty-heading`, `skip-link`, `list-structure`, `duplicate-id`, `link-name`, `button-name`, `form-label`, `iframe-title`, `landmarks`, `aria-valid`, `zoom-allowed`, `tabindex`, `meta-refresh`, `table-headers`, `autocomplete`, `media-autoplay` |

Each check ends as `pass`, `warn`, `fail`, `error` (the check itself crashed; other checks are unaffected) or `skip` (the check did not apply). If the page does not respond with 2xx, the checks that read the page content (`title`, `meta-description`, `h1`, `canonical-url`, `viewport-meta` and all accessibility checks) are skipped, since an error or block page says nothing about the real page. `warn` and `fail` come with a `remediation` hint.

The accessibility checks are static: they read the HTML as served and each lists the WCAG 2.2 success criteria it relates to (`wcag` in the JSON, `(WCAG 2.4.4, 4.1.2)` in the text output).

## Score model

Each check has a maximum number of points. `pass` earns all of them, `warn` half (rounded down), and `fail` none. `error` and `skip` checks are left out of the score entirely; for `error` the CLI also prints a warning on stderr. A category where every check was skipped scores 0 (shown as "not checked"), and the other categories are not scaled up to compensate, so a site whose page could not be checked cannot get a high score.

- **Category score** = earned points / possible points of the scored checks × category weight, rounded to an integer. Weights: Availability 15, Security 35, SEO 20, Accessibility 30 (preliminary).
- **Total score** = the sum of the _unrounded_ category scores, rounded once. The rounded category scores therefore do not always add up to the total.
- **Grade:** 90–100 Excellent, 75–89 Good, 50–74 Needs improvement, 0–49 Poor.

The weights, grade boundaries and check thresholds are constants in `src/scoring/score.ts` and `src/checks/`.

### Response time

Response time is the time from sending the final request until the response headers are received. It excludes earlier redirects, reading the body and parsing. It can include DNS and TLS setup; when a redirect stays on the same host, the connection is reused and these are not included.

## JSON output

The JSON output is a public interface (`schemaVersion` is `1`):

```json
{
  "schemaVersion": 1,
  "url": "https://example.com/",
  "finalUrl": "https://example.com/",
  "score": 73,
  "grade": "Needs improvement",
  "categories": {
    "availability": { "score": 15, "max": 15 },
    "security": { "score": 18, "max": 35 },
    "seo": { "score": 14, "max": 20 },
    "accessibility": { "score": 26, "max": 30 }
  },
  "checks": [
    {
      "id": "robots-txt",
      "category": "seo",
      "status": "fail",
      "message": "robots.txt returned HTTP 404",
      "remediation": "Add a robots.txt at the site root, ...",
      "points": 0,
      "maxPoints": 3
    },
    {
      "id": "link-name",
      "category": "accessibility",
      "status": "warn",
      "message": "1 of 10 links have no accessible name",
      "remediation": "Give every link descriptive text, ...",
      "points": 2,
      "maxPoints": 4,
      "wcag": ["2.4.4", "4.1.2"]
    }
  ],
  "meta": {
    "webgaugeVersion": "1.0.0",
    "analyzedAt": "2026-01-01T12:00:00.000Z",
    "responseTimeMs": 123,
    "truncated": false,
    "pagesAnalyzed": 1
  }
}
```

`status` is one of `pass`, `warn`, `fail`, `error`, `skip`. `remediation` is present only when there is something to fix.

With `--pages` greater than 1, `checks` holds one aggregated result per check id and an extra `pages` array lists each page (`url`, `status`, `skipped` with a reason if it was not analysed, and its `checks` as id/status/message). `meta.pagesAnalyzed` counts the start page plus the extra pages that were not skipped.

## Limitations

- The page is fetched once and analysed as served. JavaScript is not executed, so content rendered client-side (SPAs) is not seen by the SEO and accessibility checks.
- Sites that do not allow automated visitors are not scored. If the start page answers 401, 403 or 429, serves a bot-protection challenge (e.g. Cloudflare "Just a moment"), or `robots.txt` disallows the page for `WebGauge` or `*`, WebGauge prints `Access not allowed: <reason>` to stderr, writes nothing to stdout and exits with code 3. WebGauge identifies itself as `WebGauge/<version>` and never tries to bypass the protection. Note that many sites have `User-agent: *` / `Disallow: /` and only admit named search engines; ask the site owner to allow the `WebGauge` user agent. A broken `robots.txt` (4xx, 5xx, unreachable) is treated as allowing everything; on 5xx WebGauge prints a `Notice:` line to stderr. A site that answers 200 with a reduced page for bots cannot be detected reliably and is analysed as served, so the score reflects what WebGauge received rather than what a browser would see. 404 and 5xx responses are not blocks and get a normal report with a failed `http-status`.
- Requests time out after 10 s (5 s for `robots.txt`, `sitemap.xml` and the HTTP→HTTPS probe), follow at most 5 redirects, and read at most 2 MB of the page (1 MB for `robots.txt` and the sitemap). Larger pages are analysed truncated; the report then has `meta.truncated: true` and the text output says so.
- The sitemap check verifies that a sitemap exists and looks like XML; it does not validate its contents.
- Accessibility checks cover only what static HTML shows. Colour contrast, focus indicators and keyboard traps need a real browser and are not checked, so a good accessibility score does not mean the site is WCAG compliant.
- `tls-certificate` makes its own TLS handshake (no HTTP request) and validates the chain against Node's built-in CA store. Node does not fetch missing intermediate certificates the way browsers do, so a server that omits them fails here even if browsers accept it.
- `soft-404` requests one random path (`/webgauge-404-<random>`) on the site to see whether a missing page answers 404. It is skipped if `robots.txt` disallows the path.
- `www-consistency` makes one request to the www/non-www counterpart of the host. It only runs for `www.` hosts and two-label hosts like `example.com`; hosts such as `example.co.uk` are skipped because WebGauge has no public suffix list. `security-txt` requests `/.well-known/security.txt` (skipped if `robots.txt` disallows it). `duplicate-titles` and `duplicate-descriptions` need `--pages 2` or more and are skipped otherwise.
- Performance and Core Web Vitals are out of scope for v1.

## Development

```bash
npm test        # Vitest (unit, module and CLI integration tests; localhost only)
npm run lint    # ESLint
npm run format  # Prettier
npm run build   # tsc → dist/
```

The CLI integration tests build the project and run `dist/cli/index.js` against a local test server.

## License

MIT
