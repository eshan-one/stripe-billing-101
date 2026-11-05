# Day 1 guide: your first subscription in the Stripe sandbox

**Status:** Parts 0 to 3 and 5 are done. Part 4 is optional and skipped. Parts 6 to 9 are Day 2: the commands are here but **not yet run**.

This follows Stripe's guide [Sell subscriptions as a SaaS startup](https://docs.stripe.com/get-started/use-cases/saas-subscriptions), but runs it mostly from the **Stripe CLI** instead of the guide's Node app. The CLI makes the same API calls your backend will make, so you see the real requests and responses. How each section of Stripe's guide maps to what was done is in [stripe-guide-coverage.md](stripe-guide-coverage.md).

**Time:** about 2 hours for Parts 0 to 3 and 5. **Samples:** [samples/day-01](../../samples/day-01/) (trimmed and anonymised).

## Contents

1. [Before you start](#before-you-start)
2. [Part 0: Setup](#part-0-setup)
3. [Part 1: Product and price](#part-1-product-and-price)
4. [Part 2: Watch events live](#part-2-watch-events-live)
5. [Part 3: First subscription, no code](#part-3-first-subscription-no-code)
6. [Part 4: Stripe's Node app (optional)](#part-4-stripes-node-app-optional)
7. [Part 5: Payment failures and 3D Secure](#part-5-payment-failures-and-3d-secure)
8. [Reference: events](#reference-events)
9. [Reference: reading the responses](#reference-reading-the-responses)
10. [Test cards](#test-cards)
11. [Troubleshooting](#troubleshooting)
12. [What I learnt](#what-i-learnt)
13. [Day 2 (not yet run)](#day-2-not-yet-run)

## Before you start

- A Stripe account in **sandbox** mode (free). Check the Dashboard banner says "Sandbox" every time.
- macOS with Homebrew (for the CLI).
- Use **two terminal tabs**: one for commands, one for `stripe listen`.
- Keep a notes file for the IDs and event names you see.
- Never paste a secret key (`sk_test_...`) or signing secret (`whsec_...`) into chat, tickets or git.

Requests to Stripe are **form-encoded**, and responses are JSON. Nested fields use brackets, for example `line_items[0][price]=price_...`. In `zsh` the whole argument must be in quotes, otherwise the brackets are treated as a pattern and the command fails.

## Part 0: Setup

1. Install the Stripe CLI:

```bash
brew install stripe/stripe-cli/stripe
```

2. Log the CLI into your sandbox. A browser opens for you to authorise it:

```bash
stripe login
```

3. Check that it works:

```bash
stripe products list --limit 3
```

**Checkpoint:** you get a JSON list (or `"data": []`), not an authentication error.

## Part 1: Product and price

Stripe splits "what you sell" from "what it costs" (see [concepts](../concepts.md#product-and-price)).

1. In the Dashboard, open **Product catalogue** and add a product:
   - **Name:** `Plus`
   - **Price:** `8`, currency **USD**
   - **Recurring**, billed **monthly**
   - Image optional
2. Copy the price ID (`price_...`) from the price row, or list it from the CLI:

```bash
stripe prices list --limit 3
```

**Expected output:** [product.json](../../samples/day-01/product.json) and [price.json](../../samples/day-01/price.json). The fields that matter:

| Field | Value | Meaning |
|---|---|---|
| product `default_price` | `price_EXAMPLE` | The price ID. You pass it to Checkout. |
| price `unit_amount` | `800` | $8.00 in cents |
| price `recurring.interval` | `month` | Monthly |
| price `type` | `recurring` | Not a one-time price |
| `livemode` | `false` | Sandbox |

**Checkpoint:** you have a `price_...` ID in your notes, and the Dashboard shows Plus at US$8.00 per month with 0 active subscriptions.

## Part 2: Watch events live

In the **second terminal tab**, run and leave it running for the rest of the exercise:

```bash
stripe listen
```

You should see `Ready! ... Your webhook signing secret is whsec_...`. That is the same kind of secret a real webhook endpoint uses to verify signatures. Do not commit it.

Nothing prints until something happens in the sandbox. To prove it works, edit the product description in the Dashboard: a `product.updated` event appears.

## Part 3: First subscription, no code

This is what Stripe's guide does in server code: create a Checkout Session in `subscription` mode.

```bash
stripe checkout sessions create -d mode=subscription -d success_url=https://example.com/success -d "line_items[0][price]=<PRICE_ID>" -d "line_items[0][quantity]=1"
```

### If it fails with a Managed Payments error

Some accounts have **Managed Payments** enabled by default. Checkout then fails if the product has no tax code, with an error like [error-managed-payments.json](../../samples/day-01/error-managed-payments.json). Switching it off for one session changes nothing in the account:

```bash
stripe checkout sessions create -d mode=subscription -d success_url=https://example.com/success -d "line_items[0][price]=<PRICE_ID>" -d "line_items[0][quantity]=1" -d "managed_payments[enabled]=false"
```

Managed Payments is an account-level business decision (as I understand it, Stripe becomes the seller of record), so do not change the account-wide setting just to get a command working.

### The same request with curl

This is the request your backend makes through a Stripe SDK. `STRIPE_SECRET_KEY` is a sandbox key from your environment.

```bash
curl https://api.stripe.com/v1/checkout/sessions \
  -u "$STRIPE_SECRET_KEY:" \
  -d mode=subscription \
  -d success_url=https://example.com/success \
  -d "line_items[0][price]=<PRICE_ID>" \
  -d "line_items[0][quantity]=1" \
  -d "managed_payments[enabled]=false"
```

### Pay

1. Copy the `url` field from the output ([checkout-session.json](../../samples/day-01/checkout-session.json)) and open it in a browser.
2. Pay with card `4242 4242 4242 4242`, any future expiry, any 3-digit CVC, any name and email.
3. Your browser goes to `example.com/success`, which is only a placeholder.
4. In the `stripe listen` tab, note the events and their order.

### Verify

```bash
stripe subscriptions list --limit 1
```

**Expected output:** [subscription.json](../../samples/day-01/subscription.json), with `"status": "active"`.

**Checkpoint:** events include `checkout.session.completed` and `invoice.paid`, the subscription is `active`, and the Dashboard shows the new customer and **1 active** subscription on Plus. The events are in [events-first-checkout.log](../../samples/day-01/events-first-checkout.log) and explained in [Reference: events](#reference-events).

## Part 4: Stripe's Node app (optional)

Skipped on Day 1, because the CLI taught the same lesson. If you want to see the redirect flow in code:

1. On Stripe's [Checkout quickstart](https://docs.stripe.com/checkout/quickstart), pick **React** and **Node** and download the sample.
2. Put your **sandbox secret key** in its `.env` file (never in a repo), set the price ID, and set `mode` to `subscription`.
3. Add `"proxy": "http://localhost:4242"` to its `package.json`, run `npm start`, and open `http://localhost:3000/checkout`.
4. Pay with `4242 4242 4242 4242`.

Do this in a scratch folder outside this repository.

## Part 5: Payment failures and 3D Secure

Create a new session with the Part 3 command and open its URL. A failed payment leaves the page open, so **try all three cards on the same page**, in this order:

| Card | What you see |
|---|---|
| `4000 0000 0000 0002` | "Your credit card was declined" |
| `4000 0000 0000 9995` | "...declined because of insufficient funds" |
| `4000 0025 0000 3155` | A test 3D Secure window with **Fail** and **Complete**. Click **Complete**. |

The events are in [events-failed-payments-and-3ds.log](../../samples/day-01/events-failed-payments-and-3ds.log).

| Step | Attempt | Events |
|---|---|---|
| 1 | Declined | `payment_intent.payment_failed`, `charge.failed` |
| 2 | Insufficient funds | `charge.failed`, `payment_intent.payment_failed` |
| 3 | 3D Secure card | `payment_intent.requires_action` |
| 4 | Clicked Complete | `charge.succeeded`, `payment_intent.succeeded`, `payment_method.attached` |
| 5 | automatic | `checkout.session.completed`, `invoice.created`, `invoice.finalized`, `invoice.paid`, `invoice.payment_succeeded`, `customer.subscription.created` |

**Key finding:** a failed first payment creates **no subscription, no invoice and no customer**. In the Dashboard's Customers list only the successful checkouts appear. There are no `invoice.*` or `customer.subscription.*` events until the payment succeeds. So `invoice.payment_failed` is for renewals (Day 2), not for sign-up.

## Reference: events

One successful checkout produces **12 distinct events**, each delivered twice (24 lines): once as `name` and once as `v1.name`. The `v1.` copies *appear to be* the same events in Stripe's newer event format (*inferred, not confirmed*). Learn the 12 plain names and ignore the copies for now; confirm which type to subscribe to when registering a real webhook.

Logical order (arrival order differs, and is not reliable):

| Event | Plain meaning | Needed by an app? |
|---|---|---|
| `customer.created` | Stripe made a customer (`cus_...`) | Maybe, to link it to your org |
| `payment_method.attached` | The card was saved to that customer | Not now |
| `customer.updated` | Stripe filled in customer details | No |
| `customer.subscription.created` | The subscription (`sub_...`) exists | **Yes** |
| `invoice.created`, `invoice.finalized` | First invoice made and locked | No |
| `payment_intent.created`, `charge.succeeded`, `payment_intent.succeeded` | The card was charged $8 | No, the invoice events cover it |
| `invoice.paid`, `invoice.payment_succeeded` | The invoice was paid (both fire) | **Yes, use `invoice.paid`** (Stripe's guide) |
| `checkout.session.completed` | The customer finished the Checkout page | **Yes**, to link the org. Do not grant access on this alone. |

Failure events (Part 5):

| Event | Meaning |
|---|---|
| `payment_intent.payment_failed`, `charge.failed` | The card was refused |
| `payment_intent.requires_action` | The bank wants 3D Secure authentication |
| `v2.signals.account_signal.customer_risk_ready` | A Stripe risk notice. Ignore. |

Stripe's guide names three events to watch at minimum: `checkout.session.completed`, `invoice.paid` and `invoice.payment_failed`. Two were seen on Day 1; the third is a Day 2 test-clock task.

## Reference: reading the responses

### Checkout Session ([checkout-session.json](../../samples/day-01/checkout-session.json))

A one-time order form that your server creates and the customer fills in. It is **not** the subscription: `subscription` is `null` and `payment_status` is `unpaid` until the customer pays.

| Field | Value | Why it matters |
|---|---|---|
| `id` | `cs_test_...` | Store it to match a later webhook to this attempt |
| `url` | `https://checkout.stripe.com/...` | **The field the backend returns to the frontend.** The frontend redirects the browser here. |
| `status` / `payment_status` | `open` / `unpaid` | Become `complete` and `paid` after payment |
| `mode` | `subscription` | The alternative `payment` is for one-time purchases |
| `amount_total` | `800` | $8.00 in cents |
| `expires_at` | 24 hours after `created` | The link dies after that |
| `customer`, `subscription` | `null` | Filled in after payment. `customer_creation: always` makes a new customer. |
| `metadata`, `client_reference_id` | empty | **Your hook back into your own data.** Put your user or org ID here. |
| `success_url` | `https://example.com/success` | Never grant access from this page. Webhooks decide that. |
| `payment_method_types` | card, klarna, link, cashapp, amazon_pay | Restrict to what you want to accept |
| `adaptive_pricing.enabled` | `true` | Local-currency display (a currency switch on the page) |

### Subscription ([subscription.json](../../samples/day-01/subscription.json))

| Field | Value | Why it matters |
|---|---|---|
| `status` | `active` | The field your plan checks will mirror |
| `customer` | `cus_EXAMPLE` | Link back to the customer |
| `latest_invoice` | `in_EXAMPLE` | Points to the most recent invoice |
| `items.data[0].current_period_start` / `current_period_end` | 1793904890 to 1796496890 | The billing period (exactly 30 days). It lives **on the item**, not the subscription. |
| `items.data[0].quantity` | `1` | Becomes "number of seats" for per-seat pricing |
| `collection_method` | `charge_automatically` | Stripe charges the saved card each month |
| `cancel_at_period_end` | `false` | Not scheduled to cancel |
| `metadata` | `{}` | Where your org ID will go |

## Test cards

Any future expiry, any 3-digit CVC, any name and email.

| Card | Behaviour |
|---|---|
| `4242 4242 4242 4242` | Succeeds |
| `4000 0000 0000 0002` | Declined |
| `4000 0000 0000 9995` | Declined, insufficient funds |
| `4000 0025 0000 3155` | Requires 3D Secure authentication |
| `4000 0000 0000 0341` | Attaches to a customer fine, but **charging it fails** (for Day 2 failed renewals) |

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `zsh: no matches found: line_items[0]...` | Quote the whole `-d "line_items[0][price]=..."` argument |
| `the product tax code is missing ... Managed Payments` | Add `-d "managed_payments[enabled]=false"` (see Part 3) |
| `No such price` | Wrong price ID, or `stripe login` is on a different account. Re-run `stripe login`. |
| Checkout link opens but fails | The session expired after 24 hours. Create a new one. |
| `stripe listen` shows nothing | It only prints when something happens in the sandbox. Edit the product in the Dashboard to test it. |
| Every event shows up twice | Normal. The `v1.` copies are the same events in a newer format (inferred). |

## What I learnt

1. **Product vs price.** The price ID is what Checkout needs; amounts are in cents.
2. **A Checkout Session is an open order form.** The backend creates it and returns its `url`. Nothing is paid until the customer pays.
3. **One purchase produces about 12 events**, but an app handles only 3 to 5.
4. **Event order is not reliable**, and events can repeat. Read the payload and deduplicate on `evt_` ID.
5. **A failed first payment leaves nothing behind.** No customer, subscription or invoice. The page shows an error and lets the customer retry.
6. **3D Secure is one extra step, then the flow is identical.** `payment_intent.requires_action`, then success events as normal.
7. **Billing period dates are on the subscription item.**
8. **Linking Stripe to your own data is your job**, through `metadata` or `client_reference_id`, because webhooks carry no login.
9. **Managed Payments can be on by default** and breaks Checkout for products without a tax code. It is a business decision, not a coding one.

## Day 2 (not yet run)

> None of these commands has been run yet. Treat output and exact flags as unverified until run.

### Part 6: Trial

Add one parameter:

```bash
stripe checkout sessions create -d mode=subscription -d success_url=https://example.com/success -d "line_items[0][price]=<PRICE_ID>" -d "line_items[0][quantity]=1" -d "managed_payments[enabled]=false" -d "subscription_data[trial_period_days]=14"
```

Stripe's guide also shows `subscription_data[trial_end]` (a Unix timestamp) as an alternative. After checkout, the subscription should show **Trialing** with a trial end date. Note whether any invoice was charged today.

### Part 7: Test clocks (renewals and failed renewals)

Checkout cannot use test clocks, so create the customer and subscription from the Dashboard.

1. In the Dashboard, search for **Test clocks** and create one.
2. Inside the clock, add a customer with card `4242 4242 4242 4242`, then add a subscription to **Plus**.
3. **Advance time** by one month and watch `stripe listen`. Expect `invoice.paid` again.
4. Add a second customer in the same clock with card `4000 0000 0000 0341`, subscribe them to Plus and advance time. Expect `invoice.payment_failed`.

### Part 8: Customer portal

1. In the Dashboard, search for **Customer portal** and enable updating payment methods, cancelling subscriptions and invoice history.
2. Create a portal link for a customer from Part 3:

```bash
stripe billing_portal sessions create -d customer=<CUSTOMER_ID> -d return_url=https://example.com
```

3. Open the `url`, cancel the subscription, and watch the events.

### Billing cycle anchor (from Stripe's guide)

Billing can be anchored to a fixed future date with `subscription_data[billing_cycle_anchor]` (a Unix timestamp that must be in the future and before the natural next billing date):

```bash
stripe checkout sessions create -d mode=subscription -d success_url=https://example.com/success -d "line_items[0][price]=<PRICE_ID>" -d "line_items[0][quantity]=1" -d "managed_payments[enabled]=false" -d "subscription_data[billing_cycle_anchor]=$(date -v+10d +%s)"
```

### Part 9: A typical SaaS pricing page

1. In the Dashboard, create three products: **Starter** (flat, monthly), **Team** (monthly, billed per seat) and **Credit pack** (**one-time**).
2. Subscribe to Team with 5 seats: `-d "line_items[0][quantity]=5"`.
3. Buy the credit pack in one-time mode and compare its events with a subscription's:

```bash
stripe checkout sessions create -d mode=payment -d success_url=https://example.com/success -d "line_items[0][price]=<PACK_PRICE_ID>" -d "line_items[0][quantity]=1" -d "managed_payments[enabled]=false"
```
