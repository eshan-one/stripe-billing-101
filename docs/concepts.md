# Concepts

The Stripe vocabulary you need before reading any code or docs.

## Payments, Billing, Subscription

They sit at different levels: two are Stripe product areas, one is an object inside Billing.

| | What it is | Think of it as | Main objects |
|---|---|---|---|
| **Payments** | Moving money once | "Take $100 now" | PaymentIntent, Charge, Checkout Session (one-time), Payment Links |
| **Billing** | Recurring charges and invoicing | "Charge $100 every month, send invoices, retry failures" | Subscription, Invoice, Price, Customer, Customer Portal |
| **Subscription** | One record in Billing | "Acme is on the Pro plan, monthly" | Subscription, which owns the schedule, status and next renewal |

Billing sits on top of Payments. A subscription creates an invoice every period, and Stripe collects that invoice through the Payments machinery. Checkout Sessions appear in both: in `mode=payment` a session makes one payment, in `mode=subscription` it makes a subscription.

For a typical SaaS, **plans** are Billing (subscriptions) and **prepaid packs or credits** are Payments (one-time).

## Product and Price

| Object | Answers | Example | ID prefix |
|---|---|---|---|
| **Product** | What are we selling? | "Plus" | `prod_` |
| **Price** | How much, and how often? | $8.00 every month | `price_` |

A price belongs to one product, and a product can have several prices (monthly, yearly). They are separate so you can change what you charge without touching the product. Existing subscribers stay on their old price.

Key price fields (see [price.json](../samples/day-01/price.json)):

| Field | Value | Meaning |
|---|---|---|
| `unit_amount` | `800` | $8.00, in cents |
| `type` / `recurring.interval` | `recurring` / `month` | Charged every month |
| `billing_scheme` | `per_unit` | Amount = unit price x quantity. This is how per-seat pricing works. |
| `recurring.usage_type` | `licensed` | Billed upfront each period. The alternative `metered` bills after the fact, by usage. |
| `lookup_key` | `null` | Optional stable name, e.g. `starter_monthly`. IDs differ between sandbox and live, so code can look prices up by this name. |

## ID prefixes

| Prefix | Object |
|---|---|
| `prod_` | Product |
| `price_` | Price |
| `cus_` | Customer |
| `sub_` | Subscription |
| `si_` | Subscription item (one price + quantity inside a subscription) |
| `in_` | Invoice |
| `pi_` | PaymentIntent |
| `pm_` | Payment method (a saved card) |
| `cs_test_` / `cs_live_` | Checkout Session |
| `evt_` | Event (what webhooks deliver) |
| `whsec_` | Webhook signing secret (keep out of git) |
| `sk_test_` / `sk_live_` | Secret API key (keep out of git) |

## Conventions inside the API

- **Money is in cents.** `800` is $8.00.
- **Time is a Unix timestamp in seconds** (`1793902038`).
- **Requests are form-encoded, responses are JSON.** Nested fields use brackets: `line_items[0][price]=price_...`. The CLI's `-d` flag and `curl -d` both send this form.
- **Billing period dates live on the subscription item**, not on the subscription: `items.data[0].current_period_start` and `current_period_end` (see [subscription.json](../samples/day-01/subscription.json)).
- **`livemode: false`** on every object means it is sandbox data.

## Checkout modes

| `mode` | Use |
|---|---|
| `subscription` | Recurring plan |
| `payment` | One-time purchase, such as a credit pack |
| `setup` | Save a card without charging it (not explored yet) |

## Webhook principles

1. **Webhooks are the source of truth**, not the browser redirect to `success_url`.
2. **Order is not guaranteed.** In the first checkout `invoice.paid` arrived before `customer.created`. Read the data inside each event.
3. **Events can repeat.** Store each `evt_` ID and skip ones already processed.
4. **Verify the signature** with the `whsec_` secret before trusting a payload.
5. **Wait for `invoice.paid`** before granting access. `checkout.session.completed` alone is not enough for slow payment methods.
6. **A webhook has no login and no user context.** Link Stripe objects to your own user or organisation through data you set yourself, such as `metadata.orgId` or `client_reference_id`.

## A typical SaaS billing page, mapped to Stripe

| On the page | What it really is | Stripe concept |
|---|---|---|
| Starter plan with a trial and a next billing date | Flat recurring plan with a trial | Product + recurring Price, Subscription with a trial |
| Team plan, "price per user", an Add users stepper | Per-seat plan and a mid-cycle upgrade | Per-unit Price with `quantity`, plan change with proration |
| "Total due now" in an upgrade dialog | Preview of today's charge | Invoice preview (Stripe computes the proration) |
| Credit pack, e.g. 20,000 credits for $100 | One-time purchase | One-time Price x quantity, `mode=payment` |
| Auto top-up toggle | Charge a saved card when the balance is low | Saved payment method + off-session payment |
| Credit balance | Your own number | Your database, not Stripe |
| Billing history with a download link | Invoices and receipts in one list | Invoices (hosted URL or PDF) mirrored into your database by webhooks |
