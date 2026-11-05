# Day 1 samples

Responses and event logs from the Day 1 sandbox run, **trimmed and anonymised**.

- Field names and values (amounts, statuses, modes, timestamps) are real.
- Object IDs are replaced by placeholders such as `prod_EXAMPLE` and `cus_EXAMPLE`.
- The Checkout `url` is shortened and its fragment is redacted.
- JSON files keep only the fields that matter. A real response has many more, mostly `null`.
- Event logs list event names with an `evt_EXAMPLE` placeholder. Order and timing are as they arrived.

| File | From |
|---|---|
| [product.json](product.json) | `stripe products list` |
| [price.json](price.json) | `stripe prices list` |
| [checkout-session.json](checkout-session.json) | `stripe checkout sessions create` |
| [subscription.json](subscription.json) | `stripe subscriptions list` |
| [error-managed-payments.json](error-managed-payments.json) | A failed `checkout sessions create` on an account with Managed Payments on |
| [events-first-checkout.log](events-first-checkout.log) | `stripe listen` during the first successful subscription |
| [events-failed-payments-and-3ds.log](events-failed-payments-and-3ds.log) | `stripe listen` while trying a declined card, an insufficient-funds card and a 3D Secure card |
