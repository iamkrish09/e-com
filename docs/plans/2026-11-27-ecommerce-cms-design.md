# E-commerce store with custom CMS: design

Date: 2026-11-27
Status: approved by user, section by section

## Goals and scope

A single-brand online store with a custom CMS for staff. One Next.js app on Supabase Postgres. Payments through Razorpay. The store sells in one currency, chosen in the CMS (INR by default).

Version 1 includes:
- Storefront with SEO-friendly server rendering.
- Product variants (options such as size and color, each variant with its own SKU, price and stock).
- Cart for guests, sign-in with Google at checkout.
- Razorpay checkout confirmed by a signed webhook.
- CMS for commerce administration: products, categories, inventory, orders, customers, coupons, settings, staff.
- Multiple languages, controlled from the CMS.

Deferred: content management in the CMS (banners, pages, blog, page builder), order tracking, mobile OTP sign-in, zone-based shipping, GST breakdowns, multiple currencies.

## Decisions

| Topic | Decision |
|---|---|
| Store type | Single brand |
| Stack | Next.js (App Router, SSR and CSR), Supabase Postgres, Supabase Auth, Supabase Storage |
| Server side | No separate backend. Server actions and route handlers in Next.js |
| App structure | One codebase. Store on `www.*`, CMS on `admin.*`, routed by middleware |
| Payments | Razorpay, order confirmed by webhook |
| Catalog | Products with variants |
| Auth | Browse without sign-in. Google sign-in required at checkout. Mobile OTP later |
| CMS scope v1 | Commerce admin only |
| Currency | One store currency, set in the CMS |
| Languages | next-intl, languages and interface text controlled from the CMS |
| Shipping and tax | Flat shipping rate, tax-inclusive prices. Behind `calculateShipping` and `calculateTax` so they can be replaced |

## 1. Structure and hosts

```
src/
  app/
    (store)/        home, category, product, cart, checkout, account
    (admin)/        products, inventory, orders, customers, coupons, staff, settings
    api/webhooks/razorpay/
  features/         catalog, cart, checkout, orders, payments, customers,
                    coupons, inventory, auth
                    each: queries/, actions/, schemas/, components/, index.ts
  lib/              supabase clients (browser, server, admin), razorpay, env, utils
  middleware.ts     host routing and auth guards
supabase/
  migrations/  seed.sql  policies
```

- `middleware.ts` reads the host. `admin.*` maps to the `(admin)` group and `www.*` to `(store)`. Admin paths on the store host return 404, and the reverse. Locally, `admin.localhost:3000` works the same way.
- Home, category and product pages use server rendering with ISR, revalidated by tag when the CMS saves. They include metadata, canonical URLs, JSON-LD `Product` data, `hreflang` and a sitemap.
- Cart, checkout and account pages render on the client with data from server actions. The CMS is dynamic and never cached.
- Mutations are server actions validated with Zod. Webhooks are route handlers. Features import each other only through their `index.ts`.

## 2. Data model and security

Tables:
- Catalog: `categories`, `products`, `product_options`, `option_values`, `variants`, `variant_option_values`, `product_images`, and `*_translations` tables for names, descriptions, slugs and SEO fields.
- People: `profiles`, `addresses`, `staff_roles` (`admin`, `editor`).
- Cart: `carts`, `cart_items` (signed-in users only).
- Orders: `orders`, `order_items` (name, SKU and price copied at purchase), `order_events`, `payments`. Each order stores its own currency.
- Inventory: `inventory_movements`.
- Marketing: `coupons`, `coupon_redemptions`.
- Settings: `store_settings` (currency, enabled languages, default language, shipping rate, free-shipping threshold), `translations` (interface text by locale and key).
- Audit: `audit_log`.

Rules:
- Money is stored as integers in the smallest currency unit (paise for INR).
- Stock decreases in one Postgres function that locks the variant row, so two buyers cannot take the last item. Each change is recorded in `inventory_movements`.
- Row-level security is on for every table. Anonymous users read only published catalog data. Customers read and write only their own profile, addresses and cart, and read only their own orders. Orders and payments are written only by server code using the service-role key. Staff access is checked with an `is_staff()` function. Editors manage the catalog, and only admins manage staff, refunds, coupons and settings.
- A database trigger creates the `profiles` row on first sign-in.

## 3. Cart, checkout and payments

Cart:
- A guest cart is a cookie holding variant ids and quantities only. Prices are always read from the database.
- On sign-in the cookie cart merges into `carts` and `cart_items`. Quantities for the same variant are added and capped at stock.

Checkout:
1. Customer starts checkout. If not signed in, Google sign-in runs and returns to the same step.
2. Customer picks or adds an address.
3. `createOrder` re-reads prices and stock, applies the coupon, adds shipping and tax, writes an order with status `pending`, creates the Razorpay order and stores its id in `payments`.
4. The browser opens the Razorpay widget with that order id. The amount never comes from the browser.
5. `verifyPayment` checks the signature and shows the success page. It does not mark the order paid.

Webhook `/api/webhooks/razorpay`:
- Verifies the signature on the raw body. Handles `payment.captured` and `payment.failed`. Ignores events already processed, keyed by payment id.
- On capture, one transaction sets the order to `paid`, decrements stock, records the coupon redemption, writes an `order_events` row and empties the cart.

Edge cases:
- If stock ran out before capture, the order is marked `needs_review` and shown in the CMS for a manual refund. Pending orders do not reserve stock in v1.
- Pending orders with no payment after 30 minutes become `expired` through a scheduled job.
- Admins start refunds from the CMS through the Razorpay refunds API.

## 4. CMS

Access: Google sign-in on `admin.*`, allowed only for users with a `staff_roles` row. Others get 404. The first admin is created by a seed script that takes an email. Admins invite other staff.

| Area | Editor | Admin |
|---|---|---|
| Products, variants, categories, images | yes | yes |
| Inventory adjustments | yes | yes |
| Orders (view, update status) | yes | yes |
| Customers (view) | yes | yes |
| Translations | yes | yes |
| Coupons | no | yes |
| Refunds | no | yes |
| Store settings | no | yes |
| Staff management | no | yes |

Screens: dashboard (orders, revenue, low stock, `needs_review` orders), products (list, form, variants grid, images, per-language text, draft and published), categories (tree), inventory, orders (detail with status timeline, refund), customers, coupons, settings (currency, languages, shipping, translations editor).

Images go to a Supabase Storage bucket, public read and staff write, served through Next.js `Image`. Every CMS write adds an `audit_log` row.

## 5. Errors, testing, security and deployment

Errors:
- Server actions return `{ ok: true, data }` or `{ ok: false, error }`. Expected failures use error codes mapped to translated messages.
- Unexpected errors go to `error.tsx` boundaries and Sentry.
- Failed webhooks return 5xx so Razorpay retries. Retries are safe.

Testing (test-driven development):
- Unit (Vitest): price, coupon, shipping and tax calculation, cart merge, signature check.
- Database: RLS policies and the stock function, including concurrent buyers.
- Integration: `createOrder` and the webhook against a local Supabase with Razorpay mocked.
- End to end (Playwright): browse, sign in, pay in Razorpay test mode, see the order. CMS creates a product that appears on the storefront.
- Lint, typecheck, unit tests and the build must pass before work is called done.

Security:
- Service-role key and Razorpay secret exist only in server environment variables.
- Constant-time comparison for signatures. CMS routes are checked in middleware and again in each action.
- Rate limits on checkout and sign-in. Standard security headers.

Deployment: Vercel with both domains on one project. Separate Supabase projects for development and production. Migrations through the Supabase CLI. Vercel Cron for expiring pending orders.

## Open items

- Domain names for the store and the CMS.
- Razorpay account in test mode, and Google OAuth credentials.
- This folder is not a git repository. Run `git init` before the implementation plan so each task can be committed.
