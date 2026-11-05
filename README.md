# stripe-billing-101

My learning notes for **Stripe Billing**: how to sell subscriptions (and one-time packs) from a SaaS app, learnt by running everything in a Stripe **sandbox** with the Stripe CLI first, before writing app code.

> Sandbox only. No live keys, no real money. Never commit API keys or webhook secrets (`.gitignore` blocks `.env*`, `*.key`, `*.pem`).

## Where to start

| If you want to... | Read |
|---|---|
| Run a first subscription end to end | [Day 1 guide](docs/day-01/README.md) |
| Learn the vocabulary and object model | [Concepts](docs/concepts.md) |
| See how Stripe fits into an app (diagrams) | [Architecture](docs/architecture.md) |
| See how much of Stripe's official guide is covered | [Guide coverage](docs/day-01/stripe-guide-coverage.md) |
| Read example responses and event logs | [samples/day-01](samples/day-01/) |

## Progress

| Day | Focus | Status |
|---|---|---|
| Day 1 | Product and price, first subscription, failed payments, 3D Secure, event log | Done |
| Day 2 | Trial, test clocks (renewals), customer portal, per-seat and one-time products | Next |
| Later | Webhook endpoint with signature check, idempotency, a small Java demo | Planned |

## Repository layout

```text
stripe-billing-101/
├── README.md
├── .gitignore                       blocks keys and env files
├── docs/
│   ├── concepts.md                  vocabulary, objects, ID prefixes
│   ├── architecture.md              flows and object model (Mermaid)
│   └── day-01/
│       ├── README.md                Day 1 guide: commands, expected output, findings
│       └── stripe-guide-coverage.md Stripe's guide, section by section
└── samples/
    └── day-01/                      trimmed, anonymised JSON and event logs
```

New days get a sibling folder in `docs/` and `samples/` (`day-02`, ...). Anything that is not day-specific goes in `docs/` at the top level.

## Conventions

- **One folder per day**, each with its own guide and samples.
- **Samples are trimmed and anonymised**: field names and values are real, IDs are placeholders such as `prod_EXAMPLE`.
- **Placeholders in commands**: `<PRICE_ID>`, `<CUSTOMER_ID>`. Never paste a secret key.
- **Verified vs not yet run**: anything I have not run myself is marked *not yet run* or *inferred*.
- **Screenshots are left out on purpose.** Run the Day 1 guide in your own free Stripe sandbox and add your own.
- **Diagrams are Mermaid**, so GitHub renders them directly.

## Source

Based on Stripe's public guide [Sell subscriptions as a SaaS startup](https://docs.stripe.com/get-started/use-cases/saas-subscriptions) and Stripe's CLI and API docs. This is a personal study repo and is not affiliated with Stripe.
