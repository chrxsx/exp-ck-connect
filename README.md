# Private Scraping Bridge (Starter)

**Purpose:** Provide a *consumer-permissioned*, Plaid-Link-style bridge that logs into a user's **own** portal account (e.g., ProviderX) and returns a structured JSON "credit snapshot" without creating a CRA inquiry. This repo is a **starter**; selectors and flows must be customized per provider you have permission to access.

> Legal & Compliance: Many consumer portals forbid automated access. Only deploy against sites where you have **explicit permission**, and ensure **user consent**, **data minimization**, and **secure credential handling**. This starter is not advice on evading security controls and does not include any anti-detection tactics.

## Components
- **bridge/** – Node + Express API, iframe widget, BullMQ producer, credential encryption, session orchestration.
- **worker/** – Playwright worker that performs the interactive login and page traversal, parses data, and posts results back.
- **frontend/** – Minimal demo that opens the bridge widget in an iframe and listens for progress/results (postMessage).
- **schemas/** – JSON schema for `credit_snapshot` output.

## Quick Start
1. **Prereqs**: Docker, Docker Compose.
2. **Secrets**: Create `.env` in project root:
   ```bash
   ENCRYPTION_KEY=$(openssl rand -hex 32)  # must be 64 hex chars (32 bytes)
   ```
3. **Build & run**:
   ```bash
   docker compose up --build
   ```
4. **Open**: http://localhost:8082 and click **Connect Credit Account**.
5. In the widget, choose **ProviderX** and use any test creds you wired in your local dev env (see `worker/src/scrapers/providerX.js` for the target URL and selectors).

## High-Level Flow
1. Your app POSTs `/v1/sessions` to the bridge → returns `session_id` + `iframe_url`.
2. Your frontend loads `iframe_url` (hosted by **bridge**) in an `<iframe>`.
3. User selects provider and enters credentials + OTP (handled inside the iframe).
4. Bridge **encrypts** credentials, enqueues a job in Redis (BullMQ).
5. Worker consumes the job, uses **Playwright** to login, navigates to score/tradeline pages, extracts data.
6. Worker posts progress and final results back to Bridge (`/v1/sessions/:id/events`).
7. Bridge forwards progress/result to parent window via `postMessage` and exposes the final JSON to your backend via `GET /v1/sessions/:id/result`.

## Strict Security Posture
- **Ephemeral** credential handling: encrypted at rest in Redis payloads; cleared after job completion.
- **No credential storage by default**: change `SAVE_CREDS` to enable vault persistence if you have user consent.
- **CSP + SameSite cookies** on the widget. **ALLOWED_ORIGIN** restricts `postMessage` targets.
- **Audit logs** for consent and key actions.

## Customizing a Provider
- Edit `worker/src/scrapers/providerX.js`:
  - Set `LOGIN_URL` and any relevant navigation URLs.
  - Update selectors for username, password, submit, OTP input, and DOM nodes containing score/tradeline data.
- Update the **parser** to normalize output to `schemas/credit_snapshot.schema.json`.

## Notes
- This starter avoids any anti-detection or TOS-violating measures.
- Use official APIs whenever possible (e.g., Experian Connect / bureau APIs).
- Treat scraped data as **sensitive**; minimize fields and retention.


---

## React Integration (drop-in)
Use the provided `react-integration/ConnectCredit.tsx` in your site:
```tsx
import ConnectCredit from "./ConnectCredit";

export default function Page() {
  return (
    <ConnectCredit
      createSessionUrl="/api/bridge/create-session"
      bridgeOrigin={process.env.NEXT_PUBLIC_BRIDGE_ORIGIN || "http://localhost:8080"}
      onResult={(data) => {
        // Persist to your backend and proceed
        console.log("Credit snapshot", data);
      }}
    />
  );
}
```
Your `/api/bridge/create-session` should proxy to the Bridge's `POST /v1/sessions` and return `{session_id, iframe_url}`.

## Two-button flow
The component renders two buttons (Experian / Credit Karma). It appends a `?provider=...` query to the iframe URL so the widget preselects the provider.

## Hosting and GitHub
Put this repository on GitHub as a **monorepo**. Suggested structure:
```
/bridge
/worker
/frontend (demo)
/react-integration (component copied into your site repo or published as a private npm package)
/schemas
/docker-compose.yml
```
- **Local dev:** `docker compose up --build`.
- **Prod:** Run `bridge` and `worker` as services (e.g., ECS/Kubernetes). Use a managed Redis (ElastiCache).
- **GitHub Actions:** add a workflow that builds and pushes Docker images on main branch merges.

## How the scrapers work (Experian / CK)
- We prefer **network JSON** over DOM. The worker records JSON responses (XHR/fetch) during login + dashboard navigation and extracts score, accounts, and inquiries from those payloads when available.
- If network JSON is insufficient, we **fallback to DOM** with CSS selectors (fill the TODO selectors in `experian.js` and `creditkarma.js`).
- 2FA: The widget has a single OTP input that is forwarded to the worker. If the site uses step-up after login, the scraper waits for the OTP selector and submits it.
- All sessions are **ephemeral** by default. If you need refresh, add a credential vault and explicit user consent.

## Implementation checklist (handoff to agents)
- [ ] Confirm you have permission to automate Experian/CK flows with explicit consumer consent.
- [ ] Record a real login flow with DevTools → Network; identify the JSON endpoints carrying score/tradelines/inquiries.
- [ ] Fill in LOGIN_URL and selectors in `worker/src/scrapers/experian.js` and `creditkarma.js`.
- [ ] Map network payloads to `schemas/credit_snapshot.schema.json`. Keep only necessary fields.
- [ ] Update `bridge/views/widget.ejs` copy to reflect your consent language.
- [ ] Wire your site’s backend route `/api/bridge/create-session` to call Bridge `/v1/sessions`.
- [ ] Add CORS/ALLOWED_ORIGIN to match your site domain(s).
- [ ] Add logging + alerts for failures and rate-limit the worker.

## Compliance
- Show explicit consent text; log timestamp/IP and a “I am the account owner” checkbox state.
- Never store plaintext credentials; use the included AES-GCM envelope + ephemeral memory.
- Respect site terms and applicable law. Prefer official partner APIs when possible.


---

## Experian & Credit Karma connector specifics

### Login discovery
Each scraper tries several common login URLs (see `LOGIN_URLS`) and proceeds with the first page that loads.

### Selector strategy
Selectors are **arrays** of candidates; the scraper picks the first that exists:
- username: `['input[name="username"]', 'input#username', 'input[type="email"]', 'input[name="email"]']`
- password: `['input[name="password"]', 'input#password', 'input[type="password"]']`
- submit: `['button[type="submit"]', 'button[data-testid="sign-in-button"]', ...]`
- otp: `['input[name="otp"]', 'input[name="code"]', 'input#otp', 'input[name="oneTimeCode"]']`

### Data extraction
1. **Network-first**: The worker captures all `application/json` responses and searches for fields like `score`, `scoreModel`, `accounts`, `inquiries`, `profile`.   Update the mapping logic as you discover the exact JSON routes in DevTools → Network.
2. **DOM fallback**: Heuristic scan for any element containing the word “score” and a 3‑digit number between 300–900.

### Optional navigation
Post-login, the scrapers *try* a couple of likely dashboard/score routes. Adjust or remove as you confirm the real routes for your program.

### To finalize
- Do a real login (with permission) and watch the Network tab to identify the exact JSON endpoints for score/report chunks.- Add precise URL regexes and mapping into each scraper’s `for (const p of networkPayloads)` loop.- If available, prefer official partner APIs over scraping.


---

## Formatted output
Once a session completes you can fetch a human-readable summary:

- **Markdown:** `GET /v1/sessions/:id/pretty`  
- **HTML wrapper:** `GET /v1/sessions/:id/pretty.html`

These render: score (+ model), account counts, aggregate limits/balances, utilization, top-5 accounts by balance, and recent inquiries. Use the JSON at `GET /v1/sessions/:id/result` for programmatic logic, and the pretty routes for operator review or client-facing copy.


---

## Provider-specific details (HAR-derived)

### Credit Karma
- Primary API: `https://api.creditkarma.com/graphql` (GraphQL). The page calls `creditReportsV2 { creditReport { ... } }` which contains `score`, `tradelines`, `inquiries`, and `publicRecords`.- The worker captures GraphQL **request operationName** and parses batched/single JSON responses. It maps `data.creditReportsV2.creditReport.*` into the normalized schema. Fallbacks exist for legacy `getCreditReport` / `creditReport` shapes and DOM if needed.

### Experian
- Primary API observed: `https://usa.experian.com/api/report/forcereload` (JSON). Contains `reportInfo.creditFileInfo[0]` with `score`, `comparisonData.currentReport.scoreModel` (e.g., `Fico8`), `accounts[]` (name, dates, balances, limits), and optional `inquiries/publicRecords`.- The worker navigates to `https://usa.experian.com/` then `.../member/credit-report` to trigger the XHR, captures the JSON, and maps the fields.

