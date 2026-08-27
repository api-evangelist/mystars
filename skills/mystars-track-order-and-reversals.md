---
name: mystars-track-order-and-reversals
description: >-
  Track a MyStars fulfilment order to its terminal state and interpret what happened to the money
  — via the signed orderStatus webhook or by polling — including every reversal case and the one
  status that is not terminal. Use when an agent must confirm delivery or reconcile funds.
api: MyStars FaaS — Fulfilment API
base_url: https://api.mystars.tg
operations:
  - getOrder
  - listOrders
  - orderStatusWebhook
generated: '2026-08-27'
method: generated
source: openapi/mystars-faas-openapi.json + https://mystars.tg/docs
---

# Track an order and reconcile reversals

Two ways to learn an order's fate. They are equivalent; the webhook is optional.

## Option A — the signed webhook

Webhooks are **per order**, not a global subscription. Pass `callback_url` in the `createOrder`
body and the terminal event is POSTed to exactly that URL. There is **no "register webhook"
endpoint**.

Payload: `order_id`, `status` (one of `delivered`, `failed`, `reversed`, `expired`),
`failure_reason`, `purchase_tx`, `reversal_tx`.

### Verify it — and get the rollover case right

`X-Faas-Signature` is the hex **HMAC-SHA256 of the exact raw request body** under your webhook
signing secret. Sign over the raw bytes, not a re-serialised object.

The webhook secret is a **separate credential from your API key**. It is shown once, alongside the
key, on `/api_start` in the bot. `/api_rotate` rotates the **API key only**; `/api_rotate_webhook`
rotates the webhook secret.

**During the 24-hour rollover window after `/api_rotate_webhook`, `X-Faas-Signature` may carry
multiple comma-separated signatures** — the new secret's and the previous one's. Parse the header
as a comma-separated list and accept if **any** entry matches. Outside a rotation it is a single
value, so naive single-value verification appears to work fine and then breaks exactly once, on
your first rotation. Write the list-parsing version now.

### Delivery constraints

- Respond **2xx** to acknowledge. Any 2xx stops retries.
- You must respond within **5 seconds** (connect, headers and body each).
- **Redirects are not followed** — `callback_url` must be the final destination.
- Non-2xx or timeout → exponential backoff, then dead-lettered. It never blocks order processing.
- Body and signature are **stable across retries**, so deduplicate on `order_id` + `status`.

## Option B — poll `getOrder`

```
GET /v1/orders/{id}
```

Fully supported; use it when you have no public HTTPS endpoint. Polling is metered by the
order-lifecycle rate-limit bucket (60 req/min, isolated from read traffic). `listOrders` gives a
keyset-paginated, newest-first view filtered by `status` — pass `next_cursor` back as `?cursor=`;
`null` means last page.

Orders are tenant-isolated: another tenant's id returns **404**, not 403.

## Read the terminal state

| `status` | Meaning | The money |
|---|---|---|
| `delivered` | Item delivered | Spent. Reference in `purchase_tx` |
| `reversed` | Paid but undeliverable | Returned to the paying address minus the network fee. Reference in `reversal_tx` |
| `failed` | No usable payment applied | Received funds returned minus the network fee, if any arrived |
| `expired` | No payment before `expires_at` | Nothing was taken |
| `cancelled` | You cancelled it unpaid | Nothing was taken |

### `held` is **not** terminal

`held` means the order is still processing or under manual review. It resolves to `delivered` or
`reversed`. **Do not treat it as final and do not re-create the order** — a new `Idempotency-Key`
means a second order and a second charge. Keep polling, or wait for the webhook.

## Reconcile a reversal — `failure_reason`

| `failure_reason` | Terminal status | Cause | Your fix |
|---|---|---|---|
| `underpaid` | `failed` | Amount below the −1% tolerance | Re-quote; send exactly `payment.amount` |
| `overpaid` | `failed` | Amount above the +2% tolerance | Send exactly `payment.amount`; do not round up |
| `no_memo` | `failed` | Payment carried no memo | Always send `payment.memo` (it equals `order_id`) |
| `wrong_memo` | `failed` | Memo matched no order | Copy `payment.memo` verbatim; do not construct one |
| `undeliverable` | `reversed` | Paid, but could not be delivered | Pre-flight with `checkRecipient` |
| `expired` | `expired` | No matching payment in the window | Read `expires_at`; do not assume a duration |

A payment **inside** the −1% … +2% band is treated as exact — no reversal. Amounts below a small
dust threshold are not reversed, because the network fee would consume them.

## When something looks stuck

1. Re-query `getOrder`. A `held` order is still in progress — give it time.
2. If it is terminal `reversed` but the on-chain reversal has not appeared, or it stays `held` for
   an extended period, contact [support](https://t.me/Mystars_support_bot) with the `order_id`.
3. **Do not blindly retry.** A new `Idempotency-Key` creates a brand-new order and a new charge;
   reusing the **same** key safely re-queries the original.

`order_id` is a UUID that is also the on-chain payment memo, so one identifier ties the HTTP call,
the TON transaction and the support conversation together. Log it.
