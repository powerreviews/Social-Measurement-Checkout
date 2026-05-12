# Social-Measurement-Checkout — RepoDocs
_Generated on 2026-05-11_

## Summary

### Overview
This repo contains a single client-side JavaScript snippet (`checkout.js`) that customers embed on their order-confirmation page to fire a PowerReviews "Social Measurement" conversion beacon. The beacon loads `static.powerreviews.com/t/v1/tracker.js` and reports order/user metadata so the PowerReviews analytics platform can correlate reviews engagement with shopper conversion and (optionally) auto-build an order feed for review solicitation. It functions as a reference/sample integration distributed to merchants, not as a deployed service.

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | JavaScript (browser) | _Not specified_ |
| Framework | PowerReviews `tracker.js` (loaded from `static.powerreviews.com/t/v1/tracker.js`) | v1 |
| Database | _None_ | _N/A_ |
| Build Tool | _None_ | _N/A_ |
| CI/CD | GitHub Actions | _N/A_ |
| Cloud/Infra | GitHub-hosted runners (`ubuntu-latest`) | _N/A_ |

### Consumers
| Consumer | Type | How They Use It |
|----------|------|----------------|
| PowerReviews merchant clients | External (customer websites) | Paste/adapt the `checkout.js` snippet into their order-confirmation page to fire the conversion-tracking beacon |
| `static.powerreviews.com` (PowerReviews CDN, `tracker.js` v1) | External (PowerReviews-hosted asset) | The snippet loads `tracker.js` from the CDN, which then transmits conversion data back to PowerReviews analytics |
| GitHub Actions (`secrets-scan.yml`) | CI service | Runs a scheduled TruffleHog secrets scan on the repo and posts failures to Slack |
| TruffleHog (`edplato/trufflehog-actions-scan@master`) | Third-party GitHub Action | Performs regex-based secret scanning |
| Slack channel `github-token-scan` (via `rtCamp/action-slack-notify@v2.0.2`) | External service | Receives `@devops-team` alerts when secrets are detected |

### Dependencies on Org Repos
_No dependencies on other org repos detected._

### External Integrations
| Service | Purpose | Integration Type |
|---------|---------|-----------------|
| PowerReviews tracker (`static.powerreviews.com/t/v1/tracker.js`) | Conversion beacon / page-view tracking | SDK (script tag) |
| TruffleHog GitHub Action | Secrets scanning in CI | SDK (Action) |
| Slack webhook | CI failure notifications | Webhook (outbound) |

### Async & Scheduled Work
| Channel / Job | Type | Direction | Purpose |
|--------------|------|-----------|---------|
| `secret-scan` GitHub Actions workflow (cron `0 14 * * 1-5`) | Scheduled CI job | N/A (for jobs) | Weekday 14:00 UTC TruffleHog scan for committed secrets, alerting Slack on failure |

### Upgrade Alerts
| Dependency | Current Version | Issue | Severity |
|-----------|----------------|-------|----------|
| `actions/checkout@master` | `master` branch (floating) | Pinning a third-party Action to a moving `master` ref is a supply-chain risk; `actions/checkout` no longer maintains a `master` branch (renamed `main`), and recent guidance is to pin to a SHA or version tag | Critical |
| `edplato/trufflehog-actions-scan@master` | `master` branch (floating) | Third-party action pinned to a mutable branch — code can change under the workflow without review; the upstream action is unmaintained and uses the legacy TruffleHog v2 scanner (deprecated) | Critical |
| PowerReviews `tracker.js` v1 (`//static.powerreviews.com/t/v1/tracker.js`) | v1 | Protocol-relative URL (`//…`) is a legacy pattern, but the more pressing item is that the `t/v1/` path has been the integration surface unchanged since 2016 — verify whether v1 is still the supported tracker endpoint | Severe |

## API Reference

This repo exposes no programmatic API. It is a single HTML/JS snippet meant to be embedded in a merchant's order-confirmation page. The snippet calls into the externally hosted PowerReviews tracker library.

**Snippet contract — `checkout.js`**

Loads:
- `//static.powerreviews.com/t/v1/tracker.js`

Calls:
- `POWERREVIEWS.tracker.createTracker({ merchantGroupId })` — factory that returns a `tracker` instance scoped to a merchant group.
  - `merchantGroupId` *(string, required)* — PowerReviews-assigned merchant group identifier.
- `tracker.trackPageview("c", { … })` — fires a conversion (`"c"`) page-view event.

`trackPageview` payload fields used in the snippet:

| Field | Type | Description |
|-------|------|-------------|
| `merchantId` | string | PowerReviews merchant identifier |
| `locale` | string | BCP-47-style locale (e.g., `en_US`) |
| `merchantUserId` | string | Merchant's internal user ID |
| `marketingOptIn` | boolean | Whether the user opted in to marketing |
| `userEmail` | string | Shopper email |
| `userFirstName` | string | Shopper first name |
| `userLastName` | string | Shopper last name |
| `orderId` | string | Merchant's order identifier |
| `orderSubtotal` | string | Order subtotal (currency-agnostic, passed as string) |
| `orderNumberOfItems` | string | Item count |
| `orderItems` | array of arrays | Each item is a positional tuple: `[pageId, pageIdVariant, productName, qty, unitPrice]` |

Errors raised by the tracker are caught and logged via `window.console.log`.

## Architecture

### System-context diagram
```
                       (merchant order-confirmation page)
                       +-----------------------------------+
                       |  <script src="//static.power      |
                       |   reviews.com/t/v1/tracker.js">   |
                       |  POWERREVIEWS.tracker             |
                       |    .createTracker({...})          |
                       |    .trackPageview("c", {...})     |
                       +------------------+----------------+
                                          |
                                          | HTTPS (CDN load + beacon)
                                          v
                       +-----------------------------------+
                       | static.powerreviews.com (CDN)     |
                       |   /t/v1/tracker.js                |
                       +------------------+----------------+
                                          |
                                          v
                       +-----------------------------------+
                       | PowerReviews analytics platform   |
                       |   (conversion + order-feed data)  |
                       +-----------------------------------+

  Repo CI (independent of the runtime path):
  +-----------------------------+    cron 0 14 * * 1-5    +-----------------------+
  | GitHub Actions runner       |------------------------>| TruffleHog scan       |
  | (.github/workflows/         |                         +-----------+-----------+
  |  secrets-scan.yml)          |    on failure                       |
  +-----------------------------+------------------------>+-----------v-----------+
                                                          | Slack #github-token-  |
                                                          | scan (@devops-team)   |
                                                          +-----------------------+
```

### Key components
- `checkout.js` — the only runtime artifact: an HTML `<script>` block plus an IIFE that constructs a `POWERREVIEWS.tracker` and fires a `"c"` (conversion) page-view.
- `README.md` — describes the business purpose: tying review engagement to conversion, and optionally generating an automated order feed for solicitation.
- `.github/workflows/secrets-scan.yml` — scheduled security hygiene job.

### Data flow
1. Merchant renders the order-confirmation page with the snippet inlined and the per-order placeholders substituted server-side (or via templating).
2. Browser loads `tracker.js` from the PowerReviews CDN.
3. The IIFE calls `createTracker` and then `trackPageview("c", {...})`, which transmits the order/user payload to PowerReviews analytics (the destination URL is encapsulated inside `tracker.js`; this repo does not show it).
4. PowerReviews ingests the beacon as conversion data and, per the README, can also derive an order feed for review solicitation.

### CI/CD tooling
GitHub Actions (detected via `.github/workflows/secrets-scan.yml`). The pipeline is a single workflow:
- **Trigger**: cron `0 14 * * 1-5` (weekdays 14:00 UTC).
- **Steps**: `actions/checkout@master` → `edplato/trufflehog-actions-scan@master` with `--regex --entropy=False --max_depth=1`.
- **On failure**: `rtCamp/action-slack-notify@v2.0.2` posts to Slack `#github-token-scan`, addressing `@devops-team`, using `secrets.SLACK_WEBHOOK`.
There is no build, test, lint, or deploy stage in this repo — `checkout.js` is delivered as static example code.

### Test architecture
_Not present._ No test files, test runners, or test configuration exist in the repo.

### Data model / database schema
_Not applicable._ The repo owns no datastores. The only structured payload is the `trackPageview` arguments documented under [API Reference](#api-reference).

### Auth & trust boundaries
_Auth model not determinable from code._ The snippet is unauthenticated client-side code that runs in a merchant's shopper browser; any authentication of the beacon happens inside `tracker.js` (hosted externally and not in this repo) and likely relies on `merchantGroupId` / `merchantId` as opaque identifiers rather than secrets.

The repo's own trust-boundary signals:
- The CI workflow consumes two GitHub Actions secrets (`ACCESS_TOKEN`, `SLACK_WEBHOOK`).
- Both `actions/checkout` and `edplato/trufflehog-actions-scan` are pinned to `@master`, so the workflow trusts whatever those refs point to at run time.

### Data ownership
_No datastores accessed from this repo._ No DB drivers, ORMs, connection strings, migrations, or cache clients are present. The beacon payload is produced and immediately POSTed via the externally hosted `tracker.js`; this repo does not persist data.

### Deployment topology
_Deployment topology not in this repo._ There is no Dockerfile, Kubernetes manifest, Helm chart, Terraform, or deploy configuration. The artifact is a static snippet copy-pasted into merchant websites; PowerReviews' production tracker is hosted at `static.powerreviews.com` and managed elsewhere.

## Repo Activity
Derived from git history; current HEAD is `3766eda`.
- **Created**: 2016-02-12 (`6e82691` Initial commit by initial author).
- **Last meaningful change**: 2020-07-08 (`3766eda` Update secrets-scan.yml — adjusted the TruffleHog scan workflow). The single subsequent commit on 2026-03-23 (`bac7a0d`) only added a `.whitesource` config file, which is dependency-tooling configuration; the last change to actual runtime code (`checkout.js`) was 2016-02-12.
- **Activity level**: 1 commit in the last 90 days (`bac7a0d`, 2026-03-23 — `.whitesource` config added). Essentially dormant.
- **Hot spots** (top files by churn over the last 6 months):
  1. `.whitesource` — 1 commit (added)
  2. _(no other files modified in the last 6 months)_
  3. _(no other files modified in the last 6 months)_

  Over the full repo lifetime, churn is: `.github/workflows/secrets-scan.yml` (2 commits), `README.md` (2 commits), `checkout.js` (1 commit), `.whitesource` (1 commit).
- **Recent major changes**: _No major changes in the last 6 months._ The only recent commit added a `.whitesource` Mend/WhiteSource configuration file; no runtime, API, or workflow logic changed.
