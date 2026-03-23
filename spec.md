# InvoiceZap — Product Specification

## Overview

**Product Name:** InvoiceZap
**One-liner:** Send and track invoices in under 60 seconds
**Target Audience:** Freelance designers
**Pricing:** $29 one-time purchase
**Distribution:** Cloudflare Workers edge deployment
**Payments:** LemonSqueezy

InvoiceZap is a dead-simple, one-page invoice builder designed for freelance designers who lose hours chasing unpaid invoices. Unlike complex accounting suites (FreshBooks, Wave, QuickBooks), InvoiceZap focuses on one job: create a professional invoice, attach a payment link, and send it — with automatic follow-up reminders. No subscriptions, no bloat, no learning curve.

## Problem & Differentiation

**Problem Statement:** Freelancers lose hours chasing unpaid invoices because existing tools are too complex.

**Root Cause Analysis:**
- Freelance designers are visual thinkers forced into accounting-oriented interfaces
- Existing tools bundle invoicing with accounting, time-tracking, and project management — 80% of features go unused
- Monthly subscription costs ($16–70/mo) create ongoing overhead for sporadic invoicing needs
- Complex onboarding flows (client setup, tax configuration, payment gateway integration) delay time-to-first-invoice

**Hypothesized Solution:** A dead-simple one-page invoice builder with built-in payment link and auto-reminder.

**Differentiation Statement:** InvoiceZap is the only invoice tool that charges a flat $29 one-time fee with zero monthly costs, delivering a single-page invoice builder with built-in payment links and auto-reminders — eliminating the subscription fatigue that freelance designers face with FreshBooks ($23–70/mo), Wave (free but limited), and Invoice Ninja ($10–14/mo subscription).

## Market Context

| Competitor | Pricing | Key Features | Target Audience | Weakness | Source URL |
|---|---|---|---|---|---|
| FreshBooks | Lite: $23/mo, Plus: $43/mo, Premium: $70/mo (subscription) | Invoicing, expense tracking, time management, reporting, estimates, client retainers | Freelancers & SMBs | Overkill for designers who only need invoicing; client caps on lower tiers (5 on Lite); expensive at scale ($70/mo for unlimited) | https://www.freshbooks.com/pricing |
| Wave | Starter: Free, Pro: $16/mo (subscription) | Invoicing, accounting, receipt scanning, bank reconciliation, financial reports | Solo entrepreneurs & startups | No project management, no proposals, no client portals; limited automation on free plan; slow payment deposits | https://www.plutio.com/alternatives/wave |
| Invoice Ninja | Free (5 clients), Pro: $10/mo, Enterprise: $14+/mo (subscription) | Open-source, 40+ payment gateways, client portal, recurring invoices, Kanban boards | Developers & technical solopreneurs | Complex setup; self-hosting required for full control; designed for technical users, not designers; 3/5 avg rating on CloudFindr | https://invoiceninja.com/pricing-update-january-1-2026/ |
| Invoice Simple | Essentials: $4.99/mo, Plus: $13.49/mo, Premium: $19.99/mo (subscription) | Quick invoice creation, customizable templates, payment links, expense tracking | Independent contractors | Subscription model for basic functionality; limited design customization; no auto-reminders on lower tiers | https://www.invoicesimple.com/blog/freshbooks-alternative |
| HoneyBook | $14–39/mo (subscription) | Client management, estimating, invoicing, CRM, automations, collaboration tools | Solo consultants | Steep learning curve; expensive scaled subscriptions; bundles features designers don't need (CRM, contracts) | https://blog.bloom.io/invoicing-software-for-freelancers/ |
| Zoho Invoice | Free (5 clients), paid via Zoho Books $9–35/mo (subscription) | 40+ templates, recurring invoices, Zapier integration, inventory management | Product-based freelancers | Rigid design; 5-client limit on free tier; requires Zoho ecosystem buy-in for full functionality | https://www.flowlu.com/blog/finances/freelance-invoice-software/ |
| Plutio | Core: $19/mo (subscription) | All-in-one workspace: proposals, contracts, projects, time tracking, invoicing, client portals | Designers & creative freelancers | Feature overload for invoice-only needs; $19/mo ongoing cost; complex onboarding for simple invoicing | https://www.plutio.com/solutions/designers/invoicing |

**Key Finding:** Every competitor uses subscription pricing. According to Plutio's research, designers stack 4–7 disconnected tools costing $75–200/month combined (Source: plutio.com/solutions/designers/invoicing). Freelancers using dedicated invoicing tools earn 65% more annually than those using manual methods (Source: blog.bloom.io/invoicing-software-for-freelancers/). Confidence: MEDIUM — vendor-sourced statistics.

**Differentiation Statement:** InvoiceZap is the only invoice tool that charges a flat $29 one-time fee with zero monthly costs, delivering a single-page invoice builder with built-in payment links and auto-reminders — eliminating the subscription fatigue that freelance designers face with FreshBooks ($23–70/mo), Wave (free but limited), and Invoice Ninja ($10–14/mo subscription).

## Architecture

```
Browser → Cloudflare Worker (edge API) → KV Store (sessions / invoices / purchases / rate-limit)
                                       → LemonSqueezy API (checkout creation)
                                       ← LemonSqueezy Webhook (order_created)
Push to main → GitHub Actions → Wrangler deploy → Cloudflare Workers
```

**Request Flow:**
1. User visits landing page (`GET /`) → Worker serves static HTML from bundled assets
2. User clicks "Buy Now" → `POST /api/checkout` → Worker calls LS `POST /v1/checkouts` → returns redirect URL
3. LemonSqueezy processes payment → `POST /api/webhooks/ls` → Worker verifies HMAC-SHA256 → stores purchase in KV
4. User accesses app (`GET /app`) → auth middleware reads session → purchase middleware verifies purchase → serves app
5. User creates invoice → `POST /api/invoices` → Worker stores in KV
6. User sends invoice → `POST /api/invoices/:id/send` → Worker generates shareable payment link
7. Auto-reminders fire via scheduled Worker cron trigger → checks overdue invoices → sends reminder notifications

## Tech Stack

| Technology | Rationale |
|---|---|
| **Node.js (TypeScript)** | Type safety for financial data handling; first-class Cloudflare Workers support via wrangler; matches developer's core expertise. |
| **Cloudflare Workers** | Zero cold-start edge compute ensures sub-50ms response times globally; built-in KV integration eliminates need for external database; generous free tier (100K requests/day). |
| **Cloudflare KV** | Eventually-consistent key-value store ideal for session, invoice, and purchase data; global replication at no extra cost. |
| **Tailwind CSS** | Utility-first CSS enables rapid UI prototyping for the one-page invoice builder without custom CSS bloat. |
| **shadcn/ui** | Pre-built accessible components (buttons, forms, modals) accelerate UI development while maintaining polished look for designer audience. |
| **LemonSqueezy** | Merchant-of-record handles EU VAT compliance, tax remittance, and payment processing — critical for European market focus; simple API for one-time purchases. |
| **GitHub Actions** | Native CI/CD with Wrangler integration; free for public repos; secrets management built in. |

## Data Model

| KV Namespace | Key Pattern | Value Schema | TTL |
|---|---|---|---|
| `SESSIONS` | `sessions:{sessionId}` | `{ "userId": string, "email": string, "createdAt": string (ISO 8601), "expiresAt": string (ISO 8601) }` | 86400 (24h) |
| `USERS` | `users:{email}` | `{ "userId": string, "email": string, "passwordHash": string, "createdAt": string (ISO 8601) }` | 0 (permanent) |
| `INVOICES` | `invoices:{userId}:{invoiceId}` | `{ "invoiceId": string, "userId": string, "clientName": string, "clientEmail": string, "items": [{ "description": string, "quantity": number, "unitPrice": number }], "currency": string, "status": "draft" | "sent" | "viewed" | "paid" | "overdue", "createdAt": string, "dueDate": string, "sentAt": string | null, "paidAt": string | null, "reminderCount": number, "paymentLink": string | null, "notes": string }` | 0 (permanent) |
| `INVOICES_INDEX` | `invoice_index:{userId}` | `{ "invoiceIds": string[] }` | 0 (permanent) |
| `PURCHASES` | `purchases:{userId}:{productId}` | `{ "orderId": string, "variantId": string, "purchasedAt": string (ISO 8601), "status": "active" | "refunded" }` | 0 (permanent) |
| `RATELIMIT` | `ratelimit:{ip}` | `{ "count": number, "windowStart": string (ISO 8601) }` | 60 (1 min) |

## API Routes

| Method | Path | Purpose | Auth | Request Body | Response | LS Integration |
|---|---|---|---|---|---|---|
| GET | `/` | Serve landing page | No | — | `200 text/html` | — |
| GET | `/app` | Serve invoice builder app | Yes | — | `200 text/html` | Purchase gate |
| POST | `/api/auth/register` | Create user account | No | `{ "email": string, "password": string }` | `201 { "userId": string, "sessionId": string }` | — |
| POST | `/api/auth/login` | Authenticate user | No | `{ "email": string, "password": string }` | `200 { "sessionId": string }` | — |
| POST | `/api/auth/logout` | Destroy session | Yes | — | `200 { "ok": true }` | — |
| GET | `/api/invoices` | List user invoices | Yes | — | `200 { "invoices": Invoice[] }` | Purchase gate |
| POST | `/api/invoices` | Create invoice | Yes | `{ "clientName": string, "clientEmail": string, "items": LineItem[], "currency": string, "dueDate": string, "notes?": string }` | `201 { "invoice": Invoice }` | Purchase gate |
| GET | `/api/invoices/:id` | Get invoice by ID | Yes | — | `200 { "invoice": Invoice }` | Purchase gate |
| PUT | `/api/invoices/:id` | Update invoice | Yes | `{ "clientName?": string, "clientEmail?": string, "items?": LineItem[], "dueDate?": string, "notes?": string }` | `200 { "invoice": Invoice }` | Purchase gate |
| DELETE | `/api/invoices/:id` | Delete invoice | Yes | — | `200 { "ok": true }` | Purchase gate |
| POST | `/api/invoices/:id/send` | Send invoice to client | Yes | — | `200 { "sentAt": string, "paymentLink": string }` | Purchase gate |
| POST | `/api/invoices/:id/remind` | Send payment reminder | Yes | — | `200 { "reminderCount": number }` | Purchase gate |
| GET | `/api/invoices/:id/public` | Public invoice view for clients | No | — | `200 text/html` | — |
| POST | `/api/checkout` | Create LS checkout session | Yes | `{ "userId": string }` | `200 { "checkoutUrl": string }` | Creates LS checkout |
| POST | `/api/webhooks/ls` | Receive LS webhook events | No (HMAC) | LS event payload | `200 { "ok": true }` | Receives order_created |
| GET | `/api/health` | Health check | No | — | `200 { "status": "ok", "timestamp": string }` | — |

## LemonSqueezy Integration

### (a) Checkout Creation — `POST /api/checkout`

1. Authenticated user sends `POST /api/checkout` with their `userId`.
2. Worker calls LemonSqueezy `POST /v1/checkouts` with variant ID and custom user data.
3. Worker returns `{ "checkoutUrl": "[LS checkout URL]" }` to client.
4. Client redirects to LemonSqueezy-hosted checkout page.

### (b) Webhook Receipt — `POST /api/webhooks/ls`

1. LemonSqueezy sends POST to `/api/webhooks/ls` with event payload.
2. Worker extracts `X-Signature` header.
3. Worker computes HMAC-SHA256 of raw request body using `LEMON_SQUEEZY_WEBHOOK_SECRET` env var.
4. Worker compares computed hash with `X-Signature` (constant-time comparison).
5. **If signature mismatch:** return `401 Unauthorized`.
6. **If valid:** parse event type:
   - `order_created`: Extract `user_id` from `meta.custom_data`. Store in KV `purchases:{userId}:{productId}` with `status: "active"`.
   - `subscription_created` / `subscription_updated`: Update purchase status.
   - `subscription_cancelled`: Set status to `"refunded"`.
7. Return `200 { "ok": true }`.

### (c) Purchase Gate Middleware

1. On every protected route (`/app`, `/api/invoices/*`):
2. Auth middleware extracts `sessionId` from cookie → reads `sessions:{sessionId}` from KV → extracts `userId`.
3. Purchase middleware reads `purchases:{userId}:{PRODUCT_ID}` from KV.
4. **If active purchase exists:** proceed to handler.
5. **If no purchase:** return `403 { "error": "Purchase required", "checkoutUrl": "..." }`.

## Deployment Configuration

**wrangler.toml:**
```toml
name = "invoicezap"
main = "src/index.ts"
compatibility_date = "2025-12-01"

[triggers]
crons = ["0 9 * * *"]

[[kv_namespaces]]
binding = "SESSIONS"
id = "TBD_SESSIONS_KV_ID"

[[kv_namespaces]]
binding = "USERS"
id = "TBD_USERS_KV_ID"

[[kv_namespaces]]
binding = "INVOICES"
id = "TBD_INVOICES_KV_ID"

[[kv_namespaces]]
binding = "INVOICES_INDEX"
id = "TBD_INVOICES_INDEX_KV_ID"

[[kv_namespaces]]
binding = "PURCHASES"
id = "TBD_PURCHASES_KV_ID"

[[kv_namespaces]]
binding = "RATELIMIT"
id = "TBD_RATELIMIT_KV_ID"

[vars]
LEMON_SQUEEZY_STORE_ID = ""
LEMON_SQUEEZY_PRODUCT_ID = ""
LEMON_SQUEEZY_VARIANT_ID = ""
LEMON_SQUEEZY_CHECKOUT_URL = ""
WORKER_URL = ""
```

Secrets (set via `wrangler secret put`, never in code):
- `LEMON_SQUEEZY_API_KEY`
- `LEMON_SQUEEZY_WEBHOOK_SECRET`

## Environment Variables

| Variable Name | Used By | Required |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | GitHub Actions deploy via Wrangler | Yes |
| `CLOUDFLARE_ACCOUNT_ID` | GitHub Actions deploy via Wrangler | Yes |
| `LEMON_SQUEEZY_API_KEY` | Worker: checkout creation | Yes |
| `LEMON_SQUEEZY_STORE_ID` | Worker: checkout creation | Yes |
| `LEMON_SQUEEZY_PRODUCT_ID` | Worker: purchase gate | Yes |
| `LEMON_SQUEEZY_VARIANT_ID` | Worker: checkout creation | Yes |
| `LEMON_SQUEEZY_CHECKOUT_URL` | Worker: purchase gate redirect | Yes |
| `LEMON_SQUEEZY_WEBHOOK_SECRET` | Worker: webhook HMAC verification | Yes |
| `WORKER_URL` | Worker: checkout redirect URL | Yes |

## Open Questions

- [ASSUMPTION] `pricing_model` is `one-time` as provided | Verify this is the final model — subscription could increase LTV
- [ASSUMPTION] `price_usd` is $29 | Validate against competitor landscape — $29 one-time is significantly cheaper long-term but may signal low perceived value
- [ASSUMPTION] Currency is USD | Customer profile specifies Euro pricing for European markets — should this be EUR 29 instead? Verify before launch
- [ASSUMPTION] `github_repo_url` was `acme-org/invoicezap` but org doesn't exist — repo created at `mxkt/invoicezap` | Update Notion entry with correct URL
- [ASSUMPTION] Auto-reminders use Cloudflare Workers cron at 09:00 UTC daily | Verify timezone preference for European target market
- [ASSUMPTION] No email delivery service specified for sending invoices and reminders | Must choose: Cloudflare Email Workers, Resend, or Mailgun before launch
- [ASSUMPTION] LemonSqueezy credentials not yet configured in credential store | Must set LEMON_SQUEEZY_API_KEY and LEMON_SQUEEZY_STORE_ID before Phase 5 can complete
