---
name: mystars-gift-telegram-premium
description: >-
  Gift a 3, 6 or 12-month Telegram Premium subscription to an @username through the MyStars
  Fulfilment API, including the one eligibility rule that fails most Premium orders — a recipient
  with an active subscription cannot be gifted Premium. Use when an agent must buy Premium
  programmatically.
api: MyStars FaaS — Fulfilment API
base_url: https://api.mystars.tg
operations:
  - listProducts
  - getPricing
  - checkRecipient
  - createOrder
  - getOrder
  - cancelOrder
generated: '2026-08-27'
method: generated
source: openapi/mystars-faas-openapi.json + https://mystars.tg/docs
---

# Gift Telegram Premium

Premium follows the same quote → check → create → pay flow as Stars, with three differences that
matter. Read `mystars-buy-telegram-stars` for the shared mechanics; this skill covers only what
changes.

## Difference 1 — `months`, not `quantity`

Premium takes `months`, and only **3, 6 or 12**. Any other value is rejected. `quantity` does not
apply and comes back `null` on the response. `listProducts` confirms the tiers without a price.

```
GET /v1/pricing?type=premium&months=3&payment_currency=usdt_ton
```

## Difference 2 — the eligibility rule that actually bites

**A recipient who already has an active Premium subscription cannot be gifted Premium.** Telegram
blocks gifting to anyone whose subscription is still running — for example an annual plan that has
not expired. This is *Telegram's* restriction, not MyStars'.

Pre-flight it:

```
POST /v1/recipients/check
{"type":"premium","recipient":{"username":"durov"},"months":3}
```

An ineligible recipient returns `eligible: false` with `reason: "already_subscribed"` and
Telegram's verbatim wording in `telegram_message`. Show that wording to the buyer.

Skip the check and `createOrder` returns **422 `recipient_ineligible`** — **no order is created
and you are not charged**, so nothing is lost but the round trip and the buyer's confidence.

Pass `type: premium` on the check. A Stars check does **not** prove a Premium gift will be
accepted; the check runs against the exact product you intend to order, which is why `type` is
required.

Remember the check fails open: `eligible: true` with `indeterminate: true` means the probe could
not decide. Treat it as unknown and do not confirm the gift to the buyer.

## Difference 3 — nothing else

Payment, idempotency, cancellation, tolerance, reversals, webhooks and rate limits are identical
to the Stars flow. The order is cancellable only while `awaiting_payment`, bounded by `expires_at`.

## Worked order

```
POST /v1/orders
Idempotency-Key: order-<your own order id>
{"type":"premium","recipient":{"username":"durov"},"months":3,"payment_currency":"usdt_ton"}
```

For `usdt_ton`, the `payment.fee` object itemises the 1% swap + 0.5 GRAM gas that is **already
included** in `payment.amount`. `fee.total` equals `amount`; it never adds to it. Pay exactly
`amount`. For native GRAM (`ton`) there is no swap, and `fee` is `null`.
