# Wekraft SaaS: Payment Integration & Webhook Guide
> Last Updated: June 25, 2026 — Added LemonSqueezy provider, invite link fix verified

---

## 1. High-Level Architecture Overview

Wekraft uses a dynamic, location-based payment routing engine with three payment providers:
- **Razorpay** — Indian users (UPI, cards, netbanking)
- **Stripe** — Global users (international credit/debit cards)
- **LemonSqueezy** — Alternative global provider (hosted checkout, simpler merchant setup)

`
[Pricing Page / Upgrade Click]
         |
[IP Geolocation Check (ipapi.co)]
         |
   India (IN)?
   /           \
 YES            NO
  |              |
Razorpay      Stripe or LemonSqueezy
  Flow             Flow

[Daily Cron 00:30 UTC] --> Check cancelAtPeriodEnd=true AND currentPeriodEnd expired --> Downgrade to free
`

### Key Technical Decisions

1. Server-Side Pricing Control: All plans, amounts, and metadata are validated on the server. The client cannot send custom pricing.
2. Authenticated Cancel & Portal Routes: /api/payments/razorpay/cancel and /api/payments/stripe/portal require a valid Clerk session AND verify resource ownership before taking action.
3. Graceful Cancellations (No Instant Downgrades): When users cancel, their premium plans remain active with cancelAtPeriodEnd: true in Convex until their paid billing period ends.
4. Webhook Signature Security: All webhook entry points use crypto.timingSafeEqual (Razorpay) and stripe.webhooks.constructEvent (Stripe). The Razorpay verify route returns 500 if RAZORPAY_KEY_SECRET is missing — never falls back to an empty string.
5. Plan Upgrade is Server-Only: The upgradeAccount Convex mutation is an internalMutation — cannot be called from any browser client.

### Relevant File Structure

`
wekraft-saas/
├── src/
│   ├── modules/web/Pricing.tsx                         # Frontend pricing UI, checkout triggers, location routing
│   ├── modules/payments/hooks/
│   │   ├── useRazorpay.ts                              # Razorpay client hook (modal, verification flow)
│   │   └── useStripeCheckout.ts                        # Stripe client hook (redirect to checkout session)
│   └── app/api/payments/
│       ├── stripe/
│       │   ├── checkout/route.ts                       # Generates Stripe Checkout Session URL
│       │   ├── portal/route.ts                         # Generates Stripe Customer Portal URL (Cancel/Manage)
│       │   └── webhook/route.ts                        # Handles Stripe webhooks
│       ├── razorpay/
│       │   ├── subscription/route.ts                   # Calls Razorpay SDK to create a subscription order
│       │   ├── verify/route.ts                         # Validates HMAC signature, updates Convex DB on success
│       │   ├── cancel/route.ts                         # Issues cancel request to Razorpay, marks flag in DB
│       │   └── webhook/route.ts                        # Handles Razorpay webhooks
│       └── lemonsqueezy/
│           ├── checkout/route.ts                       # Creates Lemon Squeezy hosted checkout session
│           └── webhook/route.ts                        # Handles Lemon Squeezy subscription events
├── convex/
│   ├── schema.ts                                       # 'users' table definition with all subscription fields
│   ├── razorpay.ts                                     # Razorpay DB mutations (updatePlan, verifyOwner)
│   ├── lemonsqueezy.ts                                 # LemonSqueezy DB mutations (internalMutation)
│   ├── payments.ts                                     # Cron job: downgradeExpiredPlans
│   └── crons.ts                                        # Cron scheduler (runs downgradeExpiredPlans daily)
└── Docs/
    └── payment_integration.md                          # This master guide
`

---

## 2. Dynamic Location-Based Routing

The entry point for payment selection resides in src/modules/web/Pricing.tsx:

1. On page load, the component fetches https://ipapi.co/json/.
2. Extracts country_code (e.g., "US", "IN", "GB").
3. If countryCode === "IN": prices render in Rupees (Rs.) and checkout routes through useRazorpay hook.
4. If countryCode !== "IN": prices render in USD ($) and checkout routes to Stripe or LemonSqueezy.

PRODUCTION WARNING: The free tier of ipapi.co is rate-limited. If it fails, country_code defaults to null (Stripe route). To prevent Indian users from hitting Stripe (which fails on UPI/local cards), replace this with Vercel x-vercel-ip-country header or a paid geolocation provider.

---

## 3. Stripe Integration

Stripe is integrated server-side using hosted Checkout Sessions and the Customer Billing Portal.

### 3.1 Stripe Keys

Log into dashboard.stripe.com and obtain:
- STRIPE_SECRET_KEY / NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: Developers -> API Keys
- STRIPE_PLUS_PRICE_ID / STRIPE_PRO_PRICE_ID: Product Catalog -> Add Product -> create recurring monthly products -> copy Price IDs (price_...) NOT Product IDs (prod_...)
- STRIPE_WEBHOOK_SECRET: Generated when creating a webhook endpoint

### 3.2 Local Webhook Testing (Stripe CLI)

Method 1: Browser Authentication
stripe login
stripe listen --forward-to localhost:3000/api/payments/stripe/webhook

Method 2: API Key Bypass
stripe listen --api-key sk_test_YOUR_SECRET_KEY --forward-to localhost:3000/api/payments/stripe/webhook

After running, copy the whsec_... signing secret and set as STRIPE_WEBHOOK_SECRET in .env.local. Restart dev server.

### 3.3 Production Webhook Events

- checkout.session.completed — initial payment fulfillment
- customer.subscription.updated — scheduled cancellations, plan changes
- customer.subscription.deleted — downgrade when billing term ends

### 3.4 Stripe Billing Portal (Cancel / Manage)

Route: /api/payments/stripe/portal
1. Caller must be authenticated (Clerk session required -> 401 otherwise).
2. customerId in request body is verified against caller's Convex record via api.stripe.verifyCustomerOwner -> 403 if mismatch.
3. Only then is a Stripe Billing Portal session created and URL returned.

---

## 4. Razorpay Integration

Razorpay uses a hybrid flow: subscription created on the server, processed on the client via Razorpay Web SDK, then verified securely back on the server.

### 4.1 Razorpay Keys

Log into dashboard.razorpay.com:
- NEXT_PUBLIC_RAZORPAY_KEY_ID / RAZORPAY_KEY_SECRET: Account & Settings -> API Keys
- RAZORPAY_PLUS_PLAN_ID / RAZORPAY_PRO_PLAN_ID: Subscriptions -> Plans -> Create Plan (monthly, Rs. 649 / Rs. 1499) -> copy plan_... IDs
- RAZORPAY_WEBHOOK_SECRET: Set when creating a webhook

### 4.2 Local Webhook Testing (ngrok)

ngrok http 3000
Set webhook URL in Razorpay Dashboard: https://xxxx.ngrok-free.app/api/payments/razorpay/webhook

Active events:
- subscription.charged
- subscription.cancelled
- subscription.halted
- subscription.paused
- subscription.resumed

### 4.3 Razorpay Cancel Flow

Route: /api/payments/razorpay/cancel
1. Caller must be authenticated (Clerk session -> 401 otherwise).
2. subscriptionId is verified against caller's Convex record via api.razorpay.verifySubscriptionOwner -> 403 if mismatch.
3. razorpay.subscriptions.cancel(subscriptionId, true) executes — the true flag cancels at END of billing cycle, not immediately.
4. Convex is immediately patched with cancelAtPeriodEnd: true so UI shows "Ends on [Date]".

### 4.4 Razorpay Payment Signature Verification

Route: /api/payments/razorpay/verify
HMAC: SHA256(RAZORPAY_KEY_SECRET, razorpay_payment_id + "|" + razorpay_subscription_id)

Security rules:
- If RAZORPAY_KEY_SECRET is missing, route returns 500 — never falls back to empty string.
- Uses crypto.timingSafeEqual wrapped in try/catch — no pre-length-check (which would re-introduce timing side channel).
- Proper Razorpay SDK error introspection instead of generic [object Object] strings.

---

## 5. LemonSqueezy Integration (NEW)

LemonSqueezy is available as an alternative global payment provider with a simpler merchant setup than Stripe.

### 5.1 Flow

1. Client calls POST /api/payments/lemonsqueezy/checkout with { planName, planType, priceUSD, userEmail }.
2. Route requires Clerk authentication (401 if missing).
3. Route looks up user via api.user.getUserByClerkToken.
4. Creates a Lemon Squeezy hosted checkout via API v1 with:
   - checkout_data.custom: { userId: convexUserId, planType } embedded in the checkout payload.
   - product_options.redirect_url: Set to origin/dashboard?success=true.
5. Returns { url: checkoutUrl } — client redirects user to the Lemon Squeezy hosted checkout page.

### 5.2 LemonSqueezy Keys

Log into app.lemonsqueezy.com:
- LEMONSQUEEZY_API_KEY: Settings -> API
- LEMONSQUEEZY_STORE_ID: Your store ID
- LEMONSQUEEZY_PLUS_VARIANT_ID: The variant ID for the Plus plan product
- LEMONSQUEEZY_PRO_VARIANT_ID: The variant ID for the Pro plan product

### 5.3 Convex Integration

File: convex/lemonsqueezy.ts

updatePlanServerSideInternal (internalMutation):
- Called from the /api/payments/lemonsqueezy/webhook handler.
- Updates accountType, subscriptionId, customerId, subscriptionStatus, currentPeriodEnd.
- Only upgrades plan if status is "active", "on_trial", or "trialing". Otherwise preserves current plan.

### 5.4 Webhook Setup

Active events to subscribe to in Lemon Squeezy Dashboard:
- order_created — initial payment
- subscription_updated — plan changes, cancellations
- subscription_cancelled — end of billing period

Webhook URL: https://yourdomain.com/api/payments/lemonsqueezy/webhook

---

## 6. Convex Database Architecture & Subscription Tracking

Both payment integrations feed into Convex. The exact status and remaining time of a user's subscription is continuously tracked and synced.

### 6.1 User Document — Payment Fields

From convex/schema.ts:
- accountType: "free" | "plus" | "pro" — active plan tier
- subscriptionId: subscription ID from Stripe or Razorpay or LemonSqueezy
- customerId: Stripe Customer ID (used for portal redirect) or LemonSqueezy customer ID
- subscriptionStatus: "active" | "past_due" | "cancelled" | "on_trial" etc.
- subscriptionProvider: "razorpay" | "stripe" | "lemonsqueezy"
- currentPeriodEnd: Unix timestamp (ms) of billing period end
- cancelAtPeriodEnd: boolean — true when cancelled but still active until period end
- planExpiry: for temporary coupon-based upgrades

### 6.2 Graceful Downgrade Flow

1. Setting the Expiry Clock: When subscription is charged, webhook patches currentPeriodEnd.
2. Triggering Cancel Intent:
   - Stripe: Portal webhook patches cancelAtPeriodEnd: true.
   - Razorpay: Cancel API + immediate Convex patch.
   - LemonSqueezy: subscription_cancelled webhook.
3. Executing Downgrade:
   - Webhook fires on period end -> accountType: "free".
   - Daily cron at 00:30 UTC as safety net: checks cancelAtPeriodEnd === true AND currentPeriodEnd < Date.now() -> downgrade.
4. Upgrading after Canceling: updatePlanServerSide actively injects cancelAtPeriodEnd: false into the database patch.

---

## 7. Auto-Downgrade Cron (Safety Net)

File: convex/payments.ts — downgradeExpiredPlans
Schedule: Daily at 00:30 UTC (defined in convex/crons.ts)

This cron is the safety net for when provider webhooks are delayed or missed. It:
1. Queries all "plus" and "pro" users using the by_accountType index.
2. Filters those where cancelAtPeriodEnd === true AND currentPeriodEnd < Date.now().
3. Patches matching users: accountType -> "free", subscriptionStatus -> "canceled", cancelAtPeriodEnd -> false.

IMPORTANT: This cron runs in addition to (not instead of) webhook-based downgrades. Webhooks are primary; cron is the daily catch-all.

---

## 8. Complete Environment Variable Reference

| Variable | Required For | Source | Notes |
|---|---|---|---|
| NEXT_PUBLIC_RAZORPAY_KEY_ID | Client SDK + verify route | Razorpay API Keys | rzp_live_... in production |
| RAZORPAY_KEY_SECRET | Server verify + cancel routes | Razorpay API Keys | Never expose to client. Missing = 500 |
| RAZORPAY_PLUS_PLAN_ID | Subscription creation | Razorpay Plans | Must be real plan_... ID |
| RAZORPAY_PRO_PLAN_ID | Subscription creation | Razorpay Plans | Must be real plan_... ID |
| RAZORPAY_WEBHOOK_SECRET | Webhook signature verification | Razorpay Webhook Settings | Strong random secret |
| STRIPE_SECRET_KEY | All Stripe server routes | Stripe API Keys | sk_live_... in production |
| NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY | Client reference | Stripe API Keys | Currently unused client-side |
| STRIPE_WEBHOOK_SECRET | Stripe webhook verifier | Stripe CLI / Webhook Settings | whsec_... |
| STRIPE_SUCCESS_URL | Redirect after checkout | Your app URL | Use production URL, not localhost |
| STRIPE_CANCEL_URL | Redirect on checkout cancel | Your app URL | Use production URL, not localhost |
| STRIPE_PLUS_PRICE_ID | Stripe Checkout | Stripe Product Catalog | Must be price_..., NOT prod_... |
| STRIPE_PRO_PRICE_ID | Stripe Checkout | Stripe Product Catalog | Must be price_..., NOT prod_... |
| LEMONSQUEEZY_API_KEY | LemonSqueezy routes | LS Dashboard -> API | Bearer token |
| LEMONSQUEEZY_STORE_ID | LemonSqueezy checkout | LS Dashboard | Numeric store ID |
| LEMONSQUEEZY_PLUS_VARIANT_ID | LS checkout route | LS Product Catalog | Variant ID for Plus plan |
| LEMONSQUEEZY_PRO_VARIANT_ID | LS checkout route | LS Product Catalog | Variant ID for Pro plan |
| BACKEND_SECRET | Server-to-Convex auth | Generated 64-char hex | Payment mutations validate this |
| NEXT_PUBLIC_CONVEX_URL | ConvexHttpClient in API routes | Convex Dashboard | https://xxx.convex.cloud |
| NEXT_PUBLIC_APP_URL | Invite links, redirects | Your app URL | https://wekraft.xyz in production |

---

## 9. API Route Security Summary

| Route | Auth Required | Ownership Check | Notes |
|---|---|---|---|
| POST /api/payments/stripe/checkout | None | - | Rate limit recommended |
| POST /api/payments/stripe/webhook | Stripe signature | - | Public (server-to-server) |
| POST /api/payments/stripe/portal | Clerk auth | customerId ownership | 401/403 on violation |
| POST /api/payments/razorpay/subscription | None | - | Rate limit recommended |
| POST /api/payments/razorpay/verify | None | HMAC signature | Signature IS the proof |
| POST /api/payments/razorpay/cancel | Clerk auth | subscriptionId ownership | 401/403 on violation |
| POST /api/payments/razorpay/webhook | Razorpay HMAC | - | Public (server-to-server) |
| POST /api/payments/lemonsqueezy/checkout | Clerk auth | - | Returns LS checkout URL |
| POST /api/payments/lemonsqueezy/webhook | LS signature | - | Public (server-to-server) |

---

## 10. Adding New Plans (Future Expansion)

To add a new tier (e.g., "Enterprise"):

1. Schema: Add v.literal("enterprise") to accountType in convex/schema.ts.
2. Plan limits: Add "enterprise" entry to PLAN_CONFIGS in convex/pricing.ts.
3. Stripe: Create product -> get price_... ID -> set STRIPE_ENTERPRISE_PRICE_ID.
4. Razorpay: Create plan -> get plan_... ID -> set RAZORPAY_ENTERPRISE_PLAN_ID.
5. LemonSqueezy: Create variant -> get ID -> set LEMONSQUEEZY_ENTERPRISE_VARIANT_ID.
6. Checkout routes: Add planType === "enterprise" branch in all three checkout routes.
7. UI: Add new plan object to the plans array in Pricing.tsx.
8. Convex mutations: Update updatePlanServerSide arg validators in stripe.ts, razorpay.ts, and lemonsqueezy.ts to accept "enterprise".
