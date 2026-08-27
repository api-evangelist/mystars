---
name: mystars-buy-telegram-stars
description: >-
  Buy and deliver Telegram Stars to an @username through the MyStars Fulfilment API — quote the
  all-in price, confirm the recipient can receive them, create an idempotent order, and settle it
  on-chain in GRAM or USDT. Use when an agent must purchase Stars programmatically.
api: MyStars FaaS — Fulfilment API
base_url: https://api.mystars.tg
operations:
  - listProducts
  - listCurrencies
  - getPricing
  - checkRecipient
  - createOrder
  - getOrder
  - cancelOrder
generated: '2026-08-27'
method: generated
source: openapi/mystars-faas-openapi.json + https://mystars.tg/docs
---

# Buy Telegram Stars

Every call carries `X-Api-Key: $MYSTARS_API_KEY`. The key is issued in the Telegram bot
[@my_stars_tg_bot](https://t.me/my_stars_tg_bot) under **API access** — there is no dashboard and
no signup form.

## Before you start

The one irreversible step in this flow is **sending the on-chain payment**. Everything before it
— quote, recipient check, order create, cancel — is undoable while the order is still
`awaiting_payment`. Do not send funds until you have the order's `payment` block in hand.

## 1. Know the bounds (optional, cache it)

`listProducts` returns price-free catalog metadata: Stars are a continuous quantity range of
**50 – 1,000,000**; Premium is the fixed 3 / 6 / 12-month tiers. `listCurrencies` returns the
payment options (`ton` = native GRAM, `usdt_ton` = USDT as a TEP-74 jetton on TON).

Call these once and cache. They are price-free, so they never go stale the way a quote does.

## 2. Quote the price — `getPricing`

```
GET /v1/pricing?type=stars&quantity=500&payment_currency=ton
```

`amount` is the **full, all-in total** to send on-chain. There is nothing to add — the 5% MyStars
margin is already inside it, and for `usdt_ton` the `fee` object itemises the 1% swap + 0.5 GRAM
gas that is *already included*, never added.

The response also carries `quoted_at`, `valid_until` and `usdt_per_ton`. Treat `valid_until` as a
**re-quote hint, not a lock**: the price tracks the market and is recomputed roughly every minute.
The price is only locked when you create the order.

**Amounts are decimal strings.** Never parse them to a float — pass the string through.

For a storefront refreshing a whole pack catalog, use `getPricingBatch` instead: up to 200
quantities in one request, deduped and sorted, cent-for-cent identical to `getPricing`, and it
counts as **one** unit against the rate limit.

## 3. Check the recipient — `checkRecipient`

```
POST /v1/recipients/check
{"type":"stars","recipient":{"username":"durov"}}
```

Read-only and free of consequence. Pass the **same `type`** you will order with — a Stars check
does not prove a Premium gift will be accepted.

**This check fails open.** When the probe cannot reach a verdict it returns `eligible: true` with
`indeterminate: true`. That means *unknown*, not *yes*. Never show a buyer a confirmed recipient
on an indeterminate result — proceeding to `createOrder` is still safe, because the order runs its
own authoritative check, but do not promise the delivery first.

On `eligible: false`, surface `telegram_message` — Telegram's own verbatim wording — rather than
your own paraphrase.

## 4. Create the order — `createOrder`

```
POST /v1/orders
Idempotency-Key: order-<your own order id>
{"type":"stars","recipient":{"username":"durov"},"quantity":500,"payment_currency":"ton",
 "callback_url":"https://example.com/webhooks/mystars"}
```

`Idempotency-Key` is **required**. Use a **stable** key equal to your own order id.

- Same key + identical body → the original order, HTTP **200** (a first create is **201**).
- Same key + different body → **409 conflict**.
- A **new** key → a brand-new order and a **second charge**. This is the expensive mistake.

`callback_url` is optional and must be a publicly reachable `https://` URL — loopback and
private-network hosts are rejected `400`. Omit it and poll instead.

Handle these responses:

| Status | Meaning | What to do |
|---|---|---|
| 201 / 200 | Created / idempotent replay | Continue to step 5 |
| 422 `recipient_ineligible` | Recipient cannot receive it | **No order, no charge.** Show `telegram_message` |
| 503 `unavailable` | Price source or eligibility oracle down | **Retryable.** No order, no charge — retry shortly with the **same** key |
| 429 `rate_limited` | Throttled | Back off; read `Retry-After` if present |

## 5. Pay it

The response `payment` block tells you exactly how to settle:

- `pay_to_address` — the treasury wallet. **The same address for both currencies.**
- `memo` — required; it equals `order_id`.
- `amount` / `amount_units` — send **exactly** this.

For `ton`, send native GRAM with `memo` as the transfer comment. For `usdt_ton`, send a **TEP-74
jetton transfer whose destination is `pay_to_address`** (the treasury *owner* wallet) with `memo`
as the forward-payload text comment and a small non-zero `forward_ton_amount` for forwarding gas.
**Do not send USDT to a derived jetton-wallet address — the funds would be misrouted.**

Payment tolerance is **−1% … +2%**. Inside that band the payment is treated as exact. Outside it
the order ends `failed` (`underpaid` / `overpaid`) and the funds are reversed minus the network
fee. A payment with a missing or wrong memo is reversed the same way.

The official SDKs build this for you: `buildPaymentRequest(order.payment)` (TypeScript) or
`build_payment_request(order.payment)` (Python) emit a `ton://transfer/…` deeplink.

## 6. Confirm delivery

Either wait for the signed `orderStatus` webhook, or poll `getOrder`. See
`mystars-track-order-and-reversals`.

## Changed your mind before paying — `cancelOrder`

```
POST /v1/orders/{id}/cancel
```

Works **only** while `status` is `awaiting_payment`, i.e. before `expires_at`. Anything else is a
`409`. Read `expires_at` from the order — never assume a duration; the window has moved from 15
minutes to 1 hour to 2 hours across versions, and the spec deliberately no longer quotes one.

## Rate limits

`getPricing`, `getPricingBatch` and `checkRecipient` carry a tighter **60 req/min** cap on top of
the general per-tenant budget. The order lifecycle (`createOrder`, `getOrder`, `cancelOrder`) has
its **own** 60 req/min bucket, isolated from reads — a burst of quoting can never throttle an
order create. Read `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset` rather than
hard-coding, and handle a **header-less 429** too: the daily order cap and the per-recipient
concurrency guard return the envelope only.

## Errors

Every error is `{"error":{"code","message","telegram_message?"}}` — never an HTML page. Branch on
`code`: `bad_request`, `unauthorized`, `forbidden`, `not_found`, `conflict`,
`recipient_ineligible`, `rate_limited`, `unavailable`, `internal`.
