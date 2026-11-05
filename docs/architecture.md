# Architecture

High-level map of how Stripe fits into a SaaS app. This is a **working model**: Stripe-hosted Checkout, Stripe as the billing engine, and your database as a cache of Stripe's state.

## 1. Who does what

Purple boxes are the backend code you write. Everything else is frontend work or hosted by Stripe.

```mermaid
flowchart LR
  subgraph FE["Your frontend"]
    A1["1. Click Subscribe<br/>plan or credit pack"]
    A4["4. Redirect browser<br/>to the Stripe URL"]
    A10["10. Billing page<br/>asks backend for status"]
  end
  subgraph BE["Your backend (you write this)"]
    B2["2. Create session endpoint<br/>org ID in metadata"]
    B8["8. Webhook endpoint<br/>verify, dedupe, route"]
    B9["9. Update your DB<br/>plan, status, credits"]
  end
  subgraph ST["Stripe (hosted, no code)"]
    S3["3. Checkout Session<br/>returns a payment URL"]
    S5["5. Customer pays<br/>card, 3D Secure"]
    S6["6. Creates customer,<br/>subscription, invoice"]
    S7["7. Sends webhook events"]
  end
  A1 --> B2
  B2 --> S3
  B2 -->|"returns url"| A4
  A4 --> S5
  S5 --> S6 --> S7 --> B8 --> B9
  A10 -->|"reads"| B9
  classDef code fill:#EEEDFE,stroke:#534AB7,color:#26215C
  class B2,B8,B9 code
```

Backend work, in short:

| Step | What you build |
|---|---|
| 2 | An endpoint that finds the user or org, picks the price ID, sets `metadata`, calls Stripe and returns the session `url` |
| 8 | A public webhook endpoint: verify the signature, skip repeated `evt_` IDs, find the org from metadata |
| 9 | Tables for Stripe customer and subscription IDs, plan and status, credit balance, billing history |
| also | A query for the billing page and plan checks for features |

## 2. A successful subscription, over time

Verified on Day 1 (see [events-first-checkout.log](../samples/day-01/events-first-checkout.log)).

```mermaid
sequenceDiagram
  autonumber
  actor U as Customer
  participant FE as Frontend
  participant BE as Your backend
  participant ST as Stripe
  U->>FE: Click Subscribe
  FE->>BE: Create checkout session for a plan
  BE->>ST: POST /v1/checkout/sessions with price and metadata
  ST-->>BE: session id and url
  BE-->>FE: url
  FE->>U: Redirect to the Stripe page
  U->>ST: Enter card, confirm (3D Secure if asked)
  ST-->>U: Redirect to success_url
  ST->>BE: Webhooks: customer.subscription.created, invoice.paid, checkout.session.completed
  BE->>BE: Verify signature, skip repeats, find the org, save state
  FE->>BE: Billing page asks for status
  BE-->>FE: plan, status, next billing date
```

## 3. What happens when the first payment fails

Verified on Day 1 (see [events-failed-payments-and-3ds.log](../samples/day-01/events-failed-payments-and-3ds.log)). A failed first payment leaves no customer, subscription or invoice, so your database has nothing to clean up.

```mermaid
flowchart TD
  A["Customer submits card on the Stripe page"] --> B{"Bank result"}
  B -->|"Declined or insufficient funds"| C["payment_intent.payment_failed<br/>charge.failed"]
  C --> D["Page shows an error and stays open"]
  D --> A
  C --> N["No customer, subscription or invoice exists"]
  B -->|"Needs authentication"| E["payment_intent.requires_action"]
  E --> F["3D Secure window: Complete"]
  F --> G["charge.succeeded<br/>payment_intent.succeeded"]
  B -->|"Approved"| G
  G --> H["customer.subscription.created<br/>invoice.paid<br/>checkout.session.completed"]
  H --> I["Backend webhook saves state"]
```

## 4. Renewals (expected, not yet verified)

`invoice.payment_failed` belongs to renewals, not to the first payment. To be confirmed with test clocks on Day 2.

```mermaid
flowchart TD
  R["Billing period ends"] --> I["Stripe creates and finalizes the invoice"]
  I --> P{"Charge the saved card"}
  P -->|"Paid"| OK["invoice.paid"]
  OK --> S1["Backend keeps the plan active"]
  P -->|"Fails"| F["invoice.payment_failed"]
  F --> S2["Backend notifies the customer and links to the customer portal"]
```

## 5. Stripe object model

The objects your code will touch and how they link. Fields come from the Day 1 samples.

```mermaid
erDiagram
  CUSTOMER ||--o{ SUBSCRIPTION : has
  SUBSCRIPTION ||--|{ SUBSCRIPTION_ITEM : contains
  SUBSCRIPTION_ITEM }o--|| PRICE : charges
  PRICE }o--|| PRODUCT : "belongs to"
  SUBSCRIPTION ||--o{ INVOICE : generates
  INVOICE ||--o| PAYMENT_INTENT : "paid by"
  CHECKOUT_SESSION ||--o| SUBSCRIPTION : creates
  CUSTOMER {
    string id "cus_"
    string email
  }
  SUBSCRIPTION {
    string id "sub_"
    string status
    string latest_invoice
    bool cancel_at_period_end
  }
  SUBSCRIPTION_ITEM {
    string id "si_"
    int quantity
    int current_period_start
    int current_period_end
  }
  PRICE {
    string id "price_"
    int unit_amount
    string interval
  }
  PRODUCT {
    string id "prod_"
    string name
  }
  INVOICE {
    string id "in_"
  }
  PAYMENT_INTENT {
    string id "pi_"
  }
  CHECKOUT_SESSION {
    string id "cs_test_"
    string mode
    string url
  }
```

## 6. Flows that reuse the same pieces

| Flow | How it fits |
|---|---|
| Monthly renewal | Stripe charges by itself. The same webhook endpoint receives `invoice.paid` or `invoice.payment_failed`. |
| Customer portal | One more small endpoint that asks Stripe for a portal link. |
| Credit packs | Same create-session flow in `mode=payment`. The webhook adds credits to your own balance table. |
| Auto top-up | A job charges a saved card when the balance drops below a threshold. Needs idempotency so two low-balance events cannot buy two packs. |
| Upgrade with more seats | Change the subscription's price and quantity with proration. Needs an invoice preview for "total due now". |
