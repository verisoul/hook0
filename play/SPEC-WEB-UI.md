# Hook0 Play Web UI Specification

**Version**: 2.1
**Date**: 2026-04-13
**Status**: Implemented

## 1. Goal

Deliver a free, zero-signup webhook tester web UI at `play.hook0.com` (route `GET /`) that:

- Lets anyone generate a unique webhook URL instantly and inspect incoming requests in real-time
- Ranks organically for "webhook online", "webhook tester", "free webhook", "webhook inspector"
- Funnels interested users toward the Hook0 platform (sending webhooks) and the Hook0 CLI (local tunneling)

## 2. Existing Backend Surface

The web UI is a pure consumer of the already-deployed API. No new backend endpoints are required for v1.

| Endpoint | Purpose |
|---|---|
| `POST/PUT/PATCH/DELETE/GET /in/{token}/` | Receive any HTTP method as a webhook |
| `POST/PUT/PATCH/DELETE/GET /in/{token}/{*path}` | Receive with sub-path |
| `GET /api/tokens/{token}` | Session metadata (connected, created_at, counts) |
| `GET /api/tokens/{token}/webhooks` | List all stored webhooks for a token |
| `GET /api/tokens/{token}/webhooks/{id}` | Get a single webhook detail |
| `DELETE /api/tokens/{token}/webhooks/{id}` | Delete one webhook |
| `DELETE /api/tokens/{token}/webhooks` | Delete all webhooks for a token |
| `GET /ws` | WebSocket (currently used by CLI; web UI will use the same protocol) |
| `GET /view/{token}` | JSON session info (exists today, will become the HTML page in v1) |

### Token format

Tokens follow the pattern `c_<27-char-base62>` (e.g. `c_A1b2C3d4E5f6G7h8I9j0K1l2m3n`). They are generated client-side by the web UI (same alphabet/length as `relay::token::generate_token`). No server-side "create token" endpoint is needed -- sending any valid-format token to `/in/{token}/` or `/api/tokens/{token}` auto-creates the session.

### WebSocket protocol (inspection mode)

For the web UI, the WebSocket is used **in read-only inspection mode** (not CLI relay mode). The web UI connects to `/ws` and sends:

```json
{"type": "start", "version": 1, "data": {"token": "c_..."}}
```

The server responds with:

```json
{"type": "started", "version": 1, "data": {"webhook_url": "...", "view_url": "..."}}
```

Then, each incoming webhook triggers a `request` message pushed to the WebSocket:

```json
{
  "type": "request",
  "version": 1,
  "data": {
    "id": "uuid",
    "method": "POST",
    "path": "/",
    "body": "<base64>",
    "headers": {"content-type": "application/json", ...},
    "query": "foo=bar"
  }
}
```

The web UI does **not** send `response` messages (that is CLI-only). It only sends `Ping` keepalives.

**Verified**: The backend is fire-and-forget for Request messages. It does not wait for or require a Response. There is no per-request timeout. The only timeout is the idle timeout (1h default), reset by Pings. No backend changes are needed.

### WebSocket collision handling (token_in_use)

The current WebSocket handler rejects a token if it is already connected (`token_in_use` error). When this happens, the web UI must:

1. Display a clear message: "This URL is being monitored in another session"
2. Fall back to **polling mode**: `GET /api/tokens/{token}/webhooks` every 2 seconds
3. Show a badge: "Live updates unavailable (polling every 2s)"
4. Offer a "New URL" button to generate a fresh token with WebSocket support

A future version may support multiple read-only subscribers (broadcast mode).

## 3. UX Flow

### 3.1 Landing (no token)

1. User visits `play.hook0.com` (route `GET /`)
2. Page generates a token client-side using the same `c_` + 27-char base62 algorithm
3. URL updates to `play.hook0.com/#c_<token>` (hash-based routing, no server round-trip)
4. WebSocket connection is established immediately
5. The generated webhook URL `https://play.hook0.com/in/c_<token>/` is displayed prominently with a copy button

### 3.2 Shared link (token in URL)

1. User visits `play.hook0.com/#c_<token>`
2. Page reads the token from the hash
3. Fetches existing webhooks via `GET /api/tokens/{token}/webhooks`
4. Attempts WebSocket connection; if `token_in_use`, falls back to polling (see section 2)
5. Displays the feed (historical + live)

### 3.3 Receiving webhooks

1. Each incoming webhook appears at the top of the feed in real-time (WebSocket push or poll)
2. The feed is a vertical list, newest first
3. Each entry shows: HTTP method badge, path, timestamp (relative, e.g. "3s ago"), body size, content-type
4. Clicking an entry expands it to show full details (see section 5.3)

### 3.4 Empty state

When no webhooks have been received yet, display:

- The webhook URL (copy button)
- A `curl` example command pre-filled with the token:
  ```
  curl -X POST https://play.hook0.com/in/c_<token>/ \
    -H "Content-Type: application/json" \
    -d '{"hello": "world"}'
  ```
- A brief explanation: "Send any HTTP request to the URL above and it will appear here in real-time."

## 4. SEO

### 4.1 Meta tags

| Tag | Value |
|---|---|
| `<title>` | Hook0 Play - Free Webhook Tester & Inspector Online |
| `<meta name="description">` | Free webhook tester online. Generate a unique URL instantly, inspect incoming HTTP requests in real-time. No signup required. Headers, body, method -- all visible. |
| `<meta name="keywords">` | webhook tester, free webhook, webhook online, webhook inspector, webhook receiver, webhook debug, http request inspector |
| `<link rel="canonical">` | `https://play.hook0.com/` |
| Open Graph `og:title` | Hook0 Play - Free Webhook Tester |
| Open Graph `og:description` | Generate a webhook URL instantly. Inspect headers, body, and method in real-time. No signup. |
| Open Graph `og:type` | website |
| Open Graph `og:url` | `https://play.hook0.com/` |

### 4.2 Structured data (FAQPage JSON-LD)

**No competitor uses FAQPage JSON-LD schema.** This is the single highest-ROI structured data opportunity. Embed `FAQPage` schema with these 7 Q&A pairs:

1. **What is a webhook tester?** A webhook tester gives you a unique HTTPS URL that captures incoming HTTP requests so you can inspect their headers, body, method, and query parameters in real-time. Use it to verify that Stripe, GitHub, Shopify, or any other service is sending the correct payload format before you write your production webhook handler.
2. **Is Hook0 Play free?** Yes. Hook0 Play is completely free with no signup required. Generate a unique URL and start receiving webhooks instantly. The CLI tunnel for local testing is also free.
3. **How long are webhooks stored?** Webhooks are stored for up to 24 hours. Each URL keeps up to 1,000 webhooks; the oldest are evicted first when the limit is reached.
4. **Is my webhook data private?** Yes. Hook0 Play does not require an account and does not associate your webhook URL with any identity. Webhook data is stored server-side and automatically deleted after 24 hours. No client-side analytics are loaded on the page.
5. **Can I test webhooks on localhost?** Yes. Install the Hook0 CLI and run `hook0 play listen --forward http://localhost:3000/webhooks`. All requests arriving at your Hook0 Play URL will be relayed to your local server in real-time.
6. **What HTTP methods does Hook0 Play support?** All HTTP methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS, and TRACE. Any request sent to your URL is captured regardless of method.
7. **Can I self-host Hook0 Play?** Yes. Hook0 is open source (SSPL license). You can run the entire stack including Hook0 Play on your own infrastructure with Docker or Kubernetes. See the [self-hosting documentation](https://documentation.hook0.com/self-hosting).

### 4.3 Below-the-fold SEO content (~1,800 words)

7 sections of original content below the tool UI, targeting key GSC query clusters. Content must be passed through /humanizer before publishing.

#### H2: "What is a Webhook Tester?" (~200 words)
Target: webhook tester, what is a webhook, webhook receiver
- Define a webhook tester as a temporary HTTPS URL that captures HTTP requests from any service
- Explain the problem: you cannot point Stripe/GitHub/Shopify at localhost
- Mention real-time inspection of headers, body, method, query parameters
- Link to [What is Hook0](https://documentation.hook0.com/explanation/what-is-hook0)

#### H2: "How to Test Webhooks Online in 3 Steps" (~150 words)
Target: test webhooks online, webhook online, how to test webhooks
- Step 1: Copy your unique Hook0 Play URL (no signup, instant)
- Step 2: Paste it into your webhook provider settings (Stripe, GitHub, Shopify, etc.)
- Step 3: Trigger an event and watch the request appear in real-time
- Link to [Getting started](https://documentation.hook0.com/tutorials/getting-started)

#### H2: "Why Hook0 Play? Free Webhook Testing with No Limits" (~250 words)
Target: free webhook, free webhook service, webhook free, webhook.site alternative, requestbin alternative
- Comparison table: Hook0 Play vs webhook.site vs RequestBin
- Highlight: open source (SSPL), self-hostable, CLI local tunnel is free
- Name webhook.site and RequestBin explicitly (captures "[tool] alternative" queries)
- 1,000 webhooks per URL vs 50 free on webhook.site
- Link to [Comparisons](https://documentation.hook0.com/comparisons)

#### H2: "Test Webhooks Locally with the Hook0 CLI" (~200 words)
Target: test webhooks locally, webhook local tunnel, ngrok alternative
- Explain the local development problem: webhook providers need a public URL
- Show the one-liner: `hook0 play listen --forward http://localhost:3000/webhooks`
- Contrast with ngrok: free, no auth token required, purpose-built for webhook workflows
- Link to [CLI documentation](https://documentation.hook0.com/reference/sdk)

#### H2: "Supported Integrations" (~150 words)
Target: Stripe webhook testing, GitHub webhook testing, Shopify webhook tester
- Named service list: Stripe, GitHub, Shopify, Slack, Twilio, Discord, Mailchimp, PayPal, Jira, Trello, SendGrid, PagerDuty, Salesforce
- "Any service that sends HTTP POST requests to a URL works with Hook0 Play."
- Link to [Event types & subscriptions](https://documentation.hook0.com/tutorials/event-types-subscriptions)

#### H2: "Webhook Testing Questions" (~350 words)
The 7 Q&A pairs from section 4.2 rendered as HTML with FAQPage JSON-LD schema.
Each answer links to relevant documentation pages.

#### H2: "Go Further with Hook0" (~100 words)
Target: webhook platform, webhook service, webhook infrastructure, send webhooks
- Hook0 Play is the receive side. Hook0 platform handles sending webhooks from your app.
- CTA: "Try Hook0 free" → hook0.com
- CTA: "Read the docs" → documentation.hook0.com
- CTA: "Star on GitHub" → github.com/hook0/hook0

### 4.4 Semantic HTML

The page must use semantic elements (`<main>`, `<header>`, `<article>`, `<section>`, `<h1>`, `<h2>`) so that crawlers can index the static content even though the webhook feed is dynamic.

### 4.5 `<noscript>` fallback

Include a `<noscript>` block with the H1 heading, all 7 SEO sections in plain HTML, and links to documentation. This ensures crawlers without JS execution still index the page.

## 5. Features (v1)

### 5.1 Token generation

- Generated client-side in JavaScript using `crypto.getRandomValues()` for the 27 base62 characters
- Prefixed with `c_`
- Stored in `location.hash` so the URL is shareable and bookmarkable
- A "New URL" button generates a fresh token (disconnects existing WebSocket, clears feed, updates hash)

### 5.2 Webhook URL display

- Prominent card at the top of the page showing the full URL: `https://play.hook0.com/in/c_<token>/`
- Copy button with visual feedback (icon changes to checkmark for 2 seconds)
- Below the URL, a smaller line: "Any HTTP request sent to this URL will appear below in real-time."

### 5.3 Webhook detail view

When a webhook entry is expanded (clicked), show:

**Request tab (default)**:
- Method + full path + query string
- Timestamp (absolute ISO 8601 + relative)
- Headers table (key-value, monospace)
- Body display:
  - If `content-type` contains `json`: syntax-highlighted, pretty-printed JSON
  - If `content-type` contains `xml` or `html`: syntax-highlighted XML/HTML
  - If `content-type` contains `x-www-form-urlencoded`: parsed key-value table
  - Otherwise: raw text or hex dump (if binary, show first 1 KB with a "body is binary, showing base64" note)
- Body size in bytes
- A "Delete" button to remove this single webhook

**Raw tab**:
- The full base64-encoded body
- Copy-to-clipboard button for the raw body

### 5.4 Real-time feed

- WebSocket connection to `/ws` with the token
- On `request` messages: prepend the webhook to the feed list, update the counter
- Visual pulse animation on new entries (brief green left-border flash using `--color-primary`)
- Connection status indicator (green dot = connected, red dot = disconnected, yellow dot = reconnecting/polling)
- Auto-reconnect with exponential backoff: 1s, 2s, 4s, 8s, 16s, 30s (cap)
- On `token_in_use` error: fall back to polling mode (see section 2)

### 5.5 Webhook counter

Display a live count: "N webhooks received" in the header area, updating in real-time.

### 5.6 Clear all

A "Clear all" button calls `DELETE /api/tokens/{token}/webhooks` and empties the local feed.

### 5.7 Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `c` | Copy webhook URL to clipboard |
| `Escape` | Collapse expanded webhook detail |
| `j` / `k` | Navigate up/down in webhook list |
| `Enter` | Expand/collapse selected webhook |

## 6. Technical Approach

### 6.1 Serving

The HTML page is served by the existing Axum server as a new route:

- `GET /` returns the static HTML page (inline CSS + inline JS, single response, no external assets)
- `GET /` with `Accept: application/json` continues to return JSON (if a future API use-case needs it), but for `text/html` returns the page
- The route is added in `create_app()` in `lib.rs`
- **No feature flag**: the web UI is active as soon as deployed

### 6.2 Architecture

**Single self-contained HTML file**. All CSS and JS are inlined. No build system, no npm, no bundler. The HTML is:

- Embedded in the Rust binary via `include_str!("../static/index.html")` and served with `Content-Type: text/html`
- In debug mode (`#[cfg(debug_assertions)]`): read from disk for hot-reload during development

### 6.3 File structure

```
play/
  static/
    index.html       # The single-page web UI (HTML + inline CSS + inline JS)
  src/
    ...existing...
```

### 6.4 No external dependencies (no client-side analytics)

The page must not load any external CSS, JS, fonts, or analytics. Reasons:

- Performance (single request, zero waterfall)
- Privacy (no third-party tracking) — this is a differentiator
- Reliability (no CDN dependency)
- CSP-friendly

**Analytics**: Server-side metrics only (Prometheus/Axum metrics). No Matomo or client-side tracking. "Zero tracking" is a selling point vs competitors.

Exception: the Hook0 logo can be an inline SVG.

### 6.5 CSS approach

- Inline `<style>` in `<head>`
- CSS custom properties aligned with hook0.com and app.hook0.com design system (see section 9)
- Dark theme by default, with `prefers-color-scheme: light` media query for light mode
- Responsive: single-column on mobile, two-panel (feed + detail) on desktop (breakpoint: 768px)

### 6.6 JS constraints

- Vanilla JavaScript, no frameworks
- All state in a single module-scoped object
- No `eval`, no `document.write`, no inline event handlers in HTML (all bound via `addEventListener`)
- Base64 decoding via `atob()` for webhook bodies
- Token generation via `crypto.getRandomValues()`
- JSON syntax highlighting via a minimal inline function (no library), using `<pre>` with `<span>` color classes

### 6.7 WebSocket reconnection

```
on close/error:
  if was_connected:
    show "Disconnected" indicator
    attempt reconnect with exponential backoff
  if token_in_use:
    switch to polling mode (GET /api/tokens/{token}/webhooks every 2s)
    show "polling" badge
  on reconnect:
    re-send Start message
    fetch GET /api/tokens/{token}/webhooks to sync missed webhooks
    merge with local feed (deduplicate by webhook ID)
```

### 6.8 Content Security Policy

The Axum handler should set these response headers:

```
Content-Security-Policy: default-src 'none'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' wss://play.hook0.com ws://localhost:*; img-src 'self' data:; font-src 'none'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
```

## 7. Persistence: Redis

### 7.1 Decision

Replace in-memory storage with **Redis Standalone** using native Redis TTL for automatic expiration. This ensures webhooks survive pod restarts and deployments.

### 7.2 Configuration

| Parameter | Value | Rationale |
|---|---|---|
| Deployment | Redis Standalone (single pod) | Simple, sufficient for ephemeral data |
| Max memory | 256 MB | Limits cost; older data evicted via `allkeys-lru` |
| TTL per webhook | 24 hours (`EXPIRE key 86400`) | Matches current in-memory TTL |
| TTL per session | 24 hours from last activity | Same as current idle timeout |
| Max webhooks per token | 1,000 | Use Redis `LTRIM` to cap list length |
| Persistence | None (`save ""`) | Data is ephemeral; RDB/AOF unnecessary |
| Eviction policy | `allkeys-lru` | When memory is full, evict least-recently-used keys |

### 7.3 Key schema

```
session:{token}        → Hash { created_at, connected, total_webhooks, forwarded_count }  TTL 24h
webhooks:{token}       → List of webhook IDs (newest first, max 1000)                     TTL 24h
webhook:{token}:{id}   → Hash { method, path, headers, body, query, timestamp }           TTL 24h
```

### 7.4 Helm chart update

Add Redis as a dependency in the play Helm chart:

```yaml
# charts/Chart.yaml
dependencies:
  - name: redis
    version: "~19"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled

# charts/values.yaml
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false  # Internal-only, no external access
  master:
    persistence:
      enabled: false
    resources:
      requests:
        memory: 128Mi
        cpu: 50m
      limits:
        memory: 256Mi
```

### 7.5 Fallback

If Redis is unavailable, the play server falls back to in-memory storage (existing behavior). This ensures the service remains functional during Redis maintenance.

## 8. Conversion Funnel

### 8.1 Upsell banner

A non-intrusive banner at the bottom of the page (or as a subtle top-right link):

> **Need to SEND webhooks?** Hook0 is an open-source webhook infrastructure for your app. [Try Hook0 free](https://www.hook0.com)

Design: muted background, small text, dismiss-able (stores dismissal in `localStorage`).

### 8.2 CLI promotion

In the empty state and in the page footer, mention:

> **Test webhooks locally?** Use the Hook0 CLI to tunnel webhooks to your machine:
> ```
> hook0 play listen --forward http://localhost:3000/webhooks
> ```
> [Install Hook0 CLI](https://documentation.hook0.com/reference/sdk)

### 8.3 Shareable URLs

Every token URL (`play.hook0.com/#c_<token>`) is shareable. When shared, the recipient sees the same feed (in polling mode if WS is occupied). This encourages organic link sharing and backlinks.

## 9. Design System (aligned with hook0.com + app.hook0.com)

### 9.1 Layout (desktop, >= 768px)

```
+---------------------------------------------------------------+
| [Hook0 Logo]   Hook0 Play    [New URL]  [?] Help   [moon/sun] |
+---------------------------------------------------------------+
|                                                                |
|  Your webhook URL                                              |
|  +----------------------------------------------------------+ |
|  | https://play.hook0.com/in/c_abc123...  [Copy]            | |
|  +----------------------------------------------------------+ |
|  Any HTTP request sent to this URL will appear below.          |
|                                                                |
|  N webhooks received                          [Clear all]      |
+---------------------------------------------------------------+
|                                                                |
|  Webhook Feed                 |  Detail Panel                  |
|  +-------------------------+  |  +---------------------------+ |
|  | POST /  3s ago   1.2KB  |  |  | POST /                    | |
|  +-------------------------+  |  | 2026-04-10T14:32:01Z      | |
|  | GET /health  1m ago     |  |  |                           | |
|  +-------------------------+  |  | Headers                   | |
|  | PUT /users  5m ago      |  |  | content-type: app/json    | |
|  +-------------------------+  |  | x-hook0-sig: sha256=...   | |
|  |           ...            |  |  |                           | |
|  +-------------------------+  |  | Body (JSON)               | |
|                               |  | {                         | |
|                               |  |   "event": "order.paid",  | |
|                               |  |   "data": { ... }         | |
|                               |  | }                         | |
|                               |  +---------------------------+ |
+---------------------------------------------------------------+
| [SEO content sections — 7 H2 blocks, ~1800 words]             |
+---------------------------------------------------------------+
| Footer: Hook0 logo | hook0.com | Docs | GitHub | CLI          |
+---------------------------------------------------------------+
```

### 9.2 Layout (mobile, < 768px)

Single column. The feed and detail are stacked. Tapping an entry expands it inline (accordion).

### 9.3 Color palette (aligned with hook0.com design system)

**Dark mode** (default):

```css
--color-bg-primary: #0f0f13;
--color-bg-secondary: #18181f;
--color-bg-tertiary: #1e1e28;
--color-bg-elevated: #242432;
--color-text-primary: #f9fafb;
--color-text-secondary: #9ca3af;
--color-text-tertiary: #6b7280;
--color-border: #2e2e3a;
--color-border-strong: #3e3e4e;
--color-primary: #4ade80;
--color-primary-hover: #86efac;
--color-primary-light: #052e16;
--color-danger: #f87171;
--color-danger-hover: #fca5a5;
--color-warning: #fbbf24;
--color-info: #60a5fa;
--color-success: #34d399;
--color-link: #41c572;
```

**Light mode** (`prefers-color-scheme: light`):

```css
--color-bg-primary: #ffffff;
--color-bg-secondary: #f9fafb;
--color-bg-tertiary: #f3f4f6;
--color-bg-elevated: #ffffff;
--color-text-primary: #111827;
--color-text-secondary: #6b7280;
--color-text-tertiary: #9ca3af;
--color-border: #e5e7eb;
--color-border-strong: #d1d5db;
--color-primary: #22c55e;
--color-primary-hover: #16a34a;
--color-danger: #dc2626;
--color-warning: #d97706;
--color-info: #2563eb;
--color-success: #059669;
--color-link: #22c55e;
```

**HTTP Method badges** (both themes):

```css
--method-get: #34d399;
--method-post: #fbbf24;
--method-put: #a78bfa;
--method-patch: #f472b6;
--method-delete: #f87171;
--method-head: #9ca3af;
--method-options: #9ca3af;
```

### 9.4 Typography

```css
--font-sans: 'Inter Variable', 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
--font-mono: 'JetBrains Mono', 'Fira Code', ui-monospace, monospace;
```

Note: Inter and JetBrains Mono are NOT loaded externally (no external dependencies). Use the system font fallback chain. Inter is available on most modern systems; if not, the fallback stack provides a similar experience.

### 9.5 Spacing, radius, shadows

```css
--radius-sm: 4px;
--radius-md: 6px;
--radius-lg: 8px;
--radius-xl: 12px;
--radius-full: 9999px;

--shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.3);
--shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.4), 0 2px 4px -2px rgb(0 0 0 / 0.3);
--shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.4), 0 4px 6px -4px rgb(0 0 0 / 0.3);
```

### 9.6 Button styles

**Primary** (green): `background: var(--color-primary); color: white; border-radius: var(--radius-lg);`
**Secondary**: `background: var(--color-bg-primary); border: 1px solid var(--color-border); color: var(--color-text-primary);`
**Ghost**: `background: transparent; color: var(--color-text-secondary);`
**Danger**: `background: var(--color-danger); color: white;`

All buttons: `transition: background-color 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;`
Focus: `box-shadow: 0 0 0 2px var(--color-bg-primary), 0 0 0 4px var(--color-primary);`
Active: `transform: scale(0.98);`

### 9.7 Card styles

```css
background: var(--color-bg-primary);
border: 1px solid var(--color-border);
border-radius: var(--radius-xl);
box-shadow: var(--shadow-sm);
```

Interactive cards (webhook entries): `hover: border-color var(--color-border-strong); box-shadow: var(--shadow-md); transform: translateY(-1px);`

## 10. Differentiation from webhook.site

| Feature | webhook.site | Hook0 Play |
|---|---|---|
| Instant URL, no signup | Yes | Yes |
| Real-time WebSocket feed | Yes | Yes |
| Request detail inspection | Yes | Yes |
| **Local forwarding (CLI tunnel)** | Paid | Free (Hook0 CLI) |
| **Open source** | No | Yes (SSPL) |
| **Self-hostable** | No | Yes (`docker run`) |
| **Full webhook lifecycle** | Receive only | Receive + link to Hook0 for sending |
| **API-first** | Limited | Full REST API for every operation |
| **Zero client-side tracking** | No (loads analytics) | Yes (no JS tracking) |
| Custom response | Paid | Via CLI (free) |
| Max stored webhooks | 50 (free) | 1,000 |
| Retention | 7 days (free) | 24 hours (Redis-backed) |
| **Persistence across deploys** | Yes | Yes (Redis) |

## 11. Performance Budget

| Metric | Target |
|---|---|
| HTML response size (gzipped) | < 30 KB |
| Time to first meaningful paint | < 500ms |
| Time to interactive | < 800ms |
| WebSocket connection established | < 1s after page load |
| Lighthouse Performance score | >= 95 |

## 12. Accessibility

- All interactive elements must be keyboard-accessible
- ARIA labels on icon-only buttons (copy, delete, clear)
- Color is not the sole indicator for any state (method badges also have text; status indicator has text label alongside the dot)
- Focus ring visible on all interactive elements (2px outline, 2px offset)
- `role="alert"` on connection status changes
- Minimum contrast ratio 4.5:1 for text

## 13. Abuse Protection

- Existing rate limits are sufficient for v1: 100 req/s global + per-IP + per-token limits
- The WebSocket handler already tracks invalid token attempts with temporary blocking
- Redis `allkeys-lru` eviction prevents memory exhaustion
- Cloudflare WAF handles bot traffic upstream (existing infrastructure)
- No additional CAPTCHA or token limits for v1

## 14. Out of Scope (v1)

- User authentication / accounts
- Custom domains for webhook URLs
- Custom response codes/bodies from the web UI (CLI-only feature)
- Webhook replay/resend
- Team sharing / collaboration
- Rate limit display in the UI
- Client-side analytics / usage tracking (server-side metrics only)
- Export (JSON/CSV download of webhooks)
- Multi-tab support for the same token via WebSocket (polling fallback covers this)

## 15. Future Versions

These are explicitly deferred but should inform v1 architecture decisions:

- **v2**: Export webhooks as JSON/CSV, webhook replay, custom response from UI
- **v2**: Multi-subscriber WebSocket (broadcast mode) to support multiple browser tabs without polling
- **v2**: Matomo analytics integration (optional, behind consent)
- **v2**: Webhook signature verification helper (paste your secret, verify HMAC)
- **v3**: Webhook comparison (diff two requests)

## 16. Rollout

- **No feature flag**: `GET /` serves the HTML as soon as deployed
- The Helm chart update (Redis dependency) ships with the same release
- Rollback: revert the Helm release; the existing in-memory fallback keeps the service running

## 17. Implementation Checklist

1. [x] Add Redis dependency to Helm chart (section 7.4)
2. [x] Implement Redis storage backend with in-memory fallback in Rust
3. [x] Create `play/static/index.html` with the full single-page UI
4. [x] Write SEO content (~1,800 words) for the 7 below-the-fold sections; run through /humanizer
5. [x] Add `GET /` route in `create_app()` that serves the HTML with proper headers (CSP, etc.)
6. [x] Use `include_str!` to embed `index.html` in release builds; read from disk in debug builds
7. [x] Implement polling fallback for `token_in_use` WebSocket errors
8. [x] Add integration tests: `GET /` returns HTML with correct Content-Type and CSP headers
9. [x] Add integration tests: page contains expected SEO meta tags and FAQPage JSON-LD
10. [x] Add integration test: end-to-end flow (generate token, POST webhook, verify it appears via API)
11. [x] Verify existing WebSocket protocol works with the web UI usage pattern
12. [x] Test polling fallback behavior (simulate `token_in_use`, verify 2s polling) — Playwright E2E
13. [x] Test mobile layout (responsive breakpoints) — Playwright E2E (chromium + Mobile Chrome)
14. [ ] Run Lighthouse audit, verify score >= 95 — deferred to deployment
15. [x] Update Helm chart values for production deployment
