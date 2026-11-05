# Stripe guide coverage

Stripe's guide: [Sell subscriptions as a SaaS startup](https://docs.stripe.com/get-started/use-cases/saas-subscriptions). This table maps every section of it to what was done, and where it is documented. The guide implements **flat-rate pricing only**, with hosted Checkout and a Node server.

Legend: **Done** = run and verified. **Replaced** = done another way. **Next** = Day 2. **Decide** = a business choice to make for a real product. **Later** = needs a real app or a live account.

## Create a test product and price

| Guide step | Status | Notes |
|---|---|---|
| Create the "Plus" product, $8, USD | Done | [Part 1](README.md#part-1-product-and-price) |
| Upload a test product image | Done | Optional |
| Select Recurring | Done | Monthly |

## Accept subscription payments

| Guide section | Status | Notes |
|---|---|---|
| Stripe-hosted Checkout | Done | [Part 3](README.md#part-3-first-subscription-no-code) |
| Pricing table (alternative to a custom page) | Not used | A real app would use its own plan cards |
| Set up your server: Node library, create a Checkout Session, define the product, `mode=subscription`, success URL, redirect | Replaced | CLI and curl show the same API call |
| Create your checkout page: order preview page and checkout button | Later | Frontend work in a real app |
| Test your checkout page with `4242...` | Done | Part 3 |
| Test with the sample React/Node app | Skipped (optional) | [Part 4](README.md#part-4-stripes-node-app-optional) |
| Test clocks for renewals, trials and failures | Next | Day 2, Part 7 |

## Additional recommended configurations

| Guide section | Status | Notes |
|---|---|---|
| Customise the payment page (branding) | Later | Cosmetic |
| Add payment methods | Decide | Checkout showed Card, Link, Apple Pay, Amazon Pay, Cash App, Bank. A B2B product may want cards only. |
| Enable discounts (coupons, promotion codes) | Decide | |
| Add trial periods (`trial_period_days`, `trial_end`) | Next | Day 2, Part 6 |
| Add a customer portal | Next | Day 2, Part 8 |

## Monitor your subscriptions

| Guide section | Status | Notes |
|---|---|---|
| Create a webhook endpoint on the server | Later | Needs app code |
| Register it in the Dashboard and select events | Later | Needs a real URL |
| Signature verification with `STRIPE_WEBHOOK_SECRET` | Later | `stripe listen` prints a `whsec_` secret for local use |
| Watch `checkout.session.completed` | Done (observed) | [Events](README.md#reference-events) |
| Watch `invoice.paid` | Done (observed) | |
| Watch `invoice.payment_failed` | Next | Appears on renewals, so needs test clocks |
| Wait for `invoice.paid` before provisioning access | Understood | Webhook principle 5 in [concepts](../concepts.md#webhook-principles) |
| Use Entitlements to grant feature access | Decide | Stripe's feature-access system, or your own plan checks |
| Set the billing cycle (`billing_cycle_anchor`) | Next | Command in [Day 2](README.md#billing-cycle-anchor-from-stripes-guide) |

## Go live

| Guide section | Status | Notes |
|---|---|---|
| Account settings, business and personal verification, bank details, two-step authentication | Later | Only when launching a real product |
| Optional automatic tax collection | Decide | |

## Next steps

| Guide section | Status | Notes |
|---|---|---|
| Recover failed payments (Smart Retries, recovery emails) | Next | Test alongside failed renewals in Day 2 |
| Register to collect tax, enable automatic tax | Decide | |
| Cancel or refund a payment | Decide | |
| Set up payouts | Later | Live account only |
| Enable Adaptive Pricing | Decide | The currency switch seen on the Checkout page |
| Send invoices | Next | Relevant to a "download invoice" link |
| Automate revenue recognition | Later | Finance tooling |

## What the guide does not cover

The guide is flat-rate only. A fuller SaaS billing page also needs:

- **Per-seat pricing**: quantity, plan upgrades with proration, and an invoice preview for "total due now".
- **One-time credit packs**: `mode=payment` (Day 2, Part 9).
- **Auto top-up**: saving a card and charging it later without the customer present.
- **Billing history** with a download link: Stripe invoices have a hosted URL and a PDF.

Stripe docs to read for these: per-seat pricing, upgrades and downgrades (prorations), invoice previews, saving a payment method for later use, and off-session payments.
