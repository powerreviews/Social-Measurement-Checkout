# Social-Measurement-Checkout — RepoDocs
_Generated on 2026-05-11 · Consolidated on 2026-05-19 against commit 7e6a705_

## Summary

### Overview
This repo is a customer-facing reference snippet for the **PowerReviews "Social Measurement" conversion beacon** — a single `checkout.js` paste-block that merchants embed on their order-confirmation page to fire an order/user payload at the v1 `tracker.js` analytics endpoint. It is owned by the integration/customer-success surface (no clear engineering owner in git history) and sits entirely outside the org's runtime services: the beacon's destination is the externally-hosted `static.powerreviews.com/t/v1/tracker.js` shipped from `pufferfish-static` (the Maven WAR that packages the legacy v1 widget + tracker library), and the resulting beacons land in the `feed-services` / `feeds Postgres` ingestion path via the `beacon-data` SQS queue documented in the org shared-infrastructure table. Effectively this repo is documentation/sample code; no service in the org imports or builds from it.

Confluence frames the Checkout Beacon as one of three interchangeable order-ingestion paths for Follow-Up Email (the others being SFTP/CSV order feeds and the Enterprise API), pitched as "minimal engineering effort: paste a JS snippet on the order-confirmation page and order attributes are captured at checkout completion." The captured fields documented in Order Ingestion Information (PageId~variant~locale, merchant user email, first/last name, order id, order date, merchant id, merchant user id) are an exact superset of what this repo's `trackPageview("c", {...})` payload transmits, confirming the snippet's intended downstream contract. The FUE Rebuild spec (FUE Core "IN PROGRESS" under PWRE-3) explicitly lists "Checkout Beacon" as a Phase-1 supported ingestion method that the new architecture must continue to honor — the rebuild docs do not propose retiring the beacon snippet itself, only the email-sending and reporting infrastructure behind it.

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | JavaScript (browser, no transpile) | _Not specified_ |
| Framework | PowerReviews `tracker.js` (CDN-loaded) | v1 |
| Database | _None_ | _N/A_ |
| Build Tool | _None_ | _N/A_ |
| CI/CD | GitHub Actions (TruffleHog secrets scan only) | _N/A_ |
| Cloud/Infra | None in-repo (artifact is copy-paste HTML) | _N/A_ |

### Consumers
| Consumer | Type | How They Use It |
|----------|------|----------------|
| Merchant order-confirmation pages | External (customer storefronts) | Paste/template the snippet to fire the `"c"` conversion beacon |
| `pufferfish-static` (v1 `tracker.js` on `static.powerreviews.com`) | Sibling org repo (build/host of the script) | Receives the script-tag load; runs the actual beacon transmission code |
| `feed-services` (via `beacon-data` SQS) | Downstream org repo | Consumes the resulting beacon payloads as order-feed input for review solicitation (per org Shared Infrastructure: `beacon-data` SQS → `feed-services`) |
| GitHub Actions runner | CI service | Runs the scheduled TruffleHog secrets scan |
| Slack `#github-token-scan` | External service | Receives `@devops-team` alerts on scan failure |

### Dependencies on Org Repos
The base extraction marked "no dependencies," which is true at the code-level (no `package.json`, no submodules). Cross-referencing the org context surfaces two indirect runtime relationships worth recording:
| Repo | Reason |
|------|--------|
| `pufferfish-static` | Builds and publishes the `tracker.js` v1 asset that this snippet loads from `static.powerreviews.com/t/v1/tracker.js`. Org catalog explicitly calls it the "Maven-built WAR packaging legacy v1 JavaScript widget library and tracker.js analytics." |
| `feed-services` | Downstream consumer of beacons fired by this snippet — org-summary lists `beacon-data` SQS → `feed-services` as the ingest path. The README's "automatically create order feeds" claim resolves into this Ruby service. |
| `powerreviews-pufferfish` / `pufferfish-shared-services` | The tracking/conversion datastore historically lived behind the pufferfish monolith; modern path appears to flow via `feed-services` into the `feeds`/`reviews` Postgres clusters listed under Shared Infrastructure. |
| `whitesource-config` | Org-wide Mend scan policy that the `.whitesource` file (added 2026-03-23) opts into. |

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
| `actions/checkout@master` | floating `master` | Supply-chain risk: third-party Action pinned to a moving ref; upstream renamed default branch to `main`, so `master` is effectively unmaintained for this Action. Pin to a SHA or tagged release. | Critical |
| `edplato/trufflehog-actions-scan@master` | floating `master` | Unmaintained third-party Action wrapping legacy TruffleHog v2 (deprecated by the upstream project); pinned to a mutable branch so code can change under the workflow without review. Org context shows ~80 repos still using this same pattern — fleet-wide hygiene issue, not unique here. | Critical |
| `//static.powerreviews.com/t/v1/tracker.js` | v1 | Protocol-relative URL plus the v1 endpoint has been the integration surface unchanged since 2016. Verify v1 is still the supported tracker version given the modern beacon path lives in `feed-services`/`ui-library` territory. | Severe |

### Coupling Profile
| Dependency | Protocol | Frequency Pattern | Failure Mode |
|-----------|----------|-------------------|--------------|
| `pufferfish-static` (`tracker.js` v1) | sync HTTP (script tag from CDN) | per-request (each order-confirmation page view) | hard at load; the IIFE wraps the call in `try/catch` and logs via `window.console.log`, so a runtime error in `tracker.js` degrades silently to "no beacon" without breaking the merchant page |
| `feed-services` (via tracker → `beacon-data` SQS) | message queue (indirect; HTTPS beacon → SQS publish inside `tracker.js`) | event-triggered (one per checkout) | queued (eventually retries on the SQS side); from this repo's perspective the call is fire-and-forget — no acknowledgement, no retry on the client |
| GitHub Actions → TruffleHog | sync HTTP (Action invocation) | scheduled (cron `0 14 * * 1-5`) | soft (Slack-only alert on failure; no gating) |
| Slack webhook | webhook (outbound HTTPS) | event-triggered (on scan failure) | soft (best-effort notification) |

### Architectural Notes
- **Shared infrastructure**: This repo touches none directly, but its data-flow lands in two of the org's most-shared resources — the `beacon-data` SQS topic (consumed by `feed-services`) and downstream the `reviews`/`feeds` Postgres clusters (used by ~16 repos collectively). Any v1-tracker deprecation needs coordination with `pufferfish-static`, `feed-services`, and the read path in `feed-services` Sidekiq workers.
- **Bounded-context overlaps**: The snippet's payload (`merchantId`, `merchantGroupId`, `merchantUserId`, `orderId`, `orderItems[pageId, pageIdVariant, productName, qty, unitPrice]`, `userEmail`/`userFirstName`/`userLastName`, `marketingOptIn`) overlaps with: **Merchant** (canonical in `customer-account-services`, legacy in `pufferfish-shared-services`), **Order** (canonical writes in `write-services`, feeds ingested by `feed-services`), **Product/Page** (`product-services`, `core-data-services` CDM), and **User/Consumer PII** (`data-protection` is the GDPR/CCPA owner). The snippet transmits raw PII (email, first/last name) client-side — worth flagging against `data-protection`'s OneTrust/DSAR scope, since this is a public-facing PII collection surface that lives outside the platform's modern auth boundary.
- **Architectural evolution**: The runtime code (`checkout.js`) has not changed since the **2016-02-12 initial commit** — 10 years frozen. All subsequent commits are CI/policy housekeeping: the secrets-scan workflow (2020-07), a README touch-up, and the `.whitesource` config (2026-03-23). The org has since shipped a modern beacon/widget stack (`ui-library`, `ui-library-core-web-vitals`, `ui-library-diagnostics`, `ui-library-hosted-collect`) that supersedes the v1 tracker surface — this repo is the last public-facing artifact still pointing customers at `t/v1/tracker.js`. Effectively dormant integration documentation rather than active code.
- **Default order-date fallback**: Order Ingestion documents that when no order date is supplied, Feed Services synthesizes `DateTime.now - fue_wait_days.first.days` — relevant because this snippet's payload has no `orderDate` field, so any beacon-sourced order will get this fallback applied downstream.
- **Sentinel wait-time flags**: `apo_wait_time=-1&fue_wait_time=-1` are sentinel values used by both beacon and recurring-feed paths to short-circuit per-order wait timers, deferring to merchant-level `fue_wait_days` configuration. Useful context if anyone modernizes the snippet and needs to preserve semantic parity.
- **Order expiration**: Engagement Services Job Processes / FUE Testing Runbook document a 180-day order-expiry rule and a 15-hour ProcessOrder task-protection window — orders captured by this beacon are eligible for FUE solicitation for 180 days, then dropped.
- **Email-from address constraint**: FUE Rebuild notes the SendGrid-mandated from-email format `merchantname@pr.merchantname.email.powerreviews.com`. Not relevant to the snippet directly, but explains why merchant-side branding choices stop short of own-domain sending.
- **PII handling downstream**: The beacon's `userEmail`/`userFirstName`/`userLastName` payload feeds into `ss_merchant_email_customization`/`customer_engagement_order` (Email Configuration, FUE troubleshooting docs), and Email BlackListing docs describe a `global_email_blacklist` + `merchant_email_blacklist` two-step check before any FUE send — so the snippet's transmitted PII has documented suppression handling on the receiving side.
- **Modernization trajectory**: FUE Rebuild's Phase 1 (PWRE-3, IN PROGRESS) is replacing Engagement Services / EC2-cron / DynamoDB order storage with a Postgres Order Domain + SQS-based Email Service. The repo's snippet is upstream of all of this and is expected to keep working unchanged across the migration — consistent with the runtime code being 10 years frozen.

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
3. The IIFE calls `createTracker` and then `trackPageview("c", {...})`, which transmits the order/user payload to PowerReviews analytics (the destination URL is encapsulated inside `tracker.js`; this repo does not show it). Per the Data Ingestion Overview, the broader path is PDP/order page → CDN logs → aggregation → per-order ingestion events that land in Write Services and propagate into the FUE order domain; the Order Ingestion page nails down the exact transport as `/api/v2/feed?merchant_id=…&apo_wait_time=-1&fue_wait_time=-1` — meaning the v1 `tracker.js` ultimately POSTs to Write Services with FUE-disabling timing flags negated.
4. PowerReviews ingests the beacon as conversion data and, per the README, can also derive an order feed for review solicitation. This is one of three interchangeable order-ingestion paths for Follow-Up Email (the others being SFTP/CSV order feeds and the Enterprise API).

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

## Documentation Discrepancies
| Confluence States | Code Shows | Likely Reason |
|------------------|-----------|---------------|
| Checkout Beacon is "a JavaScript snippet (pixel)" — i.e., lightweight pixel request | Snippet loads a full library (`//static.powerreviews.com/t/v1/tracker.js`) and calls `POWERREVIEWS.tracker.createTracker(...).trackPageview("c", {...})` — substantially more than a pixel | Outdated doc / loose phrasing — Confluence describes the conceptual model; the actual artifact is a tracker-library script tag, not a 1×1 pixel |
| FUE Rebuild is actively replacing the surrounding architecture and Checkout Beacon is a supported Phase-1 input | The snippet's runtime code (`checkout.js`) has not changed since 2016-02-12 (10 years frozen); the org has shipped a modern `ui-library*` beacon stack that supersedes the v1 tracker surface | Repo is dormant integration documentation; modernization is happening in `pufferfish-static` / `feed-services` / `ui-library*`, not here. Confluence does not reference this specific repo as the canonical snippet source |
| Beacon "sends data to WS endpoint `/api/v2/feed?merchant_id=…&apo_wait_time=-1&fue_wait_time=-1`" | Snippet posts to whatever `tracker.js` resolves to (path encapsulated in the externally hosted library); no direct call to `/api/v2/feed` appears in this repo | Not a true conflict — the WS POST happens inside `tracker.js` (hosted in `pufferfish-static`), not in `checkout.js`. Documents two different layers of the same call chain |
| FUE primary purpose: capture orders for review-solicitation emails | REPOANALYSIS Overview frames primary purpose as "Social Measurement conversion beacon" (tying review engagement to conversion); order-feed generation is described as an optional secondary outcome | Scope changed / dual-use — same snippet serves both analytics conversion tracking AND order-feed seeding; README and Confluence emphasize different sides of the same dual-purpose payload |
| `userEmail`/`userFirstName`/`userLastName` flow into the FUE order-solicitation pipeline | Same PII fields are transmitted client-side from a public order-confirmation page; no Confluence page in the index documents this as a `data-protection`/OneTrust-scoped collection surface | Doc gap — Confluence treats the PII as ordinary order data; REPOANALYSIS Architectural Notes flags it as a public-facing PII collection point outside the modern auth boundary |

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
